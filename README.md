# Alexis Notes

Personal blog by alexis, based on [AstroPaper](https://github.com/satnaing/astro-paper) (MIT licensed). Published as a static-assets Cloudflare Worker at <https://blog.diaodaquan.dpdns.org/>.

## Local development

Requires Node.js 22.12+ and pnpm.

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

## Deploy

```sh
pnpm deploy
```

`pnpm deploy` runs the Astro build, Pagefind indexing, and Wrangler deployment. Wrangler authentication is required (`pnpm exec wrangler login`). The Worker, static asset directory, and custom domain are configured in `wrangler.jsonc`. The DNS zone must be managed in the Cloudflare account used for deployment.

The font files in `src/assets/fonts/` are bundled locally, so the build does not require access to Google Fonts. Their license is in `src/assets/fonts/OFL.txt`.

## GitHub

Repository: <https://github.com/Alexis374/blog>. Push changes with `git push origin main`.
