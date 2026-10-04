# Feature Request: Satellite Voice Turn Mirror — show SAT conversations on the Linked Screen

**Repo:** `heapsoftware/Tater-Jarvis-Screen` (jarvis_screen core)
**Proposed version:** next minor after the current core at build time (core
shipped 1.58.0 on 2026-10-04 for an unrelated music-card feature, so this
lands as **v1.59.0 or later** — feature → minor bump, per versioning
convention)
**Status:** request — needs a small Tater host-side change (see Part 1) plus a
core-side build (Part 2). Nothing in this repo implements it yet.

## Summary

Today, saying "Hey \<wake word\>" on a Tater satellite that is a screen's
**Linked Satellite** already runs the full voice pipeline (wake → STT → Hydra
→ TTS on the SAT), and the paired screen reacts **only** when the model
explicitly calls a `jarvis_screen_*` tool during the turn (card, layout,
reactor, alert, lock, say — routed via `_satellite_screen`,
`cores/jarvis_screen_core.py:3180`).

What is missing: a plain Q&A turn ("hey jarvis, what's the weather") leaves no
trace on the screen. The browser-mic voice path pushes the transcript and
response into the screen's chat feed and voice history
(`_handle_voice` / `_run_voice_turn`, `cores/jarvis_screen_core.py:7053`,
`_voice_history_append` at `:2773`), but satellite turns have no equivalent.
This feature adds that mirror, so the screen shows the SAT conversation like
it shows browser-voice conversation.

## Read this first

1. The **host side of this feature cannot be built in this repo.** The Tater
   host (`TaterTotterson/Tater`) is read-only reference; host-side wishes go
   to Tater feature requests. Part 1 below is written to be pasted into a
   Tater issue verbatim.
2. The core side (Part 2) is buildable here **only after** the host publishes
   the marker event — but it is designed to ship first and be a silent no-op
   until the event appears, so the order does not matter.
3. Line numbers below refer to jarvis_screen v1.54.0 and may drift; anchor on
   function names.

## Background: what exists today

- **Linked Satellite (per-screen, Screens tab):** `linked_satellite` profile
  field (`jarvis_screen_core.py:1007`), dropdown sourced read-only from the
  voice core's satellite registry `tater:voice:satellites:registry:v1`
  (`_satellite_options`, `:3116`). Values are announcement-target selectors
  (`voice_core:<selector>`).
- **Tool-call routing during SAT turns:** voice_core stamps the speaking
  satellite's selector into trusted `origin['device_id']`
  (`Tater/tater_voice/voice_pipeline/conversation.py:269`); the core's
  `run_hydra_kernel_tool` resolves it to the linked screen
  (`jarvis_screen_core.py:8206`).
- **Browser-voice chat feed:** `_run_voice_turn` pushes two console events
  (speaker-labelled transcript line + assistant line) and appends both sides
  to the screen's voice history.
- **Display bus:** voice_core owns `tater:display:events:v1`
  (`Tater/tater_voice/display_bus.py`); events carry `kind`, `target`,
  `title`, `message`, `description`, `meta`, `seq`, `created_at`. Unknown
  kinds are coerced to `"notification"`; `kind` must be one of the host's
  `_ALLOWED_KINDS` (`voice` is allowed).
- **The trap:** the core's watcher loop already reacts to `kind == "voice"`
  bus events by **speaking them aloud** via `announce_say`
  (`jarvis_screen_core.py:6169`). A mirror event published as a plain `voice`
  event would be TTS'd back out on the screen's targets — an echo loop through
  the very satellite the person is standing next to. The design below avoids
  this with an explicit marker.

## Proposed user-visible behavior

With "Mirror Satellite Voice Turns" enabled on a screen (new Screens-tab
checkbox, default **off**):

1. Someone says "Hey jarvis, what's the weather" at the SAT linked to that
   screen.
2. The SAT speaks the answer exactly as today (no change to TTS).
3. The screen's console/chat feed gains the same two lines the browser-mic
   path produces — `YOU (Speaker):  what's the weather` and
   `JARVIS:  Currently 72°…` — and the screen's voice history records the
   user/assistant turn.
4. No audio comes from the screen or its speakers because of the mirror; the
   mirror is display-only.

Screens with the checkbox off behave exactly as today.

## Part 1 — Tater host ask (paste into a Tater feature request)

**Title:** voice_core: publish a display-bus event per completed satellite
voice turn

**Ask:** after `_run_hydra_turn_for_voice` saves the assistant message
(`Tater/tater_voice/voice_pipeline/conversation.py`, the
`_save_history_message(conv_id, "assistant", response)` call near the end of
the turn), publish one event to the existing display bus, **only when the
turn originated from a voice satellite** (non-empty `session.selector`):

```python
from tater_voice import display_bus

display_bus.publish_display_event({
    "kind": "voice",
    "target": session.selector,          # e.g. "voice_core:esp_sat_office"
    "title": speaker_name or device_name or "Voice",
    "message": transcript,               # what the person said
    "source": "voice_core",
    "meta": {
        "voice_turn": True,              # the marker — see below
        "response": response,            # what the assistant replied (clip ~2000)
        "speaker": speaker_name,
        "device_name": device_name,
        "session_id": session.session_id,
    },
}, client=vp.redis_client)
```

Why the marker: jarvis_screen's watcher speaks every `kind:"voice"` bus event
it sees. `meta.voice_turn = True` tells the screen core *this one is a record
of a turn, not a request to talk*. Events without the marker keep today's
behavior byte-for-byte.

Notes:

- Keep the publish best-effort (`try/except`, log on failure) — a display-bus
  hiccup must never fail the voice turn.
- The 480-char `message` clip and any `meta` handling are host-side existing
  behavior; the core clamps again on its side.
- Typed/webui chats (Hydra `platform="webui"`) must NOT be published here —
  the gate is the satellite selector, not "any voice turn".

**Rejected host-side alternative:** a brand-new event `kind`. `display_bus._clean_kind`
coerces unknown kinds to `"notification"`, so the core could not reliably
recognize one; the `meta` marker is the only clean channel.

## Part 2 — jarvis_screen core build (this repo)

All changes stay in `cores/jarvis_screen_core.py` (+ its test suite). No host
files touched; if the host event never ships, the added code is dead and
harmless.

### 2.1 New per-screen setting

Add `mirror_satellite_voice` (checkbox, label **"Mirror Satellite Voice Turns
— show voice conversations spoken at this screen's Linked Satellite"**,
default `False`) to the screen profile schema alongside `linked_satellite`
(`:1007` block) and to the Screens-tab field list where `voice_followup`
lives (`:9471` area).

### 2.2 Watcher branch (the core of the feature)

In `_watcher_loop`'s display-bus dispatch (`:6160`), the existing
`kind == "voice"` branch becomes two:

```python
if kind == "voice":
    meta = event.get("meta") if isinstance(event.get("meta"), dict) else {}
    if _as_bool(meta.get("voice_turn"), False):
        _mirror_satellite_voice_turn(client, event)   # display-only, never speaks
    elif text:
        await announce_say(client, screen, text)      # unchanged legacy path
```

### 2.3 `_mirror_satellite_voice_turn`

New function; per event:

1. **Resolve the screen(s) by satellite, not by event target.** The event's
   `target` is the satellite selector, which never equals a screen name, so
   `_screen_matches_target` will not match it. Reuse the matching logic from
   `_satellite_screen` (`:3180`) factored into a helper that returns **every**
   screen whose `linked_satellite` matches the selector (bare or
   `voice_core:`-prefixed form — both are accepted there today). Mirroring to
   all matching screens is correct; a selector linked to several screens is a
   legitimate configuration.
2. For each matched screen with `mirror_satellite_voice` enabled:
   - `transcript = event.message`, `response = meta.response` (both clamped,
     transcript ~480 to match the host clip, response ~2000).
   - `_voice_history_append(client, screen, "user", transcript)` then
     `"assistant", response` — same store the browser-voice path uses
     (`:2773`), so the chat card and history stay one thread.
   - Push the same two console events `_run_voice_turn` pushes (`:7163`,
     `:7186`): speaker-labelled user line (use `meta.speaker` through
     `_screen_voice_speaker_label` `:2864`; empty → bare `YOU`) and the
     assistant line labelled by `_assistant_name_for`.
3. Do **not** call `announce_say`, `op_alert`, or any TTS path from this
   branch. Do not touch lock state — mirroring is passive and must work on
   locked screens (unlike browser voice, the person is verified by the host's
   own wake/speaker pipeline, and nothing here grants access).

### 2.4 Edge cases

- **Core restart:** existing `last_seq` + 180 s staleness skip (`:6170`) covers
  replays; no new bookkeeping.
- **Multiple turns while the watcher sleeps:** bus events are seq-ordered and
  the loop drains new rows each cycle; rapid follow-ups arrive in order.
- **Follow-up (open-mic) turns:** each turn is its own event; nothing extra.
- **Screen with a Linked Satellite but checkbox off:** matched, skipped.
- **Event with no response** (wake-only / error): mirror the transcript line
  only; skip the assistant line and the history append for that side.
- **Host never publishes:** branch never fires; version ships as a pure
  no-op.

### 2.5 Rejected core-only alternatives (why the host ask is necessary)

- **Polling `tater:voice:conv:*:history`:** the conv id is the session id or a
  client-supplied conversation id (`voice_pipeline/__init__.py:6266`), not
  derivable from the satellite selector; discovering turns would mean scanning
  arbitrary Redis keys with TTLs. Fragile and racy.
- **Hooking `run_hydra_kernel_tool`:** only fires when the model calls a
  `jarvis_screen_*` tool — exactly the turns that already show up. Plain Q&A
  never invokes it.
- **Screen-side audio capture:** the screen has no mic feed from the SAT; the
  audio lives entirely in the host's satellite pipeline.

## Test plan (core side)

1. **No-op safety:** with the host event absent, run the existing suite —
   watcher behavior unchanged.
2. **Mirror on:** fake a bus event (`kind=voice`, `target=<selector>`,
   `meta.voice_turn=True`, message+response) with a screen whose
   `linked_satellite` matches → assert two console events, two history rows,
   and that `announce_say` was **not** called (mock/spy).
3. **Legacy voice events still speak:** same event without the marker →
   `announce_say` called, no history writes.
4. **Checkbox off:** marker event + disabled screen → no feed/history writes.
5. **Two screens linked to one selector:** both mirror.
6. **Empty response / wake-only transcript:** user line only, no assistant
   history row.
7. **Staleness:** event older than 180 s → skipped, `last_seq` still advances.

## Versioning / deploy

- This repo: feature → minor bump **1.55.0** (fixes during the build bump the
  third number). Ship via the normal flow: dev here → push to
  `heapsoftware/Tater-Jarvis-Screen` (CI syncs `core_manifest.json`) →
  Cores-page **Update**.
- Tater side: the Part 1 publish rides the host's own release train; until it
  lands, the core feature is dormant.

---

## Addendum (parked 2026-10-04): SAT-mic reactor waveform

User wish: when a Linked SAT is running the voice for a screen, the arc
reactor should pulse from the **SAT's mic audio** — the person's own voice —
not only from the screen's TTS playback. Verified against Tater v1.2.7
(commit `046570b`): this is not buildable without the host, same as the
mirror itself.

### Why the host is needed (verified 2026-10-04, Tater v1.2.7)

The reactor pulse has exactly two drivers today, and both see only audio the
screen itself plays:

- browser engine — `THREE.AudioAnalyser` over the screen's own TTS playback
  (SCREEN_JS audio block, `jarvis_screen_core.py` ~:15016);
- remote engine — `compute_rms_envelope()` (~:1402) precomputed from the TTS
  wav by the core, shipped with the `speak` event and scrubbed client-side
  (`envelopeSource`, ~:15076 / ~:17406).

SAT mic/STT audio exists **only inside the host's voice pipeline process**
(`tater_voice/`); it is never published — no Redis key, bus event, or route
carries audio levels (`native_satellite._envelope` is just the websocket
message wrapper). So Part 1 must be extended.

### Host ask extension (Part 1b)

In the same satellite-gated publish as Part 1, attach the user-utterance RMS
envelope to the turn event:

```python
"meta": {
    "voice_turn": True,
    # ... existing mirror fields ...
    "rms": [0.0, 0.4, 0.9, ...],   # 50 ms windows, normalized 0..1, clipped
    "rms_ms": 50,                  # window size, so the core can rescale
}
```

- Same window format `compute_rms_envelope` already produces
  (`SPEAK_ENVELOPE_WINDOW_MS = 50`, normalized 0..1) so the client consumes
  it unchanged. A wake word / short utterance is only a few windows — keep
  the payload tiny, no wav bytes over the bus.
- **Live variant (stretch):** the pipeline now has STT streaming over the
  spud link (v1.2.6+, `VoiceSessionRuntime.spud_link_stt_stream`), so live
  audio is in-hand during encoding. A `kind:"voice"` *level* event published
  per few windows (or a small Redis pubsub off the satellite selector) would
  make the reactor react while the person still speaks. If the host prefers
  one publish only, the after-the-fact envelope still works — the pulse then
  plays over the completed utterance's window rather than live.
- Gate on the satellite selector (same as the mirror: no webui/typed turns).

### Core-side build (Part 2b)

- New per-screen screen-profile toggle (Screens tab): **Reactor SAT Pulse**,
  default off — only screens whose `linked_satellite` matches the event
  selector arm it.
- On a `voice_turn` event carrying `meta.rms` (or a live level event),
  arm the existing `envelopeSource` with `t0 = now`, `duration =
  len(rms) * window_ms / 1000` and push it as an SSE event shaped like the
  speak payload's envelope (client-side no new animation path needed — the
  pulse code at ~:15076 already scrubs by clock).
- Never touches lock state or TTS; display-only like the mirror (§2.3).
- If both a SAT pulse and a screen-speak envelope are active, max them
  (the client already does `p = Math.max(p, …)`) — SAT-mic pulse wins the
  shape, TTS keeps brightness.

### Open questions for build time

- Replay-vs-live acceptability (decides whether the level-event variant is
  worth the extra host ask).
- Whether to widen gates: Reactor Wave Gain already scales the pulse
  (~:969); SAT pulse should reuse it unchanged.
- Native-echo satellites (Echo audio integration, v1.2.4 a26ac3a) enter the
  same registry — decide at build time whether their selector turns are
  eligible (assume yes; it is registry-driven either way).