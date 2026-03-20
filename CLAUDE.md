# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Site Overview

This is a Jekyll-based personal blog ("Spirited Engineering") hosted on GitHub Pages at `spiritedengineering.net`. It uses the `minima` theme as a base with fully custom layouts and CSS overriding that theme.

## Development Commands

```bash
# Serve locally with live reload
bundle exec jekyll serve

# Build the site
bundle exec jekyll build

# Serve with drafts visible
bundle exec jekyll serve --drafts
```

The built site outputs to `_site/` (excluded from git).

## Architecture

### Theme & Styling

The site uses a **custom dark theme** ("Black & White Engineer") that overrides minima entirely:

- `assets/main.scss` — The active stylesheet. Jekyll compiles this to `/assets/main.css`, which `_layouts/default.html` loads. It owns the full visual design: CSS variables, layout, typography (Inter + JetBrains Mono), dark background (`#0d0d0d`), syntax highlighting, and all component styles.
- `assets/css/style.css` — Legacy file with old minima overrides; effectively unused since `default.html` bypasses it.

When making visual changes, edit `assets/main.scss`.

### Layout Hierarchy

```
_layouts/default.html   — base HTML shell (header, nav, main, footer)
  _layouts/home.html    — extends default; renders intro + post list
  _layouts/post.html    — extends default; renders article with tags + back link
  _layouts/page.html    — extends default; renders static pages (About, Badges)
```

Navigation items are controlled by `header_pages` in `_config.yml`. The `Home` page is excluded from nav by the layout logic.

### Posts

Posts live in `_posts/` and follow Jekyll naming: `YYYY-MM-DD-title.md`.

Standard front matter:
```yaml
---
title: "Post Title"
date: YYYY-MM-DD
author: "Duy Nguyen"
tags: ["tag1", "tag2"]
description: "Short description"
categories: ["Category"]
draft: false
---
```

Posts are listed on the home page in reverse chronological order (Jekyll default). The post date shown uses `%Y %b` format (e.g., "2025 Aug").

### Static Pages

`about.md`, `badges.md`, and `index.md` are root-level pages using `layout: page` or `layout: home`. Their permalinks and nav titles are set in front matter.

### Images

Place images in `assets/images/` and reference them in posts as `![alt](/assets/images/filename.png)`.
