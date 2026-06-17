# Audio Setup: CM6206 Quadraphonic + Subwoofer (4.1)

## Hardware

- **Sound card**: CM6206 (USB ID `0d8c:0102`, sold as ICUSBAUDIO7D)
- **Front speakers**: Edifier powered pair on green jack (Front L/R)
- **Rear speakers**: Edifier powered pair on black jack (Rear L/R)
- **Subwoofer**: B&W ASW608 on orange jack (LFE), LPF IN, crossover 80Hz
- **OS**: Kubuntu 26.04 LTS, PipeWire 1.6.2, WirePlumber 0.5.13

## Channel Mapping

The CM6206 only activates all DACs in 8-channel mode. Lower channel counts (2, 4, 6) do not route audio to all jacks. The 8-channel ALSA mapping:

| ALSA Channel | Label | Physical Jack | Connected To |
|---|---|---|---|
| 0 | Front Left | Green | Front left Edifier |
| 1 | Front Right | Green | Front right Edifier |
| 2 | Front Center | Orange | Unused (no center speaker) |
| 3 | LFE | Orange | B&W ASW608 subwoofer |
| 4 | Rear Left | Black | Rear left Edifier |
| 5 | Rear Right | Black | Rear right Edifier |
| 6 | Side Left | - | Unused |
| 7 | Side Right | - | Unused |

## Setup Steps

### 1. Switch PipeWire profile to pro-audio

The CM6206 requires the `pro-audio` profile for 8-channel output. ACP profiles (surround-41, surround-51) do not activate all DACs.

```bash
pactl set-card-profile alsa_card.usb-0d8c_USB_Sound_Device-00 pro-audio
```

### 2. Install ALSA `.asoundrc`

Defines a `surround41_cm6206` ALSA device that opens the hardware in 8-channel mode and maps 5 channels (FL FR LFE RL RR) to the correct hardware channels. This is needed for ALSA-native playback and for the `speaker-test` verification.

```bash
cp configs/asoundrc ~/.asoundrc
```

Config file: [configs/asoundrc](configs/asoundrc)

### 3. Install WirePlumber channel map rule

Pro-audio mode labels all channels as generic `AUX0`-`AUX7`. The WirePlumber rule remaps them to their real positions so PipeWire understands the layout. Also sets channelmix properties for upmix on the sink itself.

```bash
mkdir -p ~/.config/wireplumber/wireplumber.conf.d
cp configs/50-cm6206-channel-map.conf ~/.config/wireplumber/wireplumber.conf.d/
systemctl --user restart wireplumber
```

Config file: [configs/50-cm6206-channel-map.conf](configs/50-cm6206-channel-map.conf)

### 4. Install PipeWire-Pulse upmix config

Configures PipeWire's built-in channelmix to upmix stereo sources to multi-channel. Uses the PSD (Passive Surround Decoding) algorithm which extracts ambient/difference signal for rear channels. Applied system-wide via pipewire-pulse stream properties.

```bash
mkdir -p ~/.config/pipewire/pipewire-pulse.conf.d
cp configs/upmix.conf ~/.config/pipewire/pipewire-pulse.conf.d/
systemctl --user restart pipewire pipewire-pulse
```

Config file: [configs/upmix.conf](configs/upmix.conf)

### 5. Install VBAN receiver + upmix sink (network audio from Windows)

Receives 2-channel VBAN audio from a Windows PC (running Voicemeeter), then upmixes it to 4.1 using PipeWire's PSD algorithm. This makes opti function as a network audio receiver similar to a Sonos speaker.

**Audio chain**: Windows (Voicemeeter) --VBAN 2ch--> opti (vban-recv) --> vban-upmix (loopback sink) --> PSD upmix --> CM6206 (FL FR RL RR FC LFE)

```bash
# VBAN receiver module
mkdir -p ~/.config/pipewire/pipewire.conf.d
cp configs/vban-recv.conf ~/.config/pipewire/pipewire.conf.d/

# Loopback upmix sink (2ch in, 6ch out with PSD upmix)
cp configs/vban-upmix-sink.conf ~/.config/pipewire/pipewire.conf.d/

# WirePlumber routing rule (routes VBAN stream to upmix sink)
cp configs/51-vban-routing.conf ~/.config/wireplumber/wireplumber.conf.d/

# Restart everything
systemctl --user restart pipewire pipewire-pulse wireplumber
```

Config files:
- [configs/vban-recv.conf](configs/vban-recv.conf) - VBAN receiver (listens on 0.0.0.0:6980)
- [configs/vban-upmix-sink.conf](configs/vban-upmix-sink.conf) - Loopback upmix sink with PSD
- [configs/51-vban-routing.conf](configs/51-vban-routing.conf) - WirePlumber routing rule

**Windows setup (Voicemeeter Banana)**:
1. Install [Voicemeeter Banana](https://vb-audio.com/Voicemeeter/banana.htm)
2. Set A1 output to your local speakers (WDM Realtek or similar)
3. In VBAN Out tab, add a new stream:
   - Destination: opti.local (or 192.168.100.15)
   - Port: 6980
   - Channels: 2
   - Sample rate: 48000
   - Format: 16-bit
4. Enable the VBAN Out stream
5. Audio from Voicemeeter Input (default Windows playback device) will be sent to opti

**Important**: Only 2-channel VBAN works. Multi-channel VBAN (>2ch) causes severe distortion due to VB-Cable virtual I/O limitations on Windows (2ch max since Vista). The upmix is handled by PipeWire on opti instead.

### 6. Set as default sink

```bash
# Find the sink ID
wpctl status

# Set as default (ID may change, verify with wpctl status)
wpctl set-default <sink-id>
pactl set-default-sink alsa_output.usb-0d8c_USB_Sound_Device-00.pro-output-0
```

### 7. Set ALSA mixer volumes

```bash
# All channels (80% is a safe starting point)
amixer -c 1 sset 'Speaker' 80%

# PipeWire sink volume
wpctl set-volume <sink-id> 75%
```

### 8. Verify

```bash
# Direct ALSA 8-channel test (all jacks, known working)
speaker-test -c 8 -D hw:1 -t wav -l 1

# ALSA surround41 test (5 channels mapped to 8ch hardware, known working)
speaker-test -c 5 -D surround41_cm6206 -t wav -l 1

# PipeWire stereo test (front speakers)
paplay /usr/share/sounds/freedesktop/stereo/bell.oga

# PipeWire upmix test - rears (needs stereo with L/R difference)
ffmpeg -y -f lavfi -i "aevalsrc=0.7*sin(440*2*PI*t)|0.7*sin(880*2*PI*t):duration=3" -ar 48000 /tmp/test_rear_stereo.wav
paplay /tmp/test_rear_stereo.wav

# VBAN routing check
pw-link -l | grep -A2 vban-recv
pw-link -l | grep -A2 vban-upmix-output
```

## Current Status

- [x] Hardware connected (green/black/orange jacks)
- [x] Pro-audio profile activated
- [x] WirePlumber channel map rule installed
- [x] Default sink configured
- [x] ALSA 8-channel output verified (all speakers + sub)
- [x] ALSA surround41 device verified (all speakers + sub)
- [x] PipeWire stereo playback on front speakers
- [x] PipeWire upmix to rear speakers (PSD method, confirmed working)
- [x] PipeWire upmix to LFE/sub (lfe-cutoff=150, mix-lfe=true)
- [x] VBAN network audio receiver (2ch from Windows)
- [x] VBAN upmix to 4.1 via loopback sink (persistent WirePlumber routing)
- [ ] Per-channel volume tuning
- [ ] EasyEffects software crossover (optional, future)
- [ ] AirPlay receiver (shairport-sync, dormant - mDNS not advertising)

## Shairport-sync (AirPlay Receiver - Dormant)

Built from source as classic AirPlay 1 (no `--with-airplay-2`). Runs as a user service on port 5000. Currently dormant because mDNS advertisement is not working (Avahi does not register the service). The VBAN setup is the primary network audio path.

Config files:
- [configs/shairport-sync.conf](configs/shairport-sync.conf) - Shairport-sync config
- [configs/shairport-sync.service](configs/shairport-sync.service) - Systemd user service

Install:
```bash
cp configs/shairport-sync.conf /etc/shairport-sync.conf
cp configs/shairport-sync.service ~/.config/systemd/user/
systemctl --user daemon-reload
systemctl --user enable --now shairport-sync
```

## Known Quirks

- **CM6206 requires 8-channel mode**: 2/4/6 channel ALSA altsets do not route audio to all physical jacks. Only altset 1 (8ch) works.
- **Pro-audio profile tradeoff**: Loses ACP profile switching and per-port volume control. Gains raw multi-channel access.
- **No center speaker**: Channel 2 (Front Center) is unused. Stereo upmix must skip center and route to quad + sub only.
- **LFE upmix requires lfe-cutoff >= 150 and mix-lfe=true**: With `lfe-cutoff = 80`, PipeWire's channelmix did not produce LFE output from stereo sources. Setting `lfe-cutoff = 150` and `mix-lfe = true` fixed it. The hardware sub crossover is set to 80Hz, so the higher software cutoff still gets filtered correctly by the B&W's hardware crossover.
- **PSD upmix requires L/R difference**: Identical content on both channels produces no rear output. This is correct behavior (PSD extracts ambient/difference signal) but can be confusing during testing. Use `simple` method instead if you want front copied to rear regardless.
- **VBAN multi-channel distortion**: VB-Cable virtual I/O on Windows is limited to 2 channels since Windows Vista. Setting VBAN to >2 channels in Voicemeeter causes severe distortion. Use 2ch VBAN and let PipeWire upmix on the receiver side.
- **VBAN upmix routing**: WirePlumber needs `target.object = "vban-upmix"` set in both the VBAN module's `stream.props` (top-level) and the `create-stream` action. The `node.dont-fallback = true` prevents WirePlumber from falling back to the default sink when the upmix sink isn't ready yet.
- **Shairport-sync mDNS not advertising**: Classic AP1 build does not register its mDNS service with Avahi. Root cause unknown. PigeonCast (Android) could detect the device when running in AirPlay 2 mode, but AP2 mode rejects NTP-timed streams with "can not handle NTP streams" error. Classic AP1 mode supports NTP but doesn't advertise.

## Install Locations

Configs can be installed per-user or system-wide. System-wide makes the 4.1 setup available to all users on the machine.

| File | Per-User Location | System-Wide Location |
|---|---|---|
| `configs/asoundrc` | `~/.asoundrc` | `/etc/asound.conf` |
| `configs/50-cm6206-channel-map.conf` | `~/.config/wireplumber/wireplumber.conf.d/` | `/etc/wireplumber/wireplumber.conf.d/` |
| `configs/upmix.conf` | `~/.config/pipewire/pipewire-pulse.conf.d/` | `/etc/pipewire/pipewire-pulse.conf.d/` |
| `configs/vban-recv.conf` | `~/.config/pipewire/pipewire.conf.d/` | `/etc/pipewire/pipewire.conf.d/` |
| `configs/vban-upmix-sink.conf` | `~/.config/pipewire/pipewire.conf.d/` | `/etc/pipewire/pipewire.conf.d/` |
| `configs/51-vban-routing.conf` | `~/.config/wireplumber/wireplumber.conf.d/` | `/etc/wireplumber/wireplumber.conf.d/` |
| `configs/shairport-sync.conf` | - | `/etc/shairport-sync.conf` (requires sudo) |
| `configs/shairport-sync.service` | `~/.config/systemd/user/` | - |

### System-wide install (requires sudo)

```bash
sudo cp configs/asoundrc /etc/asound.conf
sudo mkdir -p /etc/wireplumber/wireplumber.conf.d
sudo cp configs/50-cm6206-channel-map.conf /etc/wireplumber/wireplumber.conf.d/
sudo mkdir -p /etc/pipewire/pipewire-pulse.conf.d
sudo cp configs/upmix.conf /etc/pipewire/pipewire-pulse.conf.d/
sudo mkdir -p /etc/pipewire/pipewire.conf.d
sudo cp configs/vban-recv.conf /etc/pipewire/pipewire.conf.d/
sudo cp configs/vban-upmix-sink.conf /etc/pipewire/pipewire.conf.d/
sudo cp configs/shairport-sync.conf /etc/shairport-sync.conf
```

After system-wide install, the per-user copies in `~/.config/` and `~/.asoundrc` can be removed since the system-level configs take precedence when no user override exists.
