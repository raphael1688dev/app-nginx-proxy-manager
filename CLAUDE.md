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
  (the 3 `patches/*.patch` are written against a specific NPM version and the
  Vite build may shift). Process: bump the ARG → build → fix any failing patch
  → verify on hardware. The simplest safe path for a big app jump is to re-sync
  from upstream `hassio-addons/addon-nginx-proxy-manager` (see below).

- **Prefer re-syncing from upstream over hand-porting.** Upstream does the hard
  work (frontend rewrites, patch maintenance, dep pins). When catching up, copy
  upstream's `proxy-manager/{Dockerfile,patches,rootfs,translations,.README.j2,DOCS.md}`
  wholesale, then re-apply the fork specifics below. That is exactly how v4.0.0
  was produced.

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
- Fork meta not present upstream: `CLAUDE.md`, `.gitignore`, root `README.md`
  "Supported architectures" note, `proxy-manager/CHANGELOG.md`.

[npm]: https://nginxproxymanager.com/
