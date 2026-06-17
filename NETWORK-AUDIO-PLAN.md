# Network Audio Receiver Plan (DIY Sonos)

## Problem

opti.local is a 4.1 speaker system (CM6206 + PipeWire) that currently plays audio from local sources only (Navidrome). The goal is to make any device on the network cast audio to it, like a Sonos speaker.

## Architecture

```
iPhone/Android (AirPlay) --> shairport-sync --> PipeWire --+
Windows PC (VBAN)          --> vban-recv module --> PipeWire --+--> CM6206 4.1
Linux PC (RTP)             --> rtp-source module --> PipeWire --+    (PSD upmix)
Navidrome                  --> PipeWire (existing) -----------+
```

All sources arrive as PipeWire source nodes. WirePlumber auto-routes them to the CM6206 pro-audio sink. Existing PSD upmix (upmix.conf) handles stereo to 4.1 for all inputs.

## Components

### 1. AirPlay Receiver (shairport-sync)

- Package: shairport-sync v4.3.7 (apt)
- Backend: pipewire (native PipeWire output, not ALSA compat layer)
- Service type: user service (required because PipeWire is a user service)
- Config: /etc/shairport-sync.conf (system-wide)
- Systemd: ~/.config/systemd/user/shairport-sync.service
- Device name: "opti" (appears in AirPlay picker)
- Key settings:
  - output_backend = "pipewire"
  - allow_session_interruption = "yes" (new AirPlay session interrupts current)
  - AirPlay 2 support (requires nqptp, check if v4.3.7 includes it)
- Android compatibility: works via AirMusic, AirScreen, Solid Stream apps
- Windows compatibility: not native (use VBAN or RTP instead)

### 2. PipeWire RTP Receiver (native module)

- Module: libpipewire-module-rtp-session (already installed on system)
- How it works: auto-discovers other PipeWire machines via mDNS, creates bidirectional RTP audio streams
- Config: /etc/pipewire/pipewire.conf.d/rtp-session.conf
- Key settings:
  - sess.media = "audio" (default is midi)
  - sess.name = "opti"
  - audio.channels = 2, audio.rate = 48000, audio.position = [ FL FR ]
- No additional software needed on sending Linux machine (same PipeWire RTP config)
- Multi-channel: supports audio.channels up to 8 with audio.position mapping

### 3. VBAN Receiver (native PipeWire module)

- Module: libpipewire-module-vban-recv (already installed on system)
- How it works: receives VBAN UDP packets, creates PipeWire source per stream
- Windows sender: VB-Audio Voicemeeter (free) or VBAN Sender
- Config: /etc/pipewire/pipewire.conf.d/vban-recv.conf
- Key settings:
  - source.ip = "0.0.0.0" (receive from any sender)
  - source.port = 6980 (VBAN default)
  - stream.rules with create-stream action
- Multi-channel: up to 8 channels at 24-bit/96kHz
- Advantage over RTP for Windows: VB-Audio Voicemeeter is widely used, free, and sends multi-channel VBAN natively

### 4. Snapcast (disabled, not removed)

- Action: sudo systemctl disable --now snapserver snapclient
- Packages remain installed, configs in ~/repos/snapcast-setup/ preserved
- Re-enable: sudo systemctl enable --now snapserver snapclient if multi-room needed later

## Files Touched

| File | Action | Stored In |
|---|---|---|
| /etc/shairport-sync.conf | Create (system-wide AirPlay config) | configs/shairport-sync.conf |
| ~/.config/systemd/user/shairport-sync.service | Create (user service unit) | configs/shairport-sync.service |
| /etc/pipewire/pipewire.conf.d/rtp-session.conf | Create (RTP receiver config) | configs/rtp-session.conf |
| /etc/pipewire/pipewire.conf.d/vban-recv.conf | Create (VBAN receiver config) | configs/vban-recv.conf |
| Snapserver/snapclient services | Disable (no file changes) | N/A |

## Design Decisions

1. System-wide configs (/etc/pipewire/, /etc/shairport-sync.conf) so all users benefit
2. User service for shairport-sync because PipeWire is a user service (system service cannot access PipeWire)
3. PSD upmix applies to all inputs automatically (existing upmix.conf) -- no per-source upmix config needed
4. No conflict resolution between sources -- PipeWire mixes concurrent streams. User manages source selection manually (do not cast from two devices simultaneously). Shairport-sync handles AirPlay session handoff.
5. VBAN over ScreamRouter for Windows -- simpler (native PipeWire module, no Docker), and VB-Audio Voicemeeter is the de facto standard on Windows
6. RTP session module for Linux to Linux -- auto-discovery via mDNS, zero config on sender side

## Implementation Order

1. Disable snapcast services
2. Install and configure shairport-sync (quickest win)
3. Configure PipeWire RTP session module
4. Configure PipeWire VBAN receiver module
5. Test each input method
6. Update README and configs in this repo

## Open Questions / Risks

- shairport-sync v4.3.7 from apt: may not include PipeWire backend (needs --with-pipewire at build time). If apt package lacks it, fall back to ALSA backend (PipeWire's ALSA compat layer works). Verify after install with shairport-sync -V.
- AirPlay 2: requires nqptp daemon. v4.3.7 may support it but needs separate package. AirPlay 1 works without it. Lower priority.
- Firewall: RTP uses multicast (224.0.0.56), VBAN uses UDP 6980. May need firewall rules if ufw is active.
- Latency: AirPlay adds ~2s latency (inherent to protocol). VBAN and RTP are sub-100ms. Not an issue for music playback, but matters for video sync.

## Install Locations

| File | Per-User Location | System-Wide Location |
|---|---|---|
| configs/shairport-sync.conf | ~/.config/shairport-sync.conf | /etc/shairport-sync.conf |
| configs/shairport-sync.service | ~/.config/systemd/user/ | N/A (user service only) |
| configs/rtp-session.conf | ~/.config/pipewire/pipewire.conf.d/ | /etc/pipewire/pipewire.conf.d/ |
| configs/vban-recv.conf | ~/.config/pipewire/pipewire.conf.d/ | /etc/pipewire/pipewire.conf.d/ |
