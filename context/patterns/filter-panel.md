# Filter Options Panel with Tag Typeahead

A collapsible filter panel pattern for list views that uses tag typeahead components for multi-select filtering, with filter badges that display active filters.

## When to Use

- List views with multiple filter criteria (3+ filters)
- Filters that support multi-select (types, categories, tags, etc.)
- When you want filters hidden by default to reduce visual clutter
- When filter state should persist across page navigation

## Implementation Steps

### 1. Controller Setup

Handle filter parameters and apply to query:

```ruby
class ItemsController < ApplicationController
  before_action :set_filter_params

  def index
    @items = query_items
    respond_to do |format|
      format.html
      format.turbo_stream
    end
  end

  private

  def set_filter_params
    # Always reject blank values from multi-select arrays
    @statuses = (params[:statuses] || []).reject(&:blank?)
    @categories = (params[:categories] || []).reject(&:blank?)
  end

  def query_items
    query = Item.all
    query = query.where(status: @statuses) if @statuses.any?
    query = query.where(category: @categories) if @categories.any?
    query.page(@page).per(@per_page)
  end
end
```

### 2. Main View

```erb
<div class="flex-1 min-h-0 flex flex-col bg-gray-50"
     data-controller="filter-panel">
  <div>
    <%# Search and Controls Row %>
    <div class="mt-6 flex flex-wrap items-center gap-4 px-4 sm:px-6 lg:px-8">
      <%# Preserve filters as hidden fields in sort/search forms %>
      <% @statuses.each do |status| %>
        <%= hidden_field_tag 'statuses[]', status %>
      <% end %>

      <%# Filter Options Button %>
      <button type="button"
              data-action="click->filter-panel#toggle"
              class="inline-flex items-center rounded-md bg-white px-3 py-2 text-sm font-semibold text-gray-900 shadow-sm ring-1 ring-inset ring-gray-300 hover:bg-gray-50">
        Filter Options
      </button>
    </div>

    <%= render 'filter_panel' %>
    <%= render 'filter_badges' %>
  </div>

  <div class="flex-1 min-h-0 overflow-y-auto">
    <%= turbo_frame_tag "items_grid" do %>
      <!-- List content -->
    <% end %>
  </div>
</div>
```

### 3. Filter Panel Partial

```erb
<%# _filter_panel.html.erb %>
<div id="filter-panel" data-filter-panel-target="panel" class="hidden mt-4 mb-4 px-4 sm:px-6 lg:px-8">
  <div class="rounded-lg bg-white shadow-sm border border-gray-200">
    <%= form_with url: items_path,
                  method: :get,
                  data: { turbo_stream: true },
                  class: "p-6" do |f| %>
      <%= f.hidden_field :search, value: params[:search] %>
      <%= f.hidden_field :sort_by, value: params[:sort_by] %>

      <h3 class="text-lg font-semibold text-gray-900 mb-6">Filter Options</h3>

      <div class="grid grid-cols-2 gap-6">
        <div>
          <%= f.label :statuses, "Status",
              class: "block text-sm font-medium text-gray-700 mb-2" %>
          <%= render 'shared/tag_typeahead',
              form: f,
              field_name: :statuses_search,
              placeholder: 'Search statuses...',
              options: Item.status.options.map { |label, value| [label, value] },
              initial_values: @statuses,
              multi_select: true,
              auto_submit: false %>
        </div>
      </div>

      <div class="mt-6 flex flex-col gap-3">
        <%= f.submit "Apply Filters",
            data: { action: "click->filter-panel#hide" },
            class: "w-full inline-flex justify-center items-center rounded-md bg-primary px-4 py-2.5 text-sm font-semibold text-white shadow-sm hover:bg-primary" %>

        <%= button_tag "Cancel",
            type: "button",
            data: { action: "click->filter-panel#hide" },
            class: "w-full inline-flex justify-center items-center rounded-md bg-white px-4 py-2.5 text-sm font-semibold text-gray-900 shadow-sm ring-1 ring-inset ring-gray-300 hover:bg-gray-50" %>
      </div>
    <% end %>
  </div>
</div>
```

### 4. Filter Badges Partial

```erb
<%# _filter_badges.html.erb %>
<div id="filter-badges">
  <% if @statuses.any? || @categories.any? %>
    <div class="mt-4 mb-4 px-4 sm:px-6 lg:px-8">
      <div class="flex flex-wrap items-center gap-2">
        <% @statuses.each do |status| %>
          <% display_name = Item.status.find_value(status)&.text %>
          <%= link_to items_path(
                search: params[:search],
                sort_by: params[:sort_by],
                statuses: @statuses - [status],
                categories: @categories
              ),
              data: { turbo_stream: true },
              class: "inline-flex items-center gap-x-1.5 rounded-full bg-indigo-50 px-3 py-1 text-xs font-medium text-indigo-700 ring-1 ring-inset ring-indigo-700/10 hover:bg-primary/10" do %>
            <span><%= display_name %></span>
            <svg class="h-3 w-3" viewBox="0 0 14 14" fill="none" stroke="currentColor" stroke-width="2">
              <path d="M4 4l6 6m0-6l-6 6" />
            </svg>
          <% end %>
        <% end %>

        <%= link_to "Clear all",
            items_path,
            data: { turbo_stream: true },
            class: "text-xs text-primary hover:text-primary font-medium" %>
      </div>
    </div>
  <% end %>
</div>
```

### 5. Turbo Stream Response

```erb
<%# index.turbo_stream.erb %>
<%= turbo_stream.replace "filter-badges" do %>
  <%= render 'filter_badges' %>
<% end %>

<%= turbo_stream.replace "items_grid" do %>
  <%= turbo_frame_tag "items_grid" do %>
    <!-- Updated content -->
  <% end %>
<% end %>
```

## Common Pitfalls

**Don't:** Forget to reject blank values from filter arrays
```ruby
# Bad - includes empty strings from form
@statuses = params[:statuses] || []
```

**Do:** Always reject blank values
```ruby
# Good - clean array of actual values
@statuses = (params[:statuses] || []).reject(&:blank?)
```

**Don't:** Use `auto_submit: true` on filter panel tag typeaheads
```erb
<!-- Bad - will submit on every selection -->
<%= render 'shared/tag_typeahead', auto_submit: true, ... %>
```

**Do:** Set `auto_submit: false` and use "Apply Filters" button
```erb
<!-- Good - user controls when to apply -->
<%= render 'shared/tag_typeahead', auto_submit: false, ... %>
<%= f.submit "Apply Filters" %>
```

**Don't:** Forget to preserve filters in other forms (sort, search)
```erb
<!-- Bad - filters lost when sorting -->
<%= form_with url: items_path do |f| %>
  <%= f.select :sort_by, ... %>
<% end %>
```

**Do:** Include filter params as hidden fields
```erb
<!-- Good - filters preserved -->
<%= form_with url: items_path do |f| %>
  <% @statuses.each do |status| %>
    <%= hidden_field_tag 'statuses[]', status %>
  <% end %>
  <%= f.select :sort_by, ... %>
<% end %>
```

**Don't:** Forget to update filter badges in turbo stream
```erb
<!-- Bad - badges don't update -->
<%= turbo_stream.replace "items_grid" do %>
  <!-- only updates grid -->
<% end %>
```

**Do:** Update both badges and grid
```erb
<!-- Good - complete UI update -->
<%= turbo_stream.replace "filter-badges" do %>
  <%= render 'filter_badges' %>
<% end %>
<%= turbo_stream.replace "items_grid" do %>
  <!-- grid content -->
<% end %>
```

**Don't:** Forget `data-action="click->filter-panel#hide"` on submit button
```erb
<!-- Bad - panel stays open after applying -->
<%= f.submit "Apply Filters" %>
```

**Do:** Close panel on submit
```erb
<!-- Good - panel closes automatically -->
<%= f.submit "Apply Filters",
    data: { action: "click->filter-panel#hide" } %>
```
