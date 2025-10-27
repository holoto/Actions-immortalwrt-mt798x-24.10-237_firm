# Guidance for AI coding agents — Actions-immortalwrt-mt798x-24.10-237_firm

This repository is a fork/template for building OpenWrt/ImmortalWrt images using GitHub Actions (derived from P3TERX/Actions-OpenWrt). The following notes explain the project's structure, important workflows, conventions, and small examples that help an AI agent be productive quickly.

Keep the instructions concise and factual. Do not suggest changes that require secrets or external network access unless the user requests it.

## Big picture
- Purpose: produce OpenWrt/ImmortalWrt firmware artifacts via GitHub Actions. The build is driven by a remote OpenWrt source repository (configurable) and local `*.config` files pushed into this repo.
- Primary components:
  - `.github/workflows/image-builder.yml` — reusable build workflow (core build steps: init environment, clone upstream OpenWrt, load feeds, copy config/files, download packages, compile, upload artifacts).
  - Per-image workflow files (examples: `360t7-24.10-6.6.yml`, `cudy-tr3000.yml`) that call the reusable builder and set env inputs.
  - `diy-part1.sh` and `diy-part2.sh` — hook scripts executed before/after feeds update to tweak feeds and packages.
  - `*.config`, `*.sh`, `*.md` files (e.g., `360t7-24.10-6.6.config`, `360t7-24.10-6.6.sh`, `360t7-24.10-6.6.md`) — per-target configuration, customizations, and release notes.

## Key files to reference when making changes
- `README.md` — high-level usage and intent (template for Actions-OpenWrt).
- `.github/workflows/image-builder.yml` — canonical build flow and environment setup; copy examples from here.
- `.github/workflows/<target>.yml` — per-device invocation; these set `REPO_URL`, `REPO_BRANCH`, `CONFIG_FILE`, `DIY_P2_SH`, etc.
- `diy-part1.sh`, `diy-part2.sh`, and per-target `*.sh` — where feed/package and makefile tweaks happen. Changes here directly affect the build inputs.
- `files/` — contents moved into OpenWrt `files/` before build; used for persistent custom files.

## Developer workflows & commands (local reproduction)
These commands are what the Action runs. Use them locally inside a clone of an OpenWrt source tree when reproducing builds:

- Initialize feeds and install:
  - ./scripts/feeds update -a
  - ./scripts/feeds install -a
- Load config and run make targets:
  - make defconfig
  - make download -j8
  - make -j$(nproc)  (fallbacks: `make -j1` or `make -j1 V=s` for verbose failures)

Notes for local runs: the workflow installs many packages and sets timezone. The builder expects an Ubuntu runner; reproducing on macOS may require Docker or an Ubuntu VM.

## Project-specific conventions & patterns
- Workflows pass build inputs through `env:` variables and `workflow_call` inputs in `image-builder.yml`. When adding a new device workflow, mirror the pattern in `360t7-24.10-6.6.yml` and call the builder.
- Hooks: `DIY_P1_SH` runs before `./scripts/feeds update -a`, and `DIY_P2_SH` runs after feeds are installed and the repository files/config have been copied. Put feed additions in `diy-part1.sh`; package/theme changes or sed edits in `diy-part2.sh` or device-specific script.
- Artifact organization: after build, the workflow sets `FIRMWARE` to `openwrt/bin/targets/*/*` and uploads that directory. Avoid touching `openwrt/bin` unless intentionally reorganizing outputs.
- Filename/date stamping: `include/image.mk` is patched in `360t7-24.10-6.6.sh` to include date in IMG_PREFIX — device scripts sometimes edit build Makefiles.

## Integration points and external dependencies
- Upstream source (REPO_URL / REPO_BRANCH) is cloned during the workflow; changes in repository layout upstream can break the builder.
- External package/theme repositories are cloned in device scripts (e.g., `luci-theme-argon`, `openwrt-passwall-packages`). Ensure URLs/branches exist before adding them.
- GitHub Actions secrets are used only for release upload (GITHUB_TOKEN provided by runner). Do not attempt to embed other secrets in the repo.

## Examples to quote in suggestions
- To add a feed, use the pattern in `diy-part1.sh` (append to feeds.conf.default or clone package repos into `package/` or `feeds/`):
  - echo 'src-git helloworld https://github.com/fw876/helloworld' >>feeds.conf.default
- To force-replace a feed or package, device scripts use `rm -rf` on `feeds/...` and `git clone` into `package/...` as shown in `360t7-24.10-6.6.sh`.

## When editing workflows or scripts, check these failure modes
- Build fails due to missing prerequisites: Actions installs many apt packages — locally you must match that environment or use runner images.
- Downloads missing in `dl/`: `make download` may leave small or corrupted files; the workflow removes files < 1KB. When diagnosing, inspect `openwrt/dl` and rerun `make download`.
- Device detection: `.config` must have CONFIG_TARGET...DEVICE...=y entries. The workflow extracts `DEVICE_NAME` by grepping `.config` — if no match, uploads will name the device `unknown`.

## What an AI agent should do (practical guardrails)
- Prefer small, incremental changes: modify device scripts or `diy-part*.sh` instead of the core `image-builder.yml` unless necessary.
- When proposing to change network URLs, point to the exact line in the device script (e.g., the `git clone` line in `360t7-24.10-6.6.sh`).
- Never hardcode secrets or tokens in files. Use workflow inputs/Secrets if the user asks to wire them.

## Useful file references (quick links)
- `.github/workflows/image-builder.yml` — main build flow
- `.github/workflows/360t7-24.10-6.6.yml` — example device workflow
- `diy-part1.sh`, `diy-part2.sh`, `360t7-24.10-6.6.sh` — hooks and package tweaks
- `README.md` — project description and template origin

If any section is unclear or you want more device-specific examples (e.g., how passwall packages are swapped), tell me which target and I will expand the instructions and add a short checklist for testing changes locally.
