---
nav_exclude: true
---

# ARTEX Field Guide — repo & publishing

This repository is a [just-the-docs](https://just-the-docs.com/) site documenting how to run
**[ARTEX](https://github.com/BaaRai/ARTEX)** (a self-hosted, LLM-driven autonomous penetration-testing
platform) against a **local, deliberately-vulnerable lab** — for authorized security education only.

The content lives in the Markdown files at the repo root (`index.md`, `setup.md`, `models.md`, …).

## Publish to GitHub Pages

1. Create a new **empty** repo on GitHub (e.g. `artex-guide`).
2. Push this directory to it:
   ```bash
   git add -A
   git commit -m "docs: ARTEX field guide"
   git branch -M main
   git remote add origin git@github.com:<your-user>/artex-guide.git
   git push -u origin main
   ```
3. In the repo on GitHub: **Settings → Pages → Build and deployment → Source: Deploy from a branch**,
   branch **`main`**, folder **`/ (root)`**. Save.
4. Wait ~1 minute. Your site is live at `https://<your-user>.github.io/artex-guide/`.

No local build is required — GitHub Pages builds the just-the-docs `remote_theme` for you.

## Preview locally (optional)

```bash
bundle install
bundle exec jekyll serve   # http://127.0.0.1:4000
```

## Editing

Each page is a plain Markdown file with a small front-matter block setting its `title` and `nav_order`.
Add a page by copying that front matter into a new `.md` file.

> **Scope note:** everything here targets a **local, isolated, intentionally-vulnerable** application
> (OWASP Juice Shop / WebGoat) that you control. Only test systems you are authorized to test.
