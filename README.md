# jeffrin-j.github.io

Source of https://jeffrin-j.github.io, built with the [Chirpy](https://github.com/cotes2020/jekyll-theme-chirpy)
Jekyll theme and deployed by GitHub Actions (`.github/workflows/pages-deploy.yml`) on every push to `main`.

- Posts go in `_posts/` as `YYYY-MM-DD-title.md`; they go live on push.
- Drafts go in `_drafts/` and are never published.
- Pages in the sidebar (About, Archives, ...) are in `_tabs/`.

Preview locally, drafts included, at http://localhost:4000 (needs only Docker):

```bash
docker run --rm -v "$PWD:/site" -w /site -p 4000:4000 ruby:3.4 sh -c "bundle install && bundle exec jekyll serve --drafts --host 0.0.0.0"
```
