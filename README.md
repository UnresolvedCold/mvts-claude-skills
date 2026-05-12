# MVTS Claude Skills

Claude Code skills for debugging and deploying MVTS (Multi-Vehicle Task Scheduler) across any GOR environment.

## Requirements

- [Claude Code](https://claude.ai/code) with an active session
- **gor-global-mcp** MCP server — auto-provisioned for GOR org members on claude.ai enterprise; no manual install needed
- **SSH access to JumpServer** — required for `stpbulk-cluster` environments only (see setup below)
- **VPN** — must be on GOR VPN for JumpServer connectivity

## Installation

```bash
# 1. Clone the repo
git clone git@github.com:greyorange/mvts-claude-skills.git ~/Projects/GreyOrange/mvts-claude-skills

# 2. Symlink all skills into Claude's commands directory
mkdir -p ~/.claude/commands
for f in ~/Projects/GreyOrange/mvts-claude-skills/*.md; do
  [[ "$(basename "$f")" == "README.md" ]] && continue
  ln -sf "$f" ~/.claude/commands/"$(basename "$f")"
done
```

> **Why symlinks?** Claude Code loads every `.md` file in `~/.claude/commands/` as a skill. Keeping the repo separate (with the README and other non-skill files) prevents them from being loaded as skills.

Then run the first-time setup inside Claude Code:

```
/mvts-debug setup
```

Claude will automatically check and configure SSH, permissions, and all prerequisites.

## First-Time Setup

Running `/mvts-debug setup` will automatically:

1. **SSH JumpServer** — add `Host JumpServer` to `~/.ssh/config` if missing
2. **SSH key** — generate `~/.ssh/id_rsa` if missing and print the public key to register
3. **MCP permissions** — add `gor-global-mcp` tool allowlist entries to `~/.claude/settings.local.json`

After setup, verify JumpServer connectivity (required for stpbulk environments):

```bash
ssh JumpServer "echo OK"
```

## Skills

### `/mvts-debug` — MVTS Debugging Guide

Step-by-step playbook for diagnosing live MVTS issues across any environment.

**Trigger**: Claude invokes this automatically when you mention MVTS issues — deadlocks, high TPQ, bots not assigned, PPS starvation, relay assignment failures, CrashLoopBackOff on `mvts-0`, etc.

**Supports all cluster types:**

| Cluster prefix | Access method |
|---------------|---------------|
| `qa4-cluster-*` | GCP SA via `kube_connect` + `kube_shell` |
| `stpbulk-cluster-*` | SSH JumpServer |

**Usage:**

```
/mvts-debug                        # load the guide
/mvts-debug qa4-relaypotepic       # load + set environment context
/mvts-debug setup                  # run first-time setup checklist
```

**Covers:**
- Pod health checks and config inspection
- Problem statement (PS) extraction and entity lookup (bots, tasks, relay points, PPS)
- IDC travel time queries
- Relay assignment debugging (HTM bots not getting assigned)
- VTM cyclic deadlock detection and traceback
- InfluxDB queries: `mvts_rtp_problem_assignment_details`, `mvts_pred_vs_real_times`, `task_cycle_times`, `mvts_relay_vtm_bot_*`
- Operator timeline gap analysis (transit overrun, PPS queue wait, +24h starvation bug)
- Config flag reference (`ENABLE_VTM_TASK_REASSIGNMENT`, `ENABLE_CYCLE_DEASSIGNMENT_FOR_FAILED_TAM`)
- CrashLoopBackOff recovery

---

### `/mvts-build` — MVTS Build & Deploy

Automates the full MVTS release workflow: commit → push → trigger GitHub Actions → monitor → deploy.

**Usage:**

```
/mvts-build
/mvts-build "fix relay assignment timeout"    # with commit message
```

**Covers:**
- Detects working worktree (`f4`, `f5`, `vrp-obts-rec`, etc.)
- Stages, commits, and pushes changes
- Triggers `create-build.yml` GitHub Actions workflow
- Monitors build progress
- Updates `values.yaml` in the deployment manifests repo and pushes to the solution branch

## Environment Examples

| Alias | Full namespace | Cluster |
|-------|---------------|---------|
| `qa4-relaypotepic` | `qa4-cluster-relaypotepic-greymatter` | `qa4-cluster` |
| `onmbulk` | `stpbulk-cluster-onmbulk-greymatter` | `stpbulk-cluster` |
| `aphrelaybulk` | `stpbulk-cluster-aphrelaybulk-greymatter` | `stpbulk-cluster` |
| `stg001` | `stg001-cluster-walmartpotstg-greymatter` | `stg001-cluster` |

Use any alias directly — the skill resolves it to the full namespace via `list_environments`.

## Updating

```bash
cd ~/Projects/GreyOrange/mvts-claude-skills
git pull
```
