# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

This is a **Jekyll blog** using the **Chirpy theme** (v7.3+). It's a starter template that provides a complete Jekyll site structure for creating technical blogs with features like table of contents, dark/light themes, comments, search, and PWA support.

## Development Commands

### Local Development
```bash
# Install dependencies
bundle install

# Start development server
bash tools/run.sh

# Start with custom host
bash tools/run.sh -H 0.0.0.0

# Start in production mode
bash tools/run.sh --production
```

### Build and Test
```bash
# Build and test the site (runs HTML proofer)
bash tools/test.sh

# Build with custom config
bash tools/test.sh -c "_config.yml,_config.local.yml"

# Manual build
JEKYLL_ENV=production bundle exec jekyll build

# Manual test
bundle exec htmlproofer _site --disable-external
```

## Site Architecture

### Core Configuration
- `_config.yml` - Main Jekyll configuration with Chirpy theme settings
- `Gemfile` - Ruby dependencies, uses jekyll-theme-chirpy gem

### Content Structure
- `_posts/` - Blog posts in Markdown (currently has placeholder)
- `_tabs/` - Navigation pages (About, Archives, Categories, Tags)
- `_data/` - Site data files (contact.yml, share.yml)
- `index.html` - Homepage using 'home' layout from theme

### Theme Integration
- Uses `jekyll-theme-chirpy` gem which provides layouts, includes, and assets
- Theme files are in gem, use `bundle info --path jekyll-theme-chirpy` to locate
- Local overrides go in `_sass/`, `assets/`, `_includes/`, `_layouts/` if needed

### Plugins and Hooks
- `_plugins/posts-lastmod-hook.rb` - Automatically sets last_modified_at from git commits
- Theme includes various Jekyll plugins for archives, SEO, PWA, etc.

### Deployment
- Configured for GitHub Pages via `.github/workflows/pages-deploy.yml`
- Builds on push to main/master branches
- Uses Ruby 3.3 and runs HTML proofer tests

## Key Features to Know

### Front Matter Conventions
- Posts: Use `layout: post` with date-prefixed filenames
- Tabs: Use `icon:` and `order:` for navigation
- All content supports Chirpy's special syntax (prompts, file paths, etc.)

### Site Configuration Areas
- SEO and social media settings in `_config.yml`
- Analytics integration (Google, GoatCounter, etc.)
- Comment systems (Disqus, Utterances, Giscus)
- PWA and offline cache settings

### Content Guidelines
- Posts go in `_posts/` with format: `YYYY-MM-DD-title.md`
- Use Chirpy's prompt syntax: `{: .prompt-info}`, `{: .filepath}`
- Images and assets in `assets/` directory


## Environmnet guidelines
- Always run conda 'source ~/miniconda3/etc/profile.d/conda.sh && conda activate jekyll-site' before running any command - like a bash command or any code