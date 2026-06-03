# Journal de Nabie

A personal blog in French for Muslim women juggling faith, studies, work,
side projects, and inner healing.

It started as a place I wanted to read, so I built it.

## What's in this repo

- **`index.html`** — the public blog. Hero, featured articles, four
  categories (*Foi · Lifestyle · Carrière & Tech · Santé mentale*),
  newsletter signup, contact form. Multiple visual themes (default,
  forest, twilight, paris) — picked from the admin, no rebuild needed.
- **`admin.html`** — a private CMS where I write, edit, and publish
  articles from the browser. Autosave, version history (5 per article),
  themes, custom logo upload, draft restoration, message inbox,
  subscriber list.
- **`SECURITY.md`** — an honest write-up of what the admin protects
  against and (more importantly) what it doesn't. Read this before
  using this pattern for anything real.
- **`LICENSE`** — All rights reserved. See file.

## Two files, no framework, no backend

The site is two HTML files. Everything else lives in
`localStorage`. No build step, no Node, no database, no deploy
pipeline. Open `index.html` and it works.

This is intentional. The blog is small. A static site can be hosted
anywhere for free, edited from any browser, and never breaks because
of a dependency upgrade. The tradeoff is that there's no real
authentication — see `SECURITY.md`.

## How the admin works

The admin password gate is a UI convenience, not a security boundary.
Articles, themes, subscriber emails, message inbox — all of it lives
in the browser's `localStorage` on whatever machine I'm using. There
is no server.

This means:

- Logging in on my phone doesn't show me my laptop's drafts.
- I export articles as JSON if I want them backed up off this device.
- Anyone with the password can edit. Anyone with browser DevTools can
  bypass the password.

That's fine for a personal blog where my voice is the moat. For
anything with subscriber data I'd hate to leak, I'd add a real
backend — `SECURITY.md` covers the options I'd reach for.

## Themes

`index.html` reads `localStorage.nb_blog_theme` before first paint, so
there's no flash of default theme. Admin "Preview" sends a
`?theme=<id>` query parameter so I can see a theme without committing
to it.

## What's mine vs what you can borrow

The code is mine, but most of the patterns are well-known
(`localStorage` CRUD, theme tokens via `data-theme`, autosave with
versioning). Look at it, learn from it, build something different
for yourself. The articles are personal — see `LICENSE`.

## Roadmap

Things I might add when life is calmer:

- Real authentication with email magic-link (Netlify Identity or
  Supabase). See `SECURITY.md` for the migration plan.
- A subscribe-by-email flow that actually emails subscribers
  (right now I just collect addresses).
- An export-to-Markdown button so articles can be moved to a real
  static-site generator later if needed.

---

[Nabintou S. Fofana](https://github.com/NabintouSFofana) · 2024–present
