# cybernook

A minimalist, dark-mode-first Jekyll blog for security research, blue-team notes
and CTF writeups. Built as a single self-contained HTML/CSS base layout that
fully overrides any Jekyll theme — no theme gem required.

## Structure

```
_config.yml            Site identity, social links, collections
_layouts/
  default.html         Sidebar + theme toggle + search/filter JS (all CSS lives here)
  post.html            Post article layout
  writeup.html         Writeup article layout
index.html             Home → Posts listing
writeups/index.html    Writeups listing
presentations/         "Under construction" page
about/                 About page
_posts/                Blog posts (Markdown + front matter)
_writeups/             Writeups (Markdown + front matter)
assets/img/            Images referenced by posts
```

## Editing your identity

Everything brandable lives in `_config.yml`:

- `title`, `description`
- `author.name`, `author.tagline`
- `social.linkedin / github / hackthebox / discord`

## Adding a post

Create `_posts/YYYY-MM-DD-my-title.md`:

```yaml
---
title: "My Post Title"
date: 2026-06-01
read_time: 5          # minutes, shown in the listing
tags: [blue team, windows]   # free-form; "blue team"/"red team" get team colors
description: "Up to two lines shown in the listing."
---

Markdown body here…
```

## Adding a writeup

Create `_writeups/YYYY-MM-DD-machine.md`:

```yaml
---
title: "Machine Name"
date: 2026-06-01
read_time: 10
difficulty: easy      # easy | medium | hard | insane → colored like HTB
platform: HackTheBox  # optional
os: Linux             # optional
tags: [web, privesc]
description: "Up to two lines shown in the listing."
---
```

## Tags & search

Tags are **not hardcoded**. Each listing page collects every tag from the front
matter of its content at build time and renders them as clickable chips under
the search bar. Add a new tag to any post and it appears automatically.

The search bar and the chips work together: typing filters by title *and* tag,
clicking chips adds tag filters, and the two combine.

## Local development

```bash
bundle install
bundle exec jekyll serve
```

Then open <http://localhost:4000/>.

## Deploying to GitHub Pages

This is configured as a **user site** served at the domain root, so the repo
must be named exactly `serg-ux.github.io` and `baseurl` is `""`. Push to `main`,
then in **Settings → Pages** set the source to **Deploy from a branch** →
`main` / `(root)`. The site publishes at `https://serg-ux.github.io/`.

If you ever move this to a normal project repo instead, set
`baseurl: "/<repo-name>"` so links and styles resolve correctly.
