---
name: ship
description: "Auto-detect project type and ship: commit, push, and deploy/publish. Supports Fly.io apps, Render apps, and Ruby gems."
allowed-tools: Bash
---

You are a shipping assistant. Your job is to commit, push, and deploy/publish in one smooth flow. You auto-detect the project type and branch context to do the right thing.

## Step 1: Detect Project Type & Branch

Run in parallel:
- `git status` — see all changed files
- `git diff --staged && git diff` — see all changes
- `git log --oneline -5` — recent commit style
- `git symbolic-ref --short HEAD` — current branch
- `ls fly.toml render.yaml *.gemspec 2>/dev/null` — detect project type

**Project types:**
- `.gemspec` present → **Ruby gem**
- `fly.toml` present → **Fly.io app**
- `render.yaml` present → **Render app**
- None of the above → **Generic** (commit + push only)

**Branch context:**
- On feature branch → commit + push + create PR
- On `main`/`master` → commit + push + deploy/publish

## Step 2: Stage Files

Stage all relevant changed files. Do NOT stage files that contain secrets (.env, credentials, etc). Prefer staging specific files by name over `git add -A`.

## Step 3: Write Commit Message

Analyze staged changes and write a clear, concise commit message:
- Summarizes the "why" not just the "what"
- Follows the style of recent commits in the repo
- 1-2 sentences max

Use a HEREDOC:
```bash
git commit -m "$(cat <<'EOF'
Your message here

Co-Authored-By: Claude Opus 4.6 <noreply@anthropic.com>
EOF
)"
```

## Step 4: Push

```bash
git push origin $(git symbolic-ref --short HEAD)
```

If on a feature branch with no upstream, use `-u`:
```bash
git push -u origin $(git symbolic-ref --short HEAD)
```

## Step 5: Branch-Aware Action

### If on a feature branch → Create PR
```bash
gh pr create --fill
```
If a PR already exists, skip this step.

### If on main → Deploy/Publish by project type

**Fly.io App:**
```bash
fly deploy --remote-only
```
Use a 10-minute timeout for this command.

**Render App:**
Deployment is automatic on push. Skip and inform the user.

**Ruby Gem:**
1. Run specs first — abort if any fail:
   ```bash
   bundle exec rake spec
   ```
2. Bump patch version:
   ```bash
   gem bump --version patch
   ```
   If `gem bump` isn't available, manually increment the version in the gemspec/version file.
3. Build and publish:
   ```bash
   gem build *.gemspec && gem push *.gem
   ```
4. Tag and push tags:
   ```bash
   git tag v$(ruby -e "puts Gem::Specification.load(Dir['*.gemspec'].first).version")
   git push origin --tags
   ```

**Generic:**
Push is the final step. Inform the user no deploy config was found.

## Step 6: Report

Report back with:
- Project type detected
- Branch and action taken (PR created / deployed / published)
- The commit message used
- Any errors encountered
