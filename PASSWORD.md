# Password & Auth — Quick Reference

This file is for **future me** when I've forgotten how the admin password works.

---

## TL;DR

- The admin uses a **password gate** to keep casual visitors out
- The password is stored as a **hash** (SHA-256), not as plain text
- To rotate the password, **change it through the Settings UI** in the admin — not by editing the code
- The real security is **Cloudflare Access** in front of `admin.html`, not the password itself

---

## What's a hash, and why hash the password?

A **hash** is a one-way fingerprint of a piece of text. Same input → same hash, every time. But you can't go backwards from the hash to the original text.

Example: SHA-256 of `Nabie2026!` is:
```
2d87fdf1f98053ab3500882892b681a120e6ee7e8a7732f0bbba5cd997d53fbf
```

That string above is 100% reversible-resistant. Someone who finds it can't recover `Nabie2026!` from it — unless they guess the password and try the hash. (Which works for weak/common passwords. So use a strong password.)

**Why hash matters here**: my admin's source code is on GitHub. Anyone can read it. If the source said `const PWD = 'Nabie2026!'`, anyone who clones the repo would know my password. By storing the hash instead, the source reveals nothing useful to a casual reader.

Also: when I change the password through the UI, the new password gets hashed before it touches `localStorage`. So even if someone opens DevTools on my browser, they see a 64-character hex string, not my real password.

**What hash does NOT do**: it does not make the admin secure. Someone determined who reads the source can:
- Try every common password against the hash (offline brute force)
- Use DevTools to set their own hash in `localStorage`
- Flip the `loggedIn` flag directly

This is why Cloudflare Access matters (see below).

---

## How to change the admin password

The right way: **don't edit the code.** Use the Settings UI.

1. Go to `admin.html`, log in
2. **Sidebar → Réglages**
3. Section **Mot de passe** → type new password (twice) → click **Mettre à jour le mot de passe**
4. Next login uses the new password. The old one stops working immediately.

The new password gets hashed in the browser before it's saved. The hash lives in `localStorage` under `nb_admin_cfg.passwordHash`. The plain text never leaves the form.

### If I lock myself out

I can reset to the default password (`Nabie2026!`) by clearing localStorage:

1. Open `admin.html`
2. Press F12 → **Application** tab → **Local Storage** → my site
3. Find `nb_admin_cfg`, right-click → **Delete**
4. Refresh the page. Default password (`Nabie2026!`) works again.

This works because clearing the config wipes the custom passwordHash, so the admin falls back to the default hash that's compiled into the source.

---

## How to rotate the DEFAULT password (the one in the source)

This is rare — I only do this if `Nabie2026!` has been exposed somewhere I care about (e.g. I committed it to a public repo and want to retire it).

### Step 1 — Pick a new default password

Something long and not memorable. Doesn't matter if I forget it — I'll be changing it in the UI right after deploying anyway. Use a password manager to generate one, or just mash the keyboard.

Example: `Nabie-Defaults-Are-Boring-9X7q`

### Step 2 — Generate its SHA-256 hash

**Windows (PowerShell):**
```powershell
$pw = "Nabie-Defaults-Are-Boring-9X7q"
$bytes = [System.Text.Encoding]::UTF8.GetBytes($pw)
$hash = [System.Security.Cryptography.SHA256]::Create().ComputeHash($bytes)
[System.BitConverter]::ToString($hash).Replace("-","").ToLower()
```

The output is a 64-character hex string.

**Mac/Linux:**
```bash
echo -n 'Nabie-Defaults-Are-Boring-9X7q' | shasum -a 256
```
The output looks like `<hash>  -`. Use just the hash part.

**Online (if I don't trust my terminal):**
- Search "SHA-256 generator" — don't pick the first result blindly, use something reputable like emn178.github.io/online-tools/sha256.html
- Paste the password, copy the hex output
- Note: typing a real password into a random website is a tiny trust leak. Fine for a *default* password (which I'll change immediately anyway). Not fine for a real production password.

### Step 3 — Update `admin.html`

Find this line near the top of the `<script>` section:
```javascript
const DEFAULT_PWD_HASH = '2d87fdf1f98053ab3500882892b681a120e6ee7e8a7732f0bbba5cd997d53fbf';
```

Replace the 64-character hex string with my new hash. Save. Push to GitHub.

### Step 4 — Log in with the new default

Open `admin.html`, type the new default password (`Nabie-Defaults-Are-Boring-9X7q`), log in.

### Step 5 — Immediately set a custom password through the UI

Réglages → Mot de passe → type a memorable strong password I'll actually use day-to-day → save.

Now: the source has a useless hash (the default that nobody will ever type), and my real password lives only as a hash in *my* browser's localStorage. Nobody else has access to either piece.

---

## The real security: Cloudflare Access

The password is one factor. By itself, it's not enough — the admin is on a public URL and the password gate runs in the browser, which means a determined attacker can bypass it.

**The real fix**: put Cloudflare Access in front of `admin.html`. This is a free service that requires anyone visiting that URL to:

1. Prove they own a specific email (Cloudflare sends a one-time code)
2. *Then* see my admin's password gate

Two factors. Real security. Free up to 50 users.

Setup steps (~30 min, one-time):

1. Buy a domain if I don't have one (~$10/year). The blog at `username.github.io` cannot be protected by Cloudflare directly — I need a custom domain.
2. Sign up at cloudflare.com → **Add a site** → enter my domain
3. Point my domain's nameservers to Cloudflare (one-time DNS change at my registrar)
4. Point Cloudflare DNS to GitHub Pages (a CNAME record — see GitHub Pages docs)
5. In Cloudflare dashboard → **Zero Trust** → pick a team name → Free plan
6. **Access → Applications → Add application** → Self-hosted
7. Application domain: `nabintousfofana.com`, path: `Mon-journal/admin.html`
8. Policy: Allow, Include → Emails → my email
9. Save

After this:
- Visiting `nabintousfofana.com/Mon-journal/admin.html` → Cloudflare email gate
- Enter the 6-digit code → see my familiar admin password gate
- Enter password → I'm in
- Visiting `nabintousfofana.com/Mon-journal/` (the blog) → normal, no gate, public

---

## Things to remember

- **The default password `Nabie2026!` is in this repo's commit history.** That means even if I rotate the default in the source today, anyone who looked at the repo earlier knows the original. This is why the hash-in-source approach is only step 1 — Cloudflare Access is step 2.
- **Custom passwords are per-browser.** If I set a custom password on my laptop, it doesn't sync to my phone. The phone still uses the default hash (or whatever was set on the phone last). That's how localStorage works.
- **The "logged in" state is also per-browser.** Login on laptop, log in separately on phone. There's no real account.
- **Subscriber emails and message inbox live in localStorage too.** They're not encrypted. If I ever want to share my browser with someone, export the data first.

---

## Files in this repo related to security

- `admin.html` — has the password gate, hashing logic, idle timeout, brute-force lockout
- `SECURITY.md` — the honest write-up of what the admin protects against and what it doesn't
- `PASSWORD.md` — this file
- `LICENSE` — content/code restrictions

---

## When something feels off

Common situations and what to do:

| Situation | What to do |
|---|---|
| I forgot my custom password | Clear `nb_admin_cfg` in DevTools → log in with default (`Nabie2026!`) → set a new custom password |
| Login screen says "Trop d'essais" | Wait 60 seconds. The lockout resets automatically. |
| Login worked yesterday, fails today | Either: I'm using the wrong browser (custom pw is per-browser), or my session expired (1h idle timeout — just log in again) |
| Someone says "I can see your password in the source!" | They see the hash, not the password. As long as my password is strong, this is fine. |
| I want to revoke a password I've used elsewhere | Set a new custom one through Settings. The old hash is overwritten immediately. |
| I'm publishing this repo for the first time | Make sure the default password in the source is something I'm OK with the world knowing, then change it via the UI immediately on the live site. |

---

Last updated: 2026
Maintained by: me, for me
