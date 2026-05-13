---
description: >
  MVTS runtime config — read and update MVTS configuration without restarting the pod.
  TRIGGER when: user wants to change a config property, check current values, enable/disable
  a feature flag, or tune any MVTS parameter on a named environment.
  TRIGGER also when: user says "can you enable [feature] in [env]", "turn on/off [feature]",
  "relay is disabled", "check what settings are active", "change the [setting] in [env]",
  "I want to adjust [something] in [env]".
  SKIP: changes that require a pod restart (Kafka topics, database connection strings).
argument-hint: "<environment> <what to change or check>"
---

# MVTS Config

Read and update MVTS runtime config via the `/mvts/config` API — no pod restart needed. Changes take effect in the next solver cycle (~13 seconds).

---

## Communication Protocol (always follow)

- Resolve the environment and read current config silently.
- Before making any change, confirm with the user in plain English what will change and what effect it will have.
- After the change, confirm success in plain English with the old and new value.
- Never show raw curl commands or JSON config dumps to the user.

---

## Step 0 — Resolve namespace (silently)

```bash
# On-prem
ssh JumpServer "kubectl get namespaces | grep -i '<env>'"

# GCP
mcp__gor-global-mcp__list_environments  # then kube_connect
```

---

## Plain-English Property Aliases

If the user refers to a setting by plain English, map it to the technical property:

| What the user might say | Technical property |
|------------------------|-------------------|
| "enable/disable relay" | `ENABLE_RELAY` |
| "turn on/off Kafka / messaging" | `ENABLE_KAFKA` |
| "transit time model" | `ENABLE_TRANSIT_TIME_MODEL` |
| "IDC multiplier / congestion factor calculation" | `ENABLE_PERIODIC_IDC_MULTIPLIER_CALCULATION` |
| "VTM reassignment / deadlock detection" | `ENABLE_VTM_TASK_REASSIGNMENT` |
| "bot cycle time multiplier / speed factor" | `BOT_CYCLE_TIME_MULTIPLIER` |
| "IDC logging to InfluxDB / metrics logging" | `LOG_TO_INFLUXDB` |
| "static aisle map / fixed bot-to-aisle assignment" | `ENABLE_STATIC_BOT_AISLE_MAP` |
| "dynamic task multiplier" | `DYNAMIC_TASK_MULTIPLIER` |
| "first tote IDC threshold" | `FIRST_TOTE_IDC_THRESHOLD` |

---

## Confirmation Flow (always do this before changing)

Before applying any change, tell the user:

> "I'm about to change **[plain-English description]** in **[env]** from `[old value]` to `[new value]`. This will take effect in the next solver cycle (~13 seconds). Should I proceed?"

Wait for confirmation, then apply.

---

## Read config (internal commands)

**All properties:**
```bash
ssh JumpServer "kubectl exec mvts-0 -n <namespace> -- bash -c 'curl -s localhost:8080/mvts/config/all'"
# GCP:
mcp__gor-global-mcp__kube_shell(command="kubectl exec mvts-0 -n <namespace> -- bash -c 'curl -s localhost:8080/mvts/config/all'")
```

**Search for specific properties:**
```bash
ssh JumpServer "kubectl exec mvts-0 -n <namespace> -- bash -c 'curl -s localhost:8080/mvts/config/all'" | \
  python3 -c "import sys,json; d=json.load(sys.stdin); [print(k,':',v) for k,v in d.items() if '<keyword>' in k.lower()]"
```

---

## Update config (internal commands)

**Single property:**
```bash
ssh JumpServer "kubectl exec mvts-0 -n <namespace> -- bash -c 'curl -s \"localhost:8080/mvts/config/set?property=<PROPERTY_NAME>&value=<value>\"'"
```

**Multiple properties:**
```bash
ssh JumpServer "kubectl exec mvts-0 -n <namespace> -- bash -c 'curl -s -X POST localhost:8080/mvts/config/set -H \"Content-Type: application/json\" -d \"{\\\"PROPERTY_1\\\": \\\"value1\\\", \\\"PROPERTY_2\\\": \\\"value2\\\"}\"'"
```

Response confirms the change:
```json
{"property": "PROPERTY_NAME", "old_value": "...", "new_value": "..."}
```

---

## Common Properties Reference

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

## Post-change confirmation (show to user)

```
**Environment**: [env name]
**Change applied**: [plain-English description, e.g. "Relay has been enabled"]
**Old value**: [value]
**New value**: [value]
**Takes effect**: Next solver cycle (~13 seconds)
**Note**: [if relevant] This change is persisted to the config file and will survive a pod restart.
For a permanent change, it also needs to be updated in values.yaml in the deployment repo.
```

---

## Notes (internal)

- Changes take effect immediately in the next solver cycle (~13s).
- Changes are persisted to `/app/data/config/local.application.properties` and survive a pod restart.
- **Exception**: If the pod is bounced via StatefulSet scale-down/up, the file may be reset from the ConfigMap. Re-apply manually after a bounce.
- For permanent changes, update `values.yaml` in the solution branch and push via CD pipeline.
