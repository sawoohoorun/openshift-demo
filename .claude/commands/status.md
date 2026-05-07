# status

Show the current state of all OpenShift resources in the active project.

Usage: /status [namespace]

Steps:
1. Run `oc project` to show the active namespace
2. Run `oc get pods,svc,routes,deployments,pvc` with wide output
3. Flag any pods that are not in Running/Completed state
4. Show recent events with `oc get events --sort-by=.lastTimestamp | tail -20`
5. Summarize what looks healthy vs. what needs attention
