# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.


---

## What Is This Project?

**Vibesbox** is a distributed home audio appliance system split across two hardware units and two git repositories within this working directory. It is a production appliance — there are no tests, no CI/CD, and no traditional build pipeline for most components.

- `VibesboxSRC/` — VibesboxSRC (Raspberry Pi 4B): source selection, sample-rate conversion, NDI/S/PDIF output
- `VibesboxDSP/` — VibesboxDSP (LattePanda Mu, Windows 11): full DSP chain, speaker output
- `README.md` — canonical system-level architecture document (the Vibesbox umbrella repo's landing page); read this first for any cross-system work

---

## System Architecture

```
Pi 4B (VibesboxSRC)                         LattePanda Mu (VibesboxDSP)
────────────────────────────────────        ────────────────────────────────────
USB Audio (UAC2)       ─┐                   NdiAsioReceiver (Scheduled Task)
Lyrion/Squeezebox       ├─► CamillaDSP      ← NDI 6ch 96kHz FLTP → ReaRoute ASIO
AirPlay (shairport)     │   96kHz FLTP      RME Digiface ASIO ← HiFiBerry S/PDIF (2ch 96kHz)
Bluetooth (bluealsa) ───┘   user-selected   TotalMix FX (hardware-side routing only)
                            2ch or 6ch out  REAPER (RME + ReaRoute, VST3 DSP chain)
                                            └─► Amplifiers → Speakers
HiFiBerry S/PDIF (2ch 96kHz, when 2ch mode) Pi Pico (SOFIAN-MIDI) ──── USB MIDI CC ──► REAPER (vol/tone real-time control)
                                            VibesboxKiosk (WinUI 3 touchscreen app)
                                            AudioFingerprintService (Scheduled Task, USB metadata)
Touchscreen UI (:80) ←─── auto_router WS (:8080)
Now Playing HTTP+WS (:8090) ────────────────────────► VibesboxKiosk subscribes
```

The **network interfaces** between the two units:
- **Audio**: either NDI `VibesboxSRC-5.1` (6ch, 96kHz, 32-bit float planar) OR S/PDIF 2ch via HiFiBerry → RME Digiface. The user selects 2ch or 6ch output in the Pi UI; the choice is independent of which source is active. CamillaDSP upmixes (Hafler) or downmixes as needed.
- **Now Playing metadata**: `nowplaying_server` on the Pi (:8090) exposes a JSON WS + HTTP API; VibesboxKiosk subscribes via WebSocket. AudioFingerprintService on the LattePanda uploads silence-gated 7-second WAV clips to `POST /api/fingerprint`; the Pi runs `shazamio` server-side and broadcasts matches. Tap is at the REAPER level, so it covers both audio paths (NDI 6ch *and* S/PDIF 2ch) — no per-transport configuration.

Both devices must be on wired Ethernet — Wi-Fi causes audible dropouts on NDI.

---

## VibesboxSRC — Key Files & Components

`VibesboxSRC/scripts/auto_router.py` — Central daemon. Polls ALSA `hw_params` every 200ms, detects active source, hot-swaps CamillaDSP config without restart, manages Bluetooth pairing state machine, runs WebSocket server on :8080 for the UI.

`VibesboxSRC/scripts/ndi_transmitter.py` — Reads CamillaDSP 6ch output from `NDITX` ALSA loopback, transmits as `VibesboxSRC-5.1` via NDI SDK v6 (ctypes/libndi.so). Auto-started/stopped by auto_router based on output mode.

`VibesboxSRC/camilladsp/*.yml` — Four routing configs: `2ch→2ch`, `2ch→6ch` (Hafler upmix), `6ch→2ch` (downmix), `6ch→6ch` (passthrough). All apply -4dB intersample headroom. CamillaDSP boots from a neutral RawFile/Null config and the auto-router hot-swaps the real config on demand.

`VibesboxSRC/ui/` — Vanilla HTML/CSS/JS dashboard served by nginx on :80. Connects to auto_router WebSocket (:8080) for state and CamillaDSP WebSocket (:5005) for RMS meters.

`VibesboxSRC/services/*.service` — systemd unit files. Boot order matters (see below).

`VibesboxSRC/install.sh` — One-time installation script (run as root on the Pi).

### Pi Boot Order (hard dependencies)
1. `alsa-loopback` — loads `snd-aloop` (4 virtual loopback cards)
2. `usb-gadget` — configures UAC2 6ch USB gadget
3. `camilladsp` — starts with neutral config
4. `auto-router` — begins polling, starts WebSocket
5. `ndi-output` — conditional; managed by auto-router
6. Source services (`squeezelite`, `shairport-sync`, `bluealsa`)
7. `greetd` + `sway` + Chromium (touchscreen dashboard, 800×480 portrait)

### ALSA Loopback Cards
| Name | Card | Source |
|------|------|--------|
| Lyrion | hw:Lyrion,1,0 (card 10) | squeezelite |
| AirPlay | hw:AirPlay,1,0 (card 11) | shairport-sync |
| Bluetooth | hw:Bluetooth,1,0 (card 12) | bluealsa |
| NDITX | card 13 | CamillaDSP 6ch output |

---

## VibesboxDSP — Key Files & Components

`VibesboxDSP/NDI ASIO Receiver/` — .NET (C#) app. Receives the NDI 6ch stream and writes it into REAPER via **ReaRoute ASIO** (not RME). Deployed as a **Scheduled Task at user logon** (not a Windows Service) because ReaRoute is session-bound — see [NDI ASIO Receiver/README.md](VibesboxDSP/NDI ASIO Receiver/README.md).

`VibesboxDSP/AudioFingerprintService/` — .NET (C#) worker. Reads a stereo tap from REAPER via **ReaRoute** (REAPER routes the USB-input track's channels 1-2 to ReaRoute outputs 1-2; the service opens ReaRoute as INPUT, reads channels 1-2). Silence-gated 7-second clips, uploads the raw WAV body to the Pi's `POST /api/fingerprint`. **The Pi side runs the actual fingerprinting via `shazamio`** and broadcasts matches itself — Windows just captures and uploads. Same Scheduled-Task pattern as NdiAsioReceiver, for the same Session-0 reason. Covers BOTH transport pipelines (6ch NDI and 2ch S/PDIF) because the tap is at REAPER level, after audio has already arrived from either path.

`VibesboxDSP/VibesboxKiosk/` — WinUI 3 touchscreen kiosk app for VibesboxDSP. Subscribes to `nowplaying_server` via WebSocket for the Now Playing card.

`VibesboxDSP/reaper-assets/` — ReaLearn presets and JSFX for VibesboxKiosk integration with REAPER.

**Critical startup constraint:** NdiAsioReceiver must be running before REAPER opens its session so the ReaRoute input track has data flowing. RME and ReaRoute are separate ASIO devices and don't contend.

---

## Common Commands

### VibesboxSRC (on the Pi)
```bash
# Check service status
systemctl status auto-router camilladsp ndi-output

# Follow auto-router logs (main daemon)
journalctl -u auto-router -f

# Reload a CamillaDSP config without restarting
# (auto_router does this automatically, but manually:)
curl -s http://localhost:5005/api/setconfig -d @camilladsp/config_2ch_2ch.yml

# Restart the UI stack
sudo systemctl restart sway

# One-time installation (as root)
sudo bash install.sh
```

### VibesboxDSP (on Windows, PowerShell as Administrator)
```powershell
# Build the .NET service
cd "VibesboxDSP\NDI ASIO Receiver"
dotnet publish -c Release -r win-x64 --self-contained -o publish\

# List available ASIO drivers
.\publish\NdiAsioReceiver.exe --list-asio

# Install / manage the Windows Service
.\install-service.ps1 install
.\install-service.ps1 status
.\install-service.ps1 remove
```

---

## Key Design Decisions

- **Dual-Brain architecture (why two devices):** The split is not a workaround — it is a requirement. VibesboxDSP's ASIO engine and VST3 chain (Dirac Live, Penteo 360) require a **fixed, stable sample rate**. VibesboxSRC (the Pi) is a dedicated normalisation engine: it resamples all incoming sources to 96 kHz using CamillaDSP so the DSP unit's clock domain never resets when a source changes.
- **Core purpose — upmixing and room correction:** While the input is typically stereo, the output is **almost always multichannel**. High-quality upmixing (Penteo 360) and room correction (Dirac Live) are only possible with commercial VST3 plugins on Windows (iLok/PACE/UA Connect licensing). Windows is the correct platform choice, not a compromise.
- **NDI transport** — HiFiBerry Digi2 Pro is stereo S/PDIF only; there is no multichannel hardware output on the Pi. NDI over Ethernet is the only viable multichannel path.
- **Double Neutral startup** — CamillaDSP boots with `RawFile`/`Null` (reads `/dev/zero`, writes `/dev/null`). This lets it start without locking any ALSA device; the auto-router hot-swaps the real config when a source becomes active.
- **Zero ALSA resampling** — `defaults.pcm.rate_converter "none"`. All SRC is done by CamillaDSP using the `Synchronous` profile.
- **"Last active wins"** — source auto-switching uses this rule; Bluetooth is always manual.
- **Ring buffer clock drift recovery** — 85ms buffer + 5s watchdog in NdiAsioReceiver handles independent Pi ALSA clock vs. LattePanda RME ASIO clock. The occasional buffer flush (brief silence, immediate recovery) is an accepted trade-off for Hi-Fi home listening where millisecond latency is not critical.
- **Hardening is deferred, not abandoned** — Shell Launcher V2, Keyboard Filter, and UWF are intentionally deferred until the DSP chain and OSC/ReaLearn mappings are finalised. UWF makes every config iteration require a commit before reboot; Shell Launcher V2 hit five separate bring-up issues during Task 8. These will be re-enabled before any uncontrolled deployment. Do not treat their absence as technical debt to clean up.

---

## Think Before Coding

**Don't assume. Don't hide confusion. Surface tradeoffs.**

Before implementing:
- State your assumptions explicitly. If uncertain, ask.
- If multiple interpretations exist, present them - don't pick silently.
- If a simpler approach exists, say so. Push back when warranted.
- If something is unclear, stop. Name what's confusing. Ask.

## Simplicity First

**Minimum code that solves the problem. Nothing speculative.**

- No features beyond what was asked.
- No abstractions for single-use code.
- No "flexibility" or "configurability" that wasn't requested.
- No error handling for impossible scenarios.
- If you write 200 lines and it could be 50, rewrite it.

Ask yourself: "Would a senior engineer say this is overcomplicated?" If yes, simplify.

## Surgical Changes

**Touch only what you must. Clean up only your own mess.**

When editing existing code:
- Don't "improve" adjacent code, comments, or formatting.
- Don't refactor things that aren't broken.
- Match existing style, even if you'd do it differently.
- If you notice unrelated dead code, mention it - don't delete it.

When your changes create orphans:
- Remove imports/variables/functions that YOUR changes made unused.
- Don't remove pre-existing dead code unless asked.

The test: Every changed line should trace directly to the user's request.

## Goal-Driven Execution

**Define success criteria. Loop until verified.**

Transform tasks into verifiable goals:
- "Add validation" → "Write tests for invalid inputs, then make them pass"
- "Fix the bug" → "Write a test that reproduces it, then make it pass"
- "Refactor X" → "Ensure tests pass before and after"

For multi-step tasks, state a brief plan:
```
1. [Step] → verify: [check]
2. [Step] → verify: [check]
3. [Step] → verify: [check]
```

Strong success criteria let you loop independently. Weak criteria ("make it work") require constant clarification.

**These guidelines are working if:** fewer unnecessary changes in diffs, fewer rewrites due to overcomplication, and clarifying questions come before implementation rather than after mistakes.