---
layout: post
title: "Hello, World"
date: 2026-07-01
tags: [meta]
---

First post. This blog is a learning log — short notes on what I'm studying, built with Jekyll and hosted on GitHub Pages.

## How I publish a new post

1. Create a Markdown file in `_posts/` named `YYYY-MM-DD-some-title.md`.
2. Add front matter at the top:

   ```yaml
   ---
   layout: post
   title: "Some Title"
   date: 2026-07-01
   tags: [topic]
   ---
   ```

3. Write the post body in Markdown below the front matter.
4. Commit and push. GitHub Pages rebuilds the site automatically in about a minute.

That's the whole workflow — no build tools needed locally.
