# AGENTS.md

This file supplements [CLAUDE.md](./CLAUDE.md) with guidance for OpenCode sessions.

## Pre-flight checks

- **Hugo is not guaranteed installed.** Run `hugo version` before any build command. CI pins Hugo **v0.157.0** (see `.github/workflows/gh-pages.yml`).
- **Theme is a git submodule.** Run `git submodule update --init` before `hugo server` or `hugo --minify`, otherwise the build fails with a missing theme error.

## Conventions not obvious from config

- **Static images** live in `static/image/YYYYMMDD/filename.jpg` and are referenced in posts as `/image/YYYYMMDD/filename.jpg`. Follow this date-prefixed directory convention for new images.
- **Post dates** use ISO 8601 with `+08:00` timezone: `2021-06-05T11:04:00+08:00`.
- **Posts are in Chinese.** There is no English content in this blog.
- **`docs/`** is a separate Hugo build output directory (not the CI deploy target). CI deploys `public/` to GitHub Pages.

## Verification

- **No test suite exists.** After creating/editing a post, verify by running `hugo server -D` and checking `http://localhost:1313`.
- **Build check**: `hugo --minify` should exit 0. If it fails, check the theme submodule first.
- The `public/` directory is build output and is gitignored (not tracked).

## Architecture

- Single Hugo site, no monorepo, no multi-language setup (`languageCode = "en-us"` in config but all content is Chinese).
- `config.toml` is the single source of truth for site config. No config directory or environment-specific overrides.
- Theme (`themes/mainroad`) is pinned via git submodule — do not edit its files directly; override in `static/` or `config.toml` params.
