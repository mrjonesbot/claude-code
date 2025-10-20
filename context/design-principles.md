[MY APP] Design Principles

This document outlines the core design principles and patterns used throughout the [MY APP] application. These principles should guide all UI/UX decisions and implementations, following S-tier SaaS dashboard standards inspired by industry leaders like Stripe, Airbnb, and Linear.

## Core Design Philosophy & Strategy

Nestingbird follows a **professional, clean, and functional** design approach optimized for accountant/property manager workflows. The interface embodies these fundamental principles:

### Primary Principles
- [ ] **Users First** - Prioritize user needs, workflows, and ease of use in every design decision
- [ ] **Meticulous Craft** - Aim for precision, polish, and high quality in every UI element and interaction
- [ ] **Speed & Performance** - Design for fast load times and snappy, responsive interactions
- [ ] **Simplicity & Clarity** - Strive for a clean, uncluttered interface with unambiguous labels, instructions, and information
- [ ] **Focus & Efficiency** - Help users achieve their goals quickly with minimal friction and unnecessary steps

### Design Standards
- [ ] **Consistency** - Maintain a uniform design language (colors, typography, components, patterns) across the entire dashboard
- [ ] **Accessibility (WCAG AA+)** - Design for inclusivity with sufficient color contrast, keyboard navigability, and screen reader compatibility
- [ ] **Opinionated Design (Thoughtful Defaults)** - Establish clear, efficient default workflows and settings, reducing decision fatigue
- [ ] **Visual Hierarchy** - Clear distinction between primary, secondary, and tertiary actions
- [ ] **Responsive Design** - Desktop-first with graceful mobile adaptation

## Design System Foundation (Tokens & Core Components)

### Color Palette & Semantic Colors

#### Primary Brand Colors
- **Indigo-600** (`#4F46E5`): Primary actions, links, focus states
                              - **Indigo-500** (`#6366F1`): Hover states for primary elements
                                                            - **Indigo-100** (`#E0E7FF`): Light backgrounds, subtle highlights
                                                                                          - **Indigo-800** (`#3730A3`): Dark text on light indigo backgrounds

#### Semantic Colors
                                                                                                                        - **Success**: Green-400 (icons), Green-100/800 (badges)
                                                                                                                        - **Danger**: Red-600 (buttons), Red-100/800 (badges)
                                                                                                                        - **Warning**: Yellow-100/800 (badges)
                                                                                                                        - **Info**: Blue-100/800 (badges)

#### Neutral Palette
                                                                                                                        - **Gray-50**: Table headers, subtle backgrounds
                                                                                                                        - **Gray-100**: Hover backgrounds, disabled states
                                                                                                                        - **Gray-200**: Borders, dividers
                                                                                                                        - **Gray-300**: Input borders
                                                                                                                        - **Gray-400**: Placeholder text, inactive icons
                                                                                                                        - **Gray-500**: Secondary text, labels
                                                                                                                        - **Gray-600**: Tertiary text
                                                                                                                        - **Gray-700**: Body text
                                                                                                                        - **Gray-900**: Headings, primary text

#### Custom OKLCH Colors
                                                                                                                        - **Copy Primary**: `oklch(37.3% 0.034 259.733)`
                                                                                                                        - **Copy Secondary**: `oklch(70.7% 0.022 261.325)`

### Typography System & Scale

#### Type Scale
                                                                                                                        - **H1**: 2.5rem (40px) - Bold, tight tracking
                                                                                                                        - **H2**: 2rem (32px) - Bold, tight line-height
                                                                                                                        - **H3**: 1.5rem (24px) - Bold, normal line-height
                                                                                                                        - **H4**: 1.125rem (18px) - Bold
                                                                                                                        - **H5**: 0.875rem (14px) - Bold, uppercase, wide tracking

#### Text Styles
                                                                                                                        - **Body**: System sans-serif, text-gray-900
                                                                                                                        - **Labels**: font-medium with mb-2
                                                                                                                        - **Meta**: text-sm text-gray-500
                                                                                                                        - **Table Headers**: text-xs uppercase tracking-wider

### Spacing System & Base Units

#### Base Unit: 0.25rem (4px)
                                                                                                                        - **Micro**: 0.5 (2px) - Badge padding
                                                                                                                        - **Small**: 2 (8px) - Button padding
                                                                                                                        - **Default**: 4 (16px) - Card padding
                                                                                                                        - **Medium**: 6 (24px) - Section spacing
                                                                                                                        - **Large**: 8 (32px) - Page sections

#### Standard Patterns
                                                                                                                        - **Buttons**: px-3 py-2
                                                                                                                        - **Badges**: px-2.5 py-0.5
                                                                                                                        - **Table Cells**: px-6 py-3
                                                                                                                        - **Form Groups**: mb-6
                                                                                                                        - **Container**: Use `.container` class for consistent page-level spacing

## Layout, Visual Hierarchy & Structure

### Page Architecture
    1. **Sidebar Navigation** (256px fixed width)
    2. **Main Content Area** (Turbo Frame #main)
    3. **Page Header** (title + actions)
4. **Content Container** (.container)

### Layout Patterns & Preferences

#### Edge-to-Edge Layouts (Preferred)
    For dashboard sections and navigation-style content, prefer **edge-to-edge layouts** that extend to container boundaries:

    ```html
    <!-- PREFERRED: Edge-to-edge with unified container -->
    <div class="bg-white border-b">
    <div class="container">
    <div class="grid grid-cols-1 lg:grid-cols-3">
    <div class="px-6 py-4 border-r border-gray-200">Content</div>
    <div class="px-6 py-4 border-r border-gray-200">Content</div>
    <div class="px-6 py-4">Content</div>
    </div>
    </div>
    </div>
    ```

    **Benefits:**
    - Creates spacious, professional feel
    - Content flows naturally across full width
    - Reduces visual clutter from individual card borders
    - Matches production design patterns

#### Individual Cards (Use Sparingly)
    Use individual cards only when content truly needs visual separation:

    ```html
    <!-- Use only when content needs distinct separation -->
    <div class="grid grid-cols-1 lg:grid-cols-3 gap-6">
    <div class="bg-white rounded-lg border border-gray-300 p-6">
    Individual card content
    </div>
    </div>
    ```

### Container Class Usage
    Always use the predefined `.container` class for consistent page width and spacing:
        - **Definition**: `mx-auto max-w-7xl sm:px-6 lg:px-8 pt-4 pb-3`
            - **Purpose**: Provides consistent max-width, centering, responsive padding, and vertical spacing
            - **Usage**: Wrap main content sections that need standard page constraints

### Responsive Breakpoints
            - **Mobile**: < 640px
            - **Tablet**: sm (640px+)
            - **Desktop**: md (768px+), lg (1024px+)

### Visual Hierarchy

#### Elevation & Shadows
            - **Base**: No shadow (flat cards)
            - **Raised**: shadow-xs (buttons)
            - **Floating**: shadow-lg (dropdowns)
            - **Modal**: shadow-xl + backdrop

#### Border Radius
            - **Small**: rounded-md (badges)
            - **Default**: rounded-lg (cards)
            - **Full**: rounded-full (buttons, pills)

#### Focus States
            - Ring: `focus:ring-2 focus:ring-indigo-500`
            - Offset: `focus:ring-offset-2`

## Core UI Components

### Buttons

#### Primary Button
            ```html
            <button class="inline-flex items-center rounded-full
            bg-indigo-600 px-3 py-2 text-sm font-semibold
            text-white shadow-xs hover:bg-indigo-500">
            Action
            </button>
            ```

#### Danger Button
            ```html
            <button class="inline-flex items-center rounded-full
            bg-red-600 px-3 py-2 text-sm font-semibold
            text-white shadow-xs hover:bg-red-500">
            Delete
            </button>
            ```

#### Link Button
            ```html
            <button class="text-indigo-600 hover:text-indigo-500">
            Link Action
            </button>
            ```

### Form Elements

#### Input Field
            ```html
            <input class="block w-full rounded-full border-0
            py-2 px-4 text-gray-900 ring-1 ring-inset
            ring-gray-300 placeholder:text-gray-400
            focus:ring-2 focus:ring-indigo-600">
            ```

#### Form Group Pattern
            ```html
            <div class="form-group mb-6">
            <label class="block font-medium mb-2">Label</label>
            <input class="input">
            </div>
            ```

### Tables

            ```html
            <div class="overflow-x-auto">
            <table class="min-w-full divide-y divide-gray-200">
            <thead class="bg-gray-50">
            <tr>
            <th class="px-6 py-3 text-left text-xs font-medium
            text-gray-500 uppercase tracking-wider">
            Header
            </th>
            </tr>
            </thead>
            <tbody class="bg-white divide-y divide-gray-200">
            <tr>
            <td class="px-6 py-3 text-sm">Content</td>
            </tr>
            </tbody>
            </table>
            </div>
            ```

### Cards
            ```html
            <div class="bg-white rounded-lg border border-gray-300 p-4">
            <!-- Card content -->
            </div>
            ```

### Badges
            ```html
            <span class="inline-flex items-center px-2.5 py-0.5
            rounded-md text-sm font-medium
            bg-indigo-100 text-indigo-800">
            Badge
            </span>
            ```

### Modals
            ```html
            <div class="fixed inset-0 bg-gray-500 bg-opacity-75">
            <div class="bg-white rounded-lg shadow-xl
            px-4 pb-4 pt-5 sm:p-6 sm:max-w-lg">
            <!-- Modal content -->
            </div>
            </div>
            ```

### Navigation Tabs
            ```html
            <!-- Desktop -->
            <nav class="flex space-x-4">
            <a class="px-3 py-2 font-medium text-sm rounded-md">Tab</a>
            </nav>

            <!-- Mobile -->
            <select class="block w-full rounded-md">
            <option>Tab</option>
            </select>
            ```

### Alerts
            ```html
            <div class="rounded-lg bg-white shadow-lg p-4">
            <div class="flex items-start">
            <svg class="h-6 w-6 text-green-400"><!-- icon --></svg>
            <div class="ml-3">
            <p class="text-sm font-medium">Success</p>
            <p class="text-sm text-gray-500">Description</p>
            </div>
            </div>
            </div>
            ```

