# Changelog

All notable changes to this add-on are documented in this file.

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
