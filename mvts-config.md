---
description: >
  MVTS Config — read and update MVTS runtime configuration without restarting the pod.
  TRIGGER when: user wants to change a config property, check current config values, enable/disable a feature flag,
  or tune any MVTS parameter on a named environment.
  SKIP: changes that require a pod restart (e.g. changing Kafka topics, database connection strings).
argument-hint: "<environment> <PROPERTY=value> [PROPERTY=value ...]"
---

# MVTS Config

Read and update MVTS runtime config via the `/mvts/config` API — no pod restart needed. Changes are applied immediately and persisted to `local.application.properties`.

---

## Step 0 — Resolve namespace

If not already known, resolve the environment to a namespace:
```bash
ssh JumpServer "kubectl get namespaces | grep -i '<env>'"
# Namespace pattern: stpbulk-cluster-<env>-greymatter
```

For GCP environments, use `mcp__gor-global-mcp__list_environments` then `kube_connect`.

---

## Read config

**All properties:**
```bash
ssh JumpServer "kubectl exec mvts-0 -n <namespace> -- bash -c 'curl -s localhost:8080/mvts/config/all'"
# GCP:
mcp__gor-global-mcp__kube_shell(command="kubectl exec mvts-0 -n <namespace> -- bash -c 'curl -s localhost:8080/mvts/config/all'")
```

**Search for specific properties** (pipe through grep/python):
```bash
ssh JumpServer "kubectl exec mvts-0 -n <namespace> -- bash -c 'curl -s localhost:8080/mvts/config/all'" | \
  python3 -c "import sys,json; d=json.load(sys.stdin); [print(k,':',v) for k,v in d.items() if '<keyword>' in k.lower()]"
```

---

## Update config

**Single property:**
```bash
ssh JumpServer "kubectl exec mvts-0 -n <namespace> -- bash -c 'curl -s \"localhost:8080/mvts/config/set?property=<PROPERTY_NAME>&value=<value>\"'"
```

**Multiple properties at once:**
```bash
ssh JumpServer "kubectl exec mvts-0 -n <namespace> -- bash -c 'curl -s -X POST localhost:8080/mvts/config/set -H \"Content-Type: application/json\" -d \"{\\\"PROPERTY_1\\\": \\\"value1\\\", \\\"PROPERTY_2\\\": \\\"value2\\\"}\"'"
```

The response confirms the change:
```json
{"property": "PROPERTY_NAME", "old_value": "...", "new_value": "..."}
```
For multi-property updates:
```json
{"PROPERTY_1": {"old_value": "...", "new_value": "..."}, "PROPERTY_2": {"old_value": "...", "new_value": "..."}}
```

---

## Common properties

| Property | Default | Description |
|----------|---------|-------------|
| `IDC_MULTIPLIER_STEP` | 0.25 | Step size for IDC multiplier adjustment per cycle |
| `IDC_MULTIPLIER_START` | 1.0 | Minimum IDC multiplier |
| `IDC_MULTIPLIER_END` | 5.0 | Maximum IDC multiplier |
| `ENABLE_PERIODIC_IDC_MULTIPLIER_CALCULATION` | true | Enable dynamic IDC multiplier recalculation |
| `DELTA_TIME_FOR_CALCULATING_IDC_MULTIPLIER` | 15m | Time window for IDC multiplier calculation |
| `SAME_RACK_QUEUE_OVERFLOW_MULTIPLIER` | 0 | Transit time multiplier when rack queue overflows |
| `SAME_RACK_QUEUE_OVERFLOW_EXTRA_ALLOWED_BOTS` | 4 | Extra bots allowed at same rack before overflow penalty |
| `ENABLE_VTM_TASK_REASSIGNMENT` | true | Enable VTM task reassignment/deadlock detection per cycle |
| `ENABLE_CYCLE_DEASSIGNMENT_FOR_FAILED_TAM` | false | Cascade deassignment when TAM chain fails |
| `ENABLE_RELAY` | true | Enable relay subsystem |
| `ENABLE_KAFKA` | true | Enable Kafka messaging |
| `ENABLE_TRANSIT_TIME_MODEL` | true | Enable transit time model |
| `BOT_CYCLE_TIME_MULTIPLIER` | 1.25 | Multiplier applied to bot cycle time estimates |
| `DYNAMIC_TASK_MULTIPLIER` | 2.5 | Multiplier for dynamic task time estimation |
| `FIRST_TOTE_IDC_THRESHOLD` | 30 | IDC threshold for first tote assignment (seconds) |
| `CONTAINER_COMPLETENESS_MULTIPLIER_RELAY` | 20 | Container completeness score multiplier for relay |

---

## Notes

- Changes take effect immediately in the next solver cycle (~13s).
- Changes are also persisted to `/app/data/config/local.application.properties` inside the pod, so they survive a pod restart.
- **Exception**: If the pod is bounced via StatefulSet scale-down/up, the persisted file may be reset from the ConfigMap. Re-apply manually if needed after a bounce.
- For permanent changes, update `values.yaml` in the solution branch and push via CD pipeline.
