# _tabs/ — Navigation Tab Pages

## Layout and Front Matter
- All tabs use `layout: page-no-sidebar` — a custom override in `_layouts/`, NOT Chirpy's default `page` layout. This removes the right sidebar/TOC panel for full-width content.
- `order` controls sidebar navigation position (0 = first). Current assignments: papers=0, writing=2, contact=3, resume=4. The homepage (`index.md` at root) occupies order=1 with `layout: home`.
- `icon` uses FontAwesome class names (e.g., `fas fa-info-circle`).
- Setting `title: " "` (a space) suppresses the auto-generated page heading — papers.md does this because it renders its own styled headings in HTML.

## Required Boilerplate
Every tab page must include this CSS override and wrapper div for consistent width/padding:

```css
.dynamic-title { padding: 5% 10% 0 10% !important; max-width: 1200px !important; margin: 0 auto !important; }
```
```html
<div style="padding: 0 10%; max-width: 1200px; margin: 0 auto;">
  <!-- page content -->
</div>
```

## papers.md — Alternating Layout Pattern
Projects alternate between two flex layouts:
1. **Left-image / right-text**: `<h3 text-align: left>`, then flex container with image (flex: 1) on left, text (flex: 2.5) on right
2. **Right-image / left-text**: `<h3 text-align: right>`, then flex container with text (flex: 2.5) on left, image (flex: 1) on right

New project entries must continue this alternating pattern.

## Image Paths
- Institutional logos → `/assets/img/`
- Project screenshots → `/assets/wordpress/`
- Resume PDF → `/assets/pdf/Alex_Mathai_CV2.pdf`
