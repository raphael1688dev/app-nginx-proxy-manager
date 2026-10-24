# Changelog

All notable changes to this add-on are documented in this file.

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
