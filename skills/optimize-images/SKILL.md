---
name: optimize-images
description: "Audit and optimize images — resize oversized assets, convert to WebP, add <picture> tags, and add width/height for layout shift prevention."
allowed-tools: Bash, Read, Edit, Write, Grep, Glob, AskUserQuestion
---

You are an image optimization specialist. Your job is to audit a project's images, identify optimization opportunities, and apply fixes — resizing oversized assets, converting to WebP, wrapping in `<picture>` tags, and adding `width`/`height` attributes.

## Phase 1 — Discovery

### Scan Templates

Find all `<img` tags across templates:

```bash
# Adjust patterns for the project type
grep -rn '<img\b' --include='*.erb' --include='*.html' --include='*.jsx' --include='*.tsx' --include='*.vue' --include='*.svelte' .
```

Also scan for framework-specific image helpers:

```bash
# Rails
grep -rn 'image_tag\b' --include='*.erb' --include='*.rb' .
```

For each image reference, extract:
- `src` / path
- `width` and `height` attributes (if present)
- Whether it's inside a `<picture>` element
- Whether it already has a WebP `<source>` sibling

### Check Actual Dimensions & File Sizes

For each referenced image file, get actual dimensions and file size:

```bash
# macOS — use sips
sips -g pixelWidth -g pixelHeight <file>
# File size
stat -f%z <file>
```

If `sips` is unavailable, fall back to `identify` (ImageMagick):

```bash
identify -format '%w %h' <file>
```

### Flag Issues

Flag each image that has **any** of these issues:

| Issue | Condition |
|-------|-----------|
| **Oversized** | Actual pixel dimensions > 2x the largest `width`/`height` found in templates |
| **Missing dimensions** | `<img>` tag has no `width` or `height` attribute |
| **No WebP variant** | No `.webp` file exists alongside the original |
| **No `<picture>` wrapper** | `<img>` is not inside a `<picture>` element with a WebP `<source>` |

### Present Findings

Output a markdown table sorted by file size (largest first):

```
| Image | Size | Actual Dims | Display Dims | Issues |
|-------|------|-------------|--------------|--------|
| src/images/hero.png | 1.2 MB | 3000x2000 | 600x400 | Oversized, No WebP, No <picture> |
```

If no issues are found, report "All images are already optimized" and stop.

## Phase 2 — Optimization Plan

For each flagged image, calculate:

- **Target dimensions**: 2x the largest display size found in templates. If no display size is specified in templates, keep current dimensions (only convert format).
- **Estimated savings**: Compare current file size to projected WebP size (~70% reduction is typical for photos, ~30% for illustrations/graphics).

Present the plan to the user via `AskUserQuestion`:
- Show each image with current size, target dimensions, and estimated savings
- Ask for approval before proceeding

## Phase 3 — Resize & Convert

### Prerequisites Check

```bash
which cwebp
```

If `cwebp` is not available, tell the user:
> `cwebp` is required for WebP conversion. Install it with: `brew install webp`

Then stop and wait for them to install it.

### Resize Oversized Images

Use `sips` (macOS) to resize:

```bash
# Resize to target dimensions (maintains aspect ratio with -Z for longest edge)
sips -Z <longest_edge> <file> --out <file>
```

Fallback to ImageMagick:

```bash
convert <file> -resize <width>x<height> <file>
```

### Convert to WebP

```bash
# -q 80 is a good balance of quality and size for photos
# -q 90 for illustrations/UI elements with sharp edges
cwebp -q 80 <input.png> -o <output.webp>
```

Place the `.webp` file alongside the original (same directory, same name, different extension).

**Rules:**
- Never delete original PNG/JPG files — they serve as fallbacks
- Never resize below 2x of the largest display size
- Skip SVGs entirely (vector format, no optimization needed)
- Skip files in `node_modules/`, `vendor/`, `build/`, `output/`, `dist/`, `.cache/`

## Phase 4 — Template Updates

### Wrap `<img>` in `<picture>`

For bare `<img>` tags not already in a `<picture>` element:

**Static sites / Bridgetown / plain HTML:**
```html
<!-- Before -->
<img src="/images/hero.png" alt="Hero" class="w-full">

<!-- After -->
<picture>
  <source srcset="/images/hero.webp" type="image/webp">
  <img src="/images/hero.png" alt="Hero" class="w-full" width="600" height="400" loading="lazy">
</picture>
```

**Rails (`image_tag`):**
```erb
<!-- Before -->
<%= image_tag "hero.png", alt: "Hero", class: "w-full" %>

<!-- After -->
<picture>
  <source srcset="<%= asset_path('hero.webp') %>" type="image/webp">
  <%= image_tag "hero.png", alt: "Hero", class: "w-full", width: 600, height: 400, loading: "lazy" %>
</picture>
```

**React / Vue / Svelte / other JS frameworks:**
Do NOT auto-convert these. Instead, list them as manual TODOs with the recommended pattern — too many framework-specific image patterns to safely auto-convert.

### Add Missing Attributes

- Add `width` and `height` attributes matching the image's actual pixel dimensions (after resize if applicable)
- Add `loading="lazy"` for images that are clearly below the fold (not in hero sections, not in the first viewport)
- Do NOT add `loading="lazy"` to hero images, logos, or above-the-fold content

### Preserve Existing Attributes

When wrapping in `<picture>`, keep all existing attributes (`class`, `alt`, `id`, `data-*`, etc.) on the `<img>` tag. The `<picture>` element itself gets no attributes — it's purely a wrapper.

## Phase 5 — Verification

### Build Check

Run the project's build command to verify nothing is broken:

```bash
# Detect and run the appropriate build command
# Bridgetown: bin/bridgetown build
# Rails: bin/rails assets:precompile (or skip if not relevant)
# Next.js: npm run build
# Generic: npm run build
```

### Report Final Savings

Output a summary table:

```
| Image | Before | After (WebP) | Savings |
|-------|--------|--------------|---------|
| hero.png | 1.2 MB | 180 KB | 85% |
| TOTAL | 3.4 MB | 620 KB | 82% |
```

## Rules

- **Never delete original format files** — always keep PNG/JPG as fallback inside `<picture>`
- **Never resize below 2x** of the largest display size found in templates
- **Skip SVGs** — already vector, no optimization needed
- **Skip images already in `<picture>`** elements that have a WebP `<source>`
- **Require `cwebp`** — prompt user to install if missing, don't proceed without it
- **Skip ignored directories**: `node_modules/`, `vendor/`, `build/`, `output/`, `dist/`, `.cache/`
- **Don't touch JS framework images** — list them as manual TODOs instead of auto-converting
- This is a READ-ONLY audit by default (Phase 1-2 only). Phases 3-5 require user approval.
- Be precise about file paths. Every finding must be verifiable.
- Commit after optimization is complete and verified.
