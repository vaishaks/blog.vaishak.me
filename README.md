# blog.vaishak.me

Personal tech blog by Vaishak Salin, built with [Astro Cactus](https://github.com/chrismwilliams/astro-theme-cactus).

## Local Development

```bash
pnpm install
pnpm dev        # starts dev server at localhost:4321
```

## Build

```bash
pnpm build      # builds to dist/
pnpm postbuild  # runs Pagefind indexing on dist/
```

## Cloudflare Pages Deployment

1. Connect the GitHub repo `vaishaks/blog.vaishak.me` to Cloudflare Pages
2. Use these build settings:
   - **Framework preset:** None (Astro)
   - **Build command:** `pnpm build && pnpm postbuild`
   - **Build output directory:** `dist`
   - **Node.js version:** `20`
3. DNS: Add a `CNAME` record for `blog` pointing to `<project>.pages.dev`
4. In Cloudflare Pages → Custom domains → add `blog.vaishak.me`
