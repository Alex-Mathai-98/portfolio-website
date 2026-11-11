# _layouts/ — Custom Layout Overrides

These layouts override the Chirpy theme defaults. Find the theme's original layouts via `bundle info --path jekyll-theme-chirpy`.

## home.html and page-no-sidebar.html
These two files are **identical**. Both render a full-width page using `class="col-12"` on the main element (the Chirpy default uses a narrower column to make room for a right-side TOC panel).

- `home.html` → used by `index.md` (the homepage)
- `page-no-sidebar.html` → used by all `_tabs/` pages

**If you change one, you must change the other.**

## Naming Clarification
"No sidebar" refers to the **right** sidebar (TOC/related-posts panel). The **left** navigation sidebar is still rendered via `{% include sidebar.html %}` and must not be removed.

## page.html
Inherits from Chirpy's `default` layout, so it keeps the right panel (Recently Updated + TOC). Tabs don't use it
(they use `page-no-sidebar`); it is for standalone long-form pages that want a TOC.

- Adds `panel_includes: [toc]`, which Chirpy's own `page` layout lacks. The TOC renders only when the page sets
  `toc: true` and has `h2`/`h3` headings, so pages without `toc: true` look exactly as before.
- Used by `talks/candidacy/candidacy.md` (`/candidacy/`, the PhD candidacy-exam paper summaries; not linked in the sidebar).
- `layout: page` also loads `page.min.js`, which gives image click-to-zoom; custom layout names would not.
- Gotcha #toc: `page.min.js` bundles Chirpy's TOC classes but never starts them (only `post.min.js` does, and it
  also needs dayjs). So `page.html` carries the post layout's TOC bar/button/popup markup plus a small inline
  tocbot start-up script (panel at >= 1200px, popup below). Keep its options in sync with the theme if upgrading.
- Gotcha #toc: for a standalone page (not a post), `page.content` is still raw Markdown when layouts run, so
  Chirpy's `contains '<h2'` check fails and no TOC appears. Keep at least one raw `<h2>` in the page body.
