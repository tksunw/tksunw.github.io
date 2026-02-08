---
title: "Migrating from Blogger to GitHub Pages with Claude Code"
date: 2026-02-07 22:00:00 -0500
categories: [Automation]
tags: [github-pages, jekyll, chirpy, claude-code, ai, blogger, migration]
---

After years of hosting my blog on Google's Blogger platform, I decided it was time to move to something more modern. I landed on [GitHub Pages](https://pages.github.com/) with the [Chirpy](https://github.com/cotes2020/jekyll-theme-chirpy) Jekyll theme -- a clean, developer-friendly setup that lives right alongside my code. To make the migration happen, I paired up with [Claude Code](https://claude.com/claude-code), Anthropic's CLI tool for working with codebases.

Here's how the session went.

## Starting Point

I had already scaffolded a Chirpy-based site from the starter template, but it had never been fully operational. No posts, no avatar, and a handful of configuration issues that needed sorting out before anything would build and deploy correctly.

## The Audit

Claude Code reviewed every file in the repo and found several issues:

- **A `.nojekyll` file** that was telling GitHub Pages to skip Jekyll processing -- on a Jekyll site
- **Outdated GitHub Actions** (checkout@v3, deploy-pages@v1, etc.) with potential security vulnerabilities
- **A command injection vulnerability** in the `posts-lastmod-hook.rb` plugin, where post paths were interpolated directly into shell commands without sanitization
- **A broken git submodule** that was defined in `.gitmodules` but never actually committed to the index
- **A POSIX-style timezone** (`est5edt`) that doesn't handle DST transitions properly
- **A stray horizontal rule** rendering at the top of the About page
- **My email address** sitting in `_config.yml` in plain text, ready for spam harvesters

All of these were fixed, committed, and pushed. The security fixes included updating all GitHub Actions to current versions and adding `Shellwords.shellescape` to the Ruby plugin.

## The Chirpy 7.x Upgrade

The site was running Chirpy 6.0, and version 7.4.1 was available with significant changes. We created a `feature/chirpy-7-upgrade` branch and tackled it:

- **Gemfile**: Updated the theme gem, bumped html-proofer from 3.x to 5.x, cleaned up deprecated platform-specific dependencies
- **`_config.yml`**: Migrated to the 7.x schema -- `google_analytics` became `analytics`, `img_cdn` became `cdn`, `comments.active` became `comments.provider`, added PWA cache configuration, removed deprecated `swcache` scopes
- **GitHub Actions workflow**: Updated to Ruby 3.3, deploy-pages@v4, and html-proofer v5 CLI flags
- **Data files**: Updated Twitter icons to the X branding (`fa-brands fa-x-twitter`)
- **`.gitignore`**: Added entries for `.jekyll-metadata` and `_sass/vendors`

We hit one snag after merging -- the deploy step failed because `upload-pages-artifact@v3` is incompatible with `deploy-pages@v2`. The v3 artifact format requires v4 of the deploy action. A quick fix and the site was building cleanly.

## Migrating the Blog Posts

With the site infrastructure solid, we turned to content. My old Blogger site at [timkennedy.net](https://www.timkennedy.net) had 13 posts spanning from 2006 to 2021. Claude Code fetched each post, converted the HTML to clean Markdown with proper fenced code blocks and language hints, assigned categories and tags, and wrote them out as Jekyll posts.

All 13 posts were migrated in a single pass:

| Category | Count | Topics |
|----------|-------|--------|
| Solaris | 5 | ZFS, Sun Cluster, OpenVPN, pyTivo, DBD::mysql |
| Linux | 3 | Bash syslog auditing, Oracle troubleshooting |
| Networking | 2 | PIA VPN on Ubiquiti, Perl port scanner |
| Automation | 2 | PowerShell Azure contexts, Sonos alarm |
| Personal | 1 | The time I broke my toe |

## The Result

From a broken scaffold to a fully operational blog with 13 migrated posts, a profile picture, dark mode, and a clean CI/CD pipeline -- all in one session. The site is live at [tksunw.github.io](https://tksunw.github.io).

Not bad for a Saturday night.
