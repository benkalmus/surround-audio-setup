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

### 2. Install WirePlumber channel map rule

Pro-audio mode labels all channels as generic `AUX0`-`AUX7`. The WirePlumber rule remaps them to their real positions so PipeWire understands the layout.

```bash
mkdir -p ~/.config/wireplumber/wireplumber.conf.d
cp configs/50-cm6206-channel-map.conf ~/.config/wireplumber/wireplumber.conf.d/
systemctl --user restart wireplumber
```

Config file: [configs/50-cm6206-channel-map.conf](configs/50-cm6206-channel-map.conf)

### 3. Set as default sink

```bash
# Find the sink ID
wpctl status

# Set as default (ID may change, verify with wpctl status)
wpctl set-default <sink-id>
pactl set-default-sink alsa_output.usb-0d8c_USB_Sound_Device-00.pro-output-0
```

### 4. Set ALSA mixer volumes

```bash
# All channels to 100% (hardware max)
amixer -c 1 sset 'Speaker' 100%

# PipeWire sink volume (perceived, 80% recommended)
wpctl set-volume <sink-id> 80%
```

### 5. Verify channel mapping

```bash
# Direct ALSA test (8 channels, all jacks)
speaker-test -c 8 -D hw:1 -t wav -l 1

# PipeWire stereo test (front speakers only until upmix configured)
paplay --device=alsa_output.usb-0d8c_USB_Sound_Device-00.pro-output-0 /usr/share/sounds/freedesktop/stereo/bell.oga
```

## Current Status

- [x] Hardware connected (green/black/orange jacks)
- [x] Pro-audio profile activated
- [x] WirePlumber channel map rule installed
- [x] Default sink configured
- [x] Channel mapping verified (8-channel speaker-test)
- [x] Stereo playback confirmed on front speakers
- [ ] Upmixing stereo to 4.1 (next step)
- [ ] Per-channel volume tuning
- [ ] EasyEffects software crossover (optional, future)
- [ ] Snapcast integration (future)

## Known Quirks

- **CM6206 requires 8-channel mode**: 2/4/6 channel ALSA altsets do not route audio to all physical jacks. Only altset 1 (8ch) works.
- **Pro-audio profile tradeoff**: Loses ACP profile switching and per-port volume control. Gains raw multi-channel access.
- **No center speaker**: Channel 2 (Front Center) is unused. Stereo upmix must skip center and route to quad + sub only.

## Files

| File | Purpose | Install Location |
|---|---|---|
| `configs/50-cm6206-channel-map.conf` | WirePlumber ALSA monitor rule for channel positions | `~/.config/wireplumber/wireplumber.conf.d/` |
