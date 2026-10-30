# my-umami

[Umami](https://umami.is), the blog's visitor statistics, at `https://stats.prabuddh.in`, with its
own PostgreSQL. It sets no cookies, so the blog needs no consent banner. Why Umami, and why
self-hosted: decision E13 in the blog's repository (`docs/decisions/infrastructure.md`).

The blog's pages load a small script from `stats.<DOMAIN>/pulse.js`, which sends each page view
to `stats.<DOMAIN>/api/pulse`. Those two addresses are public. The dashboard and everything else
on the subdomain answers only the addresses in `ADMIN_ALLOWED_IPS`, like the Wagtail admin.

**Why on its own:** while it was part of the blog's Compose project, every blog deploy restarted
it, and an Umami upgrade needed a blog deploy. Now each one stops, starts and upgrades without
the other. If Umami is down, the blog's pages load as usual and simply don't report the visit.

**How the pieces meet:**

- **`compose.yaml`:** `umami`, and its PostgreSQL, `umami-db`. Only `umami` joins the shared
  `edge` network, where Caddy reaches it; the database stays on this project's own network.
  Nothing publishes a port.
- **`caddy/Caddyfile`:** the `stats.{$DOMAIN}` site. The edge stack mounts this folder at
  `/sites/umami` and imports it. `DOMAIN`, `HSTS_SECONDS` and `ADMIN_ALLOWED_IPS` are Caddy's,
  in `/srv/edge/.env`.
- **`.env`:** the two secrets, and the addresses not to count. See `.env.example`.
- **The blog** knows only the address (`NUXT_PUBLIC_STATS_URL`, which its compose file sets to
  `https://stats.<DOMAIN>`) and the website ID (in its Site settings).

## Setting it up

Run these as the `deploy` user (`sudo -iu deploy`), like everything else in `/srv`. To move the
statistics out of the blog's folder on a server where they already run there, follow the blog's
`docs/deployment.md`, "The deploy that moves Caddy and Umami to their own stacks", instead: it
keeps the data.

1. **DNS.** An A and an AAAA record for `stats.<DOMAIN>`, the same addresses as the blog's.
   Without them Let's Encrypt can't check the name, and Caddy keeps retrying its certificate.
2. **This folder, at `/srv/umami`.** The `edge` network must exist (edge's README, step 1), and
   the server needs read access to this repository (step 2 there).

   ```sh
   sudo mkdir -p /srv/umami && sudo chown deploy:deploy /srv/umami   # as your own user
   git clone git@github-umami:PrabuddhMathur/my-umami.git /srv/umami  # as deploy
   cd /srv/umami
   cp .env.example .env && chmod 600 .env
   openssl rand -hex 32   # twice: one value for each secret
   nano .env
   docker compose up -d
   ```

3. **Caddy.** Edge's `compose.yaml` mounts this folder, and Caddy refuses to start while it's
   missing. Start Caddy now (edge's README), or, if it's already running with this folder
   mounted, load the site as in its "A site's Caddyfile changed".

## The first login

Once Caddy serves `stats.<DOMAIN>` and `ADMIN_ALLOWED_IPS` lets your devices in. On a new server
that's after the blog's Tailscale guide (`docs/admin-access.md`): until then the allowlist is
`127.0.0.1/32`, which closes the dashboard to everyone.

1. **Log in.** Open `https://stats.<DOMAIN>/` from a device with Tailscale on. Umami starts with
   the login `admin` and the password `umami`.
2. **Change the password at once.** Open your profile (the user menu, top right) → **Change
   password**. Use a long one from your password manager. **Why now:** the first password is the
   same on every Umami installation. While `ADMIN_ALLOWED_IPS` holds only Tailscale's ranges,
   nobody else can even see the login page. If you ever leave the line out, anyone could log in
   until you change it.
3. **Add the website.** **Settings → Websites → Add website**, with any name and the domain
   `prabuddh.in`. Open the website's settings (**Settings → Websites**, then **Edit**) and copy
   its **Website ID**, shown at the top.
4. **Tell the blog its ID.** In the Wagtail admin: **Settings → Site Settings → Umami website
   ID**, paste it, and save. The blog's pages now load the script (in production only). The ID
   isn't a secret, and changing it needs no deploy.
5. **Check it.** Open the blog in a private window (with Tailscale off, or with your own address
   not in `STATS_IGNORE_IPS`, below). The visit shows up in Umami's **Realtime** view within a
   few seconds.

## Day to day

Run these as `deploy` in `/srv/umami`.

- **Leaving out your own visits.** Either of these works. Both leave the browser console with an
  error for the ignored visit (a 403 from `/api/pulse`), which is harmless.
  - **By address:** in `.env`, `STATS_IGNORE_IPS=100.64.0.0/10,fd7a:115c:a1e0::/48` (Tailscale's
    ranges, comma-separated, no spaces), then `docker compose up -d`. Visits arriving from those
    addresses aren't counted: that is your devices while Tailscale is on, and none of the
    readers'. It does nothing for a visit with Tailscale off.
  - **By browser:** open the blog and, in the browser's developer console, run
    `localStorage.setItem('umami.disabled', 1)`. That browser is never counted again, with or
    without Tailscale ([Umami's documentation](https://docs.umami.is/docs/tracker-configuration)
    lists this key). `localStorage.removeItem('umami.disabled')` turns counting back on.
- **Changing `.env`:** `docker compose up -d` recreates what changed. `UMAMI_DB_PASSWORD` is the
  exception: the database keeps the password it was made with, so a new value in `.env` alone
  locks Umami out. Change it in the database first, then in `.env`, then `up -d`:

  ```sh
  docker compose exec umami-db psql -U umami -d umami -c "ALTER USER umami PASSWORD 'the-new-value'"
  ```

- **The Caddyfile changed.** After `git pull`, validate and reload Caddy as in edge's README,
  "A site's Caddyfile changed".
- **Stopping it:** `docker compose down` stops only the statistics. Caddy answers
  `stats.<DOMAIN>` with an error until `up -d`, and meanwhile the blog's pages don't report
  visits. Never add `-v`: it deletes the data.
- **Backups.** The data is in its own PostgreSQL, in the `umami_umami_db` volume. Don't copy the
  volume's files while it runs. Make a dump, then copy it off the server:

  ```sh
  docker compose exec -T umami-db pg_dump -U umami -Fc -f /tmp/umami.dump umami
  docker compose cp umami-db:/tmp/umami.dump ~/umami-$(date +%F).dump
  docker compose exec -T umami-db rm /tmp/umami.dump
  ```

  Then copy the file in `deploy`'s home folder off the server and delete it there. To restore,
  put the file back with `docker compose cp ~/the-file.dump umami-db:/tmp/umami.dump` and run
  `docker compose exec -T umami-db pg_restore -U umami -d umami --clean --if-exists /tmp/umami.dump`.
  Page-view history is the only thing in it: losing it costs you the past numbers, not the blog.
- **Upgrading Umami.** The version is the image tag in `compose.yaml`
  (`ghcr.io/umami-software/umami:3.4.1`). Umami has no long-term-support line, so look at its
  [release notes](https://github.com/umami-software/umami/releases) every month or two, and
  upgrade when one mentions security. To upgrade: read the notes (look for changes to
  environment variables and database migrations), take a backup as above, change the tag here
  and in the blog's `docs/versions.md`, then `git pull` and `docker compose up -d` on the
  server. Umami migrates its database as it starts.
- **Upgrading PostgreSQL.** Minor updates (`18.x`) are safe: change the tag and `up -d`. The major
  version (18) must not change that way, because the data folder of one major version doesn't
  open in another. A major upgrade means a dump, a new volume, and a restore into the new
  version, as above. PostgreSQL 18 is supported until 2030-11-14.
