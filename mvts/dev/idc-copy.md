---
description: >
  MVTS IDC File Copy — copies IDC distance files from mvts-0 pod (/app/data/idc) to the local
  machine for dev debugging of distance-related issues. Lists non-direction-aware IDC files,
  asks the user which bot-version file is needed, copies it + map.json from the pod to the
  jump server via kubectl cp with infinite retries, then transfers to local machine.
  TRIGGER when: user wants to copy/fetch/download IDC files from MVTS pod, replicate sim/prod
  distance issues locally, get IDC file for dev debugging.
  SKIP: IDC multiplier validation (use /mvts/dev/idc-validate); IDC sync pod crash (see reference_idc_sync_failure playbook).
argument-hint: Optional environment name (e.g. "aphrelaybulk", "stpbulk-onmbulk")
allowed-tools: ["Bash", "mcp__gor-global-mcp__list_environments", "mcp__gor-global-mcp__kube_connect",
  "mcp__gor-global-mcp__kube_shell", "mcp__gor-global-mcp__kube_exec"]
---

# MVTS IDC File Copy

Copies IDC distance files from `mvts-0:/app/data/idc` to the local machine for replicating
sim/prod distance issues in a dev environment.

---

## Background

MVTS loads IDC (Inter-Distance Cache) files at startup from `/app/data/idc/`. These files encode
the distances between grid locations and are used for transit-time estimation.

- Files are bot-version specific (e.g. `idc_v3.bin`, `idc_v4.bin`)
- **Direction-aware** files (`*direction_aware*`) are NOT understood by MVTS — always skip these
- `map.json` is always required alongside the IDC file — it maps grid coordinates and is needed
  for any local MVTS instance to function correctly

---

## Step 1 — Resolve environment and connect

Use the `mvts-debug` skill to resolve the environment alias to a namespace and connect to the pod.
If the user already provided the namespace explicitly, skip straight to Step 2.

```bash
mcp__gor-global-mcp__list_environments(query="<env_alias>")
```

---

## Step 2 — Check mvts-0 pod is running

**Before doing anything else**, verify the pod is up. If it isn't, abort immediately and tell the user.

```bash
ssh JumpServer "kubectl get pod mvts-0 -n <namespace> --no-headers 2>&1"
```

- If the output shows `Running` → proceed to Step 3.
- If the pod is missing, in `CrashLoopBackOff`, `Pending`, `Terminating`, or the statefulset
  shows `0/0` replicas → **stop here** and notify the user:

  > "mvts-0 is not running in `<namespace>` (status: `<status>`). Cannot copy IDC files.
  > Please start/scale up the MVTS pod and try again, or confirm you want to use a different environment."

  Do NOT attempt kubectl cp on a pod that isn't in Running state.

---

## Step 3 — List available IDC files

```bash
ssh JumpServer "kubectl exec mvts-0 -n <namespace> -- ls /app/data/idc"
```

Filter the output: **remove any filename containing `direction_aware`**.
Display the remaining files to the user and ask which one they need (they are named by bot version).

Example output after filtering:
```
map.json
distance_heuristic_liftdown_floor_1_ranger_version_HTM_HAI_K50_L_E1
distance_heuristic_liftup_floor_1_ranger_version_VTM_HAI_A42_TDV_E1
```

---

## Step 4 — Copy files from pod to jump server home directory

Copy **both** `map.json` and the user-selected IDC file from the pod to `~/` (home directory)
on the jump server. Users generally don't have write access to `/tmp` on the jump server.
Use `kubectl cp` with infinite retries (large files fail often).

```bash
# Copy map.json (always required)
while ! ssh JumpServer "kubectl cp <namespace>/mvts-0:/app/data/idc/map.json ~/map.json"; do
  echo "Retrying map.json copy..."; sleep 3
done

# Copy the selected IDC file
IDC_FILE="<selected_filename>"
while ! ssh JumpServer "kubectl cp <namespace>/mvts-0:/app/data/idc/${IDC_FILE} ~/${IDC_FILE}"; do
  echo "Retrying ${IDC_FILE} copy..."; sleep 3
done
```

Confirm both files landed on the jump server:
```bash
ssh JumpServer "ls -lh ~/map.json ~/${IDC_FILE}"
```

---

## Step 5 — Transfer from jump server to local machine

Ask the user where they want the files on their local machine (e.g. a specific project directory
or a default like `~/idc-files/`). Then scp both files:

```bash
LOCAL_DIR="<user_specified_local_dir>"
mkdir -p "${LOCAL_DIR}"

scp JumpServer:~/map.json "${LOCAL_DIR}/map.json"
scp "JumpServer:~/${IDC_FILE}" "${LOCAL_DIR}/${IDC_FILE}"
```

Confirm success:
```bash
ls -lh "${LOCAL_DIR}"
```

---

## Notes

- `kubectl cp` can silently truncate large files — always verify file size matches between pod
  and jump server: `ssh JumpServer "kubectl exec mvts-0 -n <ns> -- ls -lh /app/data/idc/<file>"` vs
  `ssh JumpServer "ls -lh ~/<file>"`
- If the environment's jump server isn't `JumpServer`, resolve it via the environment info or
  ask the user which jump server to use
- MVTS only reads the non-direction-aware IDC file; direction-aware files are an intermediate
  artifact and will be ignored or cause errors if loaded directly
