# scaffold

Generate OpenShift/Kubernetes manifests for a new application.

Usage: /scaffold [app-name] [image] [port]

Generates:
- Deployment (with resource requests/limits and liveness/readiness probes)
- Service
- Route (OpenShift) or Ingress (plain Kubernetes)
- ConfigMap skeleton
- Optional: PersistentVolumeClaim if storage is needed

Output files go to `./manifests/<app-name>/`.
Ask the user for any missing values before generating.
