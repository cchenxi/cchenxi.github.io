# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

This is a Hugo-based personal blog hosted at cchenxi.github.io. The blog is primarily written in Chinese and covers software engineering topics.

## Build and Deploy

- **Local development**: `hugo server -D` (D flag includes draft posts)
- **Build production**: `hugo --minify`
- **Deploy**: Push to `master` branch triggers GitHub Actions workflow that builds and deploys to GitHub Pages

## Content Structure

- **Blog posts**: `content/posts/` - All blog posts as markdown files with frontmatter
- **Static assets**: `static/` - Images and other static files (referenced as `/image/...` in posts)
- **Archetypes**: `archetypes/default.md` - Template for new post frontmatter
- **Generated site**: `public/` - Built Hugo site (not tracked in git)
- **Documentation build**: `docs/` - Another build output directory

## Theme

The blog uses the "mainroad" Hugo theme, included as a git submodule in `themes/mainroad/`. Theme updates should be handled via submodule commands.

## Creating New Posts

To create a new post, use Hugo's archetypes:
```bash
hugo new posts/my-post-title.md
```

This creates a post with frontmatter including title, date, draft status, tags, and categories.

## Frontmatter Format

Posts use YAML frontmatter with these common fields:
- `title`: Post title
- `date`: Publication date (ISO 8601 format)
- `draft`: true/false
- `tags`: Array of tags
- `categories`: Array of categories
