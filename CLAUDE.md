# CLAUDE.md

Guidance for Claude Code when working in this repository.

## What this is

Home Assistant Community Add-on that packages [Nginx Proxy Manager (NPM)][npm]
as a Home Assistant add-on. It is a fork of
`hassio-addons/addon-nginx-proxy-manager` (originally by Franck Nijhof).

The add-on lives entirely under [`proxy-manager/`](proxy-manager/):

| File | Purpose |
|------|---------|
| `Dockerfile` | Builds the image: pulls the NPM release, applies `patches/`, builds frontend/backend, installs nginx + certbot. |
| `config.yaml` | Add-on manifest: `version`, supported `arch`, ports, mappings. |
| `patches/*.patch` | Patches applied to the upstream NPM source during build. |
| `rootfs/` | Files baked into the container (s6 services, scripts). |
| `translations/` | i18n. |
| `CHANGELOG.md` | Per-version change history. |

## Critical constraints

- **Alpine package pins are exact and Alpine-version-specific.** Every `apk add`
  in the `Dockerfile` pins `name=x.y.z-rN`. These strings are valid only for the
  Alpine release of the chosen base image. **Bumping the base image major version
  means re-pinning every package** to the versions that exist in the new Alpine
  branch (look them up at <https://pkgs.alpinelinux.org/packages>), or the build
  fails. `nginx` and `nginx-mod-stream` must always be the **same** version.

- **Base image ↔ Alpine ↔ architecture mapping** (source repo:
  `hassio-addons/addon-base`):
  - base `18.x` → Alpine 3.22, supports `aarch64`/`amd64`/`armv7` (+ armhf/i386)
  - base `19.x`+ → **32-bit arches dropped**; `aarch64`/`amd64` only
  - base `21.0.0` → Alpine 3.24 (nginx 1.30.x)
  - **There is no base image that keeps `armv7` AND ships Alpine newer than
    3.22.** Newer nginx requires dropping 32-bit ARM.
  - The base image is set by `ARG BUILD_FROM` in the `Dockerfile`. There is **no
    `build.yaml`** (Supervisor deprecated it); both arches use the same multi-arch
    tag, so the single ARG default covers them. To bump the base, edit that ARG.

- **nginx version policy:** prefer the **stable** branch (even minor, e.g.
  `1.30.x`) over mainline (odd minor, e.g. `1.31.x`) for this production proxy.
  Note the Dockerfile installs nginx from Alpine's repo, which only ships stable
  — not from nginx.org.

- **Node.js + legacy OpenSSL:** the NPM frontend builds with an old webpack, so
  `NODE_OPTIONS="--openssl-legacy-provider"` is required. Keep the comment's Node
  major version in sync with the pinned `nodejs` package.

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

## Repo / git

- Remote `origin` → `https://github.com/raphael1688dev/app-nginx-proxy-manager.git`
  (default branch `main`). Authenticated via `gh` as account `raphael1688`.
- Commit/push only when the user asks.

[npm]: https://nginxproxymanager.com/
