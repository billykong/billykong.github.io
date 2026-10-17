# Billy's Tech Blog

[https://billykong.github.io/](https://billykong.github.io/), built by GitHub Pages with Jekyll and the [minima](https://github.com/jekyll/minima) theme.

## Write a post

Add `_posts/YYYY-MM-DD-slug.md` with front matter:

```yaml
---
layout: post
title: "The title"
description: "One sentence for search results and link previews."
redirect_from: /short-name/
---
```

Give every post a short address with `redirect_from`: one to three words, lowercase, hyphenated, ending in `/`, and unique across posts. It redirects to the dated URL (via `jekyll-redirect-from`, which GitHub Pages ships), so `billykong.github.io/short-name/` fits where long links get cut off, such as YouTube descriptions. Use the short address when sharing; keep the dated URL as the canonical one. Existing short addresses: `/safe-ops-lessons/`, `/defence-in-depth/`, `/planproof/`.

Add `mermaid: true` to render ```` ```mermaid ```` blocks.

## Preview locally

With Docker, no Ruby needed. Run it from this repo's folder (it mounts `$PWD`):

```bash
docker run --rm -it -p 4000:4000 -u "$(id -u):$(id -g)" -e HOME=/tmp -e BUNDLE_PATH=/tmp/bundle \
  -v "$PWD":/site -w /site ruby:3.3 \
  sh -c 'bundle install && bundle exec jekyll serve --host 0.0.0.0'
```

`-u` keeps `_site` owned by you rather than root; gems install inside the container, so each run reinstalls them (about a minute).

Then open http://localhost:4000. With Ruby installed: `bundle install && bundle exec jekyll serve`.

## Override theme files

Copy the file from the [minima](https://github.com/jekyll/minima) version GitHub Pages uses (`bundle info minima`) into the same path here, and edit the copy.
