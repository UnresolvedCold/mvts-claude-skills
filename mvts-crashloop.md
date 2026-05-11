---
description: >
  MVTS CrashLoopBackOff Debugging — diagnoses and fixes mvts-0 pod crash loops.
  TRIGGER when: mvts-0 is in CrashLoopBackOff, pod keeps restarting, MVTS is not running.
  SKIP: pod is Running — use mvts-debug instead.
---

# MVTS CrashLoopBackOff Debugging

---

## Step 1 — Resolve environment and namespace

```bash
# On-prem (stpbulk-cluster-*)
ssh JumpServer "kubectl get namespaces | grep -i '<env>'"
# Namespace pattern: stpbulk-cluster-<env>-greymatter

# GCP (qa4-cluster-*)
mcp__gor-global-mcp__list_environments  # then kube_connect
```

---

## Step 2 — Confirm the pod is crashing

**On-prem:**
```bash
ssh JumpServer "kubectl get pod mvts-0 -n <namespace>"
```

**GCP:**
```
mcp__gor-global-mcp__kube_shell(command="kubectl get pod mvts-0 -n <namespace>")
```

---

## Step 3 — Check previous container logs

**On-prem:**
```bash
ssh JumpServer "kubectl logs mvts-0 -n <namespace> --previous | tail -50"
```

**GCP:**
```
mcp__gor-global-mcp__kube_pod_logs(pod="mvts-0", namespace="<namespace>", previous=true, tail=50)
```

---

## Common cause — corrupt JAR

Error: `Invalid or corrupt jarfile app.jar`

Fix with a StatefulSet bounce:

**On-prem:**
```bash
ssh JumpServer "kubectl scale sts mvts -n <namespace> --replicas=0 && sleep 5 && kubectl scale sts mvts -n <namespace> --replicas=1"
```

**GCP:**
```
mcp__gor-global-mcp__kube_shell(command="kubectl scale sts mvts -n <namespace> --replicas=0 && sleep 5 && kubectl scale sts mvts -n <namespace> --replicas=1")
```

This forces a fresh pull/copy of the JAR. Not a code or deployment issue — safe to bounce.

---

## Verify recovery

```bash
ssh JumpServer "kubectl get pod mvts-0 -n <namespace> -w"
```

Wait for `Running 1/1`. Then run `/mvts-debug` to continue with functional debugging.

---

## Notes

- **NEVER patch the StatefulSet directly** (`kubectl set image sts/mvts ...`) — the CD pipeline will detect drift and revert it. Always update `values.yaml`, commit, and push to the solution branch.
- To verify which image is actually running: `kubectl describe pod mvts-0 -n <namespace> | grep -i 'Image:'`
