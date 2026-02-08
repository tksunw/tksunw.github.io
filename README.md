# tksunw.github.io

Personal blog — "Notes for Myself: Write or Forget"

Live at **https://tksunw.github.io**

## Stack

- [Jekyll](https://jekyllrb.com/) static site generator
- [Chirpy](https://github.com/cotes2020/jekyll-theme-chirpy) theme (v7.4.1)
- GitHub Pages with GitHub Actions CI/CD

## Local Development

```bash
bundle install
bundle exec jekyll serve
```

Site will be available at `http://127.0.0.1:4000`.

## Writing Posts

Create a new file in `_posts/` following the naming convention:

```
YYYY-MM-DD-title-slug.md
```

Front matter template:

```yaml
---
title: "Post Title"
date: YYYY-MM-DD HH:MM:SS -0500
categories: [Category]
tags: [tag1, tag2]
---
```

Existing categories: Solaris, Linux, Networking, Automation, Personal

## Deployment

Pushes to `master` automatically trigger a build and deploy via GitHub Actions. The workflow lives in `.github/workflows/pages-deploy.yml`.

## License

Content is copyright Tim Kennedy. The Chirpy theme is licensed under [MIT](https://github.com/cotes2020/chirpy-starter/blob/master/LICENSE).
