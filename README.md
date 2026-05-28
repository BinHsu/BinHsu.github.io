# binhsu.github.io

Engineering field reports — site source.

**Live:** https://binhsu.github.io

Built on [Beautiful Jekyll](https://beautifuljekyll.com/) via `remote_theme` —
no vendored theme files in this repo; GitHub Pages resolves the theme on each build.

## Adding a post

1. Drop a markdown file in `_posts/` named `YYYY-MM-DD-slug.md`
2. If the post has images, create `assets/img/posts/YYYY-MM-DD-slug/` and reference assets relatively
3. Commit + push to `main` — GitHub Pages auto-builds within ~30 seconds

## Local preview (optional)

If you want to preview before push, install Jekyll + Bundler locally, then:

```sh
bundle exec jekyll serve
```

…but for the common path, push directly — GitHub Pages is the source of truth.
