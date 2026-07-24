# Learning Log — agent guide

Personal Jekyll site published through GitHub Pages. Keep changes small,
public-safe, and easy to verify.

## Build & verify

- Run `bundle exec jekyll build` when local Ruby dependencies are available.
  GitHub Pages builds the site from `main`.
- Run `git diff --check` before handing off a change.
- Review the affected rendered page after content, layout, style, or asset
  changes. Treat `_site/` as generated output; never edit or commit it.

## Layout

- `_posts/` — dated Markdown articles with YAML front matter.
- `_layouts/` and `_sass/` — shared Jekyll presentation.
- `assets/` — shared site assets.
- `perch/` — the standalone Perch project page and its page-specific assets.
- `_reviews/` — local review notes; Markdown files here are intentionally
  ignored and must stay out of the public repository.

## Content conventions

- Keep technical claims factual and supported by the linked implementation or
  primary source.
- Preserve the author's concise, builder-led voice. Do not invent metrics,
  experience, endorsements, or project guarantees.
- Keep post filenames in `YYYY-MM-DD-title.md` form and preserve valid front
  matter (`layout`, `title`, `date`, and relevant `tags`).
- Use canonical public links and repository-relative paths for local assets.

## Repository hygiene

- This is a public repository. Never commit credentials, wallet material,
  private notes, internal plans, machine-local paths, or personal data.
- Do not commit generated/cache files such as `_site/`, `.jekyll-cache/`, or
  `.sass-cache/`.
- Avoid unrelated rewrites or formatting churn; every changed line should
  serve the requested article or site change.
