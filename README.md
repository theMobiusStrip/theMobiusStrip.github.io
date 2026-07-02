# Learning Log

My learning blog, live at **<https://themobiusstrip.github.io>**.

Built with [Jekyll](https://jekyllrb.com/) (minima theme), hosted on GitHub Pages. Pages builds the site automatically on every push to `main` — nothing to install locally.

## Writing a post

Add a Markdown file to `_posts/` named `YYYY-MM-DD-title.md` with front matter:

```yaml
---
layout: post
title: "Post Title"
date: 2026-07-01
tags: [topic]
---
```

Then commit and push.

## Optional: local preview

```sh
gem install bundler jekyll
jekyll serve
# open http://localhost:4000
```
