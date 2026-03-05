# Typeahead Select Component

A searchable select component with keyboard navigation, powered by Stimulus. Supports both single-select and multi-select modes with tag display.

## When to Use

- Dropdowns with 10+ options that benefit from search/filtering
- Forms where users need to quickly find and select from many options
- Multi-select scenarios where users can assign multiple values (teams, groups, tags)
- Replace standard `<select>` elements for better UX

## Structure

```
app/
├── views/
│   └── shared/
│       └── _tag_typeahead.html.erb    # Shared partial
├── javascript/
│   └── controllers/
│       └── tag_typeahead_controller.js # Stimulus controller
└── [your form view].html.erb           # Usage in forms
```

## Implementation

### Partial (`app/views/shared/_tag_typeahead.html.erb`)

**Parameters:**
- `form` (required): Form builder object
- `field_name` (required): Name of the field (use `_search` suffix, e.g., `staff_ids_search`)
- `options` (required): Array of options `[label, value]` or simple strings
- `placeholder` (optional): Placeholder text for search input
- `multi_select` (optional): Boolean, default `false`
- `auto_submit` (optional): Boolean, submit form on selection, default `false`
- `value` (optional): Initial value for single-select
- `initial_values` (optional): Array of initial values for multi-select

### Stimulus Controller (`app/javascript/controllers/tag_typeahead_controller.js`)

**Key methods:**
- `filter()`: Filters dropdown options based on search term
- `select()`: Handles option selection
- `clear()`: Clears selected value (single-select)
- `removeItem()`: Removes a selected tag (multi-select)
- `handleKeydown()`: Arrow key navigation and enter to select
- `highlightOption()`: Mouse hover highlighting (sets `navigatingWithKeyboard = false`)
- `highlightSelectedOption()`: Applies highlight styling and conditionally scrolls into view

## Usage Examples

#### Single-Select

```erb
<%= form_with url: some_path, method: :post do |f| %>
  <label class="block text-sm font-medium text-gray-700 mb-2">
    Assign Member
  </label>

  <%= render 'shared/tag_typeahead',
             form: f,
             field_name: :member_id_search,
             options: @members.map { |m| [m.name, m.id] },
             placeholder: 'Search members...',
             multi_select: false %>

  <%= f.submit "Assign",
              class: "mt-3 w-full inline-flex items-center justify-center rounded-md bg-indigo-600 px-3 py-2 text-sm font-semibold text-white shadow-sm hover:bg-indigo-500" %>
<% end %>
```

**Visual behavior:**
- **Before selection**: Shows search input with placeholder and magnifying glass icon
- **After selection**: Input hidden, shows gray card with bold label and remove button
- **Click remove button**: Clears selection, shows search input again

#### Multi-Select

```erb
<%= form_with url: some_path, method: :post do |f| %>
  <label class="block text-sm font-medium text-gray-700 mb-2">
    Assign to Groups
  </label>

  <%= render 'shared/tag_typeahead',
             form: f,
             field_name: :group_ids_search,
             options: @groups.map { |g| [g.name, g.id] },
             placeholder: 'Search groups...',
             multi_select: true %>

  <%= f.submit "Assign",
              class: "mt-3 w-full inline-flex items-center justify-center rounded-md bg-indigo-600 px-3 py-2 text-sm font-semibold text-white shadow-sm hover:bg-indigo-500" %>
<% end %>
```

#### Auto-Submit (for filters)

```erb
<%= form_with url: items_path,
              method: :get,
              data: { turbo_frame: "items_list" } do |f| %>
  <%= render 'shared/tag_typeahead',
             form: f,
             field_name: :category_id_search,
             options: @categories.map { |c| [c.name, c.id] },
             placeholder: 'Filter by category...',
             auto_submit: true %>
<% end %>
```

## Form Field Naming Convention

**Important:** The component uses a naming convention to separate search input from form values:

- **Search field**: `field_name_search` (e.g., `member_id_search`)
- **Hidden field (single)**: `field_name` (e.g., `member_id`)
- **Hidden fields (multi)**: `field_name[]` (e.g., `group_ids[]`)

The Stimulus controller automatically:
1. Removes `_search` from the field name
2. Creates hidden field(s) with the actual name
3. Ensures search input doesn't submit with form

## Keyboard Shortcuts

- **Arrow Down**: Open dropdown or move to next option
- **Arrow Up**: Move to previous option
- **Enter**: Select highlighted option
- **Escape**: Close dropdown
- **Tab**: Close dropdown and move to next field
- **Type to search**: Filter options in real-time

**Keyboard vs Mouse Navigation:**

The controller tracks a `navigatingWithKeyboard` flag to prevent a scroll feedback loop in long dropdown lists. When using arrow keys, `scrollIntoView({ block: "nearest" })` keeps the highlighted option visible. When hovering with the mouse, `scrollIntoView` is skipped — otherwise it repositions options away from the cursor, causing unintended highlight jumps.

```javascript
// In handleKeydown — arrow keys set the flag
this.navigatingWithKeyboard = true;

// In highlightOption — mouse hover clears it
highlightOption(event) {
  this.navigatingWithKeyboard = false;
  // ...
}

// In highlightSelectedOption — only scroll during keyboard nav
if (this.navigatingWithKeyboard) {
  selectedOption.scrollIntoView({ block: "nearest" });
}
```

## Common Pitfalls

**Don't:** Forget the `_search` suffix on field name
```erb
<!-- Bad - will conflict with hidden field -->
<%= render 'shared/tag_typeahead',
           form: f,
           field_name: :member_id,  # Missing _search
           ... %>
```

**Do:** Always use `_search` suffix for the search field
```erb
<!-- Good - search field is separate from value field -->
<%= render 'shared/tag_typeahead',
           form: f,
           field_name: :member_id_search,
           ... %>
```

**Don't:** Pass options as simple strings when you need values
```erb
<!-- Bad - can't distinguish label from value -->
<%= render 'shared/tag_typeahead',
           options: @teams.pluck(:name),
           ... %>
```

**Do:** Always provide [label, value] pairs
```erb
<!-- Good - label displays, value submits -->
<%= render 'shared/tag_typeahead',
           options: @teams.map { |t| [t.name, t.id] },
           ... %>
```

**Don't:** Place submit button in flexbox with input
```erb
<!-- Bad - layout issues -->
<div class="flex gap-2">
  <%= render 'shared/tag_typeahead', ... %>
  <%= f.submit "Assign" %>
</div>
```

**Do:** Place submit button below input with proper spacing
```erb
<!-- Good - full width, clear hierarchy -->
<%= render 'shared/tag_typeahead', ... %>
<%= f.submit "Assign", class: "mt-3 w-full ..." %>
```

**Don't:** Use multi-select for single values
```erb
<!-- Bad - user can only select one anyway -->
<%= render 'shared/tag_typeahead',
           field_name: :team_id_search,
           multi_select: true,
           ... %>
```

**Do:** Match multi-select to data model
```erb
<!-- Good - singular relationship = single-select -->
<%= render 'shared/tag_typeahead',
           field_name: :team_id_search,
           multi_select: false,
           ... %>

<!-- Good - plural relationship = multi-select -->
<%= render 'shared/tag_typeahead',
           field_name: :group_ids_search,
           multi_select: true,
           ... %>
```

**Implementation Note:**

The `selectedTags` target div is always rendered (for both single and multi-select modes) to allow the JavaScript controller to display selected items. In single-select mode, the controller hides the search input and renders a card showing the selected item. Do not conditionally render this target only for multi-select.
