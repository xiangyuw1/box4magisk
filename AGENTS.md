# AGENTS.md

## What this is

Magisk / KernelSU / APatch module that deploys proxy cores (sing-box, clash, mihomo, xray, v2ray, hysteria) on Android with transparent proxy (TPROXY/REDIRECT/TUN). Fork of CHIZI-0618/box4magisk.

Two independent codebases in one repo:
- **Shell scripts** (`box/scripts/`, `customize.sh`, `box4_service.sh`) — run on-device as root
- **WebUI** (`webui/`) — React+TypeScript SPA, builds into `webroot/` (gitignored)

## Build

```bash
# WebUI (must build first — build.sh packages webroot/)
cd webui && npm ci && npm run build && cd ..

# Module ZIP (excludes .git, .github, build.sh, box4.json, webui/, ui.ts)
sh build.sh
# Produces: box4_v<version>.zip
```

CI does exactly this in `.github/workflows/release.yml` and `prerelease.yml` (Node 20).

## Architecture

### On-device layout (`/data/adb/box/`)

```
box/
  bin/           # proxy core binaries (user-provided, not in repo)
  scripts/       # all runtime scripts
    box.config   # core selection, ports, clash API
    tproxy.conf  # transparent proxy config (interfaces, modes, app proxy)
    box.service  # start/stop/restart/status for the proxy core
    box.tproxy   # iptables/ip rules setup (~1400 lines)
    box.webui    # JSON CLI for WebUI backend
    start.sh     # boot entry, waits for unlock + boot_completed
    *.inotify    # file watchers for module enable/disable, network changes
  run/           # logs, PID files, runtime state
  <core>/        # per-core config dir (sing-box/, mihomo/, clash/, etc.)
```

### Boot flow

1. `box4_service.sh` (installed to `/data/adb/service.d/`) waits for `sys.boot_completed`
2. Calls `start.sh` → `box.service start` → `box.tproxy start`
3. `box.inotify` watches module dir for enable/disable (no reboot needed)

### WebUI ↔ Shell bridge

`box.webui` is a CLI that emits JSON. The React app calls it via KernelSU's `execCommand` API. Commands: `status`, `get-config`, `set-config`, `toggle`, `service`, `tproxy`, `download-cores`, `capabilities`, etc.

## Key conventions

- **Shell scripts use `#!/system/bin/sh`** or `#!/sbin/sh` (Android shell, not bash). No bashisms — avoid arrays, `[[`, `local` keyword is supported by mksh but test carefully.
- **Shell scripts run as root** with `PATH` extended for Magisk/KernelSU/APatch/Termux binaries.
- **Config files are sourced as shell** — `box.config` and `tproxy.conf` are `source`d, not parsed.
- **`CORE_USER_GROUP` must match** between `box.config` (`box_user_group`) and `tproxy.conf` (`CORE_USER_GROUP`). Both default to `root:net_admin`.
- **WebUI uses `@/` path alias** → `./src/` (configured in `vite.config.ts`).
- **WebUI builds to `../webroot`** with `base: './'` for KernelSU relative path support.
- **`webroot/` is gitignored** — it's a build artifact, never edit directly.
- **`box4.json`** is the online update manifest (`version`, `versionCode`, `zipUrl`, `changelog`). Update both this and `module.prop` for releases.
- **`module.prop` `versionCode`** uses date format `YYYYMMDD`.

## Modifying shell scripts

- `box.tproxy` is the most complex file. It handles iptables chain setup, ip rules, DNS hijack, per-app proxy, MAC filtering, CN IP bypass, IPv6, and performance mode. Changes here affect all proxy modes.
- `box.service` handles core lifecycle: permission checks, config validation, process management, TUN hotspot forwarding, private DNS save/restore.
- Test on a real device or emulator — iptables behavior can't be verified locally.
- DRY_RUN mode exists (`DRY_RUN=1` in tproxy.conf) — logs commands without executing.

## Modifying WebUI

- Stack: React 19, TypeScript 5.9, Vite 8, TailwindCSS 4, `kernelsu` package
- Run `npm run dev` in `webui/` for local development
- Lint: `npm run lint` (ESlint with react-hooks + react-refresh plugins)
- Type check: `tsc -b` (part of `npm run build`)
- The WebUI communicates with the shell backend via `box.webui` CLI commands

## Magisk / KernelSU / APatch differences

- **KernelSU < 10683**: service dir is `/data/adb/ksu/service.d`
- **KernelSU >= 10683 or Magisk/APatch**: service dir is `/data/adb/service.d`
- `customize.sh` renames `module.prop` name to `box4KernelSU` or `box4APatch` accordingly
- Module ID is always `box4` (in `module.prop`)

## Common pitfalls

- Don't edit files under `webroot/` — they get overwritten by `npm run build`
- Shell scripts use `busybox setuidgid` to drop privileges — the proxy core runs as `root:net_admin`, not as root
- `box.inotify` uses `source` not `.` — this is mksh-specific, both work on Android but be consistent
- The `awk '!x[$0]++'` pattern in `customize.sh` deduplicates config lines during update — don't break this
- Port conflicts: `PROXY_TCP_PORT`, `PROXY_UDP_PORT`, `DNS_PORT`, `clash_api_port` must all be different from each other
