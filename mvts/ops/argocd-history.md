---
description: >
  MVTS ArgoCD deployment history — shows what MVTS build versions were deployed to an environment
  and when, by querying the ArgoCD instance that manages that cluster.
  TRIGGER when: user asks "what build was deployed", "when was MVTS last updated", "what version
  history does this env have", "show me argocd history", "what was the previous version",
  "has MVTS been updated recently" for a named environment.
  SKIP: reading live MVTS logs or config — use /mvts/index or /mvts/ops/config instead.
argument-hint: "<environment>"
---

# MVTS ArgoCD Deployment History

Shows the history of MVTS deployments for an environment by querying the ArgoCD that manages
that cluster. Each entry shows when a deploy happened, which helm chart revision was used,
and who triggered it.

---

## Step 1 — Resolve the cluster

Use `mcp__gor-global-mcp__list_environments` or `mcp__gor-global-mcp__kube_discover` to
identify the GCP project, cluster name, and region for the environment.

```
mcp__gor-global-mcp__kube_discover(what="clusters")
```

Then connect:
```
mcp__gor-global-mcp__kube_connect(cluster="<cluster>", project="<project>", region="<region>")
```

---

## Step 2 — Find the ArgoCD URL

The ArgoCD server is exposed via an ingress in the `argocd` namespace. Get the hostname:

```bash
ssh JumpServer "kubectl get ingress --all-namespaces \
  --context <kube-context> 2>&1 | grep -i argo"
```

Expected output:
```
argocd   argocd-server   nginx   argocd-<cluster>.greymatter.greyorange.com   <IP>   80,443   Xd
```

If the cluster isn't in the jump server's kubeconfig yet, add it first:
```bash
ssh JumpServer "gcloud container clusters get-credentials <cluster> \
  --project <project> --region <region>"
```

**QA clusters** (qa4, qa3) use: `argocd-qa4.greymatter.greyorange.com` / `argocd-qa3.greymatter.greyorange.com`

---

## Step 3 — Get the ArgoCD password

The password lives in a secret in the `argocd` namespace. The MCP SA can't read secrets,
so use the jump server:

```bash
ssh JumpServer "kubectl get secret argocd-initial-admin-secret \
  -n argocd --context <kube-context> \
  -o jsonpath='{.data.password}' | base64 -d"
```

> If the secret doesn't exist, SSO is configured — ask the user for their credentials.

---

## Step 4 — Find the MVTS app in ArgoCD

Login and list apps filtered to MVTS:

```bash
TOKEN=$(curl -s -k -X POST "https://<argocd-url>/api/v1/session" \
  -H "Content-Type: application/json" \
  -d '{"username":"admin","password":"<password>"}' \
  | python3 -c "import sys,json; print(json.load(sys.stdin)['token'])")

curl -s -k "https://<argocd-url>/api/v1/applications?limit=200" \
  -H "Authorization: Bearer $TOKEN" | python3 -c "
import sys,json
apps=json.load(sys.stdin).get('items',[])
for a in apps:
    name=a['metadata']['name']
    ns=a['spec']['destination'].get('namespace','')
    if 'mvts' in name.lower():
        print(name, '|', ns)
"
```

The app name typically follows `<env-prefix>-mvts` (e.g. `apotekstg-mvts`, `apotekrelease-mvts`).

---

## Step 5 — Get and display deployment history

```bash
TOKEN=$(curl -s -k -X POST "https://<argocd-url>/api/v1/session" \
  -H "Content-Type: application/json" \
  -d '{"username":"admin","password":"<password>"}' \
  | python3 -c "import sys,json; print(json.load(sys.stdin)['token'])")

curl -s -k "https://<argocd-url>/api/v1/applications/<app-name>" \
  -H "Authorization: Bearer $TOKEN" | python3 -c "
import sys,json
a=json.load(sys.stdin)
history=a.get('status',{}).get('history',[])
print(f'{'ID':<4} {'Deployed (UTC)':<22} {'Chart revision':<20} {'Values SHA':<14} {'By'}')
print('-'*75)
for h in history:
    hid=h.get('id','')
    deployed=h.get('deployedAt','')
    sources=h.get('sources',[])
    chart_rev=next((s.get('targetRevision','') for s in sources if 'helm-charts' in s.get('repoURL','')), '')
    revisions=h.get('revisions',[])
    dep_sha=revisions[1][:12] if len(revisions)>1 else (revisions[0][:12] if revisions else '')
    initiated=h.get('initiatedBy',{})
    by='auto' if initiated.get('automated') else initiated.get('username','?')
    print(f'{hid:<4} {deployed:<22} {chart_rev:<20} {dep_sha:<14} {by}')
"
```

---

## Output format (show to user)

```
MVTS deployment history — <env> (<app-name>)

 ID  Deployed (UTC)         Chart revision       Values SHA     By
 ─────────────────────────────────────────────────────────────────
  0  2026-04-15T02:25:07Z   release-7.8.0        37824e3a843d   auto
  1  2026-04-15T02:32:03Z   release-7.8.0        37824e3a843d   admin
  2  2026-04-23T05:34:48Z   release-7.8.0        b60069ebc9df   auto
  3  2026-04-23T05:46:29Z   7.8.0.0              b60069ebc9df   admin
  4  2026-05-04T10:49:09Z   7.8.0.0              25cba253d652   admin
  5  2026-05-15T10:50:14Z   7.8.0.0              6223d8e85e9b   admin  ← current

Current running image: multifleet_planner:<tag>
ArgoCD URL: https://<argocd-url>  (login: admin / <password>)
```

- **Chart revision** = which version of the helm chart (= which MVTS feature set) was deployed
- **Values SHA** = commit in `greymatter-deployment` repo at deploy time (contains the exact image tag)
- **By** = `auto` means ArgoCD self-heal/sync; `admin` means a human triggered it

---

## Notes

- ArgoCD history only goes back as long as ArgoCD has been running on that cluster.
- The exact Docker image tag per deploy is in `greymatter-deployment` repo at the values SHA shown.
  To look it up: `git show <sha>:platform-k8s/mvts-applications/manifests/values.yaml | grep tag`
- QA4 ArgoCD (`argocd-qa4.greymatter.greyorange.com`) manages all `qa4-cluster-*` namespaces.
- Each prod cluster (e.g. `apotekinc01-cluster`) has its own ArgoCD instance at
  `argocd-<cluster-name>.greymatter.greyorange.com`.
- MCP SA cannot read secrets — always use the jump server for the password step.
