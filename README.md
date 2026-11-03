# jeffrin-j.github.io

Source of https://jeffrin-j.github.io, built by GitHub Pages (Jekyll, minima theme).

- Posts go in `_posts/` as `YYYY-MM-DD-title.md`; they go live on push.
- Drafts go in `_drafts/` and are never published.

Preview locally, drafts included, at http://localhost:4000 (needs only Docker):

```bash
docker run --rm -v "$PWD:/site" -w /site -p 4000:4000 ruby:3.3 sh -c "gem install jekyll minima jekyll-feed jekyll-seo-tag jekyll-sitemap && jekyll serve --drafts --host 0.0.0.0"
```
