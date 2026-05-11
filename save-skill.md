# Save Skill

Saves a new skill correctly: file goes in the git repo, symlink goes in `~/.claude/commands/`.

**ALWAYS use this procedure when saving any new skill. Never write skill files directly into `~/.claude/commands/`.**

---

## Procedure

### 1. Write the skill file into the repo

```bash
# Skill files live here — NOT in ~/.claude/commands/
~/Projects/GreyOrange/mvts-claude-skills/<skill-name>.md
```

### 2. Create a symlink in `~/.claude/commands/`

```bash
ln -sf ~/Projects/GreyOrange/mvts-claude-skills/<skill-name>.md ~/.claude/commands/<skill-name>.md
```

### 3. Verify

```bash
ls -la ~/.claude/commands/
# All entries must be symlinks (lrwxr-xr-x), not regular files (-rw-r--r--)
```

### 4. Commit and push

```bash
cd ~/Projects/GreyOrange/mvts-claude-skills
git add <skill-name>.md
git commit -m "Add <skill-name> skill"
git push
```

---

## Quick one-liner (after writing the file)

```bash
ln -sf ~/Projects/GreyOrange/mvts-claude-skills/<skill-name>.md ~/.claude/commands/<skill-name>.md
```

---

## Why

Skills are versioned in `git@github.com:greyorange/mvts-claude-skills.git`. The `~/.claude/commands/` directory only holds symlinks so the repo is the single source of truth and skills can be installed on a new machine with a single `git clone` + symlink loop.

Installing on a new machine:
```bash
git clone git@github.com:greyorange/mvts-claude-skills.git ~/Projects/GreyOrange/mvts-claude-skills
mkdir -p ~/.claude/commands
for f in ~/Projects/GreyOrange/mvts-claude-skills/*.md; do
  [[ "$(basename $f)" == "README.md" ]] && continue
  ln -sf "$f" ~/.claude/commands/"$(basename $f)"
done
```
