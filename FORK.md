# Walkup HQ: fork notes

Walkup HQ is a fork of [Kinwall](https://github.com/JohnDuprey/kinwall). The app, its docs and its
license (AGPL-3.0-or-later) are upstream's; this file covers only what is specific to the fork.
Everything in [AGENTS.md](AGENTS.md) and [docs/contributing/development.md](docs/contributing/development.md)
still applies.

## What differs from upstream

Kept deliberately small so upstream merges stay easy:

- `wrangler.toml`: the Worker is named `walkup-hq` (upstream: `kinwall`), so the default URL is
  `https://walkup-hq.<your-subdomain>.workers.dev`.
- A fork notice at the top of `README.md`, a pointer in `AGENTS.md`, this file, and extra
  `.gitignore` entries for local secrets, keys and databases.

Everything else is upstream's, including the in-app "Kinwall" name, package names, the Docker
image name and the D1 database name (`kinwall`). See [Left as upstream](#left-as-upstream).

## Remotes

| Remote | URL | Use |
|---|---|---|
| `origin` | `https://github.com/dustinwalkup/walkup-hq` | Your fork. Push here. |
| `upstream` | `https://github.com/JohnDuprey/kinwall` | Fetch only. Pushing is disabled locally. |

A fresh clone needs `upstream` added again:

```bash
git remote add upstream https://github.com/JohnDuprey/kinwall.git
git remote set-url --push upstream DISABLED-no-push-to-upstream
```

Pull in upstream changes with a merge, which keeps upstream's history and never needs a force push:

```bash
git fetch upstream
git switch main
git merge upstream/main      # conflicts, if any, are in the fork-only lines listed above
scripts/check.sh
git push origin main
```

## Local development

No Cloudflare account needed. Use **Node 24**, the version CI and the Docker image use
(`nvm use 24`).

```bash
cd server && npm ci && npm run dev     # API on :8080, SQLite in server/data/
cd web && npm ci && npm run dev        # UI on :5173, proxies /api to :8080
cd web && VITE_MOCK=1 npm run dev      # UI with fake data, no server
```

The same app on the Workers runtime, with a local D1 database under `.wrangler/`:

```bash
printf 'ENCRYPTION_KEY=%s\n' "$(openssl rand -base64 32)" > .dev.vars   # once; gitignored
cd server && npm run dev:worker        # builds the UI, then serves UI + API on :8787
```

Open the UI and enter the setup code printed in the terminal. Before pushing, run every check:

```bash
scripts/check.sh
```

## Deploy to Cloudflare Workers

Workers is the recommended path: it's free for a family, and the `workers.dev` URL has HTTPS from the
start, which passkeys and Google/Microsoft calendar sign-in need. You need a Cloudflare account and
the dependencies installed (above).

### First deploy

```bash
node scripts/setup-cloudflare.mjs --dry-run   # prints every step, changes nothing
node scripts/setup-cloudflare.mjs             # does it
```

The script is upstream's, documented in
[docs/getting-started/deploy-cloudflare.md](docs/getting-started/deploy-cloudflare.md). It asks for:

1. **Cloudflare login**: opens a browser. With several accounts, it asks which one.
2. **D1 location**: defaults to the region nearest your timezone (`enam`, `wnam`, `weur`, `eeur`,
   `apac`, `oc`). It creates a database named `kinwall` there.
3. **Custom domain**: leave blank to use `workers.dev`. If your account has no `workers.dev`
   subdomain yet, wrangler offers to register one.
4. **`ENCRYPTION_KEY`**: press Enter to generate one. It's printed once. **Save it in a password
   manager**: losing it means reconnecting every calendar, and health entries can't be read.
5. **`ADMIN_API_KEY`**: say yes. It's printed once and is your setup code and permanent admin key.
   Save it too.

It then deploys, sets `PUBLIC_URL`, and calls `/api/health` so the Worker applies its database
migrations. Open the URL it prints, enter the `ADMIN_API_KEY` and add a passkey.

The script writes `database_name` and `database_id` into `wrangler.toml`. The ID identifies the
database but isn't a secret. Commit it in this fork so every clone and CI deploy uses the same
database:

```bash
git add wrangler.toml && git commit -m "chore: use the walkup hq d1 database"
```

### Later deploys

```bash
npm --prefix server run deploy         # builds the UI and deploys the Worker
node scripts/setup-cloudflare.mjs      # or re-run setup; it keeps existing secrets unless you say otherwise
npx --prefix server wrangler tail      # live Worker logs
```

### Adding a domain later

```bash
node scripts/setup-cloudflare.mjs      # enter the hostname at the "Custom domain" prompt
```

The hostname must be in a zone on the same Cloudflare account. This turns off `workers.dev`,
updates `PUBLIC_URL`, and adds `routes` to `wrangler.toml` (commit it, as above).

Passkeys belong to one hostname. Passkeys made on `workers.dev` stop working on the new domain:
sign in with the `ADMIN_API_KEY` and add new ones. Google and Microsoft sign-in redirect URIs must be
updated to the new domain too. If you can, choose the domain before the family sets up passkeys.

## Docker

Still supported, unchanged from upstream:

```bash
docker compose up -d --build           # http://localhost:8080, data in ./data
docker build -t walkup-hq:local .      # check the image builds
```

Passkeys and calendar sign-in need HTTPS (or `localhost`), so a Docker install reached from other
devices needs a domain with HTTPS in front. See
[docs/self-hosting/docker-compose.md](docs/self-hosting/docker-compose.md).

## GitHub Actions in this fork

Workflows don't run in a new fork until you enable them on its **Actions** tab. Once enabled:

| Workflow | In this fork |
|---|---|
| `ci.yml` | Tests and commit-message check on every push and PR. Worth keeping. |
| `docker.yml` | On every push to `main`, builds and pushes `ghcr.io/dustinwalkup/kinwall`. |
| `release-please.yml` | Opens "chore: release x.y.z" PRs and tags releases. Needs **Settings → Actions → General → Allow GitHub Actions to create and approve pull requests**. |
| `cloudflare.yml` | Deploys on `v*` tags, only when the `CLOUDFLARE_DEPLOY` variable is `true`. Needs `CLOUDFLARE_API_TOKEN` and `CLOUDFLARE_ACCOUNT_ID` secrets (and the `CUSTOM_DOMAIN` variable if you use one). |
| `demo.yml`, `discord.yml`, `discord-sync.yml` | Skip themselves unless their variables or secrets are set. |

To turn one off, disable it on the Actions tab instead of deleting the file, so upstream merges stay
clean.

## Left as upstream

These still say Kinwall or point at upstream. They were left alone because changing them is a
product decision or would make upstream merges harder:

- `OPEN_LIBRARY_CONTACT` in `wrangler.toml` is `support@kinwall.family`, sent to Open Library with
  book lookups. Set it to a contact address for your own instance.
- **Settings → Source code** and **Help → issues** link to the upstream repository. That's accurate
  while the deployed code is upstream's. Once you change the app's code, point the source link at
  this fork: the AGPL requires offering your modified source to the people who use it.
- The in-app name, package names (`kinwall-server`, `kinwall-web`), release-please's package name,
  the Docker image and Compose service names, the D1 database name, `.github/FUNDING.yml`, links to
  `docs.kinwall.family`, and the plugin catalog and Android push endpoints on `kinwall.family`.
- A `.DS_Store` file that upstream committed at the repository root.
