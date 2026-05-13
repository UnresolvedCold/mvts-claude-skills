---
description: >
  Global skill index — entry point for all operational and engineering tasks.
  TRIGGER when: user describes a problem without naming a system, asks "what skill should I use",
  asks "how do I debug X", or starts a session with a vague problem description.
  TRIGGER also when: user says "something is wrong", "help me debug", "the warehouse is having
  issues", "I don't know which skill to use".
---

# Skill Index

Entry point for all available skills. Identify the system from what the user described, then route to the right skill.

---

## Available Systems

| System | Entry Point | Use when... |
|--------|-------------|-------------|
| **MVTS** (task scheduler, bot assignments) | `/mvts/index` | Bots not moving, orders stuck, relay issues, VTM deadlock, MVTS down |
| _(more systems coming)_ | | |

---

## If you don't know which system

Ask the user:
> "Which system are you having issues with? For example: MVTS (bot task scheduling), GMC (warehouse control), Navigation, or something else?"

Then route to the appropriate system index.

---

## Skill Directory Structure

```
/index              ← you are here (global entry point)
/mvts/index         ← all MVTS operational issues
/mvts/setup         ← first-run setup for MVTS skills
/mvts/ops/relay     ← HTM bots not picking up totes
/mvts/ops/vtm-stuck ← VTM storage bots stuck or not assigned
/mvts/ops/crash     ← MVTS service crashed / not running
/mvts/ops/config    ← change MVTS settings at runtime
/mvts/dev/influxdb          ← [eng] InfluxDB schema and queries
/mvts/dev/idc-copy          ← [eng] Copy IDC distance files from pod
/mvts/dev/idc-validate      ← [eng] Validate IDC congestion multiplier
/mvts/dev/build             ← [eng] Build and deploy MVTS
/mvts/dev/operator-time-audit ← [eng] Audit operator time inflation
```

Skills marked `[eng]` are for engineers only. Operations users should describe the problem in plain English to `/mvts/index` instead.
