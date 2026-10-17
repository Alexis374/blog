# Alexis Notes

Personal blog by alexis, based on [AstroPaper](https://github.com/satnaing/astro-paper) (MIT licensed). Published as a static-assets Cloudflare Worker at <https://blog.diaodaquan.dpdns.org/>.

## Local development

Requires Node.js 22.12+ and pnpm. Cloudflare builds use Node 24.18.0 (`.node-version`) and pnpm 11.24.0 (`packageManager` in `package.json`).

```sh
pnpm install --frozen-lockfile
pnpm dev
```

## Write a post

Add a Markdown or MDX file in `src/content/posts/`, for example `hello-world.md`:

```md
---
title: "Hello world"
pubDatetime: 2026-10-17T00:00:00+08:00
description: "My first post."
tags: [notes]
---

Hello world!
```

See `src/content.config.ts` for the complete frontmatter schema. Edit `src/content/pages/about.md` for the About page and `astro-paper.config.ts` for site metadata and features.

## Publish: Git push → Cloudflare Workers Builds

After writing a post, **do not build locally**. Commit and push the source to `main`; the connected Cloudflare pipeline builds and deploys it:

```sh
git add src/content/posts/
git commit -m "Publish new post"
git push origin main
```

The remote is HTTPS; if GitHub HTTPS authentication is not configured but your SSH key is authorized, push without changing the saved remote URL using `git -c url.git@github.com:.insteadOf=https://github.com/ push origin main`.

Connect the **existing** Worker `alexis-notes` (do not create a second Worker) to <https://github.com/Alexis374/blog> in Cloudflare **Workers & Pages → alexis-notes → Settings → Builds → Connect**. Configure:

| Setting | Value |
| --- | --- |
| Production branch | `main` |
| Root directory | `/` (repository root) |
| Build command | leave empty (as in `bill-admin`) |
| Deploy command | `pnpm deploy` |
| Build variable, if the image does not use `packageManager` | `PNPM_VERSION=11.24.0` |

Cloudflare auto-installs dependencies and then `pnpm deploy` performs Astro type-check/build, Pagefind indexing and `wrangler deploy` on Cloudflare's builder. `wrangler.jsonc` keeps the existing Worker name, `dist` static assets and custom domain. Do not configure `pnpm exec wrangler deploy` *without* a prior build: `dist/` is Git-ignored and does not exist in a fresh checkout. Monitor build history in the Worker's **Deployments** tab; a failed build does not publish the new article.

`pnpm deploy` can still be used manually for troubleshooting, but is **not** the normal publishing flow. Google Sans Code fonts are bundled in `src/assets/fonts/`, so builds do not need to access Google Fonts; the license is in `src/assets/fonts/OFL.txt`.

Repository: <https://github.com/Alexis374/blog>.
