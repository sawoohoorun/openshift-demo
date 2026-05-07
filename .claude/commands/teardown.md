# teardown

Remove all resources for the demo application from OpenShift.

Usage: /teardown [namespace]

Steps:
1. Confirm the namespace/project with the user before deleting anything
2. Delete all labeled resources: `oc delete all -l app=<name>`
3. Delete PVCs, secrets, configmaps if they carry the same label
4. Optionally delete the project itself if the user confirms
5. Verify nothing remains with `oc get all`

Safety: Always ask for confirmation before running any delete command.
