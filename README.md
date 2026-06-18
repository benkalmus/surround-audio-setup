# Audio Setup: CM6206 Surround 5.1 (with missing centre speaker)

## Hardware

- **Sound card**: CM6206 (USB ID `0d8c:0102`, sold as ICUSBAUDIO7D)
- **Front speakers**: Edifier powered pair on green jack (Front L/R)
- **Rear speakers**: Edifier powered pair on black **Surround** jack (Rear L/R in ALSA `surround51`)
- **Unused grey jack**: Back Out — only active in 7.1 / 8-channel mode
- **Subwoofer**: B&W ASW608 on orange C/SUB jack (LFE)
- **Centre speaker**: *Not present*
- **OS**: Kubuntu 26.04 LTS, PipeWire 1.6.2, WirePlumber 0.5.13

## Architecture

Instead of using the `pro-audio` profile (which requires manual ALSA mappings and loses volume control), this setup uses PipeWire's standard `analog-surround-51` ACP profile.

A unified loopback sink (`unified-upmix`) is configured as the default audio destination:
- Any stereo source connected to `unified-upmix` is upmixed to 6-channel using PipeWire's PSD algorithm (`channelmix.upmix-method = psd`).
- The loopback's playback side is mapped to `[FL FR RL RR LFE]`, deliberately omitting the **FC** channel.
- Since FC is missing from the playback stream, the orange C/SUB jack carries only the LFE subwoofer signal.
- ALSA `surround51` routes the `RL`/`RR` channels to the CM6206's **Surround** jack (black). The grey **Back Out** jack is silent in 5.1 mode.
- The centre speaker effectively works as a **phantom centre** (equal energy on FL and FR creates a central phantom image). This is fully intentional.

## Channel Mapping

The CM6206 exposes a 6-channel ALSA device in `analog-surround-51` mode. PipeWire labels these correctly:

| PW Channel | Physical Jack | Connected To | Notes |
|---|---|---|---|
| FL (0) | Green | Front left Edifier | |
| FR (1) | Green | Front right Edifier | |
| FC (2) | Orange | *Centre speaker* | **Silenced** by playback.position array |
| LFE (3) | Orange | B&W ASW608 subwoofer | Crossover 80Hz on sub |
| RL (4) | Black **Surround** | Rear left Edifier | ALSA `surround51` maps RL/RR to the side/surround pins |
| RR (5) | Black **Surround** | Rear right Edifier | |
| — | Grey **Back Out** | *Nothing* | Only active in 7.1 / 8-channel mode |

## Unified Upmix Sink

Config: [configs/unified-upmix-sink.conf](configs/unified-upmix-sink.conf)

This single sink replaces the earlier `local-upmix` and `vban-upmix` loopback sinks. All audio (local playback, Bluetooth) routes through here.

Key settings:
- `capture.props`: 6 channels `[FL FR FC LFE RL RR]`. `channelmix.upmix = true` and `channelmix.upmix-method = psd` are set on the **sink** so any connected stereo stream gets upmixed automatically.
- `playback.props`: 5 channels `[FL FR RL RR LFE]`. FC is dropped, so the orange jack's centre channel sees zero signal.
- `target.object` points directly to the hardware node `alsa_output.usb-0d8c_USB_Sound_Device-00.analog-surround-51`.

### Upmix Config Scopes

Three config files set `channelmix.*` properties, but they apply at different scopes and only activate when PipeWire needs to convert between different channel counts:

| File | Location | Scope | When it applies |
|------|----------|-------|-----------------|
| `unified-upmix-sink.conf` | `pipewire.conf.d/` | Creates the loopback **sink node** with upmix props on its capture side | Any stream connected to `unified-upmix` sink |
| `pipewire-pulse-upmix.conf` | `pipewire-pulse.conf.d/` | Default `stream.properties` for **all PulseAudio clients** | Local apps using PulseAudio API (Firefox, Spotify, etc.) |
| `client-upmix.conf` | `client.conf.d/` | Default `stream.properties` for **all PipeWire clients** | Local apps using native PipeWire API |

**Critical limitation**: The `channelmix.*` properties only activate when PipeWire needs to **convert between different channel counts** (e.g., stereo source → 6ch sink). They're part of the channel mixer, not a standalone DSP effect. If a stream arrives as 6ch matching a 6ch sink, no channel conversion occurs and these properties are bypassed entirely.

**Example**: VBAN from Windows sends 6ch audio → unified-upmix receives 6ch matching its 6ch capture format → no upmix processing → `lfe-cutoff` never applies. The LFE channel passes through with whatever the source already put in it.

## Vinyl Passthrough

UDreamer turntable (3.5mm headphone out) → CM6206 Line-In → unified-upmix → 5.1 speakers.

Script: [bin/vinyl-toggle](bin/vinyl-toggle)

Toggle on/off manually to avoid ADC hiss when turntable is idle. When ON:
1. Switches CM6206 input port from mic to line-in
2. Loads `module-loopback` (source → unified-upmix) which creates a proper PulseAudio stream
3. Stereo signal gets PSD-upmixed to 4.1 (fronts + rears + sub)

When OFF: unloads the loopback module and switches port back to mic (silent).

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
pactl set-card-profile alsa_card.usb-0d8c_USB_Sound_Device-00   analog-surround-51
```

### 2. Install unified-upmix sink (symlink, no sudo needed)

```bash
mkdir -p ~/.config/pipewire/pipewire.conf.d
ln -snf ~/repos/audio-setup-cm6206/configs/unified-upmix-sink.conf \
    ~/.config/pipewire/pipewire.conf.d/unified-upmix-sink.conf
```

### 3. Install global fallback upmix configs

These set `channelmix.upmix` for streams that don't connect through the sink directly.

```bash
mkdir -p ~/.config/pipewire/client.conf.d
mkdir -p ~/.config/pipewire/pipewire-pulse.conf.d

ln -snf ~/repos/audio-setup-cm6206/configs/client-upmix.conf \
    ~/.config/pipewire/client.conf.d/upmix.conf

ln -snf ~/repos/audio-setup-cm6206/configs/pipewire-pulse-upmix.conf \
    ~/.config/pipewire/pipewire-pulse.conf.d/upmix.conf
```

### 4. Install Bluetooth routing rule

This routes audio from any paired Bluetooth device directly into `unified-upmix`, so phone music also gets upmixed to surround.

```bash
mkdir -p ~/.config/wireplumber/wireplumber.conf.d
ln -snf ~/repos/audio-setup-cm6206/configs/60-bluetooth-route.conf \
    ~/.config/wireplumber/wireplumber.conf.d/60-bluetooth-route.conf
```

### 5. Restart services

```bash
systemctl --user restart pipewire pipewire-pulse wireplumber
```

### 6. Set unified-upmix as default sink

```bash
pactl set-default-sink unified-upmix
```

### 7. Set volumes

```bash
# CM6206 hardware volume (all channels)
amixer -c 1 sset 'Speaker' 80%

# PipeWire software sink volume
wpctl set-volume unified-upmix 0.75
```

### 8. Verify

```bash
# List sinks: should see unified-upmix and the 6ch CM6206 hardware
pactl list short sinks

# PipeWire stereo test (front + upmixed rear)
paplay /usr/share/sounds/freedesktop/stereo/bell.oga

# ALSA 5.1 speaker test (FC point will be silent)
# RL/RR tones come out the BLACK Surround jack, not the grey Back Out jack.
speaker-test -c 6 -D pipewire -t wav -l 1

# Direct ALSA test bypassing PipeWire (useful to confirm jack mapping)
speaker-test -c 6 -D surround51:CARD=ICUSBAUDIO7D,DEV=0 -t wav -l 1

# Check that unified-upmix is default
pactl info | grep "Default Sink"
```

## Configuration Files

| File | Purpose | Symlink Target |
|---|---|---|
| `configs/unified-upmix-sink.conf` | Loopback sink (stereo in → 5ch out, phantom centre) | `~/.config/pipewire/pipewire.conf.d/` |
| `configs/client-upmix.conf` | Global stream fallback upmix (client apps) | `~/.config/pipewire/client.conf.d/upmix.conf` |
| `configs/pipewire-pulse-upmix.conf` | Global stream fallback upmix (Pulse apps) | `~/.config/pipewire/pipewire-pulse.conf.d/upmix.conf` |
| `configs/60-bluetooth-route.conf` | Route Bluetooth audio → unified-upmix | `~/.config/wireplumber/wireplumber.conf.d/` |
| `configs/asoundrc` | Custom ALSA device for direct speaker test | `~/.asoundrc` |
| `bin/vinyl-toggle` | Toggle vinyl passthrough (line-in → upmix) | `~/bin/vinyl-toggle` |

All paths in the repo use absolute links (`ln -snf`) so you can change files in `~/repos/audio-setup-cm6206/configs/` without touching `~/.config/`.

## Current Status

- [x] Hardware connected (green Front, black Surround, orange C/SUB; grey Back Out unused in 5.1)
- [x] `analog-surround-51` profile active (6ch)
- [x] Unified loopback sink installed and default
- [x] Centre channel intentionally silenced (phantom centre)
- [x] LFE/subwoofer active
- [x] Global fallback upmix configs (client + pulse)
- [x] Bluetooth routing rule installed
- [x] Stereo upmix to 4.1 (quad + sub) via PSD confirmed working
- [x] ALSA `surround41_cm6206` device functional (optional direct testing)
- [ ] Network audio receiver (to replace VBAN)
- [ ] EasyEffects software crossover (optional, future)
- [ ] Per-channel volume tuning
- [x] Vinyl passthrough (turntable line-in → 5.1 upmix, toggleable)

## Removed Configs (Old Experiment)

The following files were deleted from the repo and active config directories as part of cleanup:

- `vban-recv.conf` — VBAN receiver.
- `vban-upmix-sink.conf` — Second loopback sink for VBAN.
- `51-vban-routing.conf` — WirePlumber VBAN routing rule.
- `50-cm6206-channel-map.conf` — Empty rule that did nothing useful.
- `50-cm6206-channel-map.conf` (system-wide `/etc/wireplumber/...`) — **Still exists** and targets the stale `pro-output-0` 8ch profile. Remove with sudo when convenient.

## Known Quirks

- **ICUSBAUDIO7D jack naming vs ALSA**: The black jack is labelled **Surround** and carries the side/surround channels in 5.1 mode. The grey jack is labelled **Back Out** and is only active in 7.1 / 8-channel mode. ALSA's `surround51` device sends its `RL`/`RR` channels to the black Surround jack, so plug rear speakers there. See the [StarTech manual](https://sgcdn.startech.com/005329/media/sets/icusbaudio7d_manual/icusbaudio7d__usermanual.pdf) and the [CM6206LX datasheet](http://product.ic114.com/PDF/C/CM6206-LX.PDF) (pins `SSOL`/`SSOR` are side-surround for Vista 5.1).
- **Phantom centre**: The centre speaker position is silent. Stereo vocals are phantom-imaged between FL and FR. This is standard surround practice and sounds correct for watching movies or listening to music.
- **LFE upmix**: `channelmix.lfe-cutoff = 150` and `mix-lfe = true` are required for stereo LF content to reach the sub. The B&W sub's hardware crossover at 80Hz filters the extra high-frequency content anyway.
- **Orange jack**: On the CM6206, the orange jack carries both FC and LFE. By dropping FC from playback.position, only LFE remains. This is why the sub works and the nonexistent centre stays silent.
- **PSD rear channel requirement**: PipeWire's Passive Surround Decoding only sends audio to rear channels when LEFT and RIGHT differ significantly. Identical mono content produces no rear output. This is correct behaviour.
- **Pro-audio profile abandoned**: The 8ch `pro-audio` profile worked but forced manual ALSA mappings and lost volume control. `analog-surround-51` is simpler, gives proper channel labels, and PipeWire volume control works out of the box.
