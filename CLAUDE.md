# CLAUDE.md

Guidance for Claude Code when working in this repository.

## What this is

Home Assistant Community Add-on that packages [Nginx Proxy Manager (NPM)][npm]
as a Home Assistant add-on. It is a fork of
`hassio-addons/addon-nginx-proxy-manager` (originally by Franck Nijhof).

The add-on lives entirely under [`proxy-manager/`](proxy-manager/):

| File | Purpose |
|------|---------|
| `Dockerfile` | Builds the image on **Debian** (`debian-base`): installs apt deps, adds nginx.org + NodeSource repos, pulls the NPM release, applies `patches/`, builds the frontend (Vite), installs certbot in a pip venv. |
| `config.yaml` | Add-on manifest: `version`, supported `arch`, ports, mappings. |
| `patches/*.patch` | Patches applied to the upstream NPM source during build. |
| `rootfs/` | Files baked into the container (s6 services, scripts). |
| `translations/` | i18n. |
| `CHANGELOG.md` | Per-version change history. |

## Architecture (as of v4.0.0 — Debian)

> History: v3.x was Alpine-based (`hassio-addons/base`, `apk` pins, nginx from
> Alpine). NPM v2.13.0 rewrote the frontend from webpack to Vite, which broke
> the Alpine patch set. Upstream resolved this by **rebasing the whole add-on
> onto Debian**; v4.0.0 adopts that. Do not reintroduce the Alpine approach.

## Critical constraints

- **Base image is Debian.** `ARG BUILD_FROM=ghcr.io/hassio-addons/debian-base:<ver>`
  (Debian trixie). There is **no `build.yaml`** — both supported arches
  (`aarch64`, `amd64`) use the same multi-arch tag, so the single ARG default
  covers them. To bump the base, edit that ARG. (armv7/32-bit was dropped back in
  v3.0.0 and is not coming back.)

- **Dependencies are Debian `apt` pins**, e.g. `nginx=1.31.8-1~trixie`,
  `nodejs=22.x-1nodesource1`. Exact versions are Debian-trixie-specific; look them
  up with `apt-cache policy <pkg>` in the base image, or on
  <https://packages.debian.org>. The `logrotate` pin uses a trailing `*` because
  its binNMU suffix differs per arch — keep that pattern.

- **nginx comes from nginx.org's mainline Debian repo** (added via signing key in
  the Dockerfile), not Debian's own nginx (too old) — currently `1.31.x`
  mainline. The stream module is compiled in statically, so there is **no
  `nginx-mod-stream`** package to keep in sync anymore.

- **Node.js comes from NodeSource** (`node_22.x`), because the app needs Node 22
  and Debian ships 20. npm is bundled in that package (no separate `npm` pin).
  The Vite build needs **no** `--openssl-legacy-provider` (that was a webpack-era
  workaround; it is gone).

- **certbot is a pip venv** at `/opt/certbot` (not an OS package), pinned by
  `ARG CERTBOT_VERSION` and frozen via `PIP_CONSTRAINT=/etc/certbot-constraints.txt`
  so later `pip install`s can't replace the stack.

- **The NPM app version is separate from the add-on version.**
  `ARG NGINX_PROXY_MANAGER_VERSION` pins the upstream *application*
  (`NginxProxyManager/nginx-proxy-manager` GitHub release); `config.yaml`
  `version:` is the add-on wrapper. Bumping the app is the **highest-risk** change
  (the 5 `patches/*.patch` are written against a specific NPM version and the
  Vite build may shift). Process: bump the ARG → build → fix any failing patch
  → verify on hardware. The simplest safe path for a big app jump is to re-sync
  from upstream `hassio-addons/addon-nginx-proxy-manager` (see below).

- **Prefer re-syncing from upstream over hand-porting.** Upstream does the hard
  work (frontend rewrites, patch maintenance, dep pins). When catching up, copy
  upstream's `proxy-manager/{Dockerfile,patches,rootfs,translations,.README.j2,DOCS.md}`
  wholesale — **except keep the fork's own `patches/0004-*`, `patches/0005-*`
  and `rootfs/etc/nginx/conf.d/ingress.conf`** — then re-apply the fork specifics
  below. That is exactly how v4.0.0
  was produced.

- **Check versions beyond upstream, too.** Upstream's Renovate can lag (it
  missed the Node.js 22.23.4 security release for weeks), so the fork may pin
  newer packages than upstream (since v4.2.1: `nodejs`, `apache2-utils`). On a
  re-sync, keep whichever pin is newer. Quick check: NodeSource
  `https://deb.nodesource.com/node_22.x/dists/nodistro/main/binary-arm64/Packages`;
  Debian `deb.debian.org/debian/dists/trixie{,-updates}` plus
  `security.debian.org/debian-security/dists/trixie-security`
  (`main/binary-arm64/Packages.xz`). A superseded Debian pin usually stays
  downloadable from the point release, so a stale pin rarely breaks the build.
  The other sources to check each time: certbot on PyPI
  (`https://pypi.org/pypi/certbot/json`, notes in `certbot/certbot` releases),
  nginx.org (`nginx.org/packages/mainline/debian/dists/trixie/nginx/binary-arm64/Packages`),
  `hassio-addons/addon-debian-base` releases and
  `NginxProxyManager/nginx-proxy-manager` releases. Read the release notes
  before bumping: a day-old release that fixes nothing relevant to this add-on
  can wait for upstream (e.g. certbot 5.10.0, 2026-11-10: a webroot ownership
  race that needs a less-privileged local user, which this container lacks).

## Versioning & releases

- `config.yaml` `version:` holds the add-on version (semver). Breaking changes
  (e.g. dropping an architecture) get a **major** bump.
- Record every change in `proxy-manager/CHANGELOG.md`.
- Releases are published as GitHub releases / tags (`vX.Y.Z`).

## Verifying changes

There is no Docker build environment in the typical Claude session, so package
re-pins and base bumps **cannot be verified here** — they must be built on a
machine with Docker (amd64 or aarch64) before publishing. The highest-risk areas
after a base bump are the NPM frontend `yarn build` (Node major change) and
certbot.

The fastest real verification is to install the add-on on a Home Assistant box
(add this repo as a custom repository) and watch the add-on Log — Supervisor
builds the image locally for that machine's arch.

**Verified history:**
- The `3.x` line (Alpine base 21 / nginx 1.30.2 / Node 24) built and ran clean
  on real **aarch64** hardware (HA OS 18.0).
- **`4.0.0` (Debian / NPM 2.16.0 / nginx 1.31.8 / Node 22) is hardware-verified:**
  built and ran clean on real **aarch64** hardware (HA OS 18.4, HA Core
  2026.11.0b2, Supervisor 2026.10.5) on 2026-10-31 — Vite `yarn build`, the
  nginx.org/NodeSource apt repos, and certbot in its venv all succeeded; NPM
  backend came up on :3000, DB migrations ran, logrotate clean, and the
  `app_config` map mounts data at `/config` as before.
- **`4.0.1` is hardware-verified and released (2026-11-06):** certbot 5.9.0 +
  debian-base 9.5.2 (caught up to upstream HEAD); built/ran clean on aarch64.
- **`4.1.0` is hardware-verified (2026-11-07):** Home Assistant ingress works on
  aarch64 — the interface opens from the sidebar, and port 81 keeps working.
- **`4.2.0` is hardware-verified (2026-11-08):** API tokens work on aarch64 —
  a 30-day token from the user menu authenticates `curl` calls to port 81
  (`/api/users/me`, `/api/reports/hosts`, `/api/logs/tail`, ...), a wrong password
  shows an error without logging out, and login/logout and the sidebar still work.
- **`4.2.1` is hardware-verified (2026-11-09):** Node.js 22.23.4 +
  apache2-utils 2.4.69 built and ran clean on aarch64 (HA OS 18.5.rc1, HA Core
  2026.11.0); `node --version` in the container is `v22.23.4`, API tokens still
  authenticate, and port 81 and the sidebar work.
- **`4.2.2` is hardware-verified (2026-11-10):** on aarch64 the token dialog
  stays open on an outside click or Escape, menu and dashboard links open in a
  new tab from the sidebar, the Documentation tab shows the new sections, and
  port 81 and the sidebar work as before.

## Repo / git

- Remote `origin` → `https://github.com/raphael1688dev/app-nginx-proxy-manager.git`
  (default branch `main`). Authenticated via `gh` as account `raphael1688`.
- The repo is owned by the **`raphael1688dev` org**, whose Actions policy forces
  workflow tokens to read-only. The upstream shared CI (`hassio-addons/workflows`)
  needs write scopes, so it `startup_failure`s here — CI is effectively unusable
  on this fork; verify by installing on Home Assistant instead.
- Commit/push only when the user asks.

## Fork specifics (re-apply these after any upstream re-sync)

- Root **`repository.yaml`** (name "Nginx Proxy Manager APP", this fork's url +
  maintainer) — required for Home Assistant to accept this single-add-on repo as
  a custom repository. Upstream has no such file.
- `config.yaml`: our own `version:` (semver; breaking changes → major bump) and
  `url:` pointing at this fork. Upstream ships `version: dev`.
- `config.yaml` `map:` uses the modern object form with the current type name —
  `- type: app_config` / `read_only: false` — instead of upstream's deprecated
  short form `- addon_config:rw`. Same mount (`/config`, backed by the slug's
  `/addon_configs/nginxproxymanager`), just no Supervisor deprecation warning.
  Re-apply after a re-sync until upstream adopts it.
- **No `build.yaml`** — rely on the `Dockerfile` `ARG BUILD_FROM` default (keeps
  Supervisor's deprecation warning away). Upstream still ships one.
- **Home Assistant ingress (sidebar), since v4.1.0.** Upstream declined it
  (hassio-addons/app-nginx-proxy-manager#649: the app can't run under a
  subfolder), so the fork carries three pieces that must survive a re-sync:
  - `patches/0004-Patch-support-Home-Assistant-ingress.patch` — makes the NPM
    frontend base-path aware: Vite `base: "./"`, an empty
    `<meta name="ingress-path" content="" />` in `index.html`, a
    `src/modules/BasePath.ts` module used by the API client (`base.ts`), the
    `BrowserRouter` `basename`, every `/images/...` path and the `href` of the
    menu (`NavLink.tsx`) and dashboard links (so "open in new tab" works; since
    v4.2.2), plus a nested `/<route>/(assets|images)/` rule in the admin
    `production.conf.template`.
    Written against NPM `v2.16.0`; **re-check it on every NPM app bump** (it is
    the patch most likely to break, e.g. on another frontend rewrite). Do
    **not** switch to a `<base href>` tag instead: the UI has many `href="#"`
    links that a base tag would turn into page loads.
  - `rootfs/etc/nginx/conf.d/ingress.conf` — server on `8099`, `allow
    172.30.32.2` only, proxies `/api/` to the backend on `:3000`, and
    `sub_filter`s the `X-Ingress-Path` header into that meta tag (the exact
    string must match `index.html`).
  - `config.yaml` `ingress: true`, `ingress_port: 8099`, `panel_icon`; no
    `webui`.
  - **Keep ingress on its own port; do not fold it into port 81.** Port 81 is
    open to the LAN, so it must not trust a client-supplied `X-Ingress-Path`
    (it is written into the page); 8099 only accepts the Supervisor. Port 81's
    upstream template also has absolute redirects (`/api` → `/api/`,
    `*.html` → `/$1`) that would escape to Home Assistant's root under ingress,
    and merging would need `if` logic inside NPM's template, growing patch
    0004. 8099 is not in `ports:`, so it is never exposed on the host.
  - The sidebar and port 81 keep **separate logins**: NPM stores its JWT in
    `localStorage` (`authentications`), which browsers isolate per origin, and
    the sidebar runs under Home Assistant's origin. Expected, not a bug.
  - Verify a patch change by building the frontend outside Docker:
    `git apply` patches 0001–0005 in order onto a clean NPM checkout, then
    `yarn install --frozen-lockfile --ignore-scripts && yarn locale-compile &&
    yarn build` in `frontend/`, and check `dist/index.html` and the bundles for
    leftover absolute `/assets/`, `/api/` or `/images/` paths.
  - Regenerate 0004/0005 after an edit (reproduces the committed files
    byte-for-byte when nothing changed): on a clean NPM checkout, `git apply`
    0001–0003 and commit, `git am` 0004 and 0005, edit and `git commit --amend`
    (rebase 0005 on top when 0004 changes), then per patch
    `git format-patch -1 <commit> --zero-commit --no-signature --stdout`,
    delete the ` create mode` lines and zero-pad the day in `Date:`.
  - Trust chain for `X-Ingress-Path` (checked 2026-11-10): Home Assistant Core
    sets it from the URL token after copying the client's headers, and the
    Supervisor only forwards tokens it issued, so the value written into
    `index.html` is never client-controlled on 8099.
- **API tokens in the user menu, since v4.2.0** —
  `patches/0005-Patch-add-API-tokens-to-the-user-menu.patch` (applies on top of
  0004; both touch `SiteHeader.tsx`). A user re-enters their password (+ 2FA
  code) and gets a token with an `expiry` from the allowlist `1d 7d 30d 90d 1y
  2y 3y` (enum in `schema/paths/tokens/post.json` and `tokens/2fa/post.json`).
  Design decisions to keep:
  - **No database table or migration, on purpose.** NPM runs knex
    `migrate.latest()` with list validation, so a fork-only migration could
    never be removed again (NPM refuses to start). Revocation relies on the
    existing per-request checks instead: a password change voids older tokens,
    disabling the user voids all. This keeps 0005 droppable if upstream ships
    real tokens (NginxProxyManager/nginx-proxy-manager#5975).
  - **User menu, not Settings:** Settings is admin-only, but scripts should use
    a dedicated non-admin user.
  - **`GET /tokens` ignores `?expiry=`** (upstream allowed any lifetime from a
    session token, bypassing the password check). Refresh = default 1 day.
  - **English strings only** (`locale/src/en.json`); other locales fall back to
    English via `loadMessages`, and the repo carries no Chinese text.
  - Pitfall: `formatDateTime()` treats any number as Unix *seconds*; pass the
    backend's ISO `expires` string, never `getTime()`.
  - While the token is shown the dialog uses `backdrop="static"` and
    `keyboard={false}` (since v4.2.2): an accidental close would orphan a valid
    token that can't be revoked on its own.
  - Verify backend changes without Docker: in `backend/`, `yarn install
    --ignore-scripts`, then `node validate-schema.js` (whole OpenAPI doc) and a
    small script that awaits `getCompiledSchema()` before calling
    `apiValidator(getValidationSchema("/tokens", "post"), body)` for accepted
    and rejected `expiry` values.
  - Smoke-test a token on hardware (macOS zsh): `export NPM=http://<ip>:81/api
    TOKEN='<token>'`, then `curl -s -G -H "Authorization: Bearer $TOKEN"
    $NPM/logs/tail -d type=system -d lines=20`. Use `-G -d` (or quote the URL):
    unquoted, zsh treats `?` as a glob and `&` as "run in background". Good
    read-only endpoints: `/` (no auth), `/users/me`, `/reports/hosts`,
    `/audit-log`, `/logs/tail`, `/schema`.
  - Reading API errors: a path outside `/api/` on port 81 returns the SPA's
    `index.html`, not JSON (usually `$NPM` missing `/api`). Request
    validation failures come back as a bare `400 "Internal Error"` (upstream
    hides non-public errors); the real reason is a `WARN [Express] ...` line
    in the system log (`/api/logs/tail?type=system`).
  - Harmless upstream log line at startup: `[Migrate] Current database
    version: none`. `migrate.js` calls `currentVersion()` without
    `tableName`, so it reads knex's default `knex_migrations` table instead of
    NPM's `migrations` table. The data is intact.
- `proxy-manager/DOCS.md` carries two fork-only sections, "Sidebar" and "API
  tokens" (since v4.2.2). A re-sync copies upstream's `DOCS.md` over them;
  re-add both.
- Fork meta not present upstream: `CLAUDE.md`, `.gitignore`, root `README.md`
  "Supported architectures" note, `proxy-manager/CHANGELOG.md`.

[npm]: https://nginxproxymanager.com/
