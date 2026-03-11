# Global CLAUDE.md

Shared instructions for all projects. Project-specific details live in each repo's `CLAUDE.md`.

## Core Principles

- **Simplicity First**: Make every change as simple as possible. Impact minimal code.
- **No Laziness**: Find root causes. No temporary fixes. Senior developer standards.
- **Minimal Impact**: Changes should only touch what's necessary. Avoid introducing bugs.
- **Performance Awareness**: Proactively consider performance impact — memory spikes, N+1 queries, unbounded result sets. Refer to `~/.claude/context/patterns/memory-safe-exports.md`.

## Workflow Orchestration

### 1. Plan Mode Default
- Enter plan mode for ANY non-trivial task (3+ steps or architectural decisions)
- If something goes sideways, STOP and re-plan immediately — don't keep pushing
- Use plan mode for verification steps, not just building
- Write detailed specs upfront to reduce ambiguity

### 2. Subagent Strategy
- Use subagents liberally to keep main context window clean
- Offload research, exploration, and parallel analysis to subagents
- For complex problems, throw more compute at it via subagents
- One task per subagent for focused execution

### 3. Verification Before Done
- Never mark a task complete without proving it works
- Diff behavior between main and your changes when relevant
- For non-trivial changes: pause and ask "is there a more elegant way?"
- Run tests, check logs, demonstrate correctness

### 4. Autonomous Bug Fixing
- When given a bug report: just fix it. Don't ask for hand-holding
- Point at logs, errors, failing tests — then resolve them
- Zero context switching required from the user

### 5. Auto-Branch, Commit & PR on Completion
- When work is **complete and verified**, immediately:
  1. Stage and commit all changes with a clear commit message
  2. Push the branch
  3. **Only if on `main`**: create a descriptive branch (`fix/…`, `feat/…`, `chore/…`), switch to it, and open a PR via `gh pr create`
  4. **If on a feature branch**: assume we're adding to existing work — just commit and push. Do NOT create a new branch or PR.
- Follow all commit/PR formatting rules (HEREDOC messages, Co-Authored-By, summary + test plan)

### 6. Visual Iteration Protocol
- When given a screenshot with feedback: make changes, then take a new screenshot to verify
- After 3+ rounds on the same element, stop — re-read the full context, consider the element holistically, and make one comprehensive fix
- Always reference `~/.claude/context/design-principles.md` and `/context/style-guide.md` for visual work

### 7. Plan Execution Protocol
- When given "Implement the following plan:": treat each deliverable as a discrete item
- Verify each item works before moving to the next
- No scope creep — implement exactly what the plan says, nothing more
- If a plan item is ambiguous, re-read the plan context before guessing

### 8. Error Triage Protocol
- Read the FULL error message/stack trace before acting
- Detect whether this is the same error recurring or a new one
- After 2 failed fix attempts on the same error: stop, re-diagnose from scratch, consider a different approach entirely
- Never weaken assertions or silence errors to make tests pass

### 9. Context Restore Protocol
- After `/clear` or context loss: autonomously reconstruct context from `git log --oneline -10`, `git diff`, `git branch`, and `tasks/todo.md`
- Never ask "where were we?" — figure it out from project state

### 10. Commit Cadence
- Commit after each logically complete change, not after the whole feature
- Each commit should be independently meaningful and pass tests
- This creates better git history and safer rollback points

## Task Management

- Write plan to `tasks/todo.md` with checkable items. Mark items complete as you go.
- After corrections from the user, update `tasks/lessons.md` with what went wrong and the pattern to follow.

## Visual Development

When making visual changes, refer to:
- `~/.claude/context/design-principles.md` — design checklist, layout patterns, component specs
- `/context/style-guide.md` — brand color tokens, button classes, form controls (**project-specific**)
- `~/.claude/context/patterns.md` — reusable MVC patterns (tabs, typeahead, filters, enumerations)
- `~/.claude/context/workflow.md` — browser tools, visual check steps, design review process

## Background Job & Data Processing

Refer to `~/.claude/context/patterns/memory-safe-exports.md` for all data processing patterns.

All export jobs MUST follow the Export model pattern (stream to tempfile → attach to S3 → email download link). Never use `CSV.generate` or email file attachments for exports.

## Development Standards

- **CRITICAL**: Before writing ActiveRecord queries, ALWAYS check `db/schema.rb` to verify exact column names. Do not assume column names from model methods.
- **CRITICAL**: When adding a new controller action, you MUST add the corresponding authorization method to the policy file (e.g., `new_participant` action requires `new_participant?` in the policy).
- Views: Any javascript in `.html.erb` should use a Stimulus controller (`app/javascript/controllers`).

## Internationalization (i18n)

**CRITICAL**: When making ANY copy changes to locale files:

1. **Check for other supported locales**: If the project has multiple locale files (e.g., `en.yml`, `es.yml`, `ar.yml`, `pl.yml`, `tl.yml`, `zh.yml`), ALL locales must be updated
2. **Update existing translations**: If modifying copy in `en.yml`, update the corresponding keys in all other locale files
3. **Add new translations**: If adding new copy/keys to `en.yml`, add translated versions to all other locale files
4. **Never leave locales incomplete**: Do not commit changes that only update `en.yml` — all supported locales must have translations

**Workflow for locale changes:**
- Make change to `en.yml` first
- Identify all other locale files in `config/locales/`
- Use AI translation to generate accurate translations for each locale
- Verify translations maintain context and tone
- Update all locale files in the same commit

**Exception**: Project-specific locale files (e.g., `nestingbird/config/locales/`) may have different supported languages than global context. Check the project's actual locale files, not assumptions.

## Testing Commands

```
bin/rails parallel:spec                              # all specs
bin/rails "parallel:spec['spec\/(?!features)']"      # specs except features
bin/rails "parallel:test[^test/unit]"                 # test files in test/unit
bin/rails "parallel:test[user]"                       # user-related tests
bin/rails "parallel:test['user|product']"             # user + product tests
```
