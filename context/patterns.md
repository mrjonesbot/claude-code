# Patterns

This document indexes reusable patterns used throughout the application. Each pattern is documented in its own file for easier maintenance and syncing.

## Component Patterns

| Pattern | File | Description |
|---------|------|-------------|
| [Tab Navigation](patterns/tab-navigation.md) | `patterns/tab-navigation.md` | Tabbed interfaces with URL-based navigation and lazy-loaded Turbo Frames |
| [Typeahead Select](patterns/typeahead-select.md) | `patterns/typeahead-select.md` | Searchable select with keyboard nav, single/multi-select modes |
| [Model Enumerations](patterns/model-enumerations.md) | `patterns/model-enumerations.md` | Filterable enumerations using `enumerize` gem with i18n |
| [Filter Panel](patterns/filter-panel.md) | `patterns/filter-panel.md` | Collapsible filter panel with tag typeahead and filter badges |
| [Slide-Over Overlay](patterns/slide-over-overlay.md) | `patterns/slide-over-overlay.md` | Right-anchored overlay panel using Turbo Frames |
| [Infinite Scroll](patterns/infinite-scroll.md) | `patterns/infinite-scroll.md` | Auto-loading pagination with Turbo lazy frames |

## Backend Patterns

| Pattern | File | Description |
|---------|------|-------------|
| [Memory-Safe Exports](patterns/memory-safe-exports.md) | `patterns/memory-safe-exports.md` | Disk-backed CSV/XML processing, Export model with S3 upload, streaming patterns for background jobs |

## Future Patterns to Document

- Form validation patterns
- Notification toasts
- Sortable tables
- Drag and drop
