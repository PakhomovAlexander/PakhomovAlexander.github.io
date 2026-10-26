# Personal site redirects

The main site is https://apkhmv.xyz/.

This repository keeps https://pakhomovalexander.github.io working as a redirect
site. Personal pages (including `/about/`, English pages, and the 404 fallback)
redirect to the new homepage. The podcast index and episode pages retain their
specific destinations under `https://apkhmv.xyz/podcast/`.

Redirects use an immediate HTML meta refresh, a canonical URL, and a visible
fallback link. The original content is retained here to preserve its routes.
GitHub Pages remains on its existing domain; no DNS or custom-domain change is
needed. Pushes to `main` deploy through `.github/workflows/hugo.yaml`.

## How to build

- Install Hugo extended 0.111.3 (the version pinned in the deployment workflow).
- Run `git submodule update --init --recursive`.
- Run `hugo --gc --minify --destination /tmp/gh-pages-redirect-public`.

