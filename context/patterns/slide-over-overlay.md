# Slide-Over Overlay

A slide-over panel pattern that renders content in a right-anchored overlay using Turbo Frames. The overlay slides in from the right with a backdrop, and can be dismissed via an X button or closed programmatically via Turbo Streams after form submissions.

## When to Use

- Detail views that should appear on top of the current page (voters list, edit forms, previews)
- Forms that don't warrant a full page navigation (editing a record, creating a child record)
- Secondary content that the user should be able to dismiss to return to the main view

## How It Works

The application layout includes an empty `turbo_frame_tag "overlay"` anchor. When a link with `data-turbo-frame="overlay"` is clicked, Turbo fetches the target URL and swaps the response into this frame — rendering the slide-over on top of the current page without navigation.

Closing works two ways:
1. **X button**: Links to `close_overlay_url`, which renders an empty `overlay` frame, clearing the content
2. **Turbo Stream**: After a form submission, a `turbo_stream.replace("overlay")` swaps in the empty `close_overlay` partial

## Structure

```
app/
├── controllers/
│   └── actions_controller.rb            # close_overlay action (empty method)
├── views/
│   ├── layouts/
│   │   ├── application.html.erb         # Contains empty overlay frame anchor
│   │   └── _overlay.html.erb            # Slide-over layout partial
│   ├── actions/
│   │   ├── close_overlay.html.erb       # Renders the close partial (GET endpoint)
│   │   └── _close_overlay.html.erb      # Empty turbo frame (clears overlay)
│   └── [resource]/
│       └── [action].html.erb            # Overlay content view
└── config/
    └── routes.rb                        # close_overlay route
```

## Implementation Steps

### 1. Application Layout Anchor

The application layout already includes the overlay frame anchor — no changes needed:

```erb
<%# app/views/layouts/application.html.erb %>
<main class="flex-1 bg-gray-50 overflow-y-auto">
  <%= yield %>
  <%= turbo_frame_tag "overlay" %>
</main>
```

This empty frame is the target that gets replaced when an overlay opens.

### 2. Route and Controller for Close Action

Already configured globally. The controller action is intentionally empty — Rails implicitly renders the matching view (`close_overlay.html.erb`), which in turn renders the `_close_overlay` partial (an empty turbo frame):

```ruby
# config/routes.rb
get '/close_overlay', to: 'actions#close_overlay', as: :close_overlay
```

```ruby
# app/controllers/actions_controller.rb
class ActionsController < ApplicationController
  def close_overlay; end
end
```

### 3. Trigger Link

Open the overlay from any view with a link that targets the `overlay` turbo frame:

```erb
<%= link_to "View Details",
    detail_resource_path(resource),
    data: { turbo_frame: 'overlay' },
    class: "text-indigo-600 hover:text-indigo-800" %>
```

**Key:** `data: { turbo_frame: 'overlay' }` tells Turbo to fetch the URL and swap the response into the `overlay` frame.

### 4. Overlay Content View

The target view wraps content in the `overlay` turbo frame and uses the `_overlay` layout:

```erb
<%# app/views/resources/detail.html.erb %>
<%= turbo_frame_tag 'overlay' do %>
  <%= render layout: 'layouts/overlay', locals: { title: "Detail Title", size: 'max-w-2xl' } do %>
    <div class="space-y-6">
      <!-- Your content here -->
    </div>
  <% end %>
<% end %>
```

**Layout locals:**
- `title` (required): Displayed in the slide-over header
- `size` (optional): Max width class, defaults to `'max-w-6xl'`. Common values: `'max-w-sm'`, `'max-w-2xl'`, `'max-w-4xl'`, `'max-w-6xl'`

### 5. Closing via Turbo Stream (after form submission)

When a form inside the overlay is submitted and processed, close the overlay in the turbo stream response:

```erb
<%# app/views/resources/update.turbo_stream.erb %>
<%= turbo_stream.replace("overlay", partial: 'actions/close_overlay') %>

<%# Other stream updates (refresh lists, update counters, etc.) %>
<%= turbo_stream.replace "resource_list" do %>
  <%= render 'resource_list' %>
<% end %>
```

The `_close_overlay` partial simply renders an empty turbo frame, which replaces the overlay content with nothing:

```erb
<%# app/views/actions/_close_overlay.html.erb %>
<%= turbo_frame_tag 'overlay' %>
```

## Complete Example

**Triggering the overlay (from a list view):**

```erb
<%# app/views/elections/_results.html.erb %>
<%= link_to winner.name,
    voters_choice_path(winner),
    data: { turbo_frame: 'overlay' },
    class: "font-medium text-gray-900 hover:text-indigo-600 transition-colors cursor-pointer" %>
```

**Overlay content view:**

```erb
<%# app/views/choices/voters.html.erb %>
<%= turbo_frame_tag 'overlay' do %>
  <%= render layout: 'layouts/overlay', locals: { title: "Voters for #{@choice.name}", size: 'max-w-2xl' } do %>
    <div class="space-y-6">
      <!-- Summary card, list of voters, empty state, etc. -->
    </div>
  <% end %>
<% end %>
```

## Common Pitfalls

**Don't:** Forget to wrap content in `turbo_frame_tag 'overlay'`
```erb
<!-- Bad - Turbo can't match the frame -->
<%= render layout: 'layouts/overlay', locals: { title: "Edit" } do %>
  <!-- content -->
<% end %>
```

**Do:** Always wrap in the matching turbo frame
```erb
<!-- Good - Turbo swaps into the overlay anchor -->
<%= turbo_frame_tag 'overlay' do %>
  <%= render layout: 'layouts/overlay', locals: { title: "Edit" } do %>
    <!-- content -->
  <% end %>
<% end %>
```

**Don't:** Use `turbo_stream.update` to close (leaves an empty frame element in DOM)
```erb
<!-- Avoid - inconsistent with the pattern -->
<%= turbo_stream.update "overlay", "" %>
```

**Do:** Use `turbo_stream.replace` with the close partial
```erb
<!-- Good - replaces with a clean empty frame -->
<%= turbo_stream.replace("overlay", partial: 'actions/close_overlay') %>
```

**Don't:** Forget `data: { turbo_frame: 'overlay' }` on trigger links
```erb
<!-- Bad - navigates to a full page instead of opening overlay -->
<%= link_to "Edit", edit_resource_path(resource) %>
```

**Do:** Always target the overlay frame
```erb
<!-- Good - opens as slide-over -->
<%= link_to "Edit", edit_resource_path(resource),
    data: { turbo_frame: 'overlay' } %>
```

**Don't:** Forget to close the overlay in turbo stream responses after form submissions
```erb
<!-- Bad - overlay stays open after save -->
<%= turbo_stream.replace "resource_list" do %>
  <%= render 'resource_list' %>
<% end %>
```

**Do:** Always include overlay close in turbo stream responses
```erb
<!-- Good - overlay closes and list refreshes -->
<%= turbo_stream.replace("overlay", partial: 'actions/close_overlay') %>
<%= turbo_stream.replace "resource_list" do %>
  <%= render 'resource_list' %>
<% end %>
```
