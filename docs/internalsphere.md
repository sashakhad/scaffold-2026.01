# internalsphere — Internal Apps Platform Guide

> **Local copy captured:** 2026-06-30
> **Canonical source (Notion):** https://www.notion.so/cursorai/internalsphere-348da74ef0458184af80e67adbcef6b7
>
> This is a point-in-time, offline copy for quick reference. **The Notion doc is the source of
> truth** — if anything here looks stale, check Notion and update this file (and bump the date above).
> On a live internalsphere repo, the orchestrator also ships `@internalsphere-setup` and
> `QUICKSTART.md`, which stay in sync automatically.

This guide is for anyone building internal apps at Cursor. It explains what internalsphere is, how to
get set up, how to ship changes, and answers the questions people ask most often.

---

## What is internalsphere?

**internalsphere** is the home for internal apps — little tools and dashboards that only people
inside Cursor use. It's a GitHub organization ([github.com/internalsphere](https://github.com/internalsphere))
plus a Vercel team (`anysphere-internal`) wired together so anyone can ship a real web app without
having to understand hosting, secrets management, CI/CD, or security on day one.

This is made possible by an orchestrator repo called `internal-app-orchestrator` (also called the
**"ranger"** or **"control plane"**). It watches every internal-app repo and keeps everything set up
correctly:

- Creates and links the Vercel project for you.
- Configures environment variables and secrets.
- Sets up branch protection, CODEOWNERS, and security checks.
- Deploys to **preview** on every PR and to **production** on every merge to the default branch
  (`main`).
- If any of the above drifts out of policy, it opens a PR to fix it automatically (auto-admin-merged
  by the `internalsphere-ranger` bot).

> **Mental model:** you write app code; the orchestrator handles everything else around it. You
> almost **never** edit Vercel settings, GitHub settings, or workflow files directly.

---

## First-time setup (once per laptop)

### 1. Join the GitHub org

Search Okta for `Github - Internalsphere`. If you don't have access, ping `#proj-internalsphere`.

### 2. Authorize your SSH key for internalsphere (most common setup error)

`anysphere` and `internalsphere` are separate GitHub orgs, so SSH access must be SSO-authorized for
each one. If you already use GitHub for `anysphere`, you almost certainly have a working key:

1. Go to https://github.com/settings/keys
2. Find your existing SSH key.
3. Click **Configure SSO** → enable it for `internalsphere`.
4. Test: `ssh -T git@github.com` → should say "Hi \<your-username\>!"

> If `git clone` fails with "Permission denied (publickey)" or "Repository not found" on an
> internalsphere repo, **99% of the time this step is the fix.**

### 3. Install the basics

```bash
brew install git gh sops age python pnpm node
gh auth login
```

- `git`, `gh` — version control + GitHub CLI
- `sops` + `age` — encrypt secrets before they're committed to git
- `python`, `pnpm`, `node` — for running repo scripts

### 4. Vercel access (optional)

Search Okta for `Vercel-Internal`. You'll have **Viewer** access — enough to view deploys, but **not**
enough to view env var names/values or run `vercel env pull` / `vercel dev`. Use the PR preview flow
instead.

---

## Creating a new app

1. Create an **empty** repo under the **`internalsphere`** org
   ([new repo](https://github.com/organizations/internalsphere/repositories/new)).
   - Pick a clear name like `my-cool-dashboard`.
   - Make it **Private** or **Internal**.
   - Use `main` as the branch name.
2. **Wait a minute or two.** The orchestrator picks up the new repo and bootstraps it (Vercel project,
   branch protection, CODEOWNERS, security rulesets, `app-manifest.yml`, `.sops.yaml`, `secrets/`, CI
   workflows, Cursor skills), then opens and auto-merges a bootstrap PR.
3. **Now** start building: clone, run `sh scripts/setup-repo.sh` (one-time), and push your app code
   via a PR.

> **TL;DR:** Create an empty private repo in `internalsphere` with a `main` branch, wait ~1 minute for
> the bootstrap PR to merge, then start pushing code. You **do not** scaffold platform files by hand —
> the bootstrap PR does it for you.

> ⚠️ **Do not push a full app scaffold on top of a fresh repo before the orchestrator bootstraps it.**
> The baseline commit on `main` will diverge from your push, and a hand-written `.github/workflows/ci.yml`
> will fail the `ci-required` policy check. Let the bootstrap land first, then add app code via a PR.

---

## Making changes to an existing app

1. Clone (once): `git clone git@github.com:internalsphere/<repo-name>.git && cd <repo-name>`
2. One-time per repo: `sh scripts/setup-repo.sh` (installs git hooks + local deps)
3. Branch: `git checkout -b my-change`
4. Edit, commit, push.
5. Open a PR. You'll see:
   - A **Vercel preview URL** (posted by the `internalsphere-ranger` bot).
   - **Bugbot** and **Security Bugbot** review comments.
   - CI checks (lint, secret scan, etc.).
6. Merge when green → the orchestrator deploys to production automatically.

> 🤖 If two preview-URL comments show up, use the one from the **`internalsphere-ranger`** bot — ignore
> the `Vercel` bot's.

---

## Deploying your changes

**You do not run any deploy commands. Pushing to GitHub *is* deploying.**

1. **Open or push to a PR** → Vercel publishes a **preview deploy** at a unique URL (same URL updates
   on every push to the PR).
2. **Merge the PR into `main`** → Vercel publishes a **production deploy**.

> 🚫 You can't `vercel deploy` from your laptop, and you shouldn't try. Everyone on `anysphere-internal`
> has **Viewer** access (can't deploy). Deploys must go through the managed GitHub Actions workflow so
> `secrets/` get decrypted, env vars get synced, and the audit inventory gets written.

---

## Environment variables and secrets

All env vars — sensitive or not — live in the repo **encrypted** with SOPS + age, under
`secrets/shared/`, `secrets/preview/`, or `secrets/production/`. CI decrypts them at build time and
ships them to Vercel as sensitive env vars.

```bash
python3 scripts/secrets.py list   --scope local
python3 scripts/secrets.py list   --scope remote --env production
python3 scripts/secrets.py add    --scope production --key MY_SECRET_NAME
python3 scripts/secrets.py update --scope production --key MY_SECRET_NAME
python3 scripts/secrets.py delete --scope production --key MY_SECRET_NAME
```

Scopes: `shared` (all envs), `preview`, `production`. Key names must match `^[A-Z][A-Z0-9_]*$`. Commit
the resulting `secrets/<scope>/<KEY>.sops.json`, open a PR, merge — CI syncs to Vercel.

> 🚫 **Never** paste a raw secret into `.env`, code, tests, commit messages, or `vercel env add`. The
> `secrets-guard.py` pre-commit hook, CI scanning, and Security Bugbot all catch it.

---

## Integrations (databases, caches, storage)

Declare integrations in `app-manifest.yml` under `integrations:`. Each entry is keyed by an **alias you
choose** and has a single required field, `type`:

```yaml
version: 1
integrations:
  db:
    type: supabase
  cache:
    type: upstash-kv
  assets:
    type: blob
```

`version: 1` is the **manifest schema version**, not a per-change revision — leave it at `1`.

Your alias becomes the env var prefix Vercel generates:

| Alias | Type | Example env vars |
| --- | --- | --- |
| `db` | `supabase` | `DB_POSTGRES_URL`, `DB_SUPABASE_URL`, `DB_SUPABASE_SERVICE_ROLE_KEY`, … |
| `cache` | `upstash-kv` | `CACHE_KV_REST_API_URL`, `CACHE_KV_REST_API_TOKEN` |
| `assets` | `blob` | `ASSETS_READ_WRITE_TOKEN` |

Open a PR and merge. The orchestrator provisions the backing resource and injects credentials into
Vercel. Per-integration Cursor skills ship in each repo: `supabase-database`, `upstash-redis`,
`vercel-blob-store`.

---

## Sharing with external viewers

To share a deployment with someone outside Cursor, add a time-bound grant to `external-viewers.yml`
at the repo root:

```yaml
version: 1
viewers:
  - target: my-app-preview.vercel.app
    email: partner@example.com
    expires_at: "2026-06-19T00:00:00Z"
    requested_by: "@you"
    reason: "Customer pitch review"
```

Open a PR — `@internalsphere/security` is tagged automatically. After merge, the orchestrator grants
access in Vercel and revokes it at `expires_at` or when the entry is removed. Don't make a project
public to share it.

---

## Why is my deployed app showing a 404?

The deploy went green but the page says "Not Found". Canonical checklist:
https://vercel.com/kb/guide/why-is-my-deployed-project-giving-404

Most common causes:

1. **Wrong URL.** A project's default hostname isn't always predictable. Confirm the canonical URL from
   the project's Deployments tab at `https://vercel.com/anysphere-internal/<PROJECT>/deployments`.
2. **Wrong framework preset.** For Next.js, Vercel must use the Next.js adapter. If the dashboard shows
   **Framework Preset = Other**, pin it from the repo with `vercel.json`:
   ```json
   { "framework": "nextjs" }
   ```
   On internalsphere, the orchestrator's `vercel.json` ships `{"git": {"deploymentEnabled": false}}` —
   **add** `"framework": "nextjs"` to it; don't remove the `git` key (that would re-enable Vercel's git
   integration and cause double deploys). Do **not** point `outputDirectory` at `.next` for a
   server-rendered app.
   > **This was the exact bug that 404'd our first internalsphere app.** The scaffold now ships a
   > `vercel.json` with `framework: nextjs` by default so it doesn't recur.
3. **Wrong Output Directory.** Use `"dist"`/`"public"` only when your framework really emits static
   files there. For Next.js, prefer the framework preset and omit `outputDirectory`.
4. **Wrong Root Directory** (monorepos). If build logs show `next build` at the repo root but your app
   lives in a subfolder, set `vercel.root_directory` in `app-manifest.yml` (not a `vercel.json` knob).
5. **SPA missing a rewrite.** Vite/CRA apps need a catch-all:
   ```json
   { "rewrites": [{ "source": "/(.*)", "destination": "/index.html" }] }
   ```

Still stuck? Check the dashboard's **Build Logs** and **Runtime Logs**, or ask in
`#proj-internalsphere`.

---

## FAQ highlights

- **Who can access my app?** SSO-protected to Cursor employees by default. Share externally via
  `external-viewers.yml`.
- **How do deploys happen?** Open a PR (preview) and merge into `main` (production). No local deploy
  commands.
- **Find my deployments:** https://vercel.com/anysphere-internal → click the project → **Deployments**
  (top Production entry has the canonical URL).
- **Can I run `vercel env pull` / `vercel dev` / `vercel deploy`?** No — Viewer access only. Iterate via
  PR previews.
- **Why can't I force-merge into `main`?** It's protected (PR + passing CI + CODEOWNER approval for
  managed files). Only the `internalsphere-ranger` bot admin-merges its own baseline PRs.
- **Roll back a bad deploy:** Revert the PR on GitHub and merge the revert.

---

## Guardrails (don't fight the orchestrator)

- **Never edit policy-managed files** — they're overwritten on every reconciliation:
  managed workflows (`.github/workflows/managed-app.yml` etc.), `CODEOWNERS`, `.sops.yaml`, git hooks,
  `scripts/setup-repo.sh`, `scripts/install-secrets-tooling.sh`, `scripts/app-manifest.py`,
  `scripts/secrets-guard.py`, `scripts/secrets.py`, distributed skill files, `secrets/inventory.yaml`,
  and `QUICKSTART.md`. App-specific toggles go in `app-manifest.yml`.
- **Don't add your own CI** (`.github/workflows/ci.yml`) — CI/deploy run through orchestrator reusable
  workflows; a custom one fails the `ci-required` policy check.
- **Commit `pnpm-lock.yaml`** — CI installs with `--frozen-lockfile`.
- **Don't bypass hooks** with `--no-verify` — CI catches the same issues.

---

## Where to get help

- **Slack:** `#proj-internalsphere`
- **Full guide (source of truth):** https://www.notion.so/cursorai/internalsphere-348da74ef0458184af80e67adbcef6b7
- **Orchestrator repo:** https://github.com/internalsphere/internal-app-orchestrator
- **On a live repo:** `@internalsphere-setup` (Cursor skill) and `QUICKSTART.md`
