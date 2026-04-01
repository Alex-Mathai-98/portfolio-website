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
Inherits from Chirpy's `default` layout. Currently unused by any page (all tabs use `page-no-sidebar` instead) but kept as a fallback.
