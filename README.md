# Viking Analytics Style Guide

Internal style guides for Viking Analytics codebases, published as a
Jekyll site on GitHub Pages.

**Live site:** hosted via GitHub Pages from the `main` branch.

## Repository layout

```
_config.yml          Jekyll configuration
_layouts/default.html   Single-page layout with VA brand colours
index.md             Landing page — table of available guides
pyguide.md           Python style guide (permalink: /pyguide)
```

## Adding or editing a guide

Each guide is a Markdown file at the repo root with a Jekyll permalink
in its front matter:

```yaml
---
permalink: /myguide
---
```

To add a new guide:

1. Create `<lang>guide.md` at the repo root with a `permalink`.
2. Add a row to the table in `index.md`.
3. Push to `main` — GitHub Pages rebuilds automatically.

## Local preview

```sh
bundle install        # first time only
bundle exec jekyll serve
```

Then open `http://localhost:4000`.

Requires Ruby and Bundler. If there is no `Gemfile`, create one:

```ruby
source "https://rubygems.org"
gem "github-pages", group: :jekyll_plugins
```

## Conventions

- Guides are long-form prose, not checklists — they explain *why*,
  not just *what*.
- Code examples use the language the guide covers.
- The layout uses VA brand colours defined as CSS custom properties in
  `_layouts/default.html`.
