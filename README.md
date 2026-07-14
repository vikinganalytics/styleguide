# Viking Analytics Style Guide

Style guides for Viking Analytics codebases, published at
**https://vikinganalytics.github.io/styleguide/**.

## Available guides

| Language | Source | Published |
|----------|--------|-----------|
| Python | [pyguide.md](pyguide.md) | [pyguide.html](https://vikinganalytics.github.io/styleguide/pyguide.html) |

## Adding a new guide

1. Add a `<lang>guide.md` file at the repo root.
2. Add a pandoc build step in [`.github/workflows/pages.yml`](.github/workflows/pages.yml).
3. Add a row to [`index.md`](index.md) so it appears on the landing page.
