# OpenShift Demo & AI Hosting Project

## Project Purpose
This project builds OpenShift feature demos, documentation, and AI-hosted applications on OpenShift. Tasks include deploying workloads, configuring operators, building demo scripts, and hosting AI/ML models on OpenShift.

## Key Tools & CLI
- `oc` — OpenShift CLI (primary tool for all cluster interactions)
- `kubectl` — Kubernetes CLI (fallback when `oc` isn't needed)
- `helm` — Helm chart management
- `podman` / `docker` — Container image builds
- `skopeo` — Image inspection and copying between registries
- `kustomize` — Manifest overlays

## Common Workflows

### Cluster Access
```bash
oc login --token=<token> --server=<api-url>
oc project <namespace>
oc whoami --show-server
```

### Build & Deploy Applications
```bash
# Source-to-Image (S2I) build
oc new-app <git-url> --name=<app>
oc start-build <buildconfig> --follow

# Apply manifests
oc apply -f manifests/
kustomize build overlays/dev | oc apply -f -

# Image stream tagging
oc tag <source>:<tag> <imagestream>:<tag>
```

### AI/ML Workloads on OpenShift (OpenShift AI / RHOAI)
```bash
# Check OpenShift AI operator
oc get dsci,dsc -n redhat-ods-operator

# Data Science Projects
oc get datasciencepipelineapplication -A
oc get inferenceservice -A

# Model serving (KServe / ModelMesh)
oc get servingruntime -n <namespace>
oc get inferenceservice -n <namespace>
```

### Operators & Cluster Config
```bash
oc get csv -n openshift-operators          # Installed operators
oc get operatorgroup,subscription -A       # Operator subscriptions
oc get clusteroperator                     # Core cluster operators
oc get node -o wide                        # Node status
```

### Troubleshooting
```bash
oc get events --sort-by='.lastTimestamp' -n <namespace>
oc logs <pod> -f --previous
oc describe pod <pod> -n <namespace>
oc adm must-gather                         # Full cluster diagnostics
oc adm top nodes/pods
```

## Project Structure Conventions
```
demo/
├── CLAUDE.md
├── manifests/          # Raw Kubernetes/OpenShift YAML
│   ├── base/           # Shared base manifests
│   └── overlays/       # Environment-specific overlays (dev/staging/prod)
├── demos/              # Step-by-step demo scripts and guides
│   ├── ai-hosting/     # AI model serving demos
│   ├── operators/      # Operator demo scripts
│   └── pipelines/      # Tekton/OpenShift Pipelines demos
├── docs/               # Feature documentation and architecture diagrams
├── scripts/            # Automation and helper scripts
├── helm/               # Helm charts
└── notebooks/          # Jupyter notebooks for AI/data science demos
```

## OpenShift-Specific Patterns

### Security Context Constraints (SCC)
Always check SCC requirements before deploying. OpenShift is stricter than vanilla Kubernetes:
```bash
oc adm policy who-can use scc anyuid
oc adm policy add-scc-to-serviceaccount anyuid -z <sa> -n <ns>
```

### Routes vs Ingress
Prefer OpenShift `Route` over `Ingress` for demos:
```yaml
apiVersion: route.openshift.io/v1
kind: Route
metadata:
  name: <app>
spec:
  to:
    kind: Service
    name: <service>
  tls:
    termination: edge
```

### ImageStreams
Use ImageStreams for internal registry references, not raw image: tags.

### Resource Quotas
Always set resource requests/limits — required for most OpenShift namespaces:
```yaml
resources:
  requests:
    cpu: "100m"
    memory: "256Mi"
  limits:
    cpu: "500m"
    memory: "512Mi"
```

## AI Hosting on OpenShift — Key Components
- **OpenShift AI (RHOAI)**: Operator for full ML platform (Jupyter, pipelines, model serving)
- **KServe**: Serverless model inference (InferenceService CRD)
- **ModelMesh**: High-density model serving for many small models
- **OpenShift Pipelines (Tekton)**: CI/CD and ML training pipelines
- **OpenShift GitOps (ArgoCD)**: GitOps-based deployment of AI workloads
- **NVIDIA GPU Operator**: GPU node management for AI inference
- **Node Feature Discovery (NFD)**: Hardware feature detection for GPU scheduling

## Demo Documentation Style
- Each demo in `demos/` should have a `README.md` with prerequisites, steps, and expected output
- Include `oc` commands with comments explaining what each does
- Capture screenshots or `asciinema` recordings for presentation demos
- Note OpenShift version requirements (e.g., "Requires OCP 4.14+")

## Environment Variables (never hardcode)
```bash
CLUSTER_API_URL=
CLUSTER_DOMAIN=
OCP_VERSION=
AI_MODEL_NAMESPACE=
REGISTRY_URL=
```

## Useful OpenShift Console Paths
- Operator Hub: `/operatorhub`
- Developer perspective: `/dev`
- OpenShift AI dashboard: route in `redhat-ods-applications` namespace
- Pipelines: `/dev/pipelines`
