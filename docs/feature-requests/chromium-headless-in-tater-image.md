# Feature request: Chromium (headless) in the Tater container image

**Filed by:** jarvis_screen core maintainer · **Date:** 2026-09-21
**Upstream repo:** TaterTotterson/Tater (`Dockerfile`, `Dockerfile.nvidia`)

## The ask

Add a headless-capable Chromium browser (and its runtime shared libraries) to
the Tater container image, e.g.:

```dockerfile
RUN apt-get update && apt-get install -y --no-install-recommends \
    chromium \
 && rm -rf /var/lib/apt/lists/*
```

(on debian slim, `chromium` pulls `chromium-common`/`chromium-sandbox` and the
needed `libnss3`/`libx11-*`/`libgbm` set; add `--no-sandbox`-compatible
packaging is preferred inside containers). Roughly +250–400 MB image size.

## Why

The `jarvis_screen` core's Web Browser card is gaining a "real browser" render
mode (v1.57.0): the core drives a headless Chromium over the DevTools protocol
(CDP) — screenshots stream into the card, taps/typing are forwarded as trusted
input, site cookies are persisted per card, and the page's text becomes
available to voice turns ("what's on the page?").

This mode only works when a Chromium binary exists in the container. Today the
image (`python:3.11-slim` + a fixed apt list: ffmpeg, build tools, audio libs)
ships no browser at all, so the core falls back to a same-origin proxy mode
(pure Python) that bot-checked / SPA-heavy public sites resist.

## What works without it (already shipped)

- **proxy mode** — same-origin rewrite proxy with a server-side cookie jar
  (zero new dependencies; fine for simple server-rendered sites).
- **direct mode** — plain iframe embed (sites that allow framing only).
- **external CDP endpoint** — a core setting (`WEB_CDP_ENDPOINT`,
  `ws://host:port`) lets the core attach to any reachable headless Chromium
  outside the container, no image change needed. The local-binary detection
  (`shutil.which`) picks the in-container browser automatically once the image
  ships one — no further core work required.

## Ask summary

1. Add `chromium` (+ deps) to `Dockerfile` / `Dockerfile.nvidia`.
2. Nothing else — the core auto-detects the binary and enables the mode.
