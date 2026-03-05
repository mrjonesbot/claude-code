# Model Enumerations

A pattern for defining filterable enumerations on models using the `enumerize` gem with i18n support.

## When to Use

- Models with categorical fields (types, statuses, industries, etc.)
- Fields that need human-readable labels in the UI
- Filter/search criteria for list views

## Implementation

### 1. Define Enumerations in a Concern

```ruby
# app/models/concerns/example_enumerations.rb
module ExampleEnumerations
  extend Enumerize
  extend ActiveSupport::Concern

  enumerize :status, in: %w[
    active
    pending
    archived
  ], i18n_scope: 'status', skip_validations: true

  enumerize :category, in: %w[
    type_one
    type_two
    type_three
  ], i18n_scope: 'category', skip_validations: true
end
```

### 2. Include in Model

```ruby
class Example < ApplicationRecord
  include ExampleEnumerations
end
```

### 3. Add Locale Translations

```yaml
# config/locales/en/en.yml
en:
  enumerize:
    status:
      active: 'Active'
      pending: 'Pending'
      archived: 'Archived'
    category:
      type_one: 'Type One'
      type_two: 'Type Two'
      type_three: 'Type Three'
```

### 4. Use in Views

```erb
<%# Dropdown options %>
<%= f.select :status, Example.status.options %>

<%# Display value %>
<%= @example.status.text %>

<%# Filter badge display %>
<%= Example.status.find_value(value)&.text %>
```
