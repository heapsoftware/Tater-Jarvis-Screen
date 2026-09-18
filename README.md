# Tater Jarvis Screen Core

`cores/jarvis_screen_core.py` — a Tater core hosting a full-screen Iron-Man
style "JARVIS SCREEN" web display (default port 8610) controlled by Hydra:

- Generative card UI (text/web/YouTube/HA climate/camera/video/chart/console/
  music — the music card mirrors and controls the Tater music cores' playback,
  including browser audio)
  with per-screen profiles, layouts, and a visual "Edit Layout on Screen"
  placement editor.
- Arc reactor that pulses with TTS audio; red on access-denied.
- Per-screen unlock methods: Tater Face ID camera scan, tap, 4-digit PIN, or
  "hey jarvis" voice, with optional per-screen fail/success/unlock voices.
- In-browser voice input: tap-to-talk, openWakeWord WASM wake word, follow-up
  conversations, per-screen mic capture tuning (echo cancellation / noise
  suppression / auto gain / rate / size).
- Live MJPEG camera feeds (Home Assistant proxy or a UniFi Protect direct
  pipeline) and camera-event automations (fullscreen popups + spoken
  announcements).
- Optional per-screen IP binding (a display on a bound IP always opens that
  screen, overriding `?screen=`), with opt-in `X-Real-IP`/`X-Forwarded-For`
  trust behind a trusted LAN reverse proxy.

> The browser mic/camera need a secure context — `https://` or `localhost` —
> so a plain-http LAN origin must be allowlisted or put behind TLS.
> Full install, configuration, and usage instructions:
> [`docs/jarvis_screen.md`](docs/jarvis_screen.md).

## Install / update on a Tater host

**Store install (recommended).** On the host's Cores page, add a core shop
source with this manifest URL:

```
https://raw.githubusercontent.com/heapsoftware/Tater-Jarvis-Screen/main/core_manifest.json
```

The core then appears in the core shop from this repo; install and later
updates happen through the Cores page (**Update** button), which downloads the
`.py`, sha256-verifies it against this repo's manifest, and restarts the core
runtime. The manifest is kept in sync with the committed core on every
release — the release workflow verifies the manifest's version and sha256
against `cores/jarvis_screen_core.py` before publishing and fails loudly on a
mismatch, so the Update button never rolls the core back. If a core file is
ever missing at boot, Tater's auto-restore re-downloads it from this store.

**Manual install.** Copy `cores/jarvis_screen_core.py` from a release asset
(or from a fresh clone) into the container's cores directory. Do **not** use
the Cores-page Update button for this unless the store source above is
configured — with a stale store manifest it copies the older store-side file
and rolls the core back. The host reads the installed version from the file
itself; `core_manifest.json` regeneration is only needed when adding a
brand-new core, never for routine updates.

## Versioning

The version lives in `__version__` at the top of
`cores/jarvis_screen_core.py`. Every functional fix ships with a bump; pushes
to `main` are tagged `v<version>` and published as GitHub Releases with
auto-generated changelogs by `.github/workflows/release.yml`.