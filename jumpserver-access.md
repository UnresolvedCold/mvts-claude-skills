---
description: >
  Jump Server Access — establishes SSH access to the GreyOrange jump server (192.168.9.237)
  and gets kubectl credentials for a target cluster. Used as a prerequisite whenever kubectl
  exec/cp is needed but the MCP SA lacks container.pods.exec permission (common on QA/prod).
  The jump server uses key-pair auth configured in ~/.ssh/config as "JumpServer".
  TRIGGER when: user needs to exec into a pod, kubectl cp files, or run kubectl commands that
  require exec permission; MCP kube_exec returns Forbidden on pods/exec; user says "via jump
  server" or "SSH into".
  SKIP: read-only kubectl operations (get pods, describe, logs) that work fine via MCP kube_connect.
argument-hint: Optional cluster name and GCP project (e.g. "qa4-cluster greymatter-qa")
allowed-tools: ["Bash"]
---

# Jump Server Access

Establishes SSH connectivity to the GreyOrange jump server and fetches kubectl credentials
for a target cluster so that `kubectl exec` and `kubectl cp` commands work.

---

## Background

The MCP service account (`mcp-readonly-sa`) has read-only GKE permissions and cannot exec
into pods (`container.pods.exec` is missing). For any operation that requires exec (running
commands inside pods, copying files), commands must be routed through the SSH jump server
at `192.168.9.237`, which has full `kubectl` access via the user's gcloud credentials.

SSH config (`~/.ssh/config`) already has:
```
Host JumpServer
  HostName 192.168.9.237
  User shubham.kumar
```

---

## Step 1 — Verify SSH connectivity

```bash
ssh JumpServer "echo ok"
```

If this fails: check that the SSH key is loaded (`ssh-add -l`) or that `~/.ssh/config` has
the correct `IdentityFile` entry.

---

## Step 2 — Get cluster credentials on the jump server

Before `kubectl` works for a new cluster, credentials must be fetched on the jump server:

```bash
# QA clusters (qa4-cluster, qa3-cluster)
ssh JumpServer "gcloud container clusters get-credentials qa4-cluster --region us-central1 --project greymatter-qa"

# Staging / Sam's Club ATL (stg001-cluster)
ssh JumpServer "gcloud container clusters get-credentials stg001-cluster --region us-east4 --project gm-prod-sams-atl"

# Apotek prod
ssh JumpServer "gcloud container clusters get-credentials <cluster> --region <region> --project gm-prod-apotek"
```

GCP project → cluster mapping:
| Project | Cluster | Region |
|---------|---------|--------|
| `greymatter-qa` | `qa4-cluster`, `qa3-cluster` | `us-central1` |
| `gm-prod-sams-atl` | `samsatlprod-cluster`, `stg001-cluster` | `us-east4` |
| `gm-prod-apotek` | `apotekprod-cluster` | (check with `gcloud container clusters list`) |
| `gm-prod-walmart*` | walmart prod clusters | varies |

---

## Step 3 — Run kubectl commands via SSH

Once credentials are fetched, prefix all kubectl commands with `ssh JumpServer`:

```bash
# Check pod status
ssh JumpServer "kubectl get pods -n <namespace>"

# Exec into a pod
ssh JumpServer "kubectl exec -it mvts-0 -n <namespace> -- bash"

# Run a one-shot command
ssh JumpServer "kubectl exec mvts-0 -n <namespace> -- ls /app/data/idc"
```

---

## Step 4 — kubectl cp with infinite retry

`kubectl cp` frequently fails mid-transfer for large files. Always wrap in a retry loop:

```bash
# Copy from pod to jump server /tmp
while ! ssh JumpServer "kubectl cp <namespace>/mvts-0:/path/to/file /tmp/file"; do
  echo "Retrying..."; sleep 3
done

# Verify file landed and size looks right
ssh JumpServer "ls -lh /tmp/file"

# Compare with size in pod
ssh JumpServer "kubectl exec mvts-0 -n <namespace> -- ls -lh /path/to/file"
```

---

## Step 5 — Copy from jump server to local machine

```bash
scp JumpServer:/tmp/file ~/local/destination/
```

---

## Troubleshooting

| Symptom | Fix |
|---------|-----|
| `Permission denied (publickey)` | Run `ssh-add ~/.ssh/id_rsa` or check `IdentityFile` in `~/.ssh/config` |
| `namespaces "..." not found` | Run Step 2 first — credentials not fetched for this cluster yet |
| `Forbidden: pods/exec` | You're using MCP kube_exec — route through jump server instead |
| `setlocale: LC_ALL` warning | Benign warning from jump server shell init, safe to ignore |
| `kubectl cp` hangs/truncates | Use the retry loop in Step 4; verify file sizes match |
