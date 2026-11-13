# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this repo is

CleanYourTV strips bloatware from Android TV / Google TV over Wi-Fi using standard ADB tooling (`pm disable-user`), then installs a small native launcher styled after Apple TV. Nothing is rooted, uninstalled, or flashed — everything is reversible with `tools/restore.sh`.

## Two modes of operation

| You were asked to | What to do |
|---|---|
| **Clean up a real TV** | Follow `PROMPT.md` exactly. Use existing `tools/` and `launcher/`, only edit `launcher/.../values/config.xml` or a theme's `config.json`. Do not modify repo docs, tests, or other code. |
| **Change this repo** | Read `AGENTS.md` for the full rules, then the relevant workflow in `docs/workflows/`. |

## Build and test commands

### Python tools setup
```bash
python3 -m venv tools/venv && tools/venv/bin/pip install -r tools/requirements.txt
```

### Run tests (as CI runs them)
```bash
tools/venv/bin/python -m unittest discover -s tests -v
```
CI also sets `CLEANYOURTV_STRICT=1`. On Windows, tests that drive `tools/*.sh` are skipped (need POSIX shell).

### Build the launcher
Requires JDK 17–22 and Android SDK platform 34 with build-tools 34.0.0+.
```bash
# Build and install to a TV:
TV_HOST=<ip> launcher/build-and-install.sh

# Build only (no TV needed):
cd launcher && ./gradlew --no-daemon assembleRelease

# Full check before a launcher PR:
cd launcher && ./gradlew --no-daemon testReleaseUnitTest lintRelease assembleRelease
```
Lint must have zero errors. APK must be under 576 KB (589,824 bytes).

Without JDK/SDK locally: `tools/docker-build.sh launcher` builds in Docker.

On macOS: `brew install openjdk@17 android-commandlinetools`, then `sdkmanager "platforms;android-34" "build-tools;34.0.0"`. Set `JAVA_HOME=$(brew --prefix openjdk@17)`.

### Generate TV tables
```bash
tools/venv/bin/python -I tools/gen_tables.py
```
The brand grid in READMEs and the table in `examples/README.md` are generated from `examples/*/tv.json`. Never edit those tables by hand — they are regenerated after merging.

## Architecture

- **`tools/`** — Python ADB client (`tv.py`), debloat/restore shell scripts (`disable.sh`, `restore.sh`), and utilities. Shell scripts have PowerShell twins (`.ps1`). `tv.py` is a pure-Python ADB client that works where the stock `adb` binary is blocked.
- **`launcher/`** — Android TV launcher app. Package `com.gathin.tvhome`. Plain Java, AndroidX RecyclerView + DynamicAnimation. No Kotlin, no Compose, no `INTERNET` permission, no background services. `minSdk` 26. Config lives in `app/src/main/res/values/config.xml`.
- **`themes/`** — Each folder has a `config.json` + required `screenshot.jpg`. Pushed to TV at runtime via `tools/theme.sh`, no rebuild needed. Keys documented in `themes/README.md`.
- **`addons/`** — Optional module-card apps, each its own Gradle project, never built into the launcher.
- **`examples/`** — One folder per tested TV model with `debloat.txt` and `tv.json`. The source of truth for the generated tables.
- **`tests/`** — Python `unittest` suite covering list format, README tables, themes, add-ons, Home guard logic, and tool behavior against fake TVs.
- **`docs/workflows/`** — Step-by-step recipes: `tv-list.md`, `launcher-switch.md`, `translation.md`, `theme.md`, `addon.md`, `showcase.md`, `press.md`, `docs-notes.md`, `input-sources.md`, `perf-ab.md`.

## Key constraints

- **Disable only, never uninstall.** `pm disable-user --user 0`. Everything must be undoable by `tools/restore.sh`.
- **No network in the launcher.** No `INTERNET` permission, no analytics, no background services, no blur, no video. Targets 1 GB RAM TVs.
- **No new launcher dependencies** without an issue discussion first.
- **No prebuilt APKs or binaries** in the repo.
- **New behavior is off by default.** A switch must be opt-in via theme or `config.xml`.
- **Never disable Android core packages** (permission controller, package installer, ext services/shared, module metadata, systemui, shell, phone, settings, tv.settings, `android`, most `com.android.providers.*`). The TV may not boot. `tools/disable.sh` refuses such lists.
- **Protected packages in scripts and `PROMPT.md` safety rules** are maintained by the maintainer only — do not modify in a PR.
- **APK size gate:** 576 KB max. CI enforces this.
- **No secrets/identifiers in commits:** no IPs, Wi-Fi names, emails, serial numbers, Android IDs, MAC addresses, or ADB keys.

## Commit style

Titles: `area: what changed (optional #issue)` — e.g. `examples: add Sony X90L, Android TV 12`, `Launcher: show_inputs_pill, to hide the Inputs pill`. Areas: `examples`, `Launcher`, `docs`, `tools`, `tests`, `themes`, `addons`, `SHOWCASE`, `README`, `Press`.

## CI

- `.github/workflows/ci.yml` — tests, launcher build, APK size gate
- `.github/workflows/addons.yml` — add-on builds
- `.github/workflows/maintainer-bot.yml` — PR review bot applying `.github/maintainer/MAINTAINER.md`
