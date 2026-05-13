---
description: >
  MVTS pod crash recovery — diagnoses and fixes mvts-0 pod crash loops.
  TRIGGER when: mvts-0 is in CrashLoopBackOff, pod keeps restarting, MVTS is not running.
  TRIGGER also when: user says "MVTS is down", "the scheduler stopped", "system isn't running",
  "service crashed", "[env] is not working at all", "warehouse has completely stopped".
  SKIP: pod is Running — use /mvts/index instead.
---

# MVTS Crash Recovery

---

## Communication Protocol (always follow)

- Run all diagnostic steps silently. Do NOT show kubectl commands, log output, or error codes to the user.
- Translate every finding into plain English before presenting it.
- Tell the user what you found and what you did — not how you did it.
- If you fix the problem, confirm in plain English. If you can't, provide the escalation message.

---

## Step 1 — Resolve environment and namespace (silently)

```bash
# On-prem (stpbulk-cluster-*)
ssh JumpServer "kubectl get namespaces | grep -i '<env>'"
# Namespace pattern: stpbulk-cluster-<env>-greymatter

# GCP (qa4-cluster-*)
mcp__gor-global-mcp__list_environments  # then kube_connect
```

---

## Step 2 — Confirm the pod is crashing (silently)

**On-prem:**
```bash
ssh JumpServer "kubectl get pod mvts-0 -n <namespace>"
```

**GCP:**
```
mcp__gor-global-mcp__kube_shell(command="kubectl get pod mvts-0 -n <namespace>")
```

Tell the user:
> "I can see the MVTS service in [env] is in a crash loop — it keeps starting and immediately failing. I'm reading the crash logs now."

---

## Step 3 — Read previous container logs (silently)

**On-prem:**
```bash
ssh JumpServer "kubectl logs mvts-0 -n <namespace> --previous | tail -50"
```

**GCP:**
```
mcp__gor-global-mcp__kube_pod_logs(pod="mvts-0", namespace="<namespace>", previous=true, tail=50)
```

Identify the root cause from the log output.

---

## Step 4 — Apply fix based on root cause

### Case A — Corrupt JAR (`Invalid or corrupt jarfile app.jar`)

Tell the user:
> "Found it — the MVTS application file got corrupted during the last update. This is a known issue and can be fixed with a restart. Restarting now..."

Apply the fix (StatefulSet bounce):

**On-prem:**
```bash
ssh JumpServer "kubectl scale sts mvts -n <namespace> --replicas=0 && sleep 5 && kubectl scale sts mvts -n <namespace> --replicas=1"
```

**GCP:**
```
mcp__gor-global-mcp__kube_shell(command="kubectl scale sts mvts -n <namespace> --replicas=0 && sleep 5 && kubectl scale sts mvts -n <namespace> --replicas=1")
```

This forces a fresh pull/copy of the JAR. Safe to do — not a code issue.

Then wait for the pod to reach `Running 1/1`:
```bash
ssh JumpServer "kubectl get pod mvts-0 -n <namespace>"
```

On success, tell the user:
> "The MVTS service in [env] has been restarted and is now running normally. It will take about 30–60 seconds to fully initialize before it starts assigning bots."

### Case B — OOMKilled (out of memory)

Tell the user:
> "MVTS ran out of memory and the system terminated it. This can sometimes be resolved with a simple restart, but it may recur if the memory limit is too low."

Apply the same StatefulSet bounce as Case A. Monitor after recovery.

If it crashes again → use escalation template below.

### Case C — Unknown / unrecognized error

Tell the user:
> "The service crashed with an error I haven't seen before. I'm not able to fix this automatically — it needs an engineer to look at the logs."

Use the escalation template below.

---

## Step 5 — Verify recovery (silently)

Check pod status after the bounce:
```bash
ssh JumpServer "kubectl get pod mvts-0 -n <namespace>"
```

- `Running 1/1` → recovered. Tell the user.
- Still crashing → use escalation template.

---

## Escalation Message Template

Show this when the issue cannot be auto-resolved:

```
To: MVTS Engineering Team
Environment: [env name]
Time detected: [timestamp]

The MVTS task scheduler in [env] is in a crash loop and could not be restarted automatically.

Error summary: [1-2 plain-English sentences describing the crash, e.g. "The service started but immediately failed with an out-of-memory error. A restart was attempted but it crashed again within 30 seconds."]

What was tried: Pod restart (StatefulSet scale down/up)

Impact: No bot assignments are being made in [env]. All warehouse movement is paused.

Action needed: Please review the pod logs and determine the root cause.
Logs command: kubectl logs mvts-0 -n [namespace] --previous
```

---

## Diagnosis Summary (show to user at the end)

```
**Environment**: [env name]
**Status**: [Recovered / Still crashing / Needs engineer]
**What happened**: [plain English, e.g. "The MVTS service file got corrupted and the pod kept failing to start."]
**What I did**: [e.g. "Restarted the service. It is now running normally."]
**What happens next**: [e.g. "Bots will resume normal assignments within 60 seconds." / "See escalation message above — engineering team needs to investigate."]
```

---

## Important notes (internal)

- **NEVER patch the StatefulSet directly** (`kubectl set image sts/mvts ...`) — the CD pipeline will detect drift and revert it. Always update `values.yaml`, commit, and push to the solution branch.
- To verify which image is actually running: `kubectl describe pod mvts-0 -n <namespace> | grep -i 'Image:'`
