# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

Personal blog ("Notes for Myself") at https://tksunw.github.io. Jekyll static site using the Chirpy theme gem (v7.4.1) — theme layouts/includes/assets come from the gem, not this repo. Deployed to GitHub Pages via GitHub Actions on push to `master`.

## Commands

```bash
bundle install              # install dependencies
bundle exec jekyll serve    # local dev server at http://127.0.0.1:4000
bundle exec jekyll build    # build to _site/
```

CI test (what the workflow runs after build):

```bash
bundle exec htmlproofer _site --disable-external \
  --ignore-urls "/^http:\/\/127.0.0.1/,/^http:\/\/0.0.0.0/,/^http:\/\/localhost/"
```

There is no other test or lint suite.

## Writing posts

- File: `_posts/YYYY-MM-DD-title-slug.md`. Post date must not be in the future or Jekyll silently skips it.
- Front matter:

```yaml
---
title: "Post Title"
date: YYYY-MM-DD HH:MM:SS -0500
categories: [Category]
tags: [tag1, tag2]
---
```

- Existing categories: Solaris, Linux, Networking, Automation, Personal.
- Post images go in `assets/img/posts/`.
- Layout, comments, TOC, and `/posts/:title/` permalinks are set by defaults in `_config.yml` — don't repeat them in front matter.

## Architecture notes

- `_plugins/posts-lastmod-hook.rb` sets `last_modified_at` from git history at build time. It uses `Shellwords.shellescape` deliberately (upstream Chirpy doesn't) — keep it.
- `_tabs/` holds the sidebar pages (about, archives, categories, tags) as a Jekyll collection.
- Deployment: `.github/workflows/pages-deploy.yml` builds with Ruby 3.3, tests with html-proofer, deploys with `upload-pages-artifact@v3` + `deploy-pages@v4` (these versions must stay paired — v2 deploy can't read v3 artifacts). Repo Settings > Pages source must be "GitHub Actions", not branch-based.
- Workflow triggers only on pushes to `main`/`master`; feature branches need manual `workflow_dispatch` or a merge to test deployment.
- Do not add a `.nojekyll` file — it breaks Jekyll processing.
- No git submodules; a broken `assets/lib` submodule was removed, don't reintroduce it.
- Email is intentionally absent from `_config.yml` (spam harvesting). Comments and analytics are intentionally unconfigured.
