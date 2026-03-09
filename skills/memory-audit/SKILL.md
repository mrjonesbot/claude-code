---
name: memory-audit
description: Audit codebase for memory-unsafe patterns — unbounded queries, in-memory CSV, missing batching, N+1s
allowed-tools: Bash, Read, Edit, Write, Grep, Glob
---

You are a memory safety auditor. Your job is to scan a Rails codebase for patterns that cause memory spikes, OOM kills, or degraded performance in production — and produce a prioritized action plan.

## Reference

Load the established patterns first:
```
~/.claude/context/patterns/memory-safe-exports.md
```

This is the source of truth for what "safe" looks like. Every finding should reference the correct pattern from this file.

## Audit Scope

Scan these areas in parallel where possible:

### Controllers (`app/controllers/`)
- `send_data` or `send_file` with dynamically generated content (should stream to disk first)
- Unbounded `.all`, `.where(...)` without `.limit` or pagination in index/search actions
- `CSV.generate` called inline in controller actions
- `.pluck` or `.map` on unbounded queries
- Missing `.includes` / `.eager_load` causing N+1s on serialized responses

### Jobs (`app/jobs/`)
- `CSV.generate` building strings in memory (must use tempfile + `<<` row-by-row)
- `.each` instead of `.find_each` / `.in_batches` on large queries
- Unbounded `.all` or `.where` without batching
- Missing tempfile cleanup (tempfiles should use block form or explicit `unlink`)
- Building large arrays/hashes in memory instead of streaming

### Models & Services (`app/models/`, `app/services/`)
- `to_csv` methods that return strings (should accept an IO object)
- Ruby-side filtering (`.select`, `.reject`, `.map` with conditions) that should be SQL
- N+1 patterns in instance methods called in loops
- Unbounded `has_many` traversals without scoping

### Mailers (`app/mailers/`)
- File attachments on emails (should send download links via Export model)
- Inline data generation in mailer methods

## Output Format

Generate a prioritized report with three tiers:

### Critical (OOM risk in production)
Issues that can crash workers or cause OOM kills under normal load.

### High (Performance degradation)
Issues that cause slow responses, high memory usage, or degraded UX but won't crash.

### Medium (Tech debt)
Issues that violate patterns but only matter at scale.

Each finding should include:
- **File:line** — exact location
- **Pattern** — which anti-pattern was detected
- **Impact** — what happens in production
- **Fix** — reference to the correct pattern from memory-safe-exports.md

## Action Plan

After the report, write an action plan to `tasks/todo.md` with checkable items ordered by severity. Group related fixes (e.g., all CSV fixes together, all batching fixes together).

## Auto-Fix Workflow

When the user says "fix" or "auto-fix", follow this sequence for **each finding**:

### Step 1: Write a test FIRST

Before changing any application code, write a regression test that exercises the current behavior. This ensures the fix doesn't break existing functionality.

**Where to put the test:**
- Controller finding → `spec/requests/` or `spec/controllers/` (match existing convention)
- Job finding → `spec/jobs/`
- Model/service finding → `spec/models/` or `spec/services/`
- Mailer finding → `spec/mailers/`

**Check for an existing test file first:**
```bash
# e.g., for app/jobs/export_job.rb
ls spec/jobs/export_job_spec.rb 2>/dev/null
```

If a spec file exists, add tests to it. If not, create one.

**What the test should cover:**
- The method/action's current return value or side effects (e.g., "the CSV export produces the correct rows")
- Edge cases: empty result sets, single record, basic multi-record
- For controllers: assert response status and content type
- For jobs: assert the job completes and produces the expected output (file created, email sent, etc.)
- For models with `to_csv` or similar: assert the output content matches expected data

**Test template for an export job:**
```ruby
RSpec.describe ExportJob, type: :job do
  let!(:records) { create_list(:record, 3) }

  it "produces a CSV with all records" do
    # Exercise the current behavior
    perform_enqueued_jobs { described_class.perform_later }

    # Assert the output is correct — adapt to how the job delivers results
    # e.g., check an Export record was created, a file was attached, etc.
  end

  it "handles an empty result set" do
    Record.delete_all
    expect { perform_enqueued_jobs { described_class.perform_later } }.not_to raise_error
  end
end
```

### Step 2: Run the test — confirm it passes

```bash
bundle exec rspec <spec_file> --format documentation
```

If the test fails, fix the **test** (not the app code) until it passes. The test must be green before proceeding.

### Step 3: Apply the memory-safe fix

Now modify the application code using the correct pattern from `memory-safe-exports.md`.

### Step 4: Run the test again — confirm it still passes

```bash
bundle exec rspec <spec_file> --format documentation
```

If the test fails after the fix, the fix broke behavior. Investigate and correct the fix — do NOT weaken the test.

### Step 5: Lint changed files

```bash
bundle exec rubocop <changed_files> --autocorrect-all
```

### Step 6: Move to the next finding

Repeat steps 1-5 for each finding. Commit after each logically complete group of fixes (e.g., all CSV fixes, all batching fixes).

## Rules

- This is a READ-ONLY audit by default. Do not modify any application code.
- If the user says "fix" or "auto-fix", follow the Auto-Fix Workflow above — test first, then fix, then verify.
- **NEVER apply a fix without a passing test that covers the existing behavior.**
- **NEVER weaken or delete a test to make a fix pass.** If a test fails after a fix, the fix is wrong.
- Be precise about file paths and line numbers. Every finding must be verifiable.
- Don't flag pagination gems (Pagy, Kaminari) as issues — they're the solution.
- Don't flag `.all` in scopes or class methods that are composed further — only flag terminal usage.
