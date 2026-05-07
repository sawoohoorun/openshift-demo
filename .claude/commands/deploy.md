# deploy

Deploy the application to OpenShift.

Usage: /deploy [app-name] [namespace]

Steps:
1. Build the container image with `podman build` or `docker build`
2. Push to the registry
3. Apply manifests with `oc apply -f`
4. Watch rollout status with `oc rollout status`
5. Report the route URL

If no app-name is given, infer from the current directory or nearest Deployment/DeploymentConfig manifest.
If no namespace is given, use the current `oc project`.
