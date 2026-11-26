# webtrees on the Home Server: Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:executing-plans (or superpowers:subagent-driven-development) to implement this plan task by task. Steps use checkbox (`- [ ]`) syntax for tracking.
>
> **You are running on `weissaudio-server` in `~/family-history`.** This file is a handoff. Everything you need is here and in the spec below. Do the tasks in order, and stop at every **🙋 OWNER** step until the owner confirms in chat.

**Goal:** A private webtrees family-tree site at https://family.weiss-audio.com. It runs in Docker on this server, is reached only through the existing Cloudflare Tunnel, has a login for each relative, and is backed up nightly.

**Architecture:** Two Docker Compose services: `webtrees` (`nathanvaughn/webtrees`), published only on `127.0.0.1:8095`, and `db` (MariaDB) on an internal network. Data is in git-ignored bind mounts under `./data/`; secrets are in a git-ignored `.env`. cloudflared (systemd, token-managed) forwards the public hostname to `http://localhost:8095`. Cron runs a backup script nightly.

**Tech Stack:** Docker 29.8 / Compose v5.5, nathanvaughn/webtrees (webtrees 2.x), MariaDB LTS, bash, cron, Cloudflare Tunnel.

**Spec:** `docs/superpowers/specs/2026-11-26-webtrees-home-server-design.md`. Read it first.

## Ground rules for the executor

- **The GitHub repo is PUBLIC.** Never commit `.env`, `data/`, backups, GEDCOM files, photos or any password. Never print a password into chat. Refer to it as "the value in `.env`".
- You can't run `sudo` (there's no password). If a step needs it, give the owner the exact command, then wait.
- Don't touch other containers, other compose projects, or the `cloudflared` service. Don't restart Docker or cloudflared. Don't edit the shared port bindings.
- Don't use ports other than `127.0.0.1:8095`. Nothing is bound to `0.0.0.0`.
- Git identity for commits in this clone is **repo-local only**: `git config user.name "weissaudio-server"` and `git config user.email "weissaudio-server@localhost"`. Don't set global config.
- If a fact here turns out wrong (an env var name, a tag, a setting key), check the upstream docs, fix it, note the change in your final report, and carry on.

## Global Constraints

- Public URL: `https://family.weiss-audio.com`. Local bind: `127.0.0.1:8095` → container port 80.
- Compose project name: `family-history`. Services: `webtrees`, `db`.
- Images pinned to exact version tags (no `latest`): the newest stable `nathanvaughn/webtrees` tag, and the newest `mariadb` LTS tag (for example `11.8.x`), checked on Docker Hub when you build.
- Inside the container: `HTTPS=false`, `HTTPS_REDIRECT=false`, `BASE_URL=https://family.weiss-audio.com`, `PRETTY_URLS=true`. Cloudflare terminates TLS.
- Data: `./data/webtrees` → `/var/www/webtrees/data`, `./data/mariadb` → `/var/lib/mysql`. Both git-ignored.
- Backups: `~/family-history-backups/YYYY-MM-DD/`, keep 14 days, cron at `17 3 * * *`.
- Privacy end state: site requires admin approval for new accounts; every tree requires sign-in (`REQUIRE_AUTHENTICATION=1`); living people hidden (`HIDE_LIVE_PEOPLE=1`).
- **No family data is imported until the privacy settings are on and verified (Task 5).**

## Review Focus

1. **Private data reachable without signing in:** an unauthenticated request for a tree page must redirect to the login page, never return tree content. Covered by `scripts/check.sh` in Task 5.
2. **Secrets leaking into git:** `.env` and `data/` must be ignored before they exist. Covered by `check.sh` (`git check-ignore`) in Task 1.
3. **Wrong URLs behind Cloudflare:** links or redirects pointing at `http://` or `localhost`, or redirect loops through the tunnel. Covered by the public `curl` checks in Task 4.
4. **A backup that "succeeds" but is empty or partial:** for example when the containers are down. `backup.sh` must exit non-zero and leave no dated folder behind. Covered in Task 3.
5. **Photos or GEDCOM files that are too big to upload:** PHP limits are raised to 64M. Checked in Task 2 (`php -i`).

---

### Task 1: Get the repo, then write the compose setup and the secrets template

**Files:**
- Create: `docker-compose.yml`, `.env.example`, `.gitignore`, `scripts/check.sh`

**Interfaces:**
- Produces: the services `webtrees` and `db`; the `.env` keys `MARIADB_ROOT_PASSWORD`, `MARIADB_PASSWORD`, `WT_ADMIN_USER`, `WT_ADMIN_NAME`, `WT_ADMIN_EMAIL`, `WT_ADMIN_PASSWORD`, `WEBTREES_TAG`, `MARIADB_TAG`; `scripts/check.sh [--public]`, which exits 0 when all checks pass and prints `FAIL: <reason>` otherwise.

- [ ] **Step 1: Get the repo into the current folder.** It may hold untracked files, so don't use `clone`:

```bash
cd ~/family-history
git init -b main
git remote add origin https://github.com/sweiss105/family-history.git
git fetch origin claude/ecstatic-faraday-ej3zdu
git checkout -b claude/ecstatic-faraday-ej3zdu FETCH_HEAD
git config user.name "weissaudio-server"; git config user.email "weissaudio-server@localhost"
```

Expected: `docs/superpowers/specs/…design.md` and this plan are present.

- [ ] **Step 2: Write `.gitignore`.** It ignores `.env`, `data/`, `*.ged`, `*.gedcom`, `backups/`, `*.sql`, `*.sql.gz` and `*.tar.gz`.

- [ ] **Step 3: Write `scripts/check.sh` (the test).** Use `set -euo pipefail`. Each check prints `ok: …` or `FAIL: …`, and the script exits 1 if any check fails. Checks:
  - `git check-ignore -q .env` and `git check-ignore -q data/x` both succeed.
  - `git ls-files` lists no `.env`, nothing under `data/`, and nothing ending in `.ged`, `.sql` or `.gz`.
  - `ss -tln` shows `127.0.0.1:8095` listening, and **no** `0.0.0.0:8095` or `[::]:8095`.
  - `curl -s -o /dev/null -w '%{http_code}' http://127.0.0.1:8095/` returns 200 or 302.
  - With `--public`:
    - `curl -sI https://family.weiss-audio.com/` returns 200 or 302, and any `Location` header starts with `https://family.weiss-audio.com`.
    - The login page body contains `password`, case-insensitive.
  - With `--public`, and when the environment variable `TREE` is set:
    - Unauthenticated `curl -s -o /dev/null -w '%{http_code} %{redirect_url}' "https://family.weiss-audio.com/tree/$TREE"` must be a 302 whose redirect URL contains `login`.

- [ ] **Step 4: Run it before anything exists.**
  Run: `bash scripts/check.sh`
  Expected: the ignore checks print `ok`, and the port and HTTP checks print `FAIL` (nothing is running yet).

- [ ] **Step 5: Write `docker-compose.yml`.**
  - `name: family-history`
  - `db`:
    - `mariadb:${MARIADB_TAG}`, with env `MARIADB_ROOT_PASSWORD`, `MARIADB_DATABASE=webtrees`, `MARIADB_USER=webtrees` and `MARIADB_PASSWORD`
    - volume `./data/mariadb:/var/lib/mysql`
    - healthcheck `healthcheck.sh --connect --innodb_initialized` (interval 10s, retries 10)
    - on network `backend` only
  - `webtrees`:
    - `nathanvaughn/webtrees:${WEBTREES_TAG}`, `depends_on: db: condition: service_healthy`, port `"127.0.0.1:8095:80"`, volume `./data/webtrees:/var/www/webtrees/data`, networks `backend`
    - env: `DB_TYPE=mysql`, `DB_HOST=db`, `DB_PORT=3306`, `DB_USER=webtrees`, `DB_PASS=${MARIADB_PASSWORD}`, `DB_NAME=webtrees`, `DB_PREFIX=wt_`, `WT_USER=${WT_ADMIN_USER}`, `WT_NAME=${WT_ADMIN_NAME}`, `WT_EMAIL=${WT_ADMIN_EMAIL}`, `WT_PASS=${WT_ADMIN_PASSWORD}`, `BASE_URL=https://family.weiss-audio.com`, `PRETTY_URLS=true`, `HTTPS=false`, `HTTPS_REDIRECT=false`, `LANG=en-US`, `PHP_UPLOAD_MAX_FILESIZE=64M`, `PHP_POST_MAX_SIZE=64M`, `PHP_MEMORY_LIMIT=512M`
  - Both services: `restart: unless-stopped`.
  - Network `backend`: a plain bridge network (webtrees needs it for the published port, so not `internal`).
  - **Check every env var name against the image README** at https://github.com/NathanVaughn/webtrees-docker before you continue.

- [ ] **Step 6: Write `.env.example`.**
  - Every key from Interfaces, with placeholder values (`change-me`).
  - Pick the tags now from Docker Hub (`nathanvaughn/webtrees` newest stable; `mariadb` newest LTS) and write them in as the defaults for `WEBTREES_TAG` and `MARIADB_TAG`.
  - Add a comment at the top: "Copy to .env, fill in, never commit."

- [ ] **Step 7: Validate.**
  Run: `cp .env.example .env.tmp && docker compose --env-file .env.tmp config -q && rm .env.tmp`
  Expected: no output and exit 0.

- [ ] **Step 8: Commit.**

```bash
git add .gitignore docker-compose.yml .env.example scripts/check.sh
git commit -m "Add docker compose setup for webtrees and MariaDB"
```

### Task 2: Create `.env` and start the containers

**Files:** Create `.env` (git-ignored, never committed). Create `data/` (git-ignored).

**Interfaces:**
- Consumes: Task 1's compose file and `.env` keys.
- Produces: webtrees running on `127.0.0.1:8095`, with the admin account created.

- [ ] **Step 1: 🙋 OWNER: ask for the admin details.** Ask the owner for the admin username, display name and email they want. **Don't** ask for a password.

- [ ] **Step 2: Create `.env`.**
  - Copy `.env.example`, then fill in the owner's answers.
  - Generate each of the three passwords with `openssl rand -base64 24 | tr -d '/+='`.
  - `chmod 600 .env`.
  - Tell the owner: "the initial admin password is in `~/family-history/.env` (`WT_ADMIN_PASSWORD`); read it with `grep WT_ADMIN_PASSWORD ~/family-history/.env`".

- [ ] **Step 3: Start.**
  Run: `mkdir -p data/webtrees data/mariadb && docker compose pull && docker compose up -d`
  Expected: both containers are `Up`, and `db` is `(healthy)` within about 60s (check with `docker compose ps`).

- [ ] **Step 4: Check the logs.**
  Run: `docker compose logs webtrees --tail 50`
  Expected: no PHP fatal errors; the automatic setup wrote `config.ini.php`.
  Then: `docker compose exec webtrees ls /var/www/webtrees/data/config.ini.php` exists.
  If the data folder's ownership blocks writes, fix it from inside the container (`docker compose exec webtrees chown -R www-data:www-data /var/www/webtrees/data`). Don't use host sudo.

- [ ] **Step 5: Check the upload limit.**
  Run: `docker compose exec webtrees php -i | grep -E 'upload_max_filesize|post_max_size'`
  Expected: both are `64M`.

- [ ] **Step 6: Run the checks.**
  Run: `bash scripts/check.sh`
  Expected: all `ok` (no `--public` yet). Locally, `/` answers with 200 or a 302 to the login page.

- [ ] **Step 7: Confirm `.env` is not staged.**
  Run: `git status --short`
  Expected: `.env` and `data/` are not listed. Nothing to commit.

### Task 3: Backup and restore scripts, plus the cron job

**Files:**
- Create: `scripts/backup.sh`, `scripts/restore.sh`

**Interfaces:**
- Produces:
  - `scripts/backup.sh [DEST_ROOT]`: `DEST_ROOT` defaults to `$HOME/family-history-backups`, and `KEEP_DAYS` (env) defaults to 14. On success it creates `DEST_ROOT/YYYY-MM-DD/{webtrees-db.sql.gz,webtrees-data.tar.gz}`, deletes dated folders older than `KEEP_DAYS`, and exits 0. On any failure it exits non-zero and leaves no partial dated folder.
  - `scripts/restore.sh <backup-dir>`: restores both files into the running stack.

- [ ] **Step 1: Write a failing test for the "containers down" case.**
  Run: `docker compose stop db && bash scripts/backup.sh /tmp/fh-bk-test; echo "exit=$?"; ls /tmp/fh-bk-test 2>/dev/null; docker compose start db`
  Expected before `backup.sh` exists: an error. Expected once it's written: `exit` is not 0, and no dated folder is listed.

- [ ] **Step 2: Implement `scripts/backup.sh`.**
  - `set -euo pipefail`, and `cd` to the repo root (the script's directory, `/..`).
  - Load `MARIADB_ROOT_PASSWORD` from `.env`. Pass it to the container through `docker compose exec -T -e MYSQL_PWD=… db`, so it never appears in `ps` output.
  - Write into `DEST_ROOT/.tmp-<date>`. Run `mariadb-dump --single-transaction --routines --triggers webtrees | gzip`, then `docker compose exec -T webtrees tar czf - -C /var/www/webtrees data`.
  - Check that both files are non-empty (`test -s`) and that `gzip -t` passes. Then `mv` the folder to `DEST_ROOT/<date>` (replacing any existing folder for that date).
  - Prune with `find DEST_ROOT -mindepth 1 -maxdepth 1 -type d -name '20*' -mtime +$KEEP_DAYS -exec rm -rf {} +`.
  - A `trap` removes `.tmp-*` on error.

- [ ] **Step 3: Run the failure test again.**
  Expected: exit is not 0, `/tmp/fh-bk-test` has no dated folder, and the `db` container restarts.

- [ ] **Step 4: Run a real backup.**
  Run: `bash scripts/backup.sh /tmp/fh-bk-test && ls -l /tmp/fh-bk-test/$(date +%F)`
  Expected: two non-empty files.

- [ ] **Step 5: Implement `scripts/restore.sh <backup-dir>`.**
  - It refuses to run without the argument, or if either file is missing.
  - It asks `Type RESTORE to continue`, unless `--yes` is given.
  - Then: `docker compose stop webtrees`, pipe the gunzipped SQL into `mariadb webtrees` in `db` (same `MYSQL_PWD` method), `docker compose start webtrees`, then pipe the tar into `docker compose exec -T webtrees tar xzf - -C /var/www/webtrees`.

- [ ] **Step 6: Test the restore round trip.** It's safe now, because no family data exists yet.
  Run: `bash scripts/restore.sh --yes /tmp/fh-bk-test/$(date +%F) && bash scripts/check.sh`
  Expected: all `ok`.
  Then: `rm -rf /tmp/fh-bk-test`.

- [ ] **Step 7: Install the cron job.** Keep any existing crontab lines.
  Run: `( crontab -l 2>/dev/null | grep -v 'family-history/scripts/backup.sh'; echo '17 3 * * * cd $HOME/family-history && bash scripts/backup.sh >> $HOME/family-history-backups/backup.log 2>&1' ) | crontab - && mkdir -p ~/family-history-backups && crontab -l | grep backup.sh`
  Expected: the line is printed once.

- [ ] **Step 8: Commit.**

```bash
git add scripts/backup.sh scripts/restore.sh
git commit -m "Add nightly backup and restore scripts"
```

### Task 4: Publish through the Cloudflare Tunnel

There's no family data yet. The admin account exists, so there's no open setup wizard to hijack.

**Interfaces:**
- Consumes: webtrees on `127.0.0.1:8095`.
- Produces: `https://family.weiss-audio.com` → `http://localhost:8095` on the server's existing tunnel.

- [ ] **Step 1: Set the site-level privacy before going public.**
  - Get the DB root password from `.env` the same way as in Task 3.
  - Run inside `db`:
    `REPLACE INTO wt_site_setting (setting_name, setting_value) VALUES ('USE_REGISTRATION_MODULE','1'),('REQUIRE_ADMIN_AUTH_REGISTRATION','1'),('SHOW_REGISTER_CAUTION','1');`
  - Then `SELECT` them back.
  - Expected: three rows with value `1`.
  - If the table or key names differ in this webtrees version, set the same options in the Control panel instead (step 2 of the 🙋 OWNER checklist in Task 5).

- [ ] **Step 2: Add the route.**
  - **If you have signed-in Cloudflare tools:** the check on 2026-11-26 found the Cloudflare API plugin (`execute`) signed in on this server.
    - The tunnel is **`weissaudio-home-server`** (id `ccf895dd-4866-4926-a7c7-09d74236b6a3`, managed from the dashboard). The zone is `weiss-audio.com`.
    - Write access ("Cloudflare Tunnel Write" and "DNS Write") is unconfirmed. If a write is refused, fall back to the dashboard steps below.
    - The configuration update replaces the whole ingress list. Leave all 11 existing hostnames (classicroom, cues, health, print, ontime, pumpkins, coursedog, ajn, telembridge, frankie, xmas) unchanged, and keep the final `http_status:404` catch-all last.
    - Read the tunnel's current ingress configuration.
    - Add `{hostname: family.weiss-audio.com, service: http://localhost:8095}` **before** the final catch-all rule, keeping every existing rule unchanged.
    - Make sure the proxied CNAME `family` → `<tunnel-id>.cfargotunnel.com` exists.
    - Show the owner the before/after ingress list and get a 🙋 OWNER "yes" before writing.
  - **Otherwise, 🙋 OWNER** in the Cloudflare dashboard:
    - Zero Trust → Networks → Tunnels → (this server's tunnel) → Public Hostname → Add.
    - Subdomain `family`, domain `weiss-audio.com`, type `HTTP`, URL `localhost:8095`.
    - Save. Wait for the owner to say it's done.

- [ ] **Step 3: Verify from outside.**
  Run: `bash scripts/check.sh --public`
  Expected: all `ok`. The public URL answers, there are no redirects to `http://` or `localhost`, and the login page is shown.

- [ ] **Step 4: 🙋 OWNER: first login.**
  - The owner opens https://family.weiss-audio.com and signs in with the admin username and the password from `.env`.
  - Then: My page → My account → change password.
  - Afterwards you set `WT_ADMIN_PASSWORD=` (empty) in `.env`, then run `docker compose up -d` to recreate webtrees with the new env. Confirm the owner can still sign in, because the stored password is in the database.

### Task 5: Privacy for the tree, then the family data

**Interfaces:**
- Consumes: the public site from Task 4.
- Produces: a tree, for example `family`, that only signed-in, approved users can see, and `TREE=<name>` for `check.sh`.

- [ ] **Step 1: 🙋 OWNER: create an empty tree.**
  - Control panel → Family trees → Create a family tree. Short name, for example `family`.
  - **Don't import anything yet.**
  - Ask the owner for the tree's short name.

- [ ] **Step 2: Set the tree's privacy.**
  - Look up the tree's `gedcom_id` (`SELECT gedcom_id, gedcom_name FROM wt_gedcom`).
  - Then run `REPLACE INTO wt_gedcom_setting (gedcom_id, setting_name, setting_value) VALUES (<id>,'REQUIRE_AUTHENTICATION','1'),(<id>,'HIDE_LIVE_PEOPLE','1');`
  - Then `SELECT` them back.
  - Expected: both are `1`.

- [ ] **Step 3: 🙋 OWNER: confirm in the UI.** Check that Control panel → the tree → Preferences → Privacy shows:
  - visitors must sign in
  - living people are private

  And that Control panel → Website preferences → Sign-in and registration shows:
  - visitors may request an account
  - the administrator must approve new accounts

- [ ] **Step 4: Run the gate test for private data.**
  Run: `TREE=<name> bash scripts/check.sh --public`
  Expected: all `ok`. An unauthenticated request for the tree is a 302 to the login page.
  **If this fails, stop.** Don't let any data be imported until it passes.

- [ ] **Step 5: 🙋 OWNER: import the family data (optional now).**
  - If the owner has a GEDCOM from Ancestry, FamilySearch, MyHeritage or desktop software: Control panel → Family trees → (tree) → Import a GEDCOM file.
  - Then run the Step 4 check again. Expected: still all `ok`.

- [ ] **Step 6: Take a backup now and check it.**
  Run: `bash scripts/backup.sh && ls -l ~/family-history-backups/$(date +%F)`
  Expected: two non-empty files.

### Task 6: README, final report, push

**Files:**
- Modify: `README.md`

- [ ] **Step 1: Write `README.md`.** Sections:
  1. What this is.
  2. First-time setup:
     - get the repo
     - `cp .env.example .env`, generate passwords, `chmod 600`
     - `docker compose up -d`
  3. Tunnel route: dashboard steps, as in Task 4 Step 2.
  4. Privacy checklist: Task 5 Step 3.
  5. Adding relatives: they click "Request a new user account", and you approve them in Control panel → Users. To remove someone, delete or disable their account.
  6. Importing a GEDCOM.
  7. Backups and restore:
     - the backup location
     - how to restore: `scripts/restore.sh ~/family-history-backups/YYYY-MM-DD`
     - a recommendation to copy backups off this machine
  8. Updating:
     - bump `WEBTREES_TAG` or `MARIADB_TAG` in `.env`
     - take a backup first
     - `docker compose pull && docker compose up -d`
     - check that `scripts/check.sh --public` passes
  9. Health check: `bash scripts/check.sh --public`.

  **No passwords, no names of relatives, no data.**

- [ ] **Step 2: Run the final verification.**
  Run: `TREE=<name> bash scripts/check.sh --public && git status --short && git ls-files`
  Expected: all `ok`; the working tree is clean apart from ignored files; `git ls-files` shows only these:
  - `.gitignore`, `.env.example`
  - `docker-compose.yml`, `README.md`
  - `scripts/*.sh`
  - `docs/…`

- [ ] **Step 3: Commit and push.**

```bash
git add README.md && git commit -m "Document setup, privacy, backups and updates"
git push -u origin claude/ecstatic-faraday-ej3zdu
```

  If the push fails for lack of GitHub credentials, **don't** try to set up credentials. Leave the commits local and say so in the report. The owner can push later, or pull the files into another session.

- [ ] **Step 4: Report to the owner.** Include:
  - the URL
  - the tree name
  - the image tags used
  - anything that differed from this plan, and why
  - whether the push succeeded
  - the 🙋 items still open
