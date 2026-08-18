# patch-upstream

Automate the fork-while-waiting-for-PR workflow: keep a local `development` branch that tracks upstream daily while your PR sits open.

## When to use

- You have a fork with an open PR to upstream
- You want local changes to keep running while waiting for merge
- You need upstream updates applied daily without manual intervention

## What it does

1. **Sets up remotes** — `origin` = upstream, `fork` = your fork
2. **Creates `development` branch** — merges your PR branch + upstream/main
3. **Adds CI workflow** — daily cron syncs upstream → development on GitHub
4. **Sets up local auto-update** — launchd agent reinstalls from git daily
5. **Pushes everything** — development branch, tags, CI workflow

## Usage

```
/patch-upstream <repo-path> <upstream-url> [pr-branch]
```

### Arguments

| Arg | Required | Description |
|-----|----------|-------------|
| `repo-path` | Yes | Local path to the forked repo |
| `upstream-url` | Yes | Upstream repo URL (e.g. `https://github.com/original/repo.git`) |
| `pr-branch` | No | Branch name with your PR changes (auto-detected if not given) |

### Examples

```bash
/patch-upstream ~/Developer/memsearch https://github.com/zilliztech/memsearch.git feat/kimi-platform
/patch-upstream ~/Developer/kimi-code https://github.com/MoonshotAI/kimi-code.git main
```

## What gets created

### `.github/workflows/sync-upstream.yml`
Daily cron (06:17 UTC) that:
- Fetches upstream/main
- Merges into development (skips on conflicts)
- Pushes to fork

### `~/.local/bin/<repo-name>-update`
Shell script that:
- Fetches upstream
- Merges into development
- Reinstalls via `uv tool install --force -e <repo>`

### `~/Library/LaunchAgents/com.<repo-name>.update.plist`
Daily launchd agent (07:23 local) that runs the update script.

## Implementation

When invoked, follow these steps:

### 1. Detect current state

```bash
cd <repo-path>
git remote -v  # check existing remotes
git branch -a  # check existing branches
git log --oneline -5  # check recent commits
```

### 2. Set up remotes

```bash
# origin should be upstream (zilliztech/memsearch)
git remote rename origin upstream 2>/dev/null || true
git remote add origin <upstream-url> 2>/dev/null || true

# fork should be your fork
git remote add fork git@github.com:<your-user>/<repo>.git 2>/dev/null || true
# or update if exists
git remote set-url fork git@github.com:<your-user>/<repo>.git
```

### 3. Create/update development branch

```bash
git fetch origin main
git checkout development 2>/dev/null || git checkout -b development
git merge origin/main --no-edit -m "merge: sync upstream $(date +%Y-%m-%d)"

# If you have a PR branch with your changes, merge it too
if [ -n "$pr-branch" ]; then
  git merge $pr-branch --no-edit -m "merge: $pr-branch changes"
fi
```

### 4. Create CI workflow

Write `.github/workflows/sync-upstream.yml`:

```yaml
name: Sync Upstream
on:
  schedule:
    - cron: "17 6 * * *"
  workflow_dispatch:
permissions:
  contents: write
jobs:
  sync:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v6
        with:
          fetch-depth: 0
          token: ${{ secrets.GITHUB_TOKEN }}
      - run: |
          git config user.name "github-actions[bot]"
          git config user.email "github-actions[bot]@users.noreply.github.com"
          git fetch origin main
          git checkout development || git checkout -b development
          if git merge origin/main --no-edit -m "merge: sync upstream $(date +%Y-%m-%d)"; then
            git push origin development
          else
            git merge --abort
            echo "Merge conflict — skipping"
          fi
```

### 5. Create local auto-update

Write `~/.local/bin/<repo-name>-update`:

```bash
#!/bin/bash
set -euo pipefail
REPO="<repo-path>"
LOG="$HOME/.local/share/<repo-name>-update.log"
echo "[$(date -Iseconds)] Starting update" >> "$LOG"
cd "$REPO"
git fetch origin main 2>> "$LOG"
git checkout development 2>> "$LOG"
if ! git merge origin/main --no-edit -m "auto-sync $(date +%Y-%m-%d)" 2>> "$LOG"; then
  git merge --abort 2>> "$LOG" || true
  exit 0
fi
uv tool install --force -e "$REPO" 2>> "$LOG"
echo "[$(date -Iseconds)] Update complete" >> "$LOG"
```

Write `~/Library/LaunchAgents/com.<repo-name>.update.plist`:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
    <key>Label</key>
    <string>com.<repo-name>.update</string>
    <key>ProgramArguments</key>
    <array>
        <string>/bin/bash</string>
        <string>$HOME/.local/bin/<repo-name>-update</string>
    </array>
    <key>StartCalendarInterval</key>
    <dict>
        <key>Hour</key>
        <integer>7</integer>
        <key>Minute</key>
        <integer>23</integer>
    </dict>
    <key>StandardOutPath</key>
    <string>$HOME/.local/share/<repo-name>-update.log</string>
    <key>StandardErrorPath</key>
    <string>$HOME/.local/share/<repo-name>-update.log</string>
    <key>EnvironmentVariables</key>
    <dict>
        <key>PATH</key>
        <string>$HOME/.local/bin:/usr/local/bin:/usr/bin:/bin</string>
        <key>HOME</key>
        <string>$HOME</string>
    </dict>
</dict>
</plist>
```

### 6. Load and push

```bash
# Make script executable
chmod +x ~/.local/bin/<repo-name>-update

# Load launchd agent
launchctl load ~/Library/LaunchAgents/com.<repo-name>.update.plist 2>/dev/null || true

# Commit and push
git add .github/workflows/sync-upstream.yml
git commit -m "ci: add daily upstream sync workflow"
git push fork development

# If PR branch exists, push to update the PR
if [ -n "$pr-branch" ]; then
  git push fork development:$pr-branch --force
fi
```

## Notes

- The CI workflow uses `GITHUB_TOKEN` which may not have `workflow` scope. If push fails with "workflow scope" error, use SSH (`git remote set-url fork git@github.com:...`)
- The launchd agent runs at 07:23 local time (offset from :00 to avoid herd)
- Merge conflicts are silently skipped (the script exits 0)
- The `uv tool install --force -e <repo>` installs from the local git repo in editable mode
