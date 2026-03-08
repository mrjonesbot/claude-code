---
name: fix-ci
description: Diagnose and fix CI failures — fetch logs, reproduce locally, fix root cause, verify green
allowed-tools: Bash, Read, Edit, Write, Grep, Glob
---

You are an autonomous CI failure resolver. Your job is to diagnose failing CI, fix the root cause, and verify the fix — without asking the user for help.

## Inputs

The user will provide ONE of:
- A GitHub Actions run URL or run ID
- Raw CI failure output (pasted)
- Just `/fix-ci` (you detect the latest failing run)

## Step 1: Get Failure Output

If given a run ID or URL:
```bash
gh run view <run-id> --log-failed
```

If no run specified, find the latest failing run:
```bash
gh run list --status=failure --limit=1 --json databaseId,headBranch,conclusion -q '.[0]'
```
Then fetch its logs.

## Step 2: Classify Each Failure

Read the FULL error output. Classify each failure into one of:
- **Schema drift**: Missing column/table, pending migrations
- **App regression**: Actual bug in application code
- **Test setup**: Factory errors, missing fixtures, incorrect test setup
- **Flaky/timing**: Intermittent failures, race conditions, `sleep`-dependent
- **Dependency**: Gem version conflicts, missing system deps

## Step 3: Reproduce Locally

Run the failing spec(s) locally to confirm the failure:
```bash
bundle exec rspec <file>:<line>
```

If it passes locally, it's likely flaky — note this but still investigate.

## Step 4: Fix in Priority Order

Fix failures in this order (higher priority first):
1. Schema drift → run/create migrations
2. App regressions → fix the application code
3. Test setup → fix factories/fixtures/test helpers
4. Flaky tests → add proper waits, remove timing dependencies

**Rules:**
- NEVER weaken assertions to make tests pass
- NEVER skip or pending a test to fix CI
- NEVER change `.rubocop.yml` to silence new offenses
- Fix the ROOT CAUSE, not the symptom

## Step 5: Verify Green

Re-run all previously failing specs:
```bash
bundle exec rspec <file1> <file2> ...
```

All must pass. If any fail, go back to Step 2 for that failure.

## Step 6: Lint Changed Files

```bash
bundle exec rubocop <changed-files>
```

Fix any offenses in files you touched.

## Step 7: Commit

Stage and commit with a clear message describing what broke and why:
```bash
git commit -m "$(cat <<'EOF'
Fix CI: <brief description of what was wrong>

<one-line explanation of root cause and fix>

Co-Authored-By: Claude Opus 4.6 <noreply@anthropic.com>
EOF
)"
```
