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

### 5. Set as default sink

```bash
# Find the sink ID
wpctl status

# Set as default (ID may change, verify with wpctl status)
wpctl set-default <sink-id>
pactl set-default-sink alsa_output.usb-0d8c_USB_Sound_Device-00.pro-output-0
```

### 6. Set ALSA mixer volumes

```bash
# All channels (80% is a safe starting point)
amixer -c 1 sset 'Speaker' 80%

# PipeWire sink volume
wpctl set-volume <sink-id> 75%
```

### 7. Verify

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
- [ ] Per-channel volume tuning
- [ ] EasyEffects software crossover (optional, future)
- [ ] Network audio receiver setup (see [NETWORK-AUDIO-PLAN.md](NETWORK-AUDIO-PLAN.md))

## Known Quirks

- **CM6206 requires 8-channel mode**: 2/4/6 channel ALSA altsets do not route audio to all physical jacks. Only altset 1 (8ch) works.
- **Pro-audio profile tradeoff**: Loses ACP profile switching and per-port volume control. Gains raw multi-channel access.
- **No center speaker**: Channel 2 (Front Center) is unused. Stereo upmix must skip center and route to quad + sub only.
- **LFE upmix requires lfe-cutoff >= 150 and mix-lfe=true**: With `lfe-cutoff = 80`, PipeWire's channelmix did not produce LFE output from stereo sources. Setting `lfe-cutoff = 150` and `mix-lfe = true` fixed it. The hardware sub crossover is set to 80Hz, so the higher software cutoff still gets filtered correctly by the B&W's hardware crossover.
- **PSD upmix requires L/R difference**: Identical content on both channels produces no rear output. This is correct behavior (PSD extracts ambient/difference signal) but can be confusing during testing. Use `simple` method instead if you want front copied to rear regardless.
- **Filter-chain module did not work**: PipeWire's `libpipewire-module-filter-chain` with builtin DSP nodes (convolver, delay, bq_lowpass) produced no output despite correct port links. Root cause unknown. The loopback module approach also failed for LFE. The working solution is system-wide channelmix via pipewire-pulse stream properties.

## Install Locations

Configs can be installed per-user or system-wide. System-wide makes the 4.1 setup available to all users on the machine.

| File | Per-User Location | System-Wide Location |
|---|---|---|
| `configs/asoundrc` | `~/.asoundrc` | `/etc/asound.conf` |
| `configs/50-cm6206-channel-map.conf` | `~/.config/wireplumber/wireplumber.conf.d/` | `/etc/wireplumber/wireplumber.conf.d/` |
| `configs/upmix.conf` | `~/.config/pipewire/pipewire-pulse.conf.d/` | `/etc/pipewire/pipewire-pulse.conf.d/` |

### System-wide install (requires sudo)

```bash
sudo cp configs/asoundrc /etc/asound.conf
sudo mkdir -p /etc/wireplumber/wireplumber.conf.d
sudo cp configs/50-cm6206-channel-map.conf /etc/wireplumber/wireplumber.conf.d/
sudo mkdir -p /etc/pipewire/pipewire-pulse.conf.d
sudo cp configs/upmix.conf /etc/pipewire/pipewire-pulse.conf.d/
```

After system-wide install, the per-user copies in `~/.config/` and `~/.asoundrc` can be removed since the system-level configs take precedence when no user override exists.
