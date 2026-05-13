---
description: >
  MVTS First-Run Setup — configures SSH JumpServer, SSH key, and MCP permissions needed for MVTS debugging.
  TRIGGER when: user says "setup", "configure this skill", or is on a new machine and MVTS tools aren't working.
  SKIP: any actual debugging task — run setup first, then use the appropriate MVTS debug skill.
---

# MVTS Setup

Configures all prerequisites for MVTS debugging. Run automatically on first invocation, or when the user says "setup".

---

## 1. SSH JumpServer (required for stpbulk-cluster access)

Check if the JumpServer entry exists:
```bash
grep -c "Host JumpServer" ~/.ssh/config 2>/dev/null || echo "0"
```

If the count is `0`, add the entry. Prompt the user once for the JumpServer IP if unknown (default: `192.168.9.237`):
```bash
mkdir -p ~/.ssh && chmod 700 ~/.ssh
cat >> ~/.ssh/config << 'EOF'

Host JumpServer
    HostName 192.168.9.237
    User shubham.kumar
    IdentityFile ~/.ssh/id_rsa
    ServerAliveInterval 60
    ServerAliveCountMax 3
EOF
chmod 600 ~/.ssh/config
```

Verify connectivity:
```bash
ssh -o ConnectTimeout=5 -o BatchMode=yes JumpServer "echo OK" 2>&1
```

If `OK` is returned, JumpServer is reachable. If not, check VPN/network or confirm the IP.

---

## 2. SSH key (if JumpServer auth fails)

```bash
ls ~/.ssh/id_rsa 2>/dev/null && echo "KEY_EXISTS" || echo "MISSING"
```

If missing, generate one and remind the user to register the public key on the JumpServer:
```bash
ssh-keygen -t rsa -b 4096 -f ~/.ssh/id_rsa -N "" -C "mvts-debug-skill"
echo "Public key to register on JumpServer:"
cat ~/.ssh/id_rsa.pub
```

---

## 3. gor-global-mcp MCP permissions

Check for missing permissions:
```bash
python3 -c "
import json, os
path = os.path.expanduser('~/.claude/settings.local.json')
try:
    d = json.load(open(path))
    allowed = d.get('permissions', {}).get('allow', [])
    needed = [
        'mcp__gor-global-mcp__list_environments',
        'mcp__gor-global-mcp__kube_connect',
        'mcp__gor-global-mcp__kube_shell',
        'mcp__gor-global-mcp__kube_pod_logs',
        'mcp__gor-global-mcp__query_metrics',
        'mcp__gor-global-mcp__run_influxdb_query_from_catalog',
        'mcp__gor-global-mcp__search_influxql_catalog',
        'mcp__gor-global-mcp__describe_measurement',
        'mcp__gor-global-mcp__health_check',
    ]
    missing = [t for t in needed if t not in allowed]
    print('MISSING:', missing if missing else 'none')
except Exception as e:
    print('ERROR:', e)
"
```

If any are missing, add them:
```bash
python3 << 'EOF'
import json, os

path = os.path.expanduser('~/.claude/settings.local.json')
needed = [
    'mcp__gor-global-mcp__list_environments',
    'mcp__gor-global-mcp__kube_connect',
    'mcp__gor-global-mcp__kube_shell',
    'mcp__gor-global-mcp__kube_pod_logs',
    'mcp__gor-global-mcp__query_metrics',
    'mcp__gor-global-mcp__run_influxdb_query_from_catalog',
    'mcp__gor-global-mcp__search_influxql_catalog',
    'mcp__gor-global-mcp__describe_measurement',
    'mcp__gor-global-mcp__health_check',
    'Bash(ssh JumpServer *)',
    'Bash(ssh -o ConnectTimeout=* JumpServer *)',
]
try:
    d = json.load(open(path))
except (FileNotFoundError, json.JSONDecodeError):
    d = {}
d.setdefault('permissions', {}).setdefault('allow', [])
for t in needed:
    if t not in d['permissions']['allow']:
        d['permissions']['allow'].append(t)
with open(path, 'w') as f:
    json.dump(d, f, indent=2)
print('Done — permissions updated')
EOF
```

---

## 4. gor-global-mcp server availability

The `gor-global-mcp` server is provisioned automatically for GOR org members — no manual install needed. If `mcp__gor-global-mcp__list_environments` is unavailable, wait 10s and retry or restart the Claude Code session.

---

## 5. Installing skills on a new machine

Skills are organized in a directory structure under `~/.claude/commands/`:

```
~/.claude/commands/
├── index.md              ← global entry point
└── mvts/
    ├── index.md          ← MVTS entry point
    ├── setup.md          ← this file
    ├── ops/              ← operations staff skills
    │   ├── relay.md
    │   ├── vtm-stuck.md
    │   ├── crash.md
    │   └── config.md
    └── dev/              ← engineering skills
        ├── influxdb.md
        ├── idc-copy.md
        ├── idc-validate.md
        ├── build.md
        └── operator-time-audit.md
```

To install from the repo:
```bash
git clone git@github.com:UnresolvedCold/mvts-claude-skills.git ~/Projects/GreyOrange/mvts-claude-skills
mkdir -p ~/.claude/commands/mvts/ops ~/.claude/commands/mvts/dev

# Link directory-structured skills
for dir in "" "mvts" "mvts/ops" "mvts/dev"; do
  src_dir=~/Projects/GreyOrange/mvts-claude-skills/$dir
  dst_dir=~/.claude/commands/$dir
  [ -d "$src_dir" ] && for f in "$src_dir"/*.md; do
    [[ "$(basename $f)" == "README.md" ]] && continue
    ln -sf "$f" "$dst_dir/$(basename $f)"
  done
done
```

Then run `/mvts/setup` to execute this checklist.

---

## 6. Check for new skills

After setup, check for new skills in the repo:

```bash
cd ~/Projects/GreyOrange/mvts-claude-skills && git fetch && git log HEAD..origin/main --oneline
```

Compare installed vs available (including subdirectories):
```bash
diff <(find ~/Projects/GreyOrange/mvts-claude-skills -name "*.md" ! -name "README.md" | sed "s|.*/mvts-claude-skills/||" | sort) \
     <(find ~/.claude/commands -name "*.md" | sed "s|.*/.claude/commands/||" | sort)
```

If any files appear only on the left side (not installed), install them:
```bash
git -C ~/Projects/GreyOrange/mvts-claude-skills pull
# Re-run the install loop from Step 5
```
