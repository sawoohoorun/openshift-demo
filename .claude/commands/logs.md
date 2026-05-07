# logs

Tail or dump logs for a pod or deployment.

Usage: /logs [pod-or-deployment-name] [--previous] [--lines N]

Steps:
1. If name is a Deployment, resolve to the latest running pod with `oc get pods -l app=<name>`
2. Stream logs with `oc logs -f` (or `--previous` if that flag is given)
3. If the pod is crash-looping, automatically fetch previous logs too
4. Highlight ERROR/WARN lines in the summary
