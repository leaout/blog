# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

This is a **Hexo v7** static blog site using the custom **ZenMind** theme (a simplified single-column theme forked from `one-paper`). It's deployed to GitHub Pages at [leaout.github.io](https://leaout.github.io). The blog is authored by a Chinese C/C++ engineer focused on database internals and backend architecture. Content is primarily in Chinese (zh-CN).

## Commands

| Command | Description |
|---------|-------------|
| `npm run build` or `hexo generate` | Build static site into `public/` |
| `npm run clean` or `hexo clean` | Remove generated files and cache |
| `npm run server` or `hexo server` | Start local dev server (default http://localhost:4000) |
| `npm run deploy` or `hexo deploy` | Deploy to GitHub Pages (pushes `public/` to `leaout.github.io.git:main`) |
| `npx hexo new post "Title"` | Create a new blog post under `source/_posts/` |
| `npx hexo new page "about"` | Create a new page under `source/` |

**Note:** `hexo server` starts a local preview. Always run `hexo generate` before `hexo deploy`.

## Project Structure

```
_config.yml                    # Hexo site config: title, URL, theme, deploy target
package.json                   # hexo-site, no tests/lint configured
scaffolds/                     # Templates for `hexo new` (post.md, page.md, draft.md)
source/
  _posts/                      # Blog posts in Markdown with YAML front matter
  about/                       # About page (hand-written HTML)
  images/                      # Post asset images
themes/
  ZenMind/                     # Custom theme (EJS templates)
    _config.yml                # Theme config (menu navigation)
    layout/                    # EJS templates: layout.ejs, index.ejs, post.ejs, archive.ejs
    source/                    # Theme static assets (CSS, JS, fonts, images)
public/                        # Generated output (gitignored)
.deploy_git/                   # Deploy branch checkout (gitignored)
db.json                        # Hexo cache (gitignored)
```

## Architecture

- **Hexo** generates a static site from Markdown posts in `source/_posts/`.
- **Front matter** (YAML between `---` delimiters) defines title, date, and tags for each post.
- **ZenMind theme** uses EJS templates with partials (`_partial/head`, `_partial/header`, `_partial/footer`, `_partial/paginator`, `_partial/post-header`).
- **Syntax highlighting** uses highlight.js (bundled in theme), with Hexo's built-in highlight/prismjs disabled in theme instructions.
- **Post asset images** are stored alongside posts via `post_asset_folder: true` in config.
- **Deployment** uses `hexo-deployer-git` to push generated HTML to a separate GitHub Pages repo (`leaout.github.io.git`).

## Theme Customization

The ZenMind theme templates are in `themes/ZenMind/layout/`. Key files:
- `layout.ejs` — main wrapper with `<head>`, `<header>`, `<body>` content yield, `<footer>`
- `index.ejs` — home page listing posts
- `post.ejs` — single post view
- `archive.ejs` — archive page
- `_partial/` — header, footer, head, paginator, and post-header partials

Theme config at `themes/ZenMind/_config.yml` defines the navigation menu items.

## Dependabot

`.github/dependabot.yml` runs daily npm dependency checks (up to 20 PRs).

## Notes

- No test framework or linter is configured.
- The `public/` and `.deploy_git/` directories are generated content (both gitignored).
- `db.json` is a Hexo cache file that can be deleted safely.
