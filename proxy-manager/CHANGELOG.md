# Changelog

All notable changes to this add-on are documented in this file.

## 4.3.1 (2026-11-20)

### ⬆️ Upgrades

- **Nginx Proxy Manager** `v2.17.0` → `v2.17.1`. A small release:
  - Fixes the SSL options (Force SSL, HTTP/2, HSTS) switching back off when a
    new certificate is requested while creating a host.
  - Moves nginx's proxy variables (`$forward_host`, `$forward_scheme`,
    `$x_forwarded_*`) from `nginx.conf` into their own include file; custom
    `http_top.conf` snippets can still redefine them.
  - Dependency updates (Vite, better-sqlite3, Tabler).
  - Its "OpenResty with backported fixes" note does not apply here: this
    add-on uses nginx from nginx.org, already at 1.31.9 with those fixes.
- **nginx** `1.31.9` → `1.31.10` (`nginx=1.31.10-1~trixie`, nginx.org
  mainline). Fixes excessive CPU use that a client could cause with
  `ssl_early_data` (CVE-2026-132779; the add-on's own configuration does not
  use that directive) and nginx hogging the CPU on SSL connections to
  backends, which proxy hosts that forward over `https` use. Ahead of
  upstream.
- **debian-base** `9.5.2` → `9.6.0` (`ARG BUILD_FROM`), matching upstream:
  the base image moves to Debian 13.8 and bashio 0.20.0 (can reach the
  Supervisor over its Unix socket, fixes a cache race). The add-on's startup
  scripts only use bashio's logging, exit, and DNS helpers, which are
  unchanged.
- **Debian 13.8** point release (2026-11-21) re-pins, needed to build at all:
  apt only installs versions still listed in the archive's package index, and
  the point release replaced `patch` `2.8-2` there, so the old pin made the
  image fail to build (`Version '2.8-2' for 'patch' was not found`).
  - `patch` `2.8-2` → `2.8-2+deb13u1` (fixes CVE-2026-115207, out-of-tree
    file creation).
  - `openssl` and `libssl-dev` `3.5.7-1~deb13u3` → `3.5.9-1~deb13u1`
    (OpenSSL 3.5.9). The old version is still listed in `trixie-security`,
    but will disappear from there too.
- Patches `0004` and `0005` apply to `v2.17.1` exactly, without changes.

### ✅ Verified before release

- The frontend type-checks and builds with no absolute `/api/`, `/assets/` or
  `/images/` paths left; all 57 frontend tests pass; the backend OpenAPI
  document validates and the API token `expiry` rules behave as before.

## 4.3.0 (2026-11-18)

### ⬆️ Upgrades

- **Nginx Proxy Manager** `v2.16.0` → `v2.17.0` (`ARG NGINX_PROXY_MANAGER_VERSION`).
  Mostly fixes, several of them security fixes: a login timing oracle is
  closed, logins are rate limited, certificate private keys are no longer
  returned by the API, validating a certificate needs a login, and backend
  dependency updates fix 15 advisories (including a body-parser DoS). Also new
  DNS providers, certificate details in the host tables, a warning before
  closing a form with unsaved changes, and new dark-mode colours.
  - ⚠️ Upstream updated several Certbot DNS plugins. The EdgeDNS, Beget and
    Infomaniak plugins have breaking changes: re-create certificates that use
    them. Check that DNS-validated certificates still renew.
- **nginx** `1.31.8` → `1.31.9` (`nginx=1.31.9-1~trixie`, nginx.org mainline),
  matching upstream. A security release fixing 10 CVEs, among them memory
  corruption in HTTP/2 flow control (rated major), use-after-free bugs in
  HTTP/2, HTTP/3 and the `resolver` directive, and a buffer overflow with
  `try_files` plus `alias`. nginx also now rejects some malformed requests,
  such as header lines without a colon.
- **certbot** `5.9.0` → `5.10.0`, matching upstream.
- **Node.js** `22.23.4` → `22.23.5` (`nodejs=22.23.5-1nodesource1`), a
  security release with several high-severity `http2` and `tls` fixes. Ahead
  of upstream.
- **apache2-utils** `2.4.69-1~deb13u1` → `2.4.71-1~deb13u1` (Debian security
  update; its CVEs are in Apache HTTP Server modules, and the add-on only uses
  `htpasswd`). Ahead of upstream.

### 🔁 Patches

- `0004` and `0005` regenerated against Nginx Proxy Manager `v2.17.0`. Their
  changes are the same; only line numbers and context moved, so they apply
  exactly instead of by fuzzy matching.
- `0005` now also carries Estonian and Vietnamese strings for the API token
  dialog. Upstream's frontend tests require those two locales to cover every
  English string; since 4.2.0 the English-only strings made those tests fail.
  The build never runs the tests, so the add-on itself was not affected. Other
  languages still show the dialog in English.

### ✅ Verified before release

- All five patches apply to `v2.17.0`; the frontend type-checks and builds,
  with no absolute `/api/`, `/assets/` or `/images/` paths left in the output.
- All 57 frontend tests pass; the backend OpenAPI document validates and the
  API token `expiry` rules accept and reject the right values.

## 4.2.2 (2026-11-10)

### 🐛 Fixes

- **API tokens: a new token can no longer be lost by a stray click.** While the
  token is shown, clicking outside the dialog or pressing Escape no longer
  closes it. The token is shown only once and can't be revoked on its own, so
  closing it by accident left a valid token nobody had. Close the dialog with
  "Done" or the close button. (patch `0005`)
- **Sidebar: menu and dashboard links now open in a new tab.** Their addresses
  lacked the Home Assistant ingress path, so "Open in new tab", middle-click
  and Ctrl/Cmd-click led to a Home Assistant error page. Normal clicks and
  port `81` were not affected. (patch `0004`)

### 📝 Documentation

- The app's Documentation tab (`DOCS.md`) now covers the sidebar and API
  tokens.
- Corrected the 4.2.1 entry below: Node.js 22.23.4 fixes 6 CVEs, not 7.

## 4.2.1 (2026-11-09)

### ⬆️ Upgrades

- Upgrade **Node.js** `22.23.3` → `22.23.4` (`nodejs=22.23.4-1nodesource1`).
  This is a Node.js security release: 6 CVEs fixed in `http`, `tls`, `dns`,
  `crypto`, the permission model and brotli, plus bundled OpenSSL 3.5.9, newer
  root certificates and undici 6.29.1. The Nginx Proxy Manager backend runs on
  Node.js.
- Upgrade **apache2-utils** `2.4.68-1~deb13u1` → `2.4.69-1~deb13u1` (Debian
  security update). Its CVEs are in Apache HTTP Server modules; the add-on only
  uses the `htpasswd` tool, so this is housekeeping.

Both pins are ahead of upstream, which is otherwise unchanged; nothing else was
updated (NPM 2.16.0, nginx 1.31.8, certbot 5.9.0 and debian-base 9.5.2 are
already the latest).

## 4.2.0 (2026-11-08)

### ✨ New

- **API tokens in the user menu.** Open the user menu → "API Tokens", confirm
  your password (and two-factor code, when enabled), choose how long the token
  lasts (1 day to 3 years), and copy the token, which is shown once. Scripts
  send it as `Authorization: Bearer <token>`.
  - New patch `0005-Patch-add-API-tokens-to-the-user-menu`: `POST /api/tokens`
    and `POST /api/tokens/2fa` accept an optional `expiry` (`1d`, `7d`, `30d`,
    `90d`, `1y`, `2y`, `3y`). Logins without it still last 1 day.
  - Tokens aren't stored, so they can't be listed or revoked one by one.
    Changing the user's password ends every token issued before it, and
    disabling the user ends all of its tokens. Give scripts a dedicated user
    that only has the permissions they need.
  - A 2- or 3-year token can expire a few hours before the displayed time:
    upstream computes the token's expiry and the displayed date slightly
    differently.

### 🔒 Security

- `GET /api/tokens` (token refresh) no longer accepts `?expiry=`. Upstream let
  any logged-in session swap its token for one of any lifetime without the
  password. A refresh now always lasts 1 day; longer tokens need the password.

Upstream has an open proposal for fuller, stored and revocable tokens
(NginxProxyManager/nginx-proxy-manager#5975); this fork-only version adds no
database changes, so it can be dropped cleanly if upstream ships one.

## 4.1.0 (2026-11-07)

### ✨ New

- **Home Assistant ingress: the interface can now be shown in the sidebar.**
  Turn on "Show in sidebar" on the add-on page; the interface then opens inside
  Home Assistant, which also works over HTTPS and remote access, with no
  separate port needed. The admin port `81` keeps working as before.
  - New patch `0004-Patch-support-Home-Assistant-ingress`: the Nginx Proxy
    Manager frontend no longer assumes it is served at `/`. It builds with a
    relative base and reads its path prefix at runtime from an `ingress-path`
    meta tag; the API client, the router and the image paths use it.
  - New `rootfs/etc/nginx/conf.d/ingress.conf`: an nginx server on the ingress
    port `8099`, reachable only by the Supervisor, that serves the interface
    and fills the meta tag from the `X-Ingress-Path` header.
  - `config.yaml`: `ingress: true`, `ingress_port: 8099`, `panel_icon: mdi:web`.
    The `webui` link is replaced by ingress ("Open Web UI" now opens it there).

Upstream declined ingress support (hassio-addons/app-nginx-proxy-manager#649),
so this is a fork-only feature.

## 4.0.1 (2026-11-05)

### ⬆️ Upgrades

- Upgrade **certbot** `5.8.0` → `5.9.0` (installed in the `/opt/certbot` pip
  venv via `ARG CERTBOT_VERSION`), matching upstream.
- Upgrade base image **`ghcr.io/hassio-addons/debian-base`** `9.5.1` → `9.5.2`
  (`ARG BUILD_FROM`), matching upstream.
- No other dependency changed: nginx is already at the latest nginx.org mainline
  `1.31.8`, and the Nginx Proxy Manager app stays at `2.16.0`.

## 4.0.0 (2026-10-24)

Re-sync with the re-architected upstream add-on. Upstream solved the NPM
v2.13+ frontend rewrite (webpack → Vite) by **rebasing the whole add-on from
Alpine onto Debian**; this release adopts that work.

### ⚠️ Breaking / major changes

- **Base image switched from Alpine to Debian.** Now builds on
  `ghcr.io/hassio-addons/debian-base:9.5.1` (Debian trixie) instead of
  `ghcr.io/hassio-addons/base` (Alpine). Supported architectures are unchanged
  (`aarch64`, `amd64`).
- **Nginx Proxy Manager app `v2.12.4` → `v2.16.0`** (frontend is now Vite +
  React 19 + TypeScript).
- **nginx `1.30.2` (Alpine, stable) → `1.31.8` (nginx.org mainline Debian repo)**.
  The stream module is compiled in statically, so `nginx-mod-stream` is gone.
- **Node.js 24 (Alpine) → 22 (NodeSource)**; the old-webpack
  `NODE_OPTIONS=--openssl-legacy-provider` workaround is removed (Vite build).
- **certbot** is now installed in a pip venv at `/opt/certbot` (certbot 5.8.0),
  pinned via `PIP_CONSTRAINT`, instead of the Alpine `certbot` package.
- Dependencies are now **Debian `apt` packages** (version-pinned to Debian
  trixie) instead of Alpine `apk` pins.

### 🔁 Patches

- Replaced the 6 Alpine-era patches with upstream's current **3**:
  `0001-Patch-data-to-config-folder`, `0002-Patch-keep-log-files-out-of-the-config-share`,
  `0003-Patch-certbot-plugin-install-failures-at-startup`. The old node-sass /
  webpack / sass-loader patches are obsolete under Vite. The logrotate and
  nginx `root`-user fixes are now carried by upstream itself.

### 🗂 Fork specifics kept

- Root `repository.yaml` (name "Nginx Proxy Manager APP"), `config.yaml` `url`
  pointing at this fork, and the single-`ARG BUILD_FROM` base (no `build.yaml`).
- `config.yaml` `map:` uses the current spec's object form
  (`- type: app_config` / `read_only: false`) instead of upstream's deprecated
  short form `- addon_config:rw`, clearing the Supervisor deprecation warning.
  Same mount point and data location (slug unchanged).

## 3.0.1 (2026-06-20)

### 🧹 Config modernisation (clears Supervisor deprecation warnings)

- Remove `build.yaml`. Both supported architectures use the same base image,
  which is now declared by the `ARG BUILD_FROM` default in the `Dockerfile`,
  as required by current Supervisor (`build.yaml` is deprecated).
- Remove the deprecated `codenotary` field from `config.yaml` (Supervisor now
  ignores it).
- Point `config.yaml` `url` at this fork.

### 🐛 Fixes

- Fix recurring logrotate error `unknown group 'npm'`. The add-on runs its
  services as `root` and never creates the upstream `npm` user/group, but the
  logrotate config shipped by NPM declares `su npm npm`. Rewrite it to
  `su root root` during build (same approach already used for `nginx.conf`),
  so nginx access/error logs in `/data/logs` rotate instead of being skipped.

## 3.0.0 (2026-06-20)

### ⚠️ Breaking changes

- **Dropped `armv7` (32-bit ARM) support.** The Home Assistant base image
  no longer ships 32-bit builds from version 19 onward, so moving to a newer
  Alpine release means 32-bit ARM can no longer be built. Devices that need
  `armv7` must stay on the **2.x** series. Supported architectures are now
  `aarch64` and `amd64` only.

### ⬆️ Upgrades

- Upgrade base image `ghcr.io/hassio-addons/base` from `18.2.0` to `21.0.0`
  (Alpine 3.22 → 3.24).
- Upgrade **nginx** `1.28.0-r3` → `1.30.2-r1` (and `nginx-mod-stream` to match)
  — moves onto the current nginx **stable** branch.
- Upgrade **Node.js** `22.16.0-r2` → `24.16.0-r0`.
- Upgrade **npm** `11.3.0-r1` → `11.12.1-r0`.
- Upgrade **Python** `3.12.12-r0` → `3.14.5-r0` (`python3` and `python3-dev`).
- Upgrade **py3-pip** `25.1.1-r0` → `26.1.2-r0`.
- Upgrade **certbot** `4.0.0-r0` → `5.6.0-r0`.
- Upgrade **OpenSSL** `3.5.4-r0` → `3.5.7-r0` (`openssl` and `openssl-dev`).
- Upgrade `build-base` `0.5-r3` → `0.5-r4`, `git` `2.49.1-r0` → `2.54.0-r0`,
  `libffi-dev` `3.4.8-r0` → `3.5.2-r1`, `apache2-utils` `2.4.65-r0` →
  `2.4.67-r0`, `libcap` `2.76-r0` → `2.78-r0`, `logrotate` `3.21.0-r1` →
  `3.22.0-r0`.
