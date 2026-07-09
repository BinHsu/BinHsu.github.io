# binhsu.github.io

Engineering field reports — site source.

**Live:** https://binhsu.github.io

Built on the [Chirpy](https://github.com/cotes2020/jekyll-theme-chirpy) theme
(gem `jekyll-theme-chirpy`, scaffolded from
[chirpy-starter](https://github.com/cotes2020/chirpy-starter)) — no vendored theme
files in this repo; the theme resolves as a gem at build time. Deployed by the
GitHub Actions workflow in `.github/workflows/pages-deploy.yml` (Pages source must
be set to "GitHub Actions").

## Adding a post

1. Drop a markdown file in `_posts/` named `YYYY-MM-DD-slug.md`
2. If the post has images, create `assets/img/posts/YYYY-MM-DD-slug/` and reference assets relatively
3. Commit + push to `main` — the Pages workflow builds and deploys in ~1-2 minutes

## Local preview (optional)

No host Ruby required — build in Docker with a repo-local gem path:

```sh
docker run --rm -p 4000:4000 -v "$PWD":/srv -w /srv -e BUNDLE_PATH=vendor/bundle \
  ruby:3.3 bash -c "bundle install && bundle exec jekyll serve --host 0.0.0.0"
```

…but for the common path, push directly — GitHub Pages is the source of truth.
