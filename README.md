# Jarvis Screen — Setup & Configuration

Jarvis Screen is a Tater **core** that hosts a full-screen, Iron-Man style
"JARVIS SCREEN" web display that Hydra (the LLM) drives generatively: it
creates and arranges cards, switches layouts, moves the arc reactor, speaks
through the screen, raises alerts, and locks/unlocks displays.

- **Core file:** `cores/jarvis_screen_core.py` — a single self-contained file
  (~23 MB; the whole frontend, vendored three.js, and the in-browser voice
  stack (onnxruntime-web + openWakeWord models) are embedded in it). One file
  is the entire install.
- **Core id / module key:** `jarvis_screen` / `jarvis_screen_core`.
- **Requires:** Tater.
- **Screen server:** port **8610** by default (configurable). The stock Tater
  compose file uses `network_mode: host`, so the screen is reachable on the
  LAN at `http://<docker-host>:8610/?screen=<profile>`.
- **State:** all persistent state lives in Redis under the `jarvis_screen:*`
  namespace (plus the host-managed `jarvis_screen_core_settings` hash).
- **As of v1.49.1.**

---

## What you get on first boot

The core seeds itself with two screen profiles and four layouts, so a default
install already renders:

| Seed | What it is |
|---|---|
| Profile `main` | "Main Display" — touch, **face** unlock, browser audio, re-locks after 300 s idle |
| Profile `desktop` | "Desktop" — desktop mode, **tap** unlock, remote audio, no auto re-lock |
| Layout `hero` | The choreographed lock screen (empty card grid) |
| Layout `home` | Default unlock layout: video + chart + JARVIS chat cards, stock ticker |
| Layout `security` | Front-door camera + door-log cards |
| Layout `greeting` *(built-in)* | Reactor + a greeting card that **speaks a line on unlock, then opens the mic**; re-merged on every load, never deletable |

Everything below those seeds is configurable.

---

## Getting the core into Tater

Tater discovers cores from `/app/cores` inside the container (overridable via
`TATER_CORE_DIR`). The Web UI core list **re-scans on every request** — a new
`*_core.py` dropped into `/app/cores` appears immediately, no restart needed.
Starting the core also sets an autostart flag, so it comes back on every
container start.

**The one caveat that shapes every option:** the stock compose file does
*not* volume-mount `/app/cores`. Anything installed into the container's
writable layer is lost when the container is **recreated** (`up --build`,
`--force-recreate`, host upgrade). For a 24/7 display use Option 3.

### Option 1 — `docker cp` (fastest, non-persistent)

```bash
# from this checkout, on the docker host:
docker cp cores/jarvis_screen_core.py tater_app:/app/cores/
```

Good for trying it out or patching a running deployment. Re-copy after every
core update; after a container recreation the core is gone.

### Option 2 — Core Shop (if you host this shop on GitHub)

If this checkout (with a matching `core_manifest.json`) is the repo Tater's
Core Shop points at, **Jarvis Screen** appears in Core Shop in the Web UI and
installs with one click (sha256-verified against the manifest). Same
persistence caveat as Option 1.

### Option 3 — Volume mount (recommended for production)

Mount a host directory over `/app/cores` so the file survives rebuilds:

```bash
mkdir -p ~/tater/cores
cp Tater_Shop/cores/jarvis_screen_core.py ~/tater/cores/
docker cp tater_app:/app/cores/__init__.py ~/tater/cores/
```

```yaml
# docker-compose.yml (keep the existing volumes)
volumes:
  - ${TATER_CORES_PATH:-./cores}:/app/cores   # host side = ~/tater/cores
```

```bash
docker compose up -d
```

Start the core once from the Web UI — it now autostarts forever, and updates
are just "replace the file, stop + start the core."

Then, in the Tater Web UI (`http://<docker-host>:8501`):

1. Open **Cores** — flip **Jarvis Screen** ON (or
   `curl -X POST http://<docker-host>:8501/api/cores/jarvis_screen_core/start`).
2. Open the screen: **`http://<docker-host>:8610/?screen=main`**.

### Verifying it's running

```bash
curl -s http://<docker-host>:8501/api/cores | python3 -m json.tool | grep -A3 jarvis_screen
curl -s http://<docker-host>:8610/api/health          # liveness
docker logs tater_app 2>&1 | grep -i "jarvis"
```

In a browser you should see the boot sequence, then the hero reactor idling on
the navy grid; after ~25 s of idle time on a locked screen it morphs into the
dormant clock disc.

---

## Setup checklist (after install)

1. **Core settings** — Tater Settings, category **"Jarvis Screen Settings"**
   (reference below). At minimum: confirm `SCREEN_PORT`, decide on
   `SCREEN_TOKEN` (leave empty for open-on-LAN).
2. **The Jarvis Screen manager tab** — the core's own Web UI tab:
   **Overview / Cards / Layouts / Screens / People / Automations / Voice**.
   Build the flow in that order: define *cards*, arrange them into
   *layouts*, then attach layouts to *screens* (each screen = one physical
   display).
3. **Speech backends** — a **TTS** backend (spoken replies, `jarvis_screen_say`,
   greeting/fail/success lines) and, for voice input, a **local STT** backend
   in Tater's speech settings. Without STT, voice screens report
   `VOICE UNAVAILABLE — NO STT BACKEND`.
4. **Face ID** (only for face unlock) — enroll faces in Tater's own
   **Settings › People › Faces** (camera or photo upload), and link each face
   to a Tater Person. Only linked faces unlock.
5. **Secure context for mic/camera** — browsers only allow `getUserMedia` on
   `https://` or `localhost`. For a kiosk on a plain-http LAN origin, put a
   TLS reverse proxy in front (see [Secure context](#secure-context-mic--camera)),
   or allowlist the origin for Chromium.

---

## Core settings reference

Tater Settings → **Jarvis Screen Settings**. These are global (per-core)
settings; per-screen overrides live on the manager's **Screens** tab.

### Network & access

| Setting | Default | Meaning |
|---|---|---|
| `SCREEN_PORT` | `8610` | Port of the screen webpage (`http://<host>:<port>/?screen=<profile>`). |
| `SCREEN_HOST` | `0.0.0.0` | Bind interface. `0.0.0.0` = reachable on the LAN. |
| `SCREEN_TOKEN` | *(empty)* | Optional shared secret. When set, the screen must be opened with `?token=<value>` and the state/event endpoints require it. |
| `TRUST_PROXY_HEADERS` | off | Take the client IP from `X-Real-IP` / `X-Forwarded-For` so per-screen **IP binding** works behind a trusted LAN reverse proxy. Keep off when the port is reachable by untrusted clients. |

### Face unlock

| Setting | Default | Meaning |
|---|---|---|
| `FACE_ENABLED` | on | Use Tater Face ID to unlock touch screens. |
| `FACE_FAIL_LIMIT` | `3` | Consecutive failed scans before the red reactor + fail voice (1–10). |
| `FACE_FAIL_COOLDOWN_S` | `4` | Seconds the screen stays locked out after hitting the fail limit (0–300). |
| `FACE_UNLOCK_IDENTITY` | on | While a screen stays face-unlocked, its voice turns tell the LLM who unlocked it. Off = the built-in voice pipeline (voiceprints) is the identity source instead. |

### Voice & cameras

| Setting | Default | Meaning |
|---|---|---|
| `VOICE_HISTORY_COUNT` | `12` | Past voice exchanges per screen replayed to the LLM as history (0–50; 0 = every turn starts fresh). The Voice tab can clear stored history. |
| `CAMERA_REFRESH_S` | `5` | Default snapshot refresh for `ha_camera` cards in snapshot mode (per-card `refresh_s` overrides). |
| `LIVE_FPS` | `8` | Frame rate for live-feed camera cards: 5 / 8 / 10 / 12 / 15 fps. 5–8 is the kiosk sweet spot; 10–15 suits desktops. |
| `LIVE_MAX_STREAMS` | `3` | Cap on simultaneously running live feeds (the same camera on several cards/screens counts once). |

### UniFi Protect (direct camera pipeline)

| Setting | Default | Meaning |
|---|---|---|
| `PROTECT_BASE_URL` | *(empty)* | e.g. `https://192.168.1.1` (`/proxy/protect` appended if absent). **Only used when Tater's own UniFi Protect integration has no credentials stored** — the integration's settings are the primary source and are reused automatically. |
| `PROTECT_API_KEY` | *(empty)* | Protect Public API key (controller → Settings → Control Plane → Integrations). Same fallback rule. |

### AI card generation

These tune the cards Hydra *generates* at runtime (as opposed to the cards
you author in the manager — those are always exactly where you put them).

| Setting | Default | Meaning |
|---|---|---|
| `CARD_LAYOUT_MODE` | `auto_switch` | When a generated card already exists on another layout: **auto-switch** flips the screen to that layout and reuses the card; **guidance** instead lists the layout card inventory in the tool descriptions so the LLM switches layouts itself. |
| `CARD_LAYOUT_MATCH` | `broad` | How auto-switch matches "already exists": **broad** = by card id, or type + ref (e.g. the exact camera entity), or a waiting same-type slot with no ref yet; **strict** = type + ref must be identical. |
| `CARD_PERSIST_AI` | on | **On:** generated cards are kept on their screen until removed. **Off:** generated cards are session-only — they vanish when the screen page next loads, and saved layouts are never touched. Either way, a saved layout card the assistant re-targets is only *borrowed* for the session — a reload restores the saved layout. |
| `AI_YOUTUBE_AUTOPLAY` | on | YouTube players JARVIS generates (or re-points) start playing by themselves (browser permitting). Saved layout cards never autoplay on their own. |
| `CARD_DRAG` | `free` | Head-dragging cards in everyday use: **free** (anywhere, even over the reactor), **avoid_reactor** (drops over the reactor snap to the nearest clear spot), **off** (cards are fixed). Dragging outside edit mode is session-visual — use *Edit Layout on Screen* to save placements. |
| `AI_SIZE_<TYPE>` | *(blank)* | Default size for AI cards of each built-in type — `AI_SIZE_TEXT`, `AI_SIZE_WEB`, `AI_SIZE_YOUTUBE`, `AI_SIZE_VIDEO`, `AI_SIZE_HA_CAMERA`, `AI_SIZE_HA_CLIMATE`, `AI_SIZE_CHART`, `AI_SIZE_CONSOLE`, `AI_SIZE_GREETING`, `AI_SIZE_ENTITY_STATE`. Value is `'WxH'` percent of screen (e.g. `46x28`), or `'50%'` = take that share of the screen area still free around the current cards (auto-shaped to the largest open rect). Blank = the built-in per-type default; a size the LLM sends with the card still wins; re-pointed layout cards keep their saved size. |

### Home Assistant

| Setting | Default | Meaning |
|---|---|---|
| `HA_BASE_URL` | *(empty)* | e.g. `http://homeassistant.local:8123`. **Only used when the HA Tater integration has no token set** — the integration's URL/token are the primary source. |
| `HA_TOKEN` | *(empty)* | Fallback long-lived token for direct HA REST access; ignored while the integration is configured. |

---

## The Jarvis Screen manager tab

The core's Web UI tab (order 40). Tab order follows the setup flow:
**Overview → Cards → Layouts → Screens → People → Automations → Voice**.
Forms open in modals ("Manage" / "Add …").

### Overview

Live status (server running, URL, port, Face ID state), per-screen cards with
the full settings summary in detail lines, and actions: **Lock / Unlock** any
screen, and **Test fail voice** on face/PIN screens (plays the
access-denied path without a camera).

### Cards

"Add a Card" opens a template-based form. Templates:

| Template | Card type | Notes |
|---|---|---|
| Text / Notes | `text` | |
| JARVIS Chat | `console` | The chat card (id `jarvis_chat`) |
| Web Browser | `web` | JARVIS can retarget it to any site |
| YouTube Player | `youtube` | Ref accepts a video id or a full URL |
| Video Stream | `video` | Direct mp4/webm URL (cameras without an entity) |
| Chart | `chart` | |
| Console Log | `console` | System feed |
| JARVIS Greeting | `greeting` | Speaks its Say Line when the layout appears, then opens the mic |
| HA Climate / Thermostat | `ha_climate` | Needs an entity (picker) |
| Camera Feed | `ha_camera` | HA or UniFi Protect; snapshot or live MJPEG (picker) |
| Entity Status | `entity_state` | Sensor / speaker readouts (picker) |

Card fields: **Card Id** (JARVIS reaches the card by id — keep it meaningful,
e.g. `front_door`), **Title**, **Tag** (the `// TAG` header line), **Badge**,
**Footer**, **Ref** (url / youtube id / entity / stream), **Text / Say Line**,
**Unit**, **Grid Position** (named 3×3 spot: top-left … bottom-right),
**Preset Size** (small / medium / wide / tall / large) — both override the raw
**X / Y / W / H (%)** when set — plus **Refresh (sec)** for snapshot/entity
cards, **Camera Feed** (`snapshot` = JPEG loop over SSE, `live` = real-time
MJPEG stream) and **Feed source** (`auto` / Home Assistant / UniFi Protect)
for `ha_camera` cards, and **Visible**.

### Layouts

Named card arrangements screens can switch to (the built-in `greeting` layout
is read-only — it is part of the unlock flow). Each layout has a **Title**, a
**Ticker Tape Items** line, and per-card **Position / Size / Remove** controls
plus a quick-add row. New layouts can be duplicated from an existing one.

### Screens

One entry per **physical display**. Each screen card carries the
**"Edit Layout on Screen"** action (layout picker + button): it switches the
screen to that layout, opens the on-screen visual editor, and saves
drag/resize placements **for this screen only** (per-screen overrides on top
of the layout's base geometry — the layout itself is untouched).

The **Manage** form is the per-screen profile (the big one — reference below).

#### Per-screen profile fields

| Field | Default | Meaning |
|---|---|---|
| **Screen Key** | — | Lowercase id in the URL: `?screen=<key>`. Set at add time. |
| **Label** | = key | Display name. |
| **Mode** | `touch` | `touch` = unlocked via the Unlock Method; `desktop` = manual lock/unlock. Auto re-lock applies to both. |
| **Unlock Method** | `face` | `face` (Tater Face ID scan; falls back to tap when unavailable) / `tap` (any tap) / `pin` (4-digit PIN pad) / `voice` (say "hey jarvis" on the lock screen — requires Voice Input `wake`). |
| **Face Scan Mode** | `constant` | `constant` = the camera scans the whole time the screen is locked; `tap` = the camera only scans while you tap. |
| **Tap-Scan Auto-Retries** | `0` | For scan mode `tap`: automatic rescans after the tap's first failed scan (0–20; each retry waits out the fail cooldown first). |
| **PIN Code** | *(blank)* | The screen's own 4-digit code (blank = none). Unlocks are anonymous — use the People-tab PIN book to record *who* unlocked. |
| **Audio Engine** | `browser` | `browser` = the screen plays voice (reactor pulses from a live FFT) / `remote` = the screen is silent by design; TTS goes to the Tater speakers and the reactor follows a precomputed envelope. |
| **Docked Reactor Corner** | `bottom-right` | Where the reactor parks when unlocked (bottom-right / bottom-left / top-left / top-right). |
| **Reactor Style (normal / idle-clock)** | `classic` / `speckled` | The reactor look: normal — *Classic Film*, *E5 Nested Arcs*, *Design 3*; idle clock — *Speckled Disc*, *Azure Haze Ring*. |
| **Reactor Label** | on / on | Show the assistant word inside the reactor on the normal look / on the idle-clock look. |
| **Reactor Word Font** | `classic` | `classic` = light letterspaced wordmark (shows the screen's word); `jarvis` = the stylized JARVIS logo (the word is locked to JARVIS). |
| **Reactor Ring Drift** | `1.0` | Speed of the Design-3 ring/arc stack (0–2.5; 1 = reference pace, 0 = still). |
| **Reactor Build (sec)** | `1.92` | Length of the unlock build-in animation (0.8–5 s). |
| **Reactor Size** | `1.0` | Multiplier on the centered reactor (lock screen + greeting; 0.25–2.0). |
| **Pulse Sensitivity** | `1.0` | How strongly the reactor follows the audio (0.1–3.0). |
| **Idle Clock Delay (sec)** | `25` | Idle time before the lock view morphs into the dormant clock disc (0 = off, 0–3600). |
| **Clock Time / Date Format** | `24h` / `wk_day_mon` | Dormant-clock formats, grouped US / Europe-UK / Asia. |
| **Default Layout (on unlock)** | `home` | Where the screen lands after *any* unlock. Pick the built-in **Greeting** layout to have the reactor speak and open the mic. |
| **Layouts** | *(empty = all)* | Which layouts this screen offers, in nav-bar order (multi-select; `hero` is always added for the lock screen). |
| **Greeting Say Line** | *(blank)* | What a Greeting card says **on this screen** (overrides the card's own line). One saying per line; several = one chosen at random; `<user>` resolves to the unlocked person's preferred address. |
| **Voice Input** | `tap` (touch) / `none` (desktop) | `none` / `tap` (tap the reactor to talk) / `wake` (browser "hey jarvis" listener — needs mic permission). |
| **Keep Mic Open After Response** | off | Follow-up conversation: after each spoken answer the mic re-opens for **Follow-up Wait (sec)** (3–120, default 10) without a wake word; each answer re-opens it. |
| **Wake Word Threshold** | `0.42` | openWakeWord score needed to fire (0.1–0.9; lower = more sensitive, more false wakes — kiosk mics differ, tune per screen). |
| **Lock on Farewell** | off | When a voice turn sounds like goodbye, the screen locks once the reply finishes. **Farewell Lock Delay** (0–10 s, default 0.5) pauses between the last word and the lock. |
| **Auto Re-lock (sec)** | `300` (touch) / `0` (desktop) | Re-locks after that many *idle* seconds (touch use resets the clock; 0 = off). |
| **Allowed Faces** | *(empty = anyone)* | Multi-select of Tater People whose enrolled faces may unlock this screen; everyone else is treated as an unrecognized stranger here. |
| **Allowed PIN Identities** | *(empty = any PIN)* | Tater People whose PIN-book PINs may unlock this screen. Unlinked PINs are only accepted while this list is empty; the screen's own PIN always unlocks. |
| **Tap Unlock Person** | *(People-tab default)* | The person a bare tap on *this* screen credits (a tap carries no per-attempt identity). Unlocks record them, the lock screen greets them, voice turns tell JARVIS who they are. |
| **Voice Unlock Person** | *(blank = anonymous)* | The person the "hey jarvis" wake-unlock on this screen credits. |
| **Assistant Name** | *(blank = JARVIS)* | What the assistant is called where it speaks on *this* screen (voice/console lines, reactor label). The top bar, boot text, and wake word stay unchanged. |
| **Linked Satellite** | *(none)* | Tater satellite (announcement-target value) linked to this screen — voice requests spoken at that satellite open here when the request doesn't name a screen. |
| **Bind to IP** | *(empty)* | Requests *from this IP* always open this screen (overrides `?screen=`; the page is redirected). One screen per IP — alphabetically-first profile wins on ties. A routing convenience, not access control. Behind a proxy, also enable `TRUST_PROXY_HEADERS`. |
| **Unlock Audio (wav)** | *(none)* | Plays when the screen unlocks (browser engine) and **ducks to 20% while JARVIS speaks**, restoring full volume after. |
| **Fail Voice (wav)** | *(none)* | Played with the red reactor when a face scan or PIN entry is rejected (face/PIN/voice screens; a tap screen can't fail). Unset = silent by design. |
| **Success Voice (wav)** | *(none)* | Played on a successful unlock of any kind. |

> The PIN code is verified server-side and **never shipped to the browser** —
> the client only ever sees a "PIN is set" flag.

### People

Faces are enrolled in Tater's own **Settings › People › Faces** (this tab
reads the same shared store); this tab owns the **PIN book** and the default
tap identity:

- **Add a PIN** — a 4-digit code, optionally linked to a Tater Person. Every
  PIN screen accepts the whole book (plus its own profile PIN); a linked PIN
  records who unlocked, an unlinked one unlocks anonymously. Wrong codes play
  the access-denied path and count toward the screen's fail limit.
- **Tap Unlock Person** — the *default* person credited for tap unlocks on
  every tap screen; each screen's own "Tap Unlock Person" overrides it.
  Unlinked = anonymous unlocks.

### Automations

Event → **fullscreen camera popup** on chosen screens, optionally with a
spoken line. Rule fields:

| Field | Meaning |
|---|---|
| **Trigger** | *Detects a person* / *Detects motion* / *Doorbell pressed* / *Recognizes a Tater Person* (with a per-rule person picker). |
| **Camera** | *Any camera* (shows the triggering event's own camera) or a pinned camera (UniFi Protect API cameras match their events directly). |
| **Pop Up On Screens** | One or more screens (multi-select). |
| **Popup Duration (sec)** | 5–120; repeat events restart the timer. |
| **Feed** | `live` (MJPEG; falls back to snapshots when the source can't stream) or `snapshot` (1 Hz stills). |
| **Rule Cooldown (sec)** | Ignore further matching events for this long (0 = every detection). |
| **Say / Say It Out Of** | Spoken line with `{camera}` / `{person}` / `{trigger}` placeholders, out of `screen` / `satellite` (the screens' Linked Satellites) / `both`. |

Each rule card has **Enable/Disable** and **Test popup** actions.

### Voice

- **History Messages Sent to the LLM (0–50)** — same as the
  `VOICE_HISTORY_COUNT` core setting, per-screen rolling window.
- **Clear Chat History** — wipes stored voice history for all screens (use
  after a turn went wrong so the poisoned exchange stops steering replies).

---

## Unlock methods, in detail

- **face** (default on touch screens) — recognition runs through Tater's
  shared Face ID store; only **linked** faces count. On a fail
  (`FACE_FAIL_LIMIT` consecutive non-matches) the reactor turns red, the
  screen's fail voice plays (if set) with the reactor pulsing on its
  waveform, and the screen re-arms after `FACE_FAIL_COOLDOWN_S`. The
  **Allowed Faces** list further gates *who* can unlock a given screen. If
  Face ID is unavailable (disabled, model missing, desktop mode) the screen
  degrades to tap-to-unlock. The browser needs **camera permission** —
  grant it once in the kiosk browser.
- **tap** — any tap unlocks. No camera, no code. Tap screens can never fail,
  so they only offer a success voice.
- **pin** — tapping opens a Jarvis-styled PIN pad (glowing dots + angular
  keys); the 4th digit submits **server-side**. A wrong code takes the same
  access-denied path as a failed face scan and counts toward the fail limit.
  The screen's own PIN plus any PIN-book entry unlock it, subject to the
  **Allowed PIN Identities** list.
- **voice** — saying "hey jarvis" on the *lock screen* unlocks, and Tater
  answers the phrase out loud (requires Voice Input `wake`; degrades to tap
  if the wake engine can't start).

Access denial turns the arc reactor **red** until the cooldown ends or the
screen unlocks. A green **"ACCESS GRANTED" / "FACE MATCHED"** shows on the
lock screen *before* the unlock transition. While the screen server is
unreachable the page re-locks and denies access ("DISCONNECTED" on tap) until
the connection recovers.

## Voice input, in detail

Per screen, `voice_input` = `none` / `tap` / `wake` (see the Screens table).

- **tap**: tap the reactor → `LISTENING` → speak → stop. The transcript lands
  in the screen console as a `YOU` line; JARVIS answers and speaks it.
- **wake**: the mic stays open listening for **"hey jarvis"** —
  openWakeWord running entirely in the browser via WebAssembly (no cloud, no
  API key). Status line shows `WAKE ARMED`. If the wake engine can't start,
  the screen degrades to tap-to-talk and says so in the console.
- **Follow-up**: with *Keep Mic Open After Response* on, the mic re-opens
  after each spoken answer for the follow-up window (no wake word needed) and
  re-arms on every answer, so a back-and-forth loops until silence.
- **Echo gating is built in** — the mic is closed while the screen is
  speaking and re-opens after the audio finishes.
- Tap-to-talk resamples mic audio to 16 kHz client-side before posting.
- **Requirements:** a local STT backend, a TTS backend for spoken answers,
  mic permission, and a **secure context** (below).

## Cameras

`ha_camera` cards work two ways:

- **snapshot** (default) — a JPEG re-fetched on a timer (`refresh_s`, default
  from `CAMERA_REFRESH_S`) pushed over SSE.
- **live** — a real-time multipart MJPEG stream served by the core itself,
  deduped by camera and torn down when no one is watching. Frame rate from
  `LIVE_FPS`, concurrency capped by `LIVE_MAX_STREAMS`.
- **Sources:** Home Assistant entities stream through HA's own MJPEG proxy;
  **UniFi Protect** devices (from Tater's integration, or the
  `PROTECT_*` fallback credentials) stream through a built-in direct
  pipeline (Public API + ffmpeg RTSPS decode). `feed_source: auto` picks per
  card.

## Home Assistant

Climate, entity-state, and snapshot-mode camera cards resolve through the
**Home Assistant Tater integration** when it is configured (its URL + token
win); otherwise the `HA_BASE_URL` / `HA_TOKEN` core settings are the direct
REST fallback. With neither configured, HA-backed cards render empty.

---

## Hydra (the LLM drives the screen)

The core registers six kernel tools; example prompts from Tater chat (voice or
Web UI):

| Tool | Example prompt → result |
|---|---|
| `jarvis_screen_card` | "What's the temperature?" → an `ha_climate` card with working +/− controls. "Show me the front door" → an `ha_camera` card (`feed_mode: live` for the real-time stream). Cards can be created, shown, hidden, updated, deleted; `position: "auto"` finds free space. |
| `jarvis_screen_layout` | "Switch to the security screen" → the named layout with the standard transition. |
| `jarvis_screen_reactor` | "Move the reactor to the top left" / "Put the reactor back" → preset or `{x, y, scale}`; recolor (cyan/red/amber/green/`#rrggbb`), relabel. |
| `jarvis_screen_say` | "Say 'right away, sir'" → TTS through the screen (browser engine) with the reactor pulsing on the waveform. |
| `jarvis_screen_alert` | "Front door left open 10 minutes" → info (blue note) / warn (amber) / critical (red flash + red reactor). |
| `jarvis_screen_lock` | "Lock the kitchen screen" / "Unlock it". |

Screen targeting: pass `screen` only when the user names one; otherwise the
tool targets the screen linked to the speaking satellite, or `main`.
Display-bus **alert** events from anywhere in Tater also flash the screen red
or amber.

Generated-card behavior is governed by the AI settings above
(`CARD_LAYOUT_MODE` / `CARD_LAYOUT_MATCH` / `CARD_PERSIST_AI` /
`AI_YOUTUBE_AUTOPLAY` / `AI_SIZE_*`): a create that matches an existing layout
card **borrows** it for the session (the screen switches to that layout and
the content is a session-only patch) — the saved layout is never written
back, and a page reload restores it.

---

## Secure context (mic & camera)

Browsers only allow `getUserMedia` on `https://` or `localhost`. A plain
`http://<lan-ip>:8610` screen loads fine, but **mic and camera are blocked**.
Two fixes:

**1. TLS in front (recommended):**

```nginx
server {
    listen 443 ssl;
    server_name hud.lan;
    ssl_certificate     /etc/nginx/certs/hud.crt;
    ssl_certificate_key /etc/nginx/certs/hud.key;

    location / {
        # Tater runs network_mode: host — the screen listens on the host.
        # (Bridge-network nginx? Use the Docker host's gateway IP.)
        proxy_pass http://127.0.0.1:8610;
        proxy_http_version 1.1;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        # The SSE stream must not be buffered, or events never arrive:
        proxy_set_header Connection "";
        proxy_buffering off;
        proxy_cache off;
        proxy_read_timeout 300s;
    }
}
```

With nginx in front, also enable **Trust Proxy IP Headers** in the core
settings so per-screen IP binding resolves the browser's IP, not nginx's.

**2. Chromium kiosk allowlist (quick and dirty):**

```bash
chromium --kiosk --start-fullscreen \
  --unsafely-treat-insecure-origin-as-secure=http://192.168.1.10:8610 \
  "http://192.168.1.10:8610/?screen=main"
```

Grant camera + mic permission once in the kiosk browser on first load.

---

## Day-to-day usage notes

- **Webpage:** `http://<docker-host>:8610/?screen=<profile>`
  (`&token=<value>` when `SCREEN_TOKEN` is set). Multiple displays = multiple
  profiles = multiple URLs. A profile with **Bind to IP** set follows the
  display instead — any request from that IP is redirected to that screen, so
  the kiosk can just open `http://<host>:8610/`.
- **Auto re-lock** applies to every unlock method; idle = no touch/pointer
  use (a throttled activity heartbeat resets it; watching or tapping counts).
- **Dormant state:** after the configured idle delay, a locked screen's
  reactor morphs into the speckled clock disc (per-screen time/date formats).
- **Alerts** from Tater's display bus flash the frame/reactor red or amber.
- **Unlock audio** keeps playing (ducked) for as long as the screen stays
  unlocked; re-locking stops it.

---

## Updating the core

1. Ship the new `cores/jarvis_screen_core.py`:
   - Option 1: `docker cp` it again.
   - Option 2: commit + push with a regenerated manifest
     (`python3 tools/generate_core_manifest.py` — the manifest pins an exact
     sha256).
   - Option 3: replace the file in the mounted host dir.
2. In the Web UI: **stop, then start** Jarvis Screen (the start path
   re-imports the module — no app restart). Open screens pick up the new
   state on their next SSE reconnect.

> **⚠ The Cores-page "Update" button is a rollback trap when the store
> manifest is stale.** It copies the *store-side* file (sha256-checked) — if
> the manifest still pins an older version, Update silently rolls the core
> **back**. For routine updates, copy the `.py` (steps 1–2 above), never
> "Update."

**Database upgrades are automatic.** On start, the core checks its schema
marker in Redis against the file's version and runs pending one-shot
migrations before the server comes up; additive fields are normalized on
read at their defaults. You never migrate by hand.

### Upgrading from Jarvis HUD

v1.47.0 renamed the core; v1.48.0 added automatic legacy-data migration:

1. Install `cores/jarvis_screen_core.py` fresh (it is a brand-new core to the
   host — `jarvis_screen`, not `jarvis_hud_core`), and start it.
2. On every boot `migrate_legacy_hud_data()` copies the old
   `jarvis_hud:*` namespace, `jarvis_hud_core_settings`, and
   `tater:cooldown:jarvis_hud_core` into the new namespace (types preserved;
   destinations that already hold real data are never clobbered).
3. Re-seed/re-check settings on the new core's settings page.
4. **Delete the old `jarvis_hud_core` from the Cores page** to purge the
   remaining legacy keys — the migration leaves sources in place on purpose.

Automation rules' stored `say_to: "hud"` is accepted as a legacy alias.

---

## Uninstall & data removal

Cores → **Jarvis Screen → Manage → Remove**. The platform stops the core,
deletes the file, and clears the autostart flag.

The **"delete data"** checkbox does a full sweep: the host derives the exact
keys generically from the core id — `jarvis_screen_core_settings`,
`tater:cooldown:jarvis_screen_core`, and **everything under `jarvis_screen:*`**
(profiles, layouts, PINs, automations, per-screen state, cards, placements,
custom wavs, the schema marker). Reads of other cores' data (e.g. `people`,
`tater:display:events:v1`) stay read-only and survive the sweep.

Without "delete data", everything is preserved in Redis — reinstalling the
core later restores the full configuration (usually what you want after an
update).

---

## Troubleshooting

| Symptom | Fix |
|---|---|
| Core not in the Web UI list | `docker exec tater_app ls /app/cores/` — file must be present and end in `_core.py`. Check `docker logs tater_app` for an import traceback (a syntax error keeps it out of the registry). |
| Core starts but no screen page | Port conflict: pick a free `SCREEN_PORT` and restart the core. Verify *inside* the container (`docker exec tater_app curl -s localhost:8610/api/health`) — if that works, it's a host firewall rule on the port. |
| Screen loads but "connecting…" | SSE failed — most often `SCREEN_TOKEN` set but the URL opened without `?token=…`, or a proxy buffering/chunking the stream. |
| Reactor never pulses on voice | Profile audio is `remote` but no TTS backend is configured (envelope events are skipped); or in `browser` mode the browser blocked audio until first interaction — any tap resumes the AudioContext. |
| No voice UI / `VOICE UNAVAILABLE — NO STT BACKEND` | No local STT backend in Tater's speech settings, `voice_input` is `none` for that profile, or the screen is locked. When armed, the status line shows `TAP TO TALK` / `WAKE ARMED`. |
| Mic blocked / no audio captured | `getUserMedia` needs a secure context — see [Secure context](#secure-context-mic--camera) — and the kiosk browser must have granted mic permission. |
| Wake word never fires | Check the screen console for `WAKE ENGINE STOPPED — TAP THE REACTOR` (graceful fallback to tap). Lower the **Wake Word Threshold** per screen (kiosk mics differ). Otherwise confirm the installed core is the full ~23 MB build — a truncated copy 404s the ONNX models. |
| Unlock audio doesn't play | Profile audio engine is `remote` (the screen stays silent by design), the file isn't a valid PCM `.wav`, or the screen re-locked while loading. |
| Fail sound doesn't play | The screen's **Fail Voice** isn't set (unset = silent by design), the file isn't a valid PCM `.wav`, or the audio engine is `remote` (the envelope pulse still works). |
| IP binding not redirecting | Behind a reverse proxy you must enable **Trust Proxy IP Headers** (otherwise the screen only sees the proxy's IP). The field is an exact match — typos never match. If several profiles share an IP, the alphabetically-first one wins. |
| Face scan does nothing | `FACE_ENABLED` off, the profile isn't a touch profile, the Face ID model isn't downloaded/enrolled (faces must be **linked** to a Person), or the kiosk browser didn't grant camera permission. Check `/api/health` → `face` status. |
| PIN pad says "NO PIN SET" | Unlock Method is `pin` but **PIN Code** is blank — set a 4-digit code on the profile. After the fail limit, a PIN screen shows "LOCKED OUT … RETRY IN nS" and re-arms after the cooldown. |
| Climate/camera cards empty | HA integration not configured in Tater **and** `HA_BASE_URL`/`HA_TOKEN` empty. Either enable the integration or set the REST fallback. |
| Live camera feed blank | Confirm the source pipeline (HA proxy vs UniFi Protect direct — `feed_source`), the `PROTECT_*` credentials when no integration is configured, and that `LIVE_MAX_STREAMS` isn't exhausted (same camera counts once). |
| Core vanished after a docker rebuild | Expected with Options 1/2 (container writable layer). Move to Option 3 (volume mount). |
| Wrong sha256 on Core Shop install | The manifest and the core file disagree. Regenerate from the same checkout (`python3 tools/generate_core_manifest.py`) and push both. |
| "Update" rolled the core back | The store manifest pinned an older file — see the [warning in Updating](#updating-the-core). Copy the `.py` instead. |

---

## Redis key map (for the curious)

| Key | Holds |
|---|---|
| `jarvis_screen_core_settings` | Core settings hash (port, token, AI sizes, …) — host-managed |
| `jarvis_screen:profiles` | Screen profiles |
| `jarvis_screen:layouts` | Layouts + base card geometry |
| `jarvis_screen:pins` | PIN book |
| `jarvis_screen:auto_rules` | Automation rules |
| `jarvis_screen:state:<screen>` | Per-screen runtime state (lock, layout, reactor) |
| `jarvis_screen:cards:<screen>` | Runtime (AI-generated) cards |
| `jarvis_screen:geom:<screen>` | Per-screen placement overrides (visual editor) |
| `jarvis_screen:ovr:<screen>` | Session-only layout-card content patches |
| `jarvis_screen:face:<screen>` | Face/PIN fail counters + lockout cooldowns |
| `jarvis_screen:voicehist:<screen>` | Voice conversation history |
| `jarvis_screen:unlockwav:<screen>` / `failwav:` / `successwav:` | Per-screen custom wavs |
| `jarvis_screen:backup` | Last exported snapshot (Backup tab machinery) |
| `jarvis_screen:speak:<id>` | TTS cache (self-expiring) |
| `jarvis_screen:schema_version` | Schema/migration marker |
| `tater:cooldown:jarvis_screen_core` | Host-managed core cooldown |
