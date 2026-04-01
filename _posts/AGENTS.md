# _posts/ — Blog Posts

## File Naming
Files MUST follow `YYYY-MM-DD-title.md` format. Jekyll derives the post date from the filename.

## Front Matter
- `layout: post` is applied automatically by `_config.yml` defaults — do not add it explicitly.
- `last_modified_at` is **auto-set** by `_plugins/posts-lastmod-hook.rb` from git history. Never set this field manually.
- `date` in front matter is optional if the filename date is sufficient.
- `categories` and `tags` are available but currently unused.

## Content Formatting Conventions
- Blockquotes (`>`) for scripture and quotations
- `&nbsp;` sequences for indentation within blockquotes
- Bold numbers (`**1**`, `**2**`) for structured verse/section numbering
- `{: .prompt-info }` for Chirpy callout blocks

## Note
`2025-02-07-Mom-Died.md` exists locally but is untracked in git (not deployed).
