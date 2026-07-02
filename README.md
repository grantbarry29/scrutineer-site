# scrutineer.sh

Landing page and blog for [Scrutineer](https://github.com/grantbarry29/scrutineer),
built with [Hugo](https://gohugo.io/) + [PaperMod](https://github.com/adityatelange/hugo-PaperMod),
deployed to GitHub Pages via `.github/workflows/hugo.yaml`.

## Local preview

```sh
brew install hugo
git submodule update --init --recursive
hugo server -D        # -D renders draft posts; http://localhost:1313
```

## Writing

Posts live in `content/blog/`. New drafts ship with `draft: true` in front matter and
only publish once that's flipped to `false`.

## Deploy

Push to `main`. The workflow builds and publishes to GitHub Pages.
One-time repo setup: Settings → Pages → Source: **GitHub Actions**, custom domain
`scrutineer.sh` (the `static/CNAME` file keeps it pinned), Enforce HTTPS once the
cert issues. DNS at the registrar: apex `A` records → `185.199.108.153`,
`185.199.109.153`, `185.199.110.153`, `185.199.111.153`, plus `www` CNAME →
`<username>.github.io`.
