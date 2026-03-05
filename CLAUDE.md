# Global CLAUDE.md

Shared instructions for all Rails projects. Project-specific details live in each repo's `CLAUDE.md`.

## Core Principles

- **Simplicity First**: Make every change as simple as possible. Impact minimal code.
- **No Laziness**: Find root causes. No temporary fixes. Senior developer standards.
- **Minimal Impact**: Changes should only touch what's necessary. Avoid introducing bugs.
- **Performance Awareness**: Proactively consider whether new features or changes could degrade performance — memory spikes, N+1 queries, unbounded result sets, or large in-memory collections. Refer to `~/.claude/context/patterns/memory-safe-exports.md` for established patterns.

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

### 3. Self-Improvement Loop
- After ANY correction from the user: update `tasks/lessons.md` with the pattern
- Write rules for yourself that prevent the same mistake
- Ruthlessly iterate on these lessons until mistake rate drops
- Review lessons at session start for relevant project

### 4. Verification Before Done
- Never mark a task complete without proving it works
- Diff behavior between main and your changes when relevant
- Ask yourself: "Would a staff engineer approve this?"
- Run tests, check logs, demonstrate correctness

### 5. Demand Elegance (Balanced)
- For non-trivial changes: pause and ask "is there a more elegant way?"
- If a fix feels hacky: "Knowing everything I know now, implement the elegant solution"
- Skip this for simple, obvious fixes — don't over-engineer
- Challenge your own work before presenting it

### 6. Autonomous Bug Fixing
- When given a bug report: just fix it. Don't ask for hand-holding
- Point at logs, errors, failing tests — then resolve them
- Zero context switching required from the user
- Go fix failing CI tests without being told how

## Task Management

1. **Plan First**: Write plan to `tasks/todo.md` with checkable items
2. **Verify Plan**: Check in before starting implementation
3. **Track Progress**: Mark items complete as you go
4. **Explain Changes**: High-level summary at each step
5. **Document Results**: Add review section to `tasks/todo.md`
6. **Capture Lessons**: Update `tasks/lessons.md` after corrections

## Visual Development

When making visual (front-end, UI/UX) changes, refer to these context files:
- `~/.claude/context/design-principles.md` — design checklist, layout patterns, component specs
- `/context/style-guide.md` — brand color tokens, button classes, form controls (source of truth for colors; **project-specific**)
- `~/.claude/context/patterns.md` — reusable MVC patterns (tabs, typeahead, filters, enumerations)
- `~/.claude/context/workflow.md` — browser tools (cmux in cmux terminal, Playwright MCP in tmux), visual check steps, design review process

Always follow established patterns for consistency across the application.

## Background Job & Data Processing

When building or modifying CSV exports, data imports, or any background job that handles medium-to-large datasets, refer to:
- `~/.claude/context/patterns/memory-safe-exports.md` — disk-backed tempfile patterns, Export model with S3 upload, streaming HTTP responses, query caching, SQL-side filtering

All export jobs MUST follow the established Export model pattern (stream to tempfile → attach to S3 → email download link). Never use `CSV.generate` or email file attachments for exports.

### Performance & Memory Guidelines

Before implementing features that touch datasets, collections, or batch operations, consider performance impact. Refer to `~/.claude/context/patterns/memory-safe-exports.md` for detailed patterns. Key principles:

- Batch large queries (`find_each`/`in_batches`), don't load unbounded result sets
- Stream large payloads to disk rather than building in memory
- Push filtering, counts, and aggregations to SQL
- Eager-load associations to avoid N+1s, but scope the base query first
- Paginate or limit results in controller actions (dashboards, reports, search)

## Development Standards

### Models
- Focus on data persistence and validation
- Use service objects for complex business logic
- Avoid complex callbacks
- **CRITICAL**: Before writing database queries with ActiveRecord, ALWAYS check `db/schema.rb` to verify the exact column names available on the table. Do not assume column names based on model methods (e.g., `full_name` might be a method, not a column)

### Controllers
- Stick to RESTful actions
- Delegate authorization to policy objects
- Keep controllers thin
- Use service objects to capture/abstract logic that would otherwise bloat controllers and don't align with just one model
- **CRITICAL**: When adding a new controller action, you MUST also add the corresponding authorization method to the policy file (e.g., `new_participant` action requires `new_participant?` method in the policy)

### Service & Business Objects
- Implement `#call` or `#run` as primary interface
- Return Result objects with success/failure states
- Document complex operations with examples

### Views
- Any javascript needed in the view layer (html.erb), should implement a Stimulus controller to house that behavior (app/javascript/controllers)

### Testing Philosophy
- Use real objects by default in tests
- WebMock for external HTTP calls
- Follow RSpec conventions for different test types (request, system, policy specs)
- Shared examples for common behaviors

### Request Specs
- Test HTTP-level behavior only
- Always test authenticated and unauthenticated states
- Use webmock helpers for API stubbing

### System Specs
- Use `system_spec_helper` by default
- Include `js: true` for JavaScript interactions
- Use `_url` helpers instead of `_path`

## Common Development Commands

### Server & Development
- Start development server: `bin/rails server` or `bin/dev` (includes Tailwind CSS watch)
- Rails console: `bin/rails console`
- Database console: `bin/rails db`

### Testing
- Run all tests: `bin/rails parallel:spec`
- bin/rails "parallel:test[^test/unit]" # every test file in test/unit folder
- bin/rails "parallel:test[user]"  # run users_controller + user_helper + user tests
- bin/rails "parallel:test['user|product']"  # run user and product related tests
- bin/rails "parallel:spec['spec\/(?!features)']" # run RSpec tests except the tests in spec/features

### Code Quality
- Run RuboCop linter: `bundle exec rubocop`
- Auto-fix RuboCop issues: `bundle exec rubocop -a`

### Asset Management
- Precompile assets: `bin/rails assets:precompile`
- Clean old assets: `bin/rails assets:clean`
- Watch Tailwind CSS changes: `bin/rails tailwindcss:watch`

### Background Jobs
- Start Solid Queue worker: `bin/rails solid_queue:start`
- Access job dashboard: Visit `/jobs` in browser (Mission Control)

## Deployment

- Platform: Fly.io (configured in `fly.toml`)
- Container: Docker (see `Dockerfile`)
- Production database: PostgreSQL
- Background job processor: Solid Queue
- File storage: AWS S3
