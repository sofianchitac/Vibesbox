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
Pi 4B (VibesboxSRC)                          LattePanda Mu (VibesboxDSP)
─────────────────────────────────────        ────────────────────────────────────
USB Audio (UAC2) ─┐ per-source               NdiAsioReceiver (Scheduled Task)
Lyrion           ├─ ardftsrc ─► PipeWire     ← NDI 6ch 96kHz FLTP → ReaRoute ASIO
AirPlay          │  (→96kHz)    sum bus       RME Digiface ASIO ← HiFiBerry S/PDIF (2ch)
Bluetooth* ──────┘              │             TotalMix FX (hardware-side routing only)
                                ▼             REAPER (RME + ReaRoute, VST3 DSP chain:
                       CamillaDSP (native       Penteo 360 upmix, Dirac room correction)
                       PipeWire backend)      └─► Amplifiers → Speakers
                       └─ outputs 6ch NDI or  Pi Pico (SOFIAN-MIDI) ─ USB MIDI CC ─► REAPER
                          2ch S/PDIF to DSP    VibesboxKiosk (WinUI 3 touchscreen app)
                       (user selects 2|6ch)    AudioFingerprintService (Scheduled Task)
Touchscreen UI (:80) ←─── source_router WS (:8080)
Now Playing HTTP+WS (:8090) ────────────────────────► VibesboxKiosk subscribes
* Bluetooth: pairing live; audio path deferred (no resampler bridge yet)
```

The **network interfaces** between the two units:
- **Audio**: either NDI `VibesboxSRC-5.1` (6ch, 96kHz, 32-bit float planar) OR S/PDIF 2ch via HiFiBerry → RME Digiface. The user selects 2ch or 6ch output in the Pi UI; the choice is independent of which source is active. The Pi does **no upmixing** — for NDI it passes the 6ch sum bus through (a stereo source sits in FL/FR, rears silent), for S/PDIF it downmixes 6ch→2ch. All stereo-to-surround upmixing happens on the LattePanda (Penteo 360).
- **Now Playing metadata**: `nowplaying_server` on the Pi (:8090) exposes a JSON WS + HTTP API; VibesboxKiosk subscribes via WebSocket. AudioFingerprintService on the LattePanda uploads silence-gated 7-second WAV clips to `POST /api/fingerprint`; the Pi runs `shazamio` server-side and broadcasts matches. Tap is at the REAPER level, so it covers both audio paths (NDI 6ch *and* S/PDIF 2ch) — no per-transport configuration.

Both devices must be on wired Ethernet — Wi-Fi causes audible dropouts on NDI.

---

## VibesboxSRC — Key Files & Components

`VibesboxSRC/scripts/source_router.py` — Central daemon (replaces the v1 `auto_router.py`). Detects source activity at the ALSA layer, starts/stops the per-source ardftsrc bridges, and links/unlinks each source into CamillaDSP over PipeWire (`pw-link`). Pushes the active CamillaDSP config, manages the Bluetooth pairing state machine, and runs the UI WebSocket on :8080. Per-source **mute = unlink**; multiple unmuted sources are summed.

`VibesboxSRC/scripts/ardftsrc_bridge.sh` + `services/ardftsrc-bridge@.service` — Per-source resampler. An ffmpeg/librempeg `ardftsrc` (DFT) filter reads a source at its native rate and emits a `source.<name>.ardftsrc` PipeWire node at 96kHz. One templated instance per active source (USB, Lyrion, AirPlay); source_router owns their lifecycle.

`VibesboxSRC/scripts/ndi_transmitter.py` — Reads CamillaDSP's 6ch output from the `NDITX` ALSA loopback, transmits as `VibesboxSRC-5.1` via the NDI SDK (ctypes/libndi.so). Started/stopped by source_router based on output mode.

`VibesboxSRC/camilladsp/dsp_2ch.yml` / `dsp_6ch.yml` — The two **static** v2 configs (CamillaDSP native PipeWire backend). `dsp_6ch` passes the 6ch sum bus through → NDI; `dsp_2ch` downmixes 6ch→2ch → S/PDIF. Their device blocks are byte-identical, so a 2ch↔6ch output toggle never reopens a PipeWire node. CamillaDSP boots config-less (`-w`) and source_router pushes the active config on connect. (The legacy v1 `*_to_*.yml` matrix is kept on disk but unused.)

`VibesboxSRC/config/pipewire/` + `config/wireplumber/` — System-mode PipeWire (the 96kHz-pinned audio graph that sums sources) and the WirePlumber rules naming the `sink.spdif`, `sink.ndi-feed`, and `source.usb` nodes and assigning output-sink clock authority.

`VibesboxSRC/ui/` — Vanilla HTML/CSS/JS dashboard served by nginx on :80. Connects to source_router WebSocket (:8080) for state and CamillaDSP's native WebSocket (:1234) for RMS meters.

`VibesboxSRC/services/*.service` — systemd unit files. Boot order matters (see below).

`VibesboxSRC/install.sh` — One-time installation script (run as root on the Pi).

### Pi Boot Order (hard dependencies)
1. `alsa-loopback` — loads `snd-aloop` (loopback cards)
2. `usb-gadget` — configures UAC2 6ch USB gadget
3. `pipewire` + `wireplumber` — system-mode audio graph (96kHz) + session manager
4. `camilladsp` — native PipeWire backend, boots config-less (`-w`)
5. `source-router` — links the graph, pushes the CamillaDSP config, starts the :8080 WebSocket
6. `ndi-output` — conditional; managed by source-router
7. Source services (`squeezelite`, `shairport-sync`, `bluealsa`) + on-demand `ardftsrc-bridge@` instances
8. `greetd` + `sway` + Chromium (touchscreen dashboard, 800×480 portrait)

### ALSA Loopback Cards
| Name | Card | Write side | Read side |
|------|------|-----------|-----------|
| Lyrion | card 10 | squeezelite | `ardftsrc-bridge@lyrion` |
| AirPlay | card 11 | shairport-sync | `ardftsrc-bridge@airplay` |
| Bluetooth | card 12 | bluealsa | unused (audio path deferred) |
| NDITX | card 13 | CamillaDSP 6ch output | `ndi_transmitter.py` |

USB Audio is **not** a loopback — `ardftsrc-bridge@usb` reads the raw `hw:UAC2Gadget` capture device directly.

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
systemctl status pipewire wireplumber source-router camilladsp ndi-output

# Follow the main daemon's logs
journalctl -u source-router -f

# Inspect the PipeWire graph (nodes + links)
XDG_RUNTIME_DIR=/run/pipewire pw-link -l

# Restart the UI stack (sway runs under the greetd session, not a system unit)
sudo systemctl restart greetd

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
- **Config-less CamillaDSP startup** — CamillaDSP boots with `-w` (no config) so it starts without locking devices; source_router pushes the active `dsp_<mode>.yml` once PipeWire is up, and that push is what creates CamillaDSP's graph nodes. (Replaces the v1 "Double Neutral" `RawFile`/`Null` boot.)
- **Explicit resampling, no hidden SRC** — `defaults.pcm.rate_converter "none"` keeps ALSA from silently resampling. Coarse rate conversion (source-native → 96kHz) is done **per-source by the ardftsrc bridges** before the PipeWire sum bus; CamillaDSP then runs 96kHz→96kHz only. PipeWire's adaptive resampler stays in the path for clock-drift correction.
- **Source summing with per-source mute** — PipeWire sums every unmuted, playing source; the UI exposes per-source mute toggles (mute = unlink) instead of the v1 "last active wins" single-source switch. Bluetooth pairing is always manual.
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