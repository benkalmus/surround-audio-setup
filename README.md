# Audio Setup: CM6206 Surround 4.0

## Hardware

- **Sound card**: CM6206 (USB ID `0d8c:0102`, sold as ICUSBAUDIO7D)
- **Front speakers**: Edifier powered pair on green jack (Front L/R)
- **Rear speakers**: Edifier powered pair on black **Surround** jack (Rear L/R)
- **Unused orange jack**: C/SUB (no subwoofer connected)
- **Unused grey jack**: Back Out, only active in 7.1 / 8-channel mode
- **OS**: Kubuntu 26.04 LTS, PipeWire 1.6.2, WirePlumber 0.5.13

## Architecture

Uses PipeWire's `analog-surround-51` ACP profile with a unified loopback sink that drops both FC and LFE channels.

The loopback sink (`unified-upmix`) is the default audio destination:
- Stereo sources are upmixed to 4 channels using PipeWire's PSD algorithm (`channelmix.upmix-method = psd`)
- The playback side maps to `[FL FR RL RR]`, omitting both FC and LFE
- The orange C/SUB jack carries no signal (both FC and LFE silenced)
- ALSA `surround51` routes RL/RR to the CM6206's black **Surround** jack

## Channel Mapping

| PW Channel | Physical Jack | Connected To | Notes |
|---|---|---|---|
| FL (0) | Green | Front left Edifier | |
| FR (1) | Green | Front right Edifier | |
| FC (2) | Orange | *Nothing* | **Silenced** by playback.position array |
| LFE (3) | Orange | *Nothing* | **Silenced** by playback.position array |
| RL (4) | Black **Surround** | Rear left Edifier | |
| RR (5) | Black **Surround** | Rear right Edifier | |

## Unified Upmix Sink

Config: [configs/unified-upmix-sink.conf](configs/unified-upmix-sink.conf)

All audio (local playback, Bluetooth) routes through this sink.

Key settings:
- `capture.props`: 5 channels `[FL FR FC RL RR]` (LFE omitted from capture). PSD upmix applied to any connected stereo stream.
- `playback.props`: 4 channels `[FL FR RL RR]`. FC and LFE are dropped, so the orange jack carries no signal.
- `target.object` points to `alsa_output.usb-0d8c_USB_Sound_Device-00.analog-surround-51`

### Upmix Config Scopes

| File | Location | Scope |
|------|----------|-------|
| `unified-upmix-sink.conf` | `pipewire.conf.d/` | Creates the loopback sink with upmix on capture side |
| `pipewire-pulse-upmix.conf` | `pipewire-pulse.conf.d/` | Default stream properties for PulseAudio clients |
| `client-upmix.conf` | `client.conf.d/` | Default stream properties for native PipeWire clients |

The `channelmix.*` properties only activate when PipeWire converts between channel counts (e.g. stereo source to 4ch sink).

## Vinyl Passthrough

UDreamer turntable (3.5mm headphone out) to CM6206 Line-In to 4.0 speakers.

Script: [bin/vinyl-toggle](bin/vinyl-toggle)

Toggle on/off to avoid ADC hiss when turntable is idle. When ON:
1. Switches CM6206 input port from mic to line-in
2. Loads `module-loopback` (source to unified-upmix)
3. Stereo signal gets PSD-upmixed to 4.0

When OFF: unloads loopback, switches port back to mic.

### Sound quality tips

- CM6206 input at 100% will clip. Start at 50-70%: `pactl set-source-volume alsa_input.usb-0d8c_USB_Sound_Device-00.analog-stereo 60%`
- Set turntable headphone volume to ~70% for best signal-to-noise ratio
- CM6206 has a budget ADC. Some noise floor is expected.

### Usage

```bash
vinyl-toggle          # toggle
vinyl-toggle on       # force on
vinyl-toggle off      # force off
vinyl-toggle status   # print ON or OFF
```

### Install

```bash
mkdir -p ~/bin
ln -snf ~/repos/audio-setup-cm6206/bin/vinyl-toggle ~/bin/vinyl-toggle
```

Ensure `~/bin` is in `$PATH` (it is by default on Kubuntu).

## Install Steps

### 1. Set PipeWire profile to analog-surround-51

```bash
pactl set-card-profile alsa_card.usb-0d8c_USB_Sound_Device-00 analog-surround-51
```

Or use KDE Plasma audio control panel and select 5.1 output.

### 2. Install unified-upmix sink

```bash
mkdir -p ~/.config/pipewire/pipewire.conf.d
ln -snf ~/repos/audio-setup-cm6206/configs/unified-upmix-sink.conf \
    ~/.config/pipewire/pipewire.conf.d/unified-upmix-sink.conf
```

### 3. Install upmix configs

```bash
mkdir -p ~/.config/pipewire/client.conf.d
mkdir -p ~/.config/pipewire/pipewire-pulse.conf.d

ln -snf ~/repos/audio-setup-cm6206/configs/client-upmix.conf \
    ~/.config/pipewire/client.conf.d/upmix.conf

ln -snf ~/repos/audio-setup-cm6206/configs/pipewire-pulse-upmix.conf \
    ~/.config/pipewire/pipewire-pulse.conf.d/upmix.conf
```

### 4. Install Bluetooth routing rule

```bash
mkdir -p ~/.config/wireplumber/wireplumber.conf.d
ln -snf ~/repos/audio-setup-cm6206/configs/60-bluetooth-route.conf \
    ~/.config/wireplumber/wireplumber.conf.d/60-bluetooth-route.conf
```

Optionally enable Bluetooth permanently:

```bash
sudo cp bluetooth/main.conf /etc/bluetooth/main.conf

bluetoothctl discoverable on
bluetoothctl agent on
bluetoothctl default-agent
```

KDE Connect toggle commands:

| Name | Command |
|------|---------|
| Bluetooth ON | rfkill unblock bluetooth |
| Bluetooth OFF | rfkill block bluetooth |

### 5. Install systemd override (prevent startup race)

USB audio devices may not be ready when PipeWire starts at boot, causing sink creation to fail with "Device or resource busy". Add a dependency on `sound.target`:

```bash
mkdir -p ~/.config/systemd/user/pipewire.service.d
ln -snf ~/repos/audio-setup-cm6206/configs/10-wait-for-sound.conf \
    ~/.config/systemd/user/pipewire.service.d/10-wait-for-sound.conf

systemctl --user daemon-reload
```

### 6. Restart services

```bash
systemctl --user restart pipewire pipewire-pulse wireplumber
```

### 7. Set default sink

```bash
pactl set-default-sink unified-upmix
```

### 8. Set volumes

```bash
amixer -c 1 sset 'Speaker' 80%
wpctl set-volume unified-upmix 0.75
```

### 9. Verify

```bash
pactl list short sinks

paplay /usr/share/sounds/freedesktop/stereo/bell.oga

speaker-test -c 6 -D pipewire -t wav -l 1

pactl info | grep "Default Sink"
```

### 10. Commit changes

After verifying everything works, commit the repo state:

```bash
cd ~/repos/audio-setup-cm6206
git add .
git commit -m "Add systemd override for USB audio startup race"
```

## Configuration Files

| File | Purpose | Symlink Target |
|------|---------|----------------|
| `configs/unified-upmix-sink.conf` | Loopback sink (stereo in to 4ch out) | `~/.config/pipewire/pipewire.conf.d/` |
| `configs/client-upmix.conf` | Stream upmix for native PipeWire clients | `~/.config/pipewire/client.conf.d/upmix.conf` |
| `configs/pipewire-pulse-upmix.conf` | Stream upmix for PulseAudio clients | `~/.config/pipewire/pipewire-pulse.conf.d/upmix.conf` |
| `configs/60-bluetooth-route.conf` | Route Bluetooth audio to unified-upmix | `~/.config/wireplumber/wireplumber.conf.d/` |
| `configs/10-wait-for-sound.conf` | PipeWire waits for sound.target before starting | `~/.config/systemd/user/pipewire.service.d/` |
| `configs/asoundrc` | Custom ALSA device for direct speaker test | `~/.asoundrc` |
| `bin/vinyl-toggle` | Toggle vinyl passthrough (line-in to 4.0) | `~/bin/vinyl-toggle` |

All paths use absolute links (`ln -snf`) so edits in `~/repos/audio-setup-cm6206/configs/` apply without touching `~/.config/`.

## Symlink Summary

Complete list of all symlinks for this setup:

```bash
# PipeWire configs
mkdir -p ~/.config/pipewire/pipewire.conf.d
mkdir -p ~/.config/pipewire/client.conf.d
mkdir -p ~/.config/pipewire/pipewire-pulse.conf.d

ln -snf ~/repos/audio-setup-cm6206/configs/unified-upmix-sink.conf \
    ~/.config/pipewire/pipewire.conf.d/unified-upmix-sink.conf

ln -snf ~/repos/audio-setup-cm6206/configs/client-upmix.conf \
    ~/.config/pipewire/client.conf.d/upmix.conf

ln -snf ~/repos/audio-setup-cm6206/configs/pipewire-pulse-upmix.conf \
    ~/.config/pipewire/pipewire-pulse.conf.d/upmix.conf

# WirePlumber config
mkdir -p ~/.config/wireplumber/wireplumber.conf.d

ln -snf ~/repos/audio-setup-cm6206/configs/60-bluetooth-route.conf \
    ~/.config/wireplumber/wireplumber.conf.d/60-bluetooth-route.conf

# Systemd override (prevents USB audio startup race)
mkdir -p ~/.config/systemd/user/pipewire.service.d

ln -snf ~/repos/audio-setup-cm6206/configs/10-wait-for-sound.conf \
    ~/.config/systemd/user/pipewire.service.d/10-wait-for-sound.conf

systemctl --user daemon-reload

# ALSA config
ln -snf ~/repos/audio-setup-cm6206/configs/asoundrc ~/.asoundrc

# Vinyl toggle script
mkdir -p ~/bin
ln -snf ~/repos/audio-setup-cm6206/bin/vinyl-toggle ~/bin/vinyl-toggle
```

## Known Quirks

- **ICUSBAUDIO7D jack naming vs ALSA**: The black jack is labelled **Surround** and carries the rear channels. The grey jack is labelled **Back Out** and is only active in 7.1 / 8-channel mode.
- **No subwoofer**: Both FC and LFE are silenced. The orange jack carries no signal.
- **PSD rear channel requirement**: PipeWire's Passive Surround Decoding only sends audio to rear channels when LEFT and RIGHT differ significantly. Identical mono content produces no rear output. This is correct behaviour.
- **USB audio startup race**: Without the systemd override, PipeWire may start before the USB device is ready, causing "Device or resource busy" errors and missing sinks. The `10-wait-for-sound.conf` override fixes this by adding `After=sound.target` to pipewire.service.
