---
name: ship
description: Commit with AI-generated message, push to main, and deploy to Fly.io
allowed-tools: Bash
---

You are a deployment assistant. Your job is to commit, push, and deploy in one smooth flow.

## Task

### Step 1: Review Changes

Run these commands in parallel:
- `git status` to see all changed files
- `git diff --staged` and `git diff` to see all changes
- `git log --oneline -5` to see recent commit message style

### Step 2: Stage Files

Stage all relevant changed files. Do NOT stage files that contain secrets (.env, credentials, etc). Prefer staging specific files by name over `git add -A`.

### Step 3: Write Commit Message

Analyze the staged changes and write a clear, concise commit message that:
- Summarizes the "why" not just the "what"
- Follows the style of recent commits in the repo
- Is 1-2 sentences max
- Ends with `Co-Authored-By: Claude Opus 4.6 <noreply@anthropic.com>`

Use a HEREDOC to pass the message:
```
git commit -m "$(cat <<'EOF'
Your message here

Co-Authored-By: Claude Opus 4.6 <noreply@anthropic.com>
EOF
)"
```

### Step 4: Push

Detect the default branch and push:
```
git push origin $(git symbolic-ref --short HEAD)
```

### Step 5: Deploy

Detect the deployment platform from the project root:
- If `fly.toml` exists → `fly deploy --remote-only` (10-minute timeout)
- If `render.yaml` exists → deployment is automatic on push, skip this step
- If no deploy config found → skip and inform the user

### Step 6: Confirm

Report back with:
- The commit message used
- The deploy status (deployed, auto-deploys on push, or no deploy config)
- Any errors encountered
