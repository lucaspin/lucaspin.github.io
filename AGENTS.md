# AGENTS.md

Guidance for AI agents and new contributors working in this repository.

## What this is

A personal static blog built with [Jekyll](https://jekyllrb.com/) (`~> 4.1.1`) and
served at `lucaspin.github.io`. Ruby dependencies are managed with Bundler
(`Gemfile` / `Gemfile.lock`). Content is written in Markdown (kramdown) and
rendered into a static site.

## Setup, build & serve

Prerequisites: Ruby and Bundler installed locally.

```sh
bundle install          # install gems pinned in Gemfile.lock
make dev.server         # run the local dev server with live reload
```

`make dev.server` is just a thin wrapper around:

```sh
bundle exec jekyll serve
```

The dev server serves the site at http://localhost:4000 and rebuilds on file
changes. To produce a one-off static build without serving:

```sh
bundle exec jekyll build
```

Notes:

- Always run Jekyll through `bundle exec` so the pinned gem versions are used.
- The build output goes to `_site/` (plus `.jekyll-cache/` and `.sass-cache/`).
  These are generated artifacts and are git-ignored — never edit or commit them.
- There is no automated test suite. "Testing" a change means running the dev
  server (or `jekyll build`) and confirming the site builds without errors and
  the affected pages render correctly.

## Repository layout

- `_config.yml` — site configuration (kramdown Markdown, `jekyll-feed` plugin,
  title/description, social handles). Jekyll must be restarted to pick up
  changes here; the live-reload server does not watch this file.
- `_posts/` — published blog posts as dated Markdown files.
- `_drafts/` — unpublished drafts (no date in the filename). Preview them with
  `bundle exec jekyll serve --drafts`.
- `_layouts/` — page templates: `default`, `home`, `page`, `post`, `spyable`.
  `post` wraps `spyable`; `page` wraps `default`.
- `_includes/` — reusable partials (currently `head.html`, which wires up CSS,
  fonts, the RSS link, and scripts).
- `public/` — static assets served as-is:
  - `public/css/` — stylesheets (`index.css` is the main one; plus code themes).
  - `public/img/` — images referenced from posts and pages.
  - `public/media/` — audio/other media.
- Top-level pages: `index.md` (uses `home` layout), `about.md` (uses `page`
  layout), `404.html`.
- `Makefile` — convenience targets (`dev.server`).
- `Gemfile` / `Gemfile.lock` — Ruby dependency definitions and lockfile.

## Writing posts

Create a file in `_posts/` named with the pattern:

```
YYYY-MM-DD-title-with-dashes.md
```

The date prefix is required by Jekyll and sets the post date and URL. Start
every post with YAML front matter, for example:

```yaml
---
layout: post
title: "Where is my SIGTERM, Docker?"
categories: [docker, kubernetes]
---
```

Conventions:

- Body is kramdown Markdown. Fenced code blocks with a language hint
  (```` ```dockerfile ````, ```` ```java ````, etc.) get syntax highlighting.
- External links commonly use `{:target="_blank"}` to open in a new tab, e.g.
  `[Mockito](https://github.com/mockito/mockito){:target="_blank"}`.
- Use `##`/`###` headings for structure within a post.

## Adding assets

Put images, audio, and other media under the matching `public/` subdirectory
(`public/img/`, `public/media/`, …) and reference them with an absolute,
`baseurl`-relative path, matching how `head.html` links assets:

```markdown
![alt text](/public/img/example.png)
```

## Conventions & scope

- Keep changes focused. This is a small content-and-templates repo; avoid
  broad refactors unless requested.
- Don't commit generated output (`_site/`, caches) or vendored gems
  (`vendor/`) — they're listed in `.gitignore`.
- When changing dependencies, edit `Gemfile` and run `bundle install` so
  `Gemfile.lock` stays in sync; commit both.
