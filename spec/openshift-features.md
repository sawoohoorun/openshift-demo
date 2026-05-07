# OpenShift Features: DevSecOps & OpenShift AI

## DevSecOps

### CI/CD Pipeline Security (OpenShift Pipelines / Tekton)

- Tekton-based CI/CD with built-in security gates that fail builds on policy violations
- Integrated pipeline stages: secret scanning, dependency analysis, image building, vulnerability scanning, CVE policy checking, image signing, and deployment verification
- **Tekton Chains** — cryptographic signing of code artifacts using sigstore
- SLSA and in-toto standards support for supply chain provenance
- Pipelines as Code (PaC) for version-controlled pipeline definitions

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
