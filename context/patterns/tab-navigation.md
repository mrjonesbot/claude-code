# Tab Navigation with Turbo Frames

A pattern for creating tabbed interfaces where each tab has its own URL and lazy-loads content via Turbo Frames.

## When to Use
- Multi-section views that benefit from URL-based navigation
- Heavy data sections that should load independently
- Interfaces where users need to bookmark/share specific tabs

## Structure

```
app/
├── controllers/
│   └── example_controller.rb
├── views/
│   └── examples/
│       ├── _header.html.erb          # Shared header/tabs/filters
│       ├── tab_one.html.erb          # Tab view
│       ├── tab_two.html.erb          # Tab view
│       ├── tab_one_data.html.erb     # Turbo frame content
│       └── tab_two_data.html.erb     # Turbo frame content
└── config/
    └── routes.rb
```

## Implementation Steps

### 1. Routes

Define routes for each tab and their turbo frame endpoints:

```ruby
scope path: 'examples', as: 'examples', controller: 'examples' do
  get '/', action: :index, as: ''
  get '/tab_one', action: :tab_one
  get '/tab_two', action: :tab_two

  # Turbo frame endpoints for lazy loading
  get '/tab_one_data', action: :tab_one_data
  get '/tab_two_data', action: :tab_two_data
end
```

**Route naming conventions:**
- Tab views: `/examples/tab_one`, `/examples/tab_two`, etc.
- Turbo frame endpoints: `/examples/tab_one_data`, `/examples/tab_two_data`, etc.
- Index redirects to default tab

### 2. Controller

Keep controller actions minimal - they just authorize and render:

```ruby
class ExamplesController < ApplicationController
  before_action :set_common_data

  # Main tab views
  def index
    authorize :example, :show?
    redirect_to examples_tab_one_path
  end

  def tab_one
    authorize :example, :show?
  end

  def tab_two
    authorize :example, :show?
  end

  # Turbo frame endpoints
  def tab_one_data
    authorize :example, :show?
  end

  def tab_two_data
    authorize :example, :show?
  end

  private

  def set_common_data
    # Shared data for all tabs
  end
end
```

### 3. Shared Header Partial

Create `_header.html.erb` with tabs, filters, and content yield:

```erb
<% current_tab ||= local_assigns[:current_tab] || 'tab_one' %>

<div class="flex flex-col h-full min-h-0 bg-gray-50">
  <%# Header %>
  <div class="bg-white border-b border-gray-200 px-4 sm:px-6 lg:px-8 py-6">
    <div>
      <h1 class="text-2xl font-semibold text-gray-900">Page Title</h1>
      <p class="mt-1 text-sm text-gray-500">Description</p>
    </div>
  </div>

  <%# Tab Navigation %>
  <div class="border-b border-gray-200 bg-white">
    <nav class="flex space-x-4 px-4 sm:px-6 lg:px-8" aria-label="Tabs">
      <%= link_to examples_tab_one_path,
                  data: { turbo_action: "advance" },
                  class: "whitespace-nowrap border-b-2 py-4 px-1 text-sm font-medium #{current_tab == 'tab_one' ? 'border-indigo-500 text-indigo-600' : 'border-transparent text-gray-500 hover:border-gray-300 hover:text-gray-700'}" do %>
        Tab One
      <% end %>
      <%= link_to examples_tab_two_path,
                  data: { turbo_action: "advance" },
                  class: "..." do %>
        Tab Two
      <% end %>
    </nav>
  </div>

  <%# Tab Content %>
  <div class="flex-1 min-h-0 overflow-y-auto px-4 sm:px-6 lg:px-8 py-6">
    <%= yield %>
  </div>
</div>
```

**Key elements:**
- `current_tab` local variable to track active tab
- `data: { turbo_action: "advance" }` on all navigation links to update URL
- `yield` for tab-specific content

### 4. Tab Views

Each tab view uses the header layout and provides content:

```erb
<%# app/views/examples/tab_one.html.erb %>
<%= render layout: 'header', locals: { current_tab: 'tab_one' } do %>
  <div class="mb-8">
    <h2 class="text-xl font-semibold text-gray-900 mb-4">Section Title</h2>

    <%= turbo_frame_tag "tab_one_data",
        src: examples_tab_one_data_path do %>
      <%= render 'shared/grid_spinner' %>
    <% end %>
  </div>
<% end %>
```

**Pattern:**
- Render `'header'` layout with `current_tab` local
- Use `turbo_frame_tag` with `src:` for lazy loading
- Show spinner while loading
- Pass filter params to turbo frame endpoint

### 5. Turbo Frame Content

Frame endpoints render just the frame content:

```erb
<%# app/views/examples/tab_one_data.html.erb %>
<%= turbo_frame_tag "tab_one_data" do %>
  <!-- Actual content -->
<% end %>
```

**Important:**
- Wrap content in matching `turbo_frame_tag`
- Frame ID must match the `src:` request
- No layout needed (rendered inside existing frame)

## Common Pitfalls

**Don't:** Use `params[:tab]` to conditionally render content in one view
```erb
<!-- Bad - one view with conditionals -->
<% if params[:tab] == 'tab_one' %>
  <%= render 'tab_one_content' %>
<% end %>
```

**Do:** Create separate views for each tab
```ruby
# Good - separate routes and views
get '/tab_one', action: :tab_one
get '/tab_two', action: :tab_two
```

**Don't:** Forget `turbo_action: "advance"` on links
```erb
<!-- Bad - URL won't update -->
<%= link_to "Tab", path %>
```

**Do:** Always use `turbo_action: "advance"` for tab links
```erb
<!-- Good - URL updates in browser -->
<%= link_to "Tab", path, data: { turbo_action: "advance" } %>
```

**Don't:** Load all tab content on page load
```erb
<!-- Bad - slow initial load -->
<%= turbo_frame_tag "tab_one_data" do %>
  <%= render 'tab_one_content' %>
<% end %>
```

**Do:** Use `src:` to lazy-load frame content
```erb
<!-- Good - fast initial load -->
<%= turbo_frame_tag "tab_one_data",
    src: tab_one_data_path do %>
  <%= render 'shared/grid_spinner' %>
<% end %>
```
