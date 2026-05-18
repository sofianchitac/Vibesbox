# Vibesbox — System Overview

> **Read this first.** This document describes the Vibesbox system at the architecture and design level — what it is, why it exists, how the two sub-projects relate, and the key decisions made along the way. Each sub-project has its own README with operational and technical detail.

---

## Architectural Philosophy: The "Stable Main Brain" Model

Vibesbox uses a **Dual-Brain architecture** to solve a fundamental challenge in digital audio: **sample rate stability**.

The core DSP chain (Room Correction via Dirac Live, Stereo-to-Multichannel Upmixing via Penteo 360) runs on Windows with professional VST3 plugins. These plugins — and the RME clock domain they operate within — require a **fixed, stable sample rate**. If the ASIO engine were forced to reset its clock every time a source changed sample rate (e.g. switching from a 44.1 kHz AirPlay stream to a 48 kHz USB source), it would interrupt the DSP chain and cause audible artefacts or require a full engine restart.

The solution is **VibesboxSRC (the Pi)**: a dedicated pre-processor that normalises all incoming sources to a fixed **96 kHz** stream before audio ever reaches the DSP unit. The VibesboxDSP (LattePanda) sees one stable clock domain at all times.

> While the input is often stereo, the **output is almost always multichannel**. The upmixing and room correction stages — the core value of the system — are only achievable on a Windows platform with commercial VST3 plugin support (iLok/PACE licensing). This is why a full PC runs the DSP chain rather than a lighter embedded alternative.

---

## What Is Vibesbox?

Vibesbox is a custom, always-on home audio appliance designed to professional studio and live-sound standards. The goal is a living room system that feels like a purpose-built piece of hardware — not a PC running audio software.

Core capabilities:

- **Multiple source inputs** — USB Audio (from a connected computer or player), Lyrion/Squeezebox, AirPlay, and Bluetooth, all selectable via a touchscreen or automatically.
- **High-quality multichannel DSP** — room correction, per-speaker management, upmixing, and dynamic limiting running on dedicated hardware with professional-grade VST3 plugins.
- **A polished touch UI** — a touchscreen on each device provides source control and system status.
- **Hardware physical controls** — a Pi Pico MIDI controller with three potentiometers provides real-time tactile control of key parameters.

The system is split into two sub-projects running on separate, dedicated hardware, with a single well-defined network interface between them.

---

## Hardware Overview

| Device | Hardware | Role |
|---|---|---|
| VibesboxSRC | Raspberry Pi 4B (4 GB) | Source selection & resampling |
| VibesboxDSP | LattePanda Mu (Intel N100) | Full DSP chain & speaker output |
| Audio I/O (primary) | RME Digiface USB | ASIO hub on the DSP unit |
| Audio I/O (secondary) | Audient iD24 (via ADAT) | Additional I/O on the DSP unit |
| Pi touchscreen | Waveshare 5" DSI LCD, 800×480 | Source selection UI on the Pi |
| LattePanda touchscreen | Salvaged iPad panel, 1024×768 multitouch | Custom UI on the DSP unit |
| MIDI controller | Pi Pico USB, device name `SOFIAN-MIDI` | 3× potentiometers for real-time DSP control |
| Speakers | Dynaudio Emit 20 (stereo), KRK Rokit 8, XTZ Sub 10 | Output transducers |

---

## The Two Sub-Projects

```
┌─────────────────────────────────────────┐            ┌──────────────────────────────────────────┐
│            VibesboxSRC                  │            │              VibesboxDSP                 │
│         (Raspberry Pi 4B)               │            │      (LattePanda Mu / Windows 11)        │
│                                         │            │                                          │
│  Audio Sources (any one active)         │            │  NdiAsioReceiver (Scheduled Task,        │
│  ┌──────────────┐                       │            │   user session — required by ReaRoute)   │
│  │ USB Audio    │──┐                    │            │  Receives NDI 6ch → writes to ReaRoute   │
│  │ (UAC2)       │  │                    │            │  ↓                                       │
│  ├──────────────┤  │                    │            │  REAPER (RME ASIO for hardware I/O;      │
│  │ Lyrion /     │──┤  CamillaDSP        │            │   ReaRoute as a parallel ASIO transport  │
│  │ Squeezebox   │  │  Synchronous       │            │   for NDI 6ch inputs)                    │
│  ├──────────────┤  │  resample          │            │  ↓                                       │
│  │ AirPlay      │──┤  → 96 kHz          │            │  Full DSP chain (Penteo upmix, Dirac     │
│  │ (shairport)  │  │  → upmix/downmix   │            │   room correction, crossovers, limiting) │
│  ├──────────────┤  │  per user-selected │            │  ↓                                       │
│  │ Bluetooth    │──┘  output channels   │            │  ASIO outputs → Amplifiers → Speakers    │
│  │ (bluealsa)   │                       │            │                                          │
│  └──────────────┘                       │            │  Pi Pico MIDI (CC0 vol, CC1/CC2 tone)    │
│                                         │            │  VibesboxKiosk (WinUI 3, 1024×768)       │
│  Output (user selects 2ch or 6ch in the │            │  AudioFingerprintService taps REAPER     │
│  Pi UI; works for ANY source):          │            │   → uploads WAV to Pi /api/fingerprint   │
│   • 2ch mode: HiFiBerry S/PDIF → RME    │  NDI 6ch   │                                          │
│   • 6ch mode: NDI to NdiAsioReceiver   ─┼──────────► │                                          │
│                                         │  S/PDIF 2ch│                                          │
│                                        ─┼──────────► │                                          │
│  auto_router.py — source detection,     │            │                                          │
│  config switching, Bluetooth mgmt,      │            │                                          │
│  WebSocket state → touchscreen UI       │            │                                          │
│                                         │            │                                          │
│  Now Playing pipeline:                  │            │                                          │
│   nowplaying_server (HTTP+WS :8090)     │  WS push   │                                          │
│   metadata_orchestrator → producers    ─┼──────────► │  VibesboxKiosk subscribes to WS          │
│   (shairport / lms / bluealsa /         │            │  (NowPlayingCard — Acrylic overlay on    │
│    fingerprint / generic)               │            │    the Spectrum row, auto-fade 5s)       │
│   shazamio runs server-side on POST     │            │                                          │
│   /api/fingerprint                      │ WAV upload │                                          │
│                                        ◄┼──────────  │                                          │
└─────────────────────────────────────────┘            └──────────────────────────────────────────┘
                    │                                             │
                    └──────────── Local wired Ethernet ───────────┘
```

> **Topology note (important).** There is no source-to-transport mapping. *Any* source (USB Audio, Lyrion, AirPlay, Bluetooth) can be routed via either output transport (2ch S/PDIF or 6ch NDI). The user picks **2ch or 6ch** in the Pi touchscreen UI; CamillaDSP handles upmixing (2ch → 6ch via Hafler) or downmixing (6ch → 2ch) accordingly. The four CamillaDSP configs cover all combinations.

---

## VibesboxSRC — Source Selection & Resampling

**Repository:** [VibesboxSRC](https://github.com/sofianchitac/VibesboxSRC)

The Pi acts as a headless source-selection and resampling appliance. All audio inputs are normalised to a single internal format and transmitted to the DSP unit over the network (6ch) or S/PDIF (2ch).

**Architecture — key components:**

- **CamillaDSP (v4.1.3)** — open-source Rust-based DSP engine running as a systemd service. Performs all sample-rate conversion using the `Synchronous` resampler profile. All routing configurations (`2ch→2ch`, `2ch→6ch`, `6ch→2ch`, `6ch→6ch`) apply -4 dB intersample headroom protection before the mixer stage.

- **auto_router.py** — the central Python daemon. Polls ALSA `hw_params` and the UAC2 Gadget capture rate control every 200 ms to detect which source is active, then hot-swaps the live CamillaDSP configuration without restarting the process. Also manages Bluetooth power and pairing state, and exposes a WebSocket server on port 8080 for the touchscreen UI.

- **USB Audio Gadget (UAC2)** — the Pi presents itself to a connected source device (iPad, PC, etc.) as a 6-channel USB audio device (using `libcomposite`/`dwc2`), supporting sample rates from 44.1 kHz to 192 kHz. `camilladsp-controller` watches the incoming rate and updates CamillaDSP in real time.

- **ndi_transmitter.py** — reads CamillaDSP's 6ch processed output from the `NDITX` ALSA loopback and transmits it as the NDI source `VibesboxSRC-5.1` using the NDI SDK v6 via ctypes. Stopped automatically when 2ch output mode is selected to save CPU.

- **Touchscreen UI** — a vanilla HTML/CSS/JavaScript web dashboard served by nginx (port 80), rendered in Chromium kiosk mode via Sway on the 800×480 Waveshare DSI display setuped in portrait mode. Connects to the auto-router WebSocket for live state and to CamillaDSP's native WebSocket for RMS level meters.

**Routing matrix:**

| Input | Output | Path | Use case |
|---|---|---|---|
| 2ch (Lyrion / AirPlay / Bluetooth) | 2ch | CamillaDSP → HiFiBerry S/PDIF | 2ch input to REAPER via S/PDIF |
| 2ch | 6ch | CamillaDSP + Hafler upmix → NDI | 6ch upmixed input to REAPER via NDI |
| 6ch (USB) | 6ch | CamillaDSP passthrough → NDI | 6ch input to REAPER via NDI |
| 6ch (USB) | 2ch | CamillaDSP downmix → HiFiBerry S/PDIF | 2ch downmixed input to REAPER via S/PDIF |

**Source priority:** "last active wins" for auto-switching. Bluetooth is always manual (requires explicit UI tap to activate or pair).

**Key design decision — Double Neutral startup:** CamillaDSP starts with a `RawFile`/`Null` neutral config (reading from `/dev/zero`, writing to `/dev/null`). This allows it to boot in milliseconds without locking any ALSA device. The auto-router swaps in the real hardware config only when a source becomes active, and returns to neutral when all sources go idle — releasing ALSA handles so sample-rate changes can propagate.

---

## VibesboxDSP — The DSP Processing Unit

**Repository:** [VibesboxDSP](https://github.com/sofianchitac/VibesboxDSP) (NdiAsioReceiver + REAPER session)

The LattePanda Mu runs Windows 11 and hosts the entire DSP chain. It is configured and optimised as a near-real-time audio appliance.

**Architecture — key components:**

- **NdiAsioReceiver** (Scheduled Task at user logon, [VibesboxDSP/NDI ASIO Receiver]) — a .NET app that receives the `VibesboxSRC-5.1` NDI stream and writes it into REAPER via the **ReaRoute ASIO** virtual driver (NOT the RME ASIO driver). REAPER reads the 6 channels as software inputs alongside its main RME Digiface ASIO device, leaving all RME hardware loopback channels free for hardware sources (S/PDIF, ADAT). Runs as a Scheduled Task — not a Windows Service — because ReaRoute is session-bound and only routes audio between processes in the same Windows user session. Automatic reconnect on source loss. Includes a ring buffer watchdog that detects and resolves NDI/ASIO clock drift.

- **TotalMix FX** — RME's hardware mixer application. Routes RME hardware inputs (S/PDIF, ADAT) into REAPER and routes REAPER's RME ASIO outputs to the speaker amps. The NDI path bypasses TotalMix entirely (NDI → ReaRoute → REAPER); TotalMix is only on the hardware-I/O side.

- **REAPER** — runs as a scheduled task at startup and hosts the VST3 plugin DSP chain. Also receives MIDI CC from the Pi Pico controller for real-time parameter control.

- **Pi Pico MIDI controller** — three physical potentiometers, presented to Windows as a USB MIDI device (`SOFIAN-MIDI`), sending MIDI CC0 (master volume), CC1 and CC2 (tone parameters) on channel 1.

See [VibesboxDSP/README.md](VibesboxDSP/README.md) for full operational detail.

---

## The NDI Transport — Design Decision

**Why NDI?**

NDI (Network Device Interface) is a professional IP audio/video protocol by Vizrt designed for zero-configuration LAN transport of uncompressed streams. For this system:

- The HiFiBerry Digi2 Pro fitted to the Pi is a **stereo S/PDIF output only**. There is no multichannel hardware output path on the Pi. This is the fundamental, non-negotiable constraint that drives the entire transport architecture.
- NDI carries **6 channels of 32-bit float audio at 96 kHz** over standard Ethernet with no special hardware.
- NDI handles the network cleanly: mDNS-based source discovery, reliable stream framing, and straightforward reconnection.

**Alternatives considered:**

| Option | Reason rejected |
|---|---|
| S/PDIF (HiFiBerry) | Stereo only — hardware constraint, not negotiable |
| AES67 / Dante | Significant cost and complexity for a home system |
| JACK over network | Requires JACK on both ends; fragile on Windows |
| USB audio from Pi to LattePanda | USB host/device topology doesn't fit; latency unpredictable |

**Clock domain bridging:**

The Pi runs its own ALSA clock; the LattePanda runs on the RME ASIO clock (which ReaRoute follows). These are independent and will drift. NdiAsioReceiver handles this with:

1. A ring buffer (~85 ms at 96 kHz) that absorbs NDI jitter and clock drift.
2. A watchdog that monitors fill level every 5 seconds and flushes the buffer when drift accumulates beyond 93% capacity (brief silence then immediate recovery).

**Accepted trade-off:** NDI does not carry a native clock signal, so buffer flushes (brief silence, immediate recovery) are the mechanism for managing long-term drift. This is a known and acceptable trade-off for **Hi-Fi home listening**, where millisecond-level latency is not a critical constraint as it would be in a live performance context. The stability of the fixed 96 kHz chain on the DSP side is prioritised above all else.

---

## NDI Stream Parameters

| Parameter | Value |
|---|---|
| NDI source name | `VibesboxSRC-5.1` |
| Channels | 6 (5.1 layout: FL, FR, FC, LFE, RL, RR) |
| Sample rate | 96 000 Hz |
| Format | 32-bit float planar (FLTP) |
| Chunk / period size | 1 024 frames (~10.7 ms) |
| Transport | NDI SDK v6 via libndi.so (Pi) / Processing.NDI.Lib.x64.dll (Windows) |

---

## Network Topology

Both devices must be on the same wired Ethernet segment. Wi-Fi introduces jitter that will exceed the ring buffer tolerance and cause audible dropouts.

```
Pi 4B ──────── Ethernet switch ──────── LattePanda Mu
 (vibesbox-src.local)                   (vibesbox-dsp)
 NDI TX: VibesboxSRC-5.1   ─────────►  NdiAsioReceiver (Scheduled Task)
 auto_router WS:    :8080   (LAN)       REAPER + TotalMix FX
 nowplaying WS+HTTP :8090               ReaRoute ASIO + RME ASIO
 CamillaGUI:        :5005               AudioFingerprintService (Scheduled Task)
```

NDI source discovery uses mDNS/Avahi — no static IP configuration required, provided both devices are on the same subnet.

---

## Boot Sequence

Understanding the startup order matters because audio services have hard dependencies.

**Pi (VibesboxSRC):**
1. `alsa-loopback.service` — loads `snd-aloop` with 4 virtual loopback cards (Lyrion, AirPlay, Bluetooth, NDITX).
2. `usb-gadget.service` — configures the UAC2 6ch USB gadget via `libcomposite`.
3. `camilladsp.service` — starts CamillaDSP with the neutral `RawFile`/`Null` config (`-w` flag keeps WebSocket port alive).
4. `auto-router.service` — connects to CamillaDSP, begins polling, starts WebSocket server on :8080.
5. `ndi-output.service` — starts only when 6ch output mode is active; stopped by auto-router when 2ch is selected.
6. Source services (`squeezelite`, `shairport-sync`, `bluealsa`) start independently.
7. `greetd` + `sway` + Chromium — launches the touchscreen dashboard.

**LattePanda (VibesboxDSP):**
1. `NdiAsioReceiver` Scheduled Task triggers at user logon — opens **ReaRoute ASIO** (NOT RME), outputs silence, begins NDI discovery. Must be in the user session (not Session 0) so ReaRoute can route audio to REAPER.
2. TotalMix FX — handles only the RME hardware-I/O side (S/PDIF in from Pi, analogue/ADAT out to amps).
3. REAPER — started via Windows Task Scheduler at login. Opens RME Digiface ASIO as its primary device for hardware I/O, and reads ReaRoute as a secondary device for the NDI 6ch input.
4. `AudioFingerprintService` Scheduled Task triggers at user logon — reads ReaRoute input channels 1-2 (where REAPER has been routed the USB-input track's front L/R, pre-FX). Silence-gated 7-second WAV clips are uploaded to the Pi's `POST /api/fingerprint`. **The Pi runs `shazamio` server-side** and broadcasts matches; the Windows side is a thin capture-and-upload client. The tap captures any source the user is listening to via REAPER, so it works for both NDI 6ch and S/PDIF 2ch transport paths.

**Critical ordering:** NdiAsioReceiver must be running before REAPER opens its session, so that REAPER's ReaRoute input track has data flowing. The RME Digiface ASIO driver is independent of ReaRoute — they're separate ASIO devices and don't contend.

---

## Status & Future Work

### V1.0 — Operational

The system is currently deployed and in daily use. The VibesboxKiosk (WinUI 3 touchscreen app) is running on the LattePanda. Both sub-projects are stable.

The **Now Playing pipeline** is fully operational:
- Pi side: `nowplaying_server` + `metadata_orchestrator` + four producers (shairport, lms, bluealsa, generic) deployed as systemd services. Each transport-tier source (AirPlay / Lyrion / Bluetooth) emits structured metadata; USB Audio falls through to fingerprinting.
- Windows side: `AudioFingerprintService` taps REAPER via ReaRoute, uploads silence-gated 7-second WAV clips to the Pi. The Pi runs `shazamio` server-side on the `/api/fingerprint` endpoint and broadcasts matches over the same WS channel as the other producers. Works for any audio reaching REAPER, regardless of which transport (NDI 6ch or S/PDIF 2ch) the Pi is currently using.
- Kiosk side: WinUI 3 `NowPlayingCard` overlays the SpectrumView with an Acrylic backdrop, auto-fades after 5 s, re-triggers on track change or 5 s before track end, tap-toggles. Three layouts (generic, centered-text, album-art) chosen by payload shape. Bluetooth swaps the orange palette to cyan.

### Deferred Hardening (planned, not abandoned)

The following Windows IoT lockdown features were **strategically deferred** during bring-up to avoid iteration friction while the OSC/ReaLearn mappings and DSP chain are being finalised. They remain the plan for final deployment:

- **Shell Launcher V2** — replaces `explorer.exe` with the kiosk EXE as the user shell. Encountered five separate bring-up issues during Task 8; documented in `README-Kiosk.md` for future revisit.
- **Keyboard Filter** — blocks Win key, Alt+Tab, and other escape routes on the touchscreen panel.
- **Unified Write Filter (UWF)** — freezes the C: drive against power-cut corruption. Not enabled during stabilisation because every OS-level change requires a UWF commit; re-enable before any deployment in an uncontrolled environment.

### Unified Control

- **VibesboxKiosk enhancements** — a web-based remote interface accessible from any device on the LAN.
