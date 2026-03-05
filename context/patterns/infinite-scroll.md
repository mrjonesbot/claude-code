# Infinite Scroll Pagination

A pattern for automatically loading more records as the user scrolls, using Turbo's native lazy-loading frames and Turbo Streams. No custom Stimulus controller is needed — Turbo handles the IntersectionObserver behavior natively via `loading: :lazy`.

## When to Use

- List views with many records where traditional page-number pagination feels clunky
- Tables or card grids where seamless scrolling improves UX
- Replace `pagy_nav` click-based pagination with auto-loading

## How It Works

1. After the initial list of records, render a `turbo_frame_tag` with a unique ID (`pagination-{next_page}`), a `src:` pointing to the same action with `format: :turbo_stream`, and `loading: :lazy`
2. When the user scrolls to the frame, Turbo automatically fetches the URL
3. The turbo_stream response **appends** new rows to the list container and **replaces** the pagination frame with the next one (or removes it on the last page)
4. This chains indefinitely — each new pagination frame triggers the next page when scrolled into view

## Structure

```
app/
├── controllers/
│   └── items_controller.rb              # Sets @pagy, @items, @next_page_params
├── views/
│   └── items/
│       ├── _items_table.html.erb        # Table with tbody ID + pagination frame
│       └── index.turbo_stream.erb       # Appends rows + replaces pagination frame
```

## Implementation Steps

### 1. Controller

Set up pagination and build next-page params. The key is including `format: :turbo_stream` in the params so the lazy-loaded frame triggers the turbo_stream response:

```ruby
class ItemsController < ApplicationController
  def index
    @items = current_account.items
                            .reorder(sort_column(Item, default: 'created_at') => sort_direction(default: 'desc'))

    @page = params[:page]&.to_i || 1
    @pagy, @items = pagy(@items, page: @page)

    # Build next page URL params for infinite scrolling
    if @pagy.next
      @next_page_params = {
        page: @pagy.next,
        sort: params[:sort],
        direction: params[:direction],
        search: params[:search],
        format: :turbo_stream          # Critical: triggers turbo_stream response
      }.compact
    end

    respond_to do |format|
      format.html
      format.turbo_stream
    end
  end
end
```

**Key points:**
- `@pagy.next` returns the next page number, or `nil` if on the last page
- `format: :turbo_stream` ensures the lazy frame request gets a turbo_stream response
- Include all current filter/sort/search params so they carry forward
- `respond_to` must handle both `html` (initial load) and `turbo_stream` (subsequent pages)

### 2. List Partial (HTML)

Render the table with an identifiable `<tbody>` and the pagination frame after it:

```erb
<%# app/views/items/_items_table.html.erb %>
<%= turbo_frame_tag 'items' do %>
  <table class="min-w-full border-separate border-spacing-0">
    <thead>
      <tr>
        <th class="...">Name</th>
        <th class="...">Date</th>
        <%# ... other columns %>
      </tr>
    </thead>
    <tbody id="items-tbody">
      <% @items.each do |item| %>
        <%= render 'item_row', item: item %>
      <% end %>

      <% if @items.empty? %>
        <tr>
          <td colspan="4" class="px-3 py-12 text-center text-sm text-gray-500">
            No items found
          </td>
        </tr>
      <% end %>
    </tbody>
  </table>

  <%# Infinite Scroll Pagination %>
  <% if @pagy&.next %>
    <%= turbo_frame_tag "pagination-#{@pagy.next}",
        src: items_path(@next_page_params),
        loading: :lazy do %>
      <div class="flex items-center justify-center py-4">
        <svg class="animate-spin h-5 w-5 text-primary" xmlns="http://www.w3.org/2000/svg" fill="none" viewBox="0 0 24 24">
          <circle class="opacity-25" cx="12" cy="12" r="10" stroke="currentColor" stroke-width="4"></circle>
          <path class="opacity-75" fill="currentColor" d="M4 12a8 8 0 018-8V0C5.373 0 0 5.373 0 12h4zm2 5.291A7.962 7.962 0 014 12H0c0 3.042 1.135 5.824 3 7.938l3-2.647z"></path>
        </svg>
      </div>
    <% end %>
  <% end %>
<% end %>
```

**Key elements:**
- `<tbody id="items-tbody">` — the append target for new rows
- `turbo_frame_tag "pagination-#{@pagy.next}"` — unique ID per page prevents conflicts
- `src: items_path(@next_page_params)` — URL includes `format: :turbo_stream`
- `loading: :lazy` — Turbo only fetches when the frame scrolls into view
- Spinner content inside the frame shows while loading

### 3. Turbo Stream Response

The turbo_stream template handles two concerns: appending new rows and chaining to the next page:

```erb
<%# app/views/items/index.turbo_stream.erb %>
<% if @page > 1 %>
  <%# Append new rows to the table body %>
  <%= turbo_stream.append 'items-tbody' do %>
    <% @items.each do |item| %>
      <%= render 'item_row', item: item %>
    <% end %>
  <% end %>

  <%# Replace current pagination frame with the next one (or nothing if last page) %>
  <%= turbo_stream.replace "pagination-#{@page}" do %>
    <% if @pagy.next %>
      <%= turbo_frame_tag "pagination-#{@pagy.next}",
          src: items_path(@next_page_params),
          loading: :lazy do %>
        <div class="flex items-center justify-center py-4">
          <svg class="animate-spin h-5 w-5 text-primary" xmlns="http://www.w3.org/2000/svg" fill="none" viewBox="0 0 24 24">
            <circle class="opacity-25" cx="12" cy="12" r="10" stroke="currentColor" stroke-width="4"></circle>
            <path class="opacity-75" fill="currentColor" d="M4 12a8 8 0 018-8V0C5.373 0 0 5.373 0 12h4zm2 5.291A7.962 7.962 0 014 12H0c0 3.042 1.135 5.824 3 7.938l3-2.647z"></path>
          </svg>
        </div>
      <% end %>
    <% end %>
  <% end %>
<% else %>
  <%# Page 1: Full table replacement (for filter/search/sort changes) %>
  <%= turbo_stream.replace 'items' do %>
    <%= render 'items_table' %>
  <% end %>
<% end %>
```

**Key logic:**
- `@page > 1`: Infinite scroll — append rows and chain pagination
- `@page == 1`: Filter/search/sort change — replace the entire table (resets the scroll chain)
- `turbo_stream.replace "pagination-#{@page}"` replaces the *current* frame (the one that triggered this request) with a *new* frame for the next page
- If `@pagy.next` is nil (last page), the replace block renders nothing, removing the pagination frame entirely

## Handling Filters, Search, and Sort

When filters or search change, the request should reset to page 1 and replace the full table. The `@page > 1` check in the turbo_stream template handles this automatically:

```ruby
# In the controller — page always defaults to 1 for new filter/search requests
@page = params[:page]&.to_i || 1
```

Existing filter/search forms should continue to work as-is. They submit to the same `index` action at page 1, which replaces the entire table frame (including a fresh pagination chain).

## Common Pitfalls

**Don't:** Forget `format: :turbo_stream` in next page params
```ruby
# Bad - lazy frame gets HTML response, breaks the chain
@next_page_params = { page: @pagy.next }
```

**Do:** Always include `format: :turbo_stream`
```ruby
# Good - lazy frame gets turbo_stream response
@next_page_params = { page: @pagy.next, format: :turbo_stream }.compact
```

**Don't:** Use the same pagination frame ID for every page
```erb
<!-- Bad - Turbo can't distinguish pages, breaks chaining -->
<%= turbo_frame_tag "pagination", src: ..., loading: :lazy do %>
```

**Do:** Use unique per-page IDs
```erb
<!-- Good - each page gets its own frame -->
<%= turbo_frame_tag "pagination-#{@pagy.next}", src: ..., loading: :lazy do %>
```

**Don't:** Use `turbo_stream.update` to chain pagination
```erb
<!-- Bad - update replaces inner content, not the frame itself -->
<%= turbo_stream.update "pagination-#{@page}" do %>
```

**Do:** Use `turbo_stream.replace` so the frame element itself is swapped
```erb
<!-- Good - replaces the entire frame element with the next one -->
<%= turbo_stream.replace "pagination-#{@page}" do %>
```

**Don't:** Forget to handle page 1 differently in the turbo_stream template
```erb
<!-- Bad - appending on page 1 duplicates the initial rows -->
<%= turbo_stream.append 'items-tbody' do %>
  <% @items.each do |item| %>
    <%= render 'item_row', item: item %>
  <% end %>
<% end %>
```

**Do:** Replace the full table on page 1 (filter/sort/search), append on page 2+
```erb
<!-- Good - page 1 resets, page 2+ appends -->
<% if @page > 1 %>
  <%= turbo_stream.append 'items-tbody' do %>
    ...
  <% end %>
<% else %>
  <%= turbo_stream.replace 'items' do %>
    <%= render 'items_table' %>
  <% end %>
<% end %>
```

**Don't:** Forget to carry forward filter/sort params in `@next_page_params`
```ruby
# Bad - subsequent pages lose the current filters
@next_page_params = { page: @pagy.next, format: :turbo_stream }
```

**Do:** Include all active filter/sort/search params
```ruby
# Good - filters persist across all pages
@next_page_params = {
  page: @pagy.next,
  search: params[:search],
  sort: params[:sort],
  direction: params[:direction],
  format: :turbo_stream
}.compact
```
