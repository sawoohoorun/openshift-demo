# OpenShift Features: DevSecOps & OpenShift AI

## DevSecOps

### CI/CD Pipeline Security (OpenShift Pipelines / Tekton)

#### Pipelines as Code (PaC)

Pipelines as Code stores Tekton pipeline definitions in a `.tekton/` directory alongside source code. A `Repository` CRD configures the Git provider, webhook URL, and authentication token. On each Git event the matching PipelineRun is created and results are reported back as PR checks.

**Key annotations on PipelineRun templates:**

| Annotation | Purpose |
|---|---|
| `pipelinesascode.tekton.dev/on-event` | Trigger on `pull_request`, `push`, etc. |
| `pipelinesascode.tekton.dev/on-target-branch` | Limit to specific branches (e.g., `[main]`) |
| `pipelinesascode.tekton.dev/task` | Inline a remote task from Artifact Hub by name |
| `pipelinesascode.tekton.dev/pipeline` | Share a pipeline definition across repositories via URL |
| `pipelinesascode.tekton.dev/max-keep-runs` | Auto-prune old PipelineRuns; keep N successful runs |
| `pipelinesascode.tekton.dev/target-namespace` | Restrict execution namespace (prevents privilege escalation) |

**ChatOps commands** from PR/MR comments: `/test`, `/retest`, `/cancel`, `[skip ci]`

Supported Git providers: GitHub, GitLab, Bitbucket Cloud, Bitbucket Data Center, Forgejo.

---

#### Security Gate Pipeline Stages

A typical RHTAP-aligned secure pipeline runs these ordered gates — each stage blocks promotion if it fails:

1. **Secret scanning** — detect leaked credentials or API keys in source
2. **Dependency analysis (SCA)** — identify vulnerable third-party packages
3. **Hermetic build** — compile/package with network access disabled (see below)
4. **SBOM generation** — produce a full software bill of materials (Syft)
5. **Vulnerability scanning** — scan the built image for CVEs (Grype / ACS Scanner)
6. **CVE policy check** — enforce severity thresholds (e.g., block if CVSS ≥ 8.0)
7. **Image signing** — cryptographically sign the image with cosign/Tekton Chains
8. **SLSA provenance attestation** — attach a signed in-toto provenance record
9. **Enterprise Contract check** — validate signature + attestation + policy rules
10. **Deployment manifest check** — validate Kubernetes manifests against ACS policies (`roxctl deployment check`)
11. **Deployment + verification** — promote image; verify rollout health

---

#### Hermetic Builds

Hermetic builds run with no network access during the build phase. Tekton's extended entrypoint binary creates a new Linux network namespace for each step — the same isolation mechanism used by containers. Any task that tries to fetch dependencies at runtime fails loudly, forcing all inputs to be declared explicitly.

- Configured per-TaskRun with `hermetic: true`
- Does **not** apply to sidecar containers
- Currently alpha/experimental in upstream Tekton
- Defense-in-depth measure, not a full security boundary (complements SELinux/seccomp)

---

#### SBOM Generation

Syft scans built images and generates a Software Bill of Materials listing every package, library, and OS component.

- Detects **28+ package ecosystems**: Alpine, Debian, RPM, npm, pip, Maven, Go modules, Rust, Ruby, PHP, .NET, Java, etc.
- Runs fully offline with no cloud dependencies
- Output formats:

| Format | Standard Body | Primary Use |
|---|---|---|
| **SPDX JSON** | Linux Foundation | Government compliance (NTIA, EO 14028) — rich license fields |
| **CycloneDX** | OWASP Foundation | Vulnerability identification — component-focused |
| **Syft JSON** | Anchore | Highest fidelity internal pipelines; lossless intermediate format |

SBOMs are attached to the image as OCI attestations following the in-toto SPDX predicate spec:

```bash
# Generate and attest
syft [image] -o spdx-json > sbom.json
cosign attest --type spdxjson --attestation sbom.json [IMAGE]

# Verify downstream
cosign verify-attestation --type spdxjson [IMAGE]
```

---

#### Tekton Chains — Signing & SLSA Provenance

Tekton Chains is a CRD controller that watches every TaskRun/PipelineRun completion and automatically signs artifacts, generates in-toto attestations, and uploads them to a transparency log.

**SLSA provenance formats supported:**

| Config value | SLSA spec version | Notes |
|---|---|---|
| `slsa/v2alpha4` | v1.0 (latest) | Adds optional `isBuildArtifact` flag |
| `slsa/v2alpha3` | v1.0 | Uses latest Tekton v1 objects |
| `slsa/v1` / `in-toto` | v0.2 | Legacy alias |

**Artifact identification:** Chains captures any TaskRun result ending in `_DIGEST` (e.g., `IMAGE_DIGEST=sha256:abc…`) as a provenance subject, paired with the matching `IMAGE` result for the artifact reference.

**Signing options:**

- **Keyless (Fulcio)** — Chains requests an OIDC token from the cluster provider; Fulcio issues a short-lived X.509 cert; cert + payload + signature are stored as base64 annotations on the run object. Enable: `signers.x509.fulcio.enabled: "true"` in `chains-config`.
- **KMS** — Supports GCP KMS (`gcpkms://…`), AWS KMS (`awskms://…`), Azure Key Vault (`azurekms://…`), and HashiCorp Vault (`hashivault://…`).

**Transparency log (Rekor):** Enable with `transparency.enabled: "true"`. After upload, Chains adds annotation `chains.tekton.dev/transparency: https://rekor.sigstore.dev/<logIndex>`. Searchable at https://rekor.tlog.dev.

**Verification:**

```bash
cosign verify --key cosign.pub [IMAGE]
cosign verify-attestation --key cosign.pub --type slsaprovenance [IMAGE]
```

**Key `chains-config` ConfigMap settings:**

| Key | Example value |
|---|---|
| `artifacts.taskrun.format` | `slsa/v2alpha4` |
| `artifacts.taskrun.storage` | `oci` |
| `artifacts.pipelinerun.format` | `slsa/v2alpha4` |
| `transparency.enabled` | `"true"` |
| `transparency.url` | `https://rekor.sigstore.dev` |
| `signers.x509.fulcio.enabled` | `"true"` |
| `signers.kms.kmsref` | `gcpkms://projects/…` |

---

#### Enterprise Contract (EC) — Policy-as-Code Promotion Gates

Enterprise Contract enforces policy rules written in **Rego** (Open Policy Agent language), bundled as OCI artifacts and executed with Conftest at every promotion boundary (dev → stage → stage → prod).

**Built-in rule categories (41+ rules):**
- Container image signed by a trusted build system
- SLSA provenance present and valid (v1.0+)
- Builder identity (`builder.id`) matches an allowlisted value
- Attestation signature verifiable with the expected key or keyless cert chain
- License compliance (no GPL in proprietary software, etc.)
- SBOM attached and meets minimum package coverage

**Promotion flow:**
```
Dev build → EC validates: signature + attestation + builder + policies
     ↓ PASS                         ↓ FAIL
  Merge to stage              Pipeline blocked; fix required → rebuild
     ↓
Stage build → EC validates again
     ↓ PASS
  Merge to production
```

**Blocking mechanism:** EC returns a non-zero exit code on policy failure → the pipeline Step fails → PipelineRun halts → no image promoted. Manual remediation + full rebuild required.

---

#### Security Task Catalog (Artifact Hub / Tekton Hub)

Reusable security tasks are referenced via the Hub resolver:

```yaml
taskRef:
  resolver: hub
  params:
  - name: catalog
    value: artifact-hub
  - name: name
    value: grype
  - name: version
    value: "0.1"
```

**Key security tasks:**

| Task | Tool | What it does |
|---|---|---|
| `grype` | Anchore Grype | CVE scan of image or filesystem; enforces severity thresholds |
| `syft` | Anchore Syft | SBOM generation (SPDX, CycloneDX, JSON) |
| `acs-image-check` | roxctl | Checks image against ACS policies; fails on violations |
| `acs-deployment-check` | roxctl | Validates Kubernetes manifests against ACS policies |
| `cosign-sign` | cosign | Signs OCI image; attaches signature to registry manifest |
| `show-sbom` | — | Lists all SBOM components; outputs machine-readable audit trail |
| `git-clone` | — | Authenticated source checkout |
| `buildah` | Buildah | Rootless OCI image builds |

### Container Image Security & Registry

- **Red Hat Quay** integration — image scanning at push time (build-time) and at deployment (deploy-time)
- Continuously watches images for newly published CVEs even when not actively deployed
- Registry whitelisting — container runtime restricted to pull only from approved registries
- Image signature verification before deployment

### Red Hat Advanced Cluster Security (ACS / RHACS)

- **Scanner V4** — merges StackRox scanner with Clair V4, backed by Trivy
- Vulnerability scanning across all image layers, packages, OS components, and language dependencies (Go, Java, Node, Python, etc.)
- Build, deploy, and runtime policy enforcement lifecycle
- On-demand and scheduled vulnerability scans
- Real-time threat detection and automated response in running containers
- Risk prioritization across all discovered CVEs
- Network policy visualization and enforcement

### Compliance Automation

- **OpenShift Compliance Operator** — automated scanning using OpenSCAP (NIST-certified)
- Scans Kubernetes/OpenShift API resources, pods, roles, cluster nodes, and configurations
- Supported frameworks:
  - CIS Benchmark (OCP4-CIS profile)
  - FedRAMP High SCAP profile
  - NIST SP 800-53
- `ComplianceRemediation` objects for automated fix suggestions with admin-controlled approval
- `CustomRule` CRD for writing platform-specific compliance checks
- CI/CD pipeline integration — compliance checks run as gates before production promotion

### Identity, Access & Audit

- Fine-grained **RBAC** (Role-Based Access Control) across namespaces and cluster resources
- Identity-based audit trails for all user actions
- Security Context Constraints (SCC) — restrict pod privileges at the platform level
- Integration with enterprise identity providers (LDAP, OIDC, SAML)

### Runtime & Platform Security

- **CRI-O** container runtime — minimal attack surface vs. Docker
- Containers run as non-root by default
- Network policies for namespace-level traffic isolation
- SELinux and seccomp profiles enforced per workload
- Immutable infrastructure patterns with GitOps (ArgoCD / OpenShift GitOps)

### Supply Chain Security

- **OpenShift Platform Plus** bundle: OCP + ACS + ACM + Quay + OpenShift Data Foundation
- End-to-end software supply chain protection: build → sign → scan → verify → deploy
- Trusted artifact promotion with signed container images
- Integration with Sigstore (cosign, rekor) for keyless signing

---

## OpenShift AI (RHOAI)

### Model Serving & Inference

- **KServe-based single-model serving** — each LLM deployed with its own dedicated resources
- Default integration with Red Hat OpenShift Serverless and Service Mesh for autoscaling and mTLS
- Raw deployment mode for environments without Serverless operator
- **Multi-model serving platform** (ModelMesh) for efficient co-hosting of smaller models
- REST and gRPC prediction APIs

**Inference Runtimes:**
- OpenVINO Model Server (default)
- vLLM runtime for high-throughput parallelized LLM serving
- NVIDIA Triton Inference Server
- Custom runtime support via ServingRuntime CRD

### LLM Fine-Tuning & Training

- **Kubeflow Trainer** — distributed multi-node fine-tuning with fault tolerance and checkpoint management
- **Training Hub** — algorithm-focused interface for post-training LLM techniques:
  - Supervised Fine-Tuning (SFT)
  - LoRA / QLoRA (parameter-efficient fine-tuning)
  - Orthogonal Subspace Fine-Tuning (OSFT)
- `ClusterTrainingRuntime` objects — platform engineers define runtimes, data scientists write portable training code without custom images
- Four-stage scaling path: local workstation → RHOAI Notebooks → Kubeflow Trainer → AI Pipelines

### ML Pipelines & Orchestration

- **Kubeflow Pipelines v2** backed by Argo Workflows
- Directed acyclic graph (DAG) execution model
- End-to-end ML workflow orchestration: data prep → training → evaluation → registration → serving
- **Elyra** JupyterLab extension — drag-and-drop pipeline composition from notebooks
- YAML-based pipeline definitions for GitOps compatibility
- Quality gates: only models passing evaluation thresholds are promoted to serving
- Automated approval workflows and audit trails before production deployment

### Data Science Workbenches

- Project-isolated Jupyter notebook environments (workbenches) with dedicated compute
- Multiple workbench instances per project
- Preconfigured notebook images:
  - Standard Data Science
  - TensorFlow
  - PyTorch
  - TrustyAI (model explainability)
  - CUDA (GPU-accelerated)
- Custom notebook image support via ImageStream

### GPU & Hardware Acceleration

- NVIDIA GPU support via Red Hat certified **GPU Operator**
- GPU resource scheduling and multi-tenancy (MIG partitioning support)
- GPU-backed workbenches and training jobs
- Distributed GPU fleet orchestration for large-scale training
- Support for AMD ROCm and Intel Gaudi accelerators (via hardware profiles)

### Model Registry & Lifecycle Management

- Centralized **Model Registry** for versioning, metadata, and provenance tracking
- Pipeline integration for automated model flow from training to serving
- Model lifecycle: experiment → register → stage → approve → serve
- Audit trail from training run to deployed inference endpoint
- **llm-d** framework for distributed inference fleet management at scale

### Monitoring & Trustworthy AI

- **TrustyAI** — model explainability, fairness metrics, and bias detection
- Drift detection on live model predictions
- Performance monitoring dashboards per model endpoint
- Integration with OpenShift monitoring (Prometheus / Grafana)

### Multi-Tenancy & Organization

- Project-based isolation for data science teams
- RBAC integration: admin, data scientist, and editor roles per project
- Resource quota management per project namespace
- Shared cluster resources with per-team guardrails
- GitOps-compatible configuration for all RHOAI resources

---

## References

- [Red Hat OpenShift AI Product Page](https://www.redhat.com/en/products/ai/openshift-ai)
- [RHOAI Self-Managed Documentation](https://docs.redhat.com/en/documentation/red_hat_openshift_ai_self-managed)
- [Red Hat Advanced Cluster Security Documentation](https://docs.openshift.com/acs/latest)
- [OpenShift Compliance Operator Documentation](https://docs.redhat.com/en/documentation/openshift_container_platform/latest/html/security_and_compliance/compliance-operator)
- [Multicluster DevSecOps Validated Patterns](https://validatedpatterns.io/patterns/devsecops/)
- [AI on OpenShift](https://ai-on-openshift.io)
