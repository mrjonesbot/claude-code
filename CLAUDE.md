# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

[MY APPLICATION] is a Ruby on Rails 8.0.1 application for [DESCRIBE BUSINESS USE CASE]. It uses PostgreSQL for data persistence, Hotwire (Turbo & Stimulus) for frontend interactivity, and [CSS FRAMEWORK] for styling.

## Visual Development

### Design Principles
- Comprehensive design checklist in `/context/design-principles.md`
- Brand style guide in `/context/style-guide.md`
- When making visual (front-end, UI/UX) changes, always refer to these files for guidance

### Quick Visual Check
IMMEDIATELY after implementing any front-end change:
1. **Identify what changed** - Review the modified components/pages
2. **Navigate to affected pages** - Use `mcp__playwright__browser_navigate` to visit each changed view
3. **Verify design compliance** - Compare against `/context/design-principles.md` and `/context/style-guide.md`
4. **Validate feature implementation** - Ensure the change fulfills the user's specific request
5. **Check acceptance criteria** - Review any provided context files or requirements
6. **Capture evidence** - Take full page screenshot at desktop viewport (1440px) of each changed view
7. **Check for errors** - Run `mcp__playwright__browser_console_messages`

This verification ensures changes meet design standards and user requirements.

### Comprehensive Design Review
Invoke the `@agent-design-review` subagent for thorough design validation when:
- Completing significant UI/UX features
- Before finalizing PRs with visual changes

## Architecture Overview

### Core Domain Models
- **Product**: Central entity with image processing, categorization, and tracking
- **Account**: Multi-tenant structure for user organizations
- **Plan**: Subscription management with Stripe integration via Pay gem
- **Board**: Product collections/wishlists functionality
- **Location**: Physical or virtual locations with scheduling
- **Blog**: Content management for marketing

### Service Layer
The application uses service objects for complex operations:
- **GPT Service** (`app/services/gpt.rb`): AI-powered features
- **ProcessProductPage** (`app/services/process_product_page.rb`): Web scraping and product import
- **ProcessProductImages** (`app/services/process_product_images.rb`): Image manipulation and storage

### Key Technologies & Gems
- **Authentication**: Passwordless gem for magic link auth
- **Payments**: Pay gem with Stripe integration
- **File Storage**: Active Storage with AWS S3
- **Search**: Ransack for filtering and search
- **Analytics**: Ahoy for tracking, Blazer for business intelligence
- **PDF Generation**: FerrumPDF for document generation
- **Background Jobs**: Solid Queue for async processing
- **Components**: ViewComponent for reusable UI components
- **Error Tracking**: AppSignal for monitoring

### Frontend Architecture
- **Hotwire Stack**: Turbo for SPA-like navigation, Stimulus for JavaScript
- **Tailwind CSS 4.2**: Utility-first CSS framework
- **Import Maps**: Native ES modules without bundling

## Development Standards

### Feature development
- When starting a new feature or scope of work, create a todo list of all tasks

### Models
- Focus on data persistence and validation
- Use service objects for complex business logic
- Avoid complex callbacks

### Controllers
- Stick to RESTful actions
- Delegate authorization to policy objects
- Keep controllers thin
- Use service objects to capture/abstract logic that would otherwise bloat controllers and don't align with just one model

### Service & Business Objects
- Implement `#call` or `#run` as primary interface
- Return Result objects with success/failure states
- Document complex operations with examples

### Views
- Any javascript needed in the view layer (html.erb), should implement a Stimulus controller to house that behavior (app/javascript/controllers)

### Service & Business Objects
● Implement `#call` or `#run` as primary interface
● Return Result objects with success/failure states
● Document complex operations with examples

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

## External Integrations

The application integrates with several external services:
- **Twilio**: SMS notifications
- **Affinda**: Document processing
- **UKG**: HR/recruiting system
- **Adept**: Skills extraction
- **Stripe**: Payment processing
- **Postmark**: Transactional email
- **AWS S3**: File storage

When testing integrations, use the provided WebMock helpers in the test suite rather than direct stubs.

## Deployment

- Platform: Fly.io (configured in `fly.toml`)
- Container: Docker (see `Dockerfile`)
- Production database: PostgreSQL
- Background job processor: Solid Queue
- File storage: AWS S3

