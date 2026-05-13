# MVTS Build & Deploy

> **Engineer-only tool**: This skill is for MVTS engineers. Operations users should use `/mvts/index` for operational issues.

Automate the full MVTS workflow: commit → push → trigger GitHub Actions build → monitor → optionally deploy to a solution branch.

## Constants
- **MVTS repo base**: `/Users/shubham.kumar/Projects/GreyOrange/mvts/`
- **Deployment manifests**: `/Users/shubham.kumar/Projects/GreyOrange/greymatter-deployment-github/platform-k8s/mvts-applications/manifests/values.yaml`
- **GitHub repo**: `greyorange/vrp-obts`
- **Build workflow**: `create-build.yml`
- **Container image prefix**: `us-docker.pkg.dev/greymatter-development/apps/multifleet_planner:`

## Step 1 — Identify working directory

The user may be in a worktree (`f4`, `f5`, `vrp-obts-rec`, etc.). Detect the current branch:

```bash
cd <current_directory_or_ask_user>
git branch --show-current
```

If the user didn't specify which worktree, ask: "Which worktree are you building from? (vrp-obts-rec, f4, f5, etc.)"

## Step 2 — Commit & push

1. Show `git status` so the user can confirm what's being committed.
2. Ask for a commit message if one wasn't provided as an argument to this skill.
3. Stage and commit:
   ```bash
   git add -p   # or ask user to confirm specific files
   git commit -m "<message>"
   git push
   ```
   If there's nothing to commit, skip to Step 3.

## Step 3 — Trigger the Create Build workflow

Trigger with the current branch as `target-branch` and `push_artifacts: true`:

```bash
gh workflow run create-build.yml \
  --repo greyorange/vrp-obts \
  --field target-branch=<current_branch> \
  --field push_artifacts=true
```

Wait ~5 seconds, then fetch the run ID of the just-triggered run:

```bash
gh run list --workflow create-build.yml --repo greyorange/vrp-obts \
  --limit 1 --json databaseId,status,createdAt \
  --jq '.[0]'
```

Report the run ID and URL to the user:
`https://github.com/greyorange/vrp-obts/actions/runs/<run_id>`

## Step 4 — Monitor build in background

Poll every 60 seconds until `status` is `completed`:

```bash
gh run view <run_id> --repo greyorange/vrp-obts --json status,conclusion \
  --jq '"Status: \(.status) | Conclusion: \(.conclusion)"'
```

While monitoring, continue the conversation — do NOT block. When the build completes:
- If `conclusion == "success"`: proceed to extract build version.
- If `conclusion == "failure"`: report failure and link to the run. Stop.

## Step 5 — Extract build version

Download the `workflow-results` artifact and parse `build_vsn`:

```bash
TMPDIR=$(mktemp -d)
gh run download <run_id> --repo greyorange/vrp-obts \
  --name workflow-results --dir "$TMPDIR"
cat "$TMPDIR/workflow-results.json" | python3 -c "import sys,json; d=json.load(sys.stdin); print(d['build_vsn'])"
```

Report: "Build succeeded. Version: `<build_vsn>`"

## Step 6 — Ask about deployment

Ask the user:
> "Build version `<build_vsn>` is ready. Do you want to deploy it? If yes, which solution branch? (e.g. `qa3-adidassanity`, `qa4-walmartpotepic`, `stpbulk-onmbulk`, etc.)"

If the user says no, stop here.

## Step 7 — Update deployment manifest

1. Switch to the deployment repo and checkout the requested branch:
   ```bash
   cd /Users/shubham.kumar/Projects/GreyOrange/greymatter-deployment-github
   git fetch origin
   git checkout <solution_branch>
   git pull origin <solution_branch>
   ```

2. Update `values.yaml` — replace the existing image tag with the new build version:
   - File: `platform-k8s/mvts-applications/manifests/values.yaml`
   - Find the line: `repository: us-docker.pkg.dev/greymatter-development/apps/multifleet_planner:<old_version>`
   - Replace `<old_version>` with `<build_vsn>`

3. Show the diff to the user before committing:
   ```bash
   git diff platform-k8s/mvts-applications/manifests/values.yaml
   ```

4. Confirm with the user, then commit and push:
   ```bash
   git add platform-k8s/mvts-applications/manifests/values.yaml
   git commit -m "chore: bump MVTS to <build_vsn>"
   git push origin <solution_branch>
   ```

5. Report: "Deployed `<build_vsn>` to `<solution_branch>`. Push complete."

## Notes
- Always confirm the diff before pushing the deployment change.
- If the solution branch doesn't exist locally, use `git checkout -b <branch> origin/<branch>`.
- If the user wants to deploy to multiple branches, repeat Step 7 for each.
