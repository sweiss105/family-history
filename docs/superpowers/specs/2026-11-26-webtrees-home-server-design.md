# Family history site: webtrees on the home server

Date: 2026-11-26 · Status: approved

## Intent

A private family-tree website that relatives can browse. Access is limited to
people the owner approves. It is self-hosted on the owner's existing home
server and reached at **https://family.weiss-audio.com**. Cloudflare Access is
**not** used for the login; webtrees' own accounts control who gets in.

## Goals

- Relatives can browse the tree, people and photos after signing in.
- Each relative has their own account, which the owner approves (and can remove).
- Nothing is visible to anyone who isn't signed in. Living people are hidden as
  an extra safeguard.
- Nightly backups, plus written steps for restoring from them.
- The setup can be rebuilt from this repo plus a passwords file.

## Non-goals

- A custom-built website (webtrees provides the UI).
- Public pages, or search-engine visibility.
- Off-site backup copies (worth adding later).

## Anti-goals

- No family data, photos, GEDCOM files, passwords or `.env` files in this
  repository. **The repository is public.**
- Nothing exposed on the home network or the internet other than through the
  existing Cloudflare Tunnel.

## Constraints (from server recon, 2026-11-26)

- Host: `weissaudio-server`, Ubuntu 26.04, 14 GB RAM, 134 GB free on `/`.
  Docker 29.8 and Compose v5.5. User `weissaudio` can run docker without sudo.
- `cloudflared` runs as a systemd service with a **token** (remotely managed).
  Its public-hostname routes live in the Cloudflare dashboard/API, not in a
  local file.
- Port **8095** on 127.0.0.1 is free. 8080, 8088, 8090, 8443 and 41xx–43xx are
  taken.
- No existing MariaDB/MySQL or webtrees containers.
- Git on the server has no user identity configured. The server only *pulls*
  this repo; commits are made elsewhere.

## Approaches considered

1. **Static site + shared password (StatiCrypt):** free, no server. Rejected:
   one shared password, and no per-person accounts.
2. **Host-level basic auth (Netlify/Vercel paid, or nginx):** rejected in favour
   of a purpose-built genealogy app.
3. **webtrees, self-hosted on the home server behind the existing Cloudflare
   Tunnel (chosen):** mature genealogy software with per-user accounts, approval
   of new users, and privacy rules for living people. No new hosting cost; reuses
   the existing tunnel and domain.

## Design

**Containers** (`docker-compose.yml`, project name `family-history`):

- `webtrees`: community image `nathanvaughn/webtrees` (webtrees publishes no
  official image), pinned to a specific stable version tag. Published only as
  `127.0.0.1:8095 -> 80`. `BASE_URL=https://family.weiss-audio.com`. HTTPS and
  HTTPS-redirect are off inside the container, because Cloudflare terminates TLS.
- `db`: `mariadb`, pinned LTS tag, on an internal network with no published
  ports. Health check so webtrees starts only after the database is ready.
- Both use `restart: unless-stopped`.

**Data:** bind mounts under `./data/` (git-ignored): `data/webtrees` (webtrees
data folder: config, media, uploads) and `data/mariadb`.

**Secrets:** `.env` (git-ignored), created from a committed `.env.example`. It
holds the DB passwords and the initial webtrees admin credentials. The image's
automatic first-run setup creates the database config and the admin account
from these values. After the first login, the owner changes the admin password
in webtrees and can blank `WT_PASS` in `.env`.

**Cloudflare:** one public hostname on the existing tunnel:
`family.weiss-audio.com` → `http://localhost:8095`. It is added once the
admin account exists and account approval is switched on, but **before any
family data is imported**. That way there's no open setup wizard to hijack,
and nothing private can be seen. Setup happens through the real URL, because
webtrees redirects everything to `BASE_URL`. Data is imported only after a
test confirms that signed-out visitors are sent to the login page. Added by a Claude session that has working
Cloudflare tools if one is available; otherwise by the owner in the dashboard.
The README documents both ways.

**Privacy settings** (applied in the webtrees control panel and documented in
the README):

- Tree privacy: visitors must sign in to see anything.
- Site: allow users to request accounts; the administrator must approve each
  one; new users are emailed on approval if mail is set up (optional).
- Living people: show only to members (default rules; confirm they're on).

**Backups** (`scripts/backup.sh`, run nightly by the `weissaudio` user's
crontab, for example at 03:17):

- `mariadb-dump --single-transaction` from the `db` container, gzipped.
- A tar.gz of `data/webtrees` (config + media).
- Written to `~/family-history-backups/YYYY-MM-DD/`. Backups older than 14 days
  are deleted.
- `scripts/restore.sh` (or documented commands) restores a chosen day.

**Docs:** `README.md` covering:

- what this is
- first-time setup on the server
- adding the tunnel route
- the privacy checklist
- adding and approving relatives
- importing a GEDCOM
- backups and restore
- updating webtrees and MariaDB (bump the tags, `docker compose pull && up -d`)

## Who does what

- **Claude (this repo):** writes all the files, commits and pushes to branch
  `claude/ecstatic-faraday-ej3zdu`.
- **Claude session on the server** (`~/family-history`):
  - clones the repo
  - creates `.env` with generated passwords
  - starts the containers and verifies webtrees answers on 127.0.0.1:8095
  - installs the cron job and runs one test backup
  - adds the tunnel route if it has Cloudflare tools
- **Owner:**
  - any `sudo` steps
  - signing in as admin and applying or confirming the privacy settings
  - the Cloudflare dashboard step if Claude can't do it
  - approving relatives

## Left to the builder

- The exact `nathanvaughn/webtrees` and `mariadb` tags: the latest stable, as
  checked on Docker Hub at build time.
- The exact environment-variable names, checked against the image's README.
- The cron time and the backup file naming.
