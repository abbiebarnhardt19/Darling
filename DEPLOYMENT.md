# Deployment & Security Guide

This project (**Darling**) is a static website (`index.html`, `styles.css`, plus
page files) that talks to a [Supabase](https://supabase.com) backend from the
browser. This guide covers two things:

1. **Hosting it for free using GitHub** (GitHub Pages).
2. **Handling configuration/secrets safely** with a `.env`-style pattern and
   `.gitignore`.

---

## 1. Get a hosted server using GitHub (GitHub Pages)

GitHub can host static sites (HTML/CSS/JS) for free with **GitHub Pages**.
There is no separate server to rent — GitHub serves your files over HTTPS and
gives you a public URL like `https://<username>.github.io/<repo>/`.

> Pages only serves **static files**. It does not run server-side code. That's
> fine here because Darling does all its dynamic work through Supabase from the
> browser. (See the security note in section 2 about what that means for keys.)

### Steps

1. Push your site to the `main` branch of this repo (it already lives there).
2. On GitHub, open the repo and go to **Settings → Pages**.
3. Under **Build and deployment → Source**, choose **Deploy from a branch**.
4. Set **Branch** to `main` and the folder to `/ (root)`, then click **Save**.
5. Wait ~1–2 minutes. Refresh the Pages settings page and you'll see:
   *"Your site is live at `https://<username>.github.io/Darling/`."*
6. Open that URL to confirm the site loads.

### Useful extras

- **Custom domain:** In **Settings → Pages → Custom domain**, enter your domain
  (e.g. `abbieandben.com`), save, and add the DNS records GitHub shows you at
  your domain registrar. Then tick **Enforce HTTPS**.
- **Auto-redeploy:** Every push to `main` automatically rebuilds and republishes
  the site. No manual deploy needed.
- **Check status:** The **Actions** tab shows each Pages deployment and any
  errors.

---

## 2. Configuration & secrets — `.env`, `config.js`, and `.gitignore`

### The most important thing to understand first

This is a **static, browser-only** site. Any file the browser loads
(`config.js`, JavaScript, etc.) is **fully visible to anyone** who visits the
site — "View Source" reveals it. **You cannot hide a real secret in a static
front-end.** A `.env` file does *not* make values secret on a static site; it
only keeps them out of your Git history.

Because of that, the rules are:

- ✅ The Supabase **anon / public key** is *designed* to be public. It is safe to
  ship to the browser **as long as you turn on Row Level Security (RLS)** on
  every table in Supabase. RLS is what actually protects your data.
- ❌ **Never** put the Supabase **`service_role` key** (or any private API key,
  password, or token) in front-end code. It bypasses RLS and would expose all
  your data. Service-role keys belong only on a real backend / serverless
  function.

### How this repo is set up

The site reads its Supabase values from a `config.js` file that is **not
committed** to Git:

```html
<script src="https://cdn.jsdelivr.net/npm/@supabase/supabase-js@2"></script>
<script src="config.js"></script>
<script>
  const sb = supabase.createClient(SUPABASE_URL, SUPABASE_ANON_KEY);
</script>
```

A committed template, `config.example.js`, shows what `config.js` should
contain. To run the site:

```bash
cp config.example.js config.js
# then edit config.js and paste in your real Supabase URL + anon key
```

`config.js` is listed in `.gitignore`, so your local copy is never pushed.

> ⚠️ **GitHub Pages caveat:** because `config.js` is gitignored, it will **not**
> be deployed to Pages, so the live site won't have its Supabase config. Pick
> one of these options for the live site:
>
> - **Option A (simplest, recommended for this project):** Since the anon key is
>   public anyway, commit a `config.js` with the *anon* key directly. The
>   protection comes from RLS, not from hiding the key. To do this, remove
>   `config.js` from `.gitignore` and commit it.
> - **Option B (keep it out of Git):** Use a GitHub Actions workflow that writes
>   `config.js` from a repo **Secret** during deploy (see below). Use this if
>   you'd rather not have the key in the repo history at all.

### Using a real `.env` file (for any backend / build step you add later)

If you later add a Node/serverless backend or a build tool (Vite, etc.), use a
proper `.env` file for real secrets:

1. Create `.env` in the project root:

   ```dotenv
   SUPABASE_URL=https://your-project-ref.supabase.co
   SUPABASE_ANON_KEY=your-public-anon-key
   # Backend-only — NEVER expose to the browser:
   SUPABASE_SERVICE_ROLE_KEY=your-secret-service-role-key
   ```

2. Make sure `.env` is in `.gitignore` (it already is in this repo).
3. Commit a `.env.example` with **placeholder** values so teammates know which
   variables are needed, without leaking real ones.
4. Load it server-side (e.g. `require('dotenv').config()` in Node).

### The `.gitignore`

This repo includes a `.gitignore` that keeps secrets and local files out of Git:

```gitignore
.env
.env.*
!.env.example
config.js          # Supabase URL + anon key for this site
.DS_Store
node_modules/
```

If you ever committed a secret by accident, **rotate it immediately** (generate
a new key in the Supabase dashboard) — removing it from a later commit does not
remove it from Git history.

### Optional: inject `config.js` at deploy time with GitHub Actions (Option B)

1. In GitHub: **Settings → Secrets and variables → Actions → New repository
   secret.** Add `SUPABASE_URL` and `SUPABASE_ANON_KEY`.
2. Add `.github/workflows/deploy.yml`:

   ```yaml
   name: Deploy to GitHub Pages
   on:
     push:
       branches: [main]
   permissions:
     contents: read
     pages: write
     id-token: write
   jobs:
     deploy:
       runs-on: ubuntu-latest
       environment:
         name: github-pages
         url: ${{ steps.deployment.outputs.page_url }}
       steps:
         - uses: actions/checkout@v4
         - name: Generate config.js from secrets
           run: |
             cat > config.js <<EOF
             const SUPABASE_URL = "${{ secrets.SUPABASE_URL }}";
             const SUPABASE_ANON_KEY = "${{ secrets.SUPABASE_ANON_KEY }}";
             EOF
         - uses: actions/upload-pages-artifact@v3
           with:
             path: .
         - id: deployment
           uses: actions/deploy-pages@v4
   ```

3. In **Settings → Pages → Source**, choose **GitHub Actions**. Now each push to
   `main` rebuilds `config.js` from your Secrets and deploys.

---

## Quick checklist

- [ ] GitHub Pages enabled (Settings → Pages → `main` / root).
- [ ] Site loads at `https://<username>.github.io/Darling/`.
- [ ] `.gitignore` present and excludes `.env` and `config.js`.
- [ ] `config.example.js` committed; real `config.js` created locally.
- [ ] Row Level Security **enabled** on all Supabase tables.
- [ ] No `service_role` key anywhere in front-end code.
