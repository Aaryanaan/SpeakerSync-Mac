# bt-audio-sync — Setup & Usage Guide

Free, local alternative to Airfoil. Routes system audio to two Bluetooth speakers with per-speaker delay tuning so they stay in sync.

---

## Contents

- [Prerequisites](#prerequisites)
- [macOS Setup](#macos-setup)
- [Windows Setup](#windows-setup)
- [Linux Setup](#linux-setup)
- [CLI Commands](#cli-commands)
- [Launch Flags](#launch-flags)
- [Delay Tuning](#delay-tuning)
- [Stereo & DSP](#stereo--dsp)
- [Troubleshooting](#troubleshooting)

---

## Prerequisites

- Python 3.8+ (`python3 --version`)
- Two Bluetooth speakers that can connect to your computer at the same time
- ~5 minutes

---

## macOS Setup

### 1. Install dependencies

```bash
pip install sounddevice numpy
```

### 2. Install BlackHole (virtual audio loopback)

```bash
brew install blackhole-2ch
```

Or download the installer from https://existential.audio/blackhole/.

### 3. Create a Multi-Output Device

So you can still hear audio on your laptop while the app reads from BlackHole.

1. Open **Audio MIDI Setup** (Spotlight → "Audio MIDI Setup")
2. Click **+** at the bottom-left → **Create Multi-Output Device**
3. Check both:
   - **BlackHole 2ch**
   - **MacBook Pro Speakers** (optional — for local monitoring)
4. **System Settings → Sound → Output** → select the **Multi-Output Device**

### 4. Pair both speakers

**System Settings → Bluetooth** → pair each speaker. Both must show "Connected" simultaneously.

> Do NOT use JBL Connect+ / Auracast. We're bypassing that to control timing per speaker.

### 5. Find device indices

```bash
python3 -c "import sounddevice; print(sounddevice.query_devices())"
```

Note three numbers: BlackHole (input), Speaker A, Speaker B.

### 6. Launch

```bash
python3 bt_audio_sync.py --input <BlackHole_idx> --output-a <SpeakerA_idx> --output-b <SpeakerB_idx>
```

Play music from any app. See [Delay Tuning](#delay-tuning) to sync them.

### macOS-specific issues

| Symptom | Fix |
|---------|-----|
| No audio | Sound output is set to speakers directly. Switch back to Multi-Output Device. |
| Both speakers won't pair simultaneously | Built-in adapter limitation. Use a USB Bluetooth 5.0+ dongle. |
| `OSError: PortAudio library not found` | `brew install portaudio` |
| Clicks/pops | You used BlackHole 16ch or 64ch. Reinstall `blackhole-2ch`. |
| App lists no BlackHole device | Restart your Mac after install (kernel extension needs to load). |

---

## Windows Setup

### 1. Install dependencies

```powershell
pip install sounddevice numpy
```

### 2. Install VB-Cable (virtual audio loopback)

1. Download from https://vb-audio.com/Cable/
2. Run installer **as Administrator**
3. **Reboot** (required)

### 3. Route system audio through VB-Cable

1. Right-click speaker icon in taskbar → **Sound Settings**
2. Set **Output** to **CABLE Input (VB-Audio Virtual Cable)**

### 4. (Optional) Hear audio locally too

Right-click **CABLE Output** → Properties → **Listen** tab → check "Listen to this device" → pick your headphones.

### 5. Pair both speakers

**Settings → Bluetooth & devices → Add device** for each. Both must show "Connected" at the same time.

### 6. Find device indices

```powershell
python -c "import sounddevice; print(sounddevice.query_devices())"
```

Look for **CABLE Output** (input), Speaker A, Speaker B.

### 7. Launch

```powershell
python bt_audio_sync.py --input <CABLE_idx> --output-a <SpeakerA_idx> --output-b <SpeakerB_idx>
```

### Windows-specific issues

| Symptom | Fix |
|---------|-----|
| `CABLE Output` not listed | Reboot. VB-Cable installs a driver that needs a restart. |
| No audio | Check **Sound Settings → Output** is **CABLE Input** (not your speakers). |
| Speakers stutter | Run `python bt_audio_sync.py --blocksize 1024` (or 2048). |
| Both speakers won't pair simultaneously | Stock Windows BT often supports only one A2DP. Use a USB BT 5.0+ dongle. |
| `WASAPI` errors | Sample rate mismatch. Try `--sample-rate 44100`. |

---

## Linux Setup

(Tested on Ubuntu 22.04+ and Fedora 34+ with PipeWire.)

### 1. Install dependencies

```bash
# Debian / Ubuntu
sudo apt install libportaudio2 portaudio19-dev
pip install sounddevice numpy

# Fedora
sudo dnf install portaudio portaudio-devel
pip install sounddevice numpy
```

### 2. Create a virtual sink (loopback)

```bash
pactl load-module module-null-sink \
  sink_name=virtual_out \
  sink_properties=device.description="BT_Sync_Virtual"

pactl set-default-sink virtual_out
```

The app reads from **"Monitor of BT_Sync_Virtual"**.

### 3. (Optional) Make the sink persistent

Add to `~/.config/pipewire/pipewire.conf.d/virtual-sink.conf`:

```
context.modules = [
    {
        name = libpipewire-module-loopback
        args = {
            node.name = "bt_sync_virtual"
            node.description = "BT Sync Virtual Output"
            capture.props = { media.class = "Audio/Sink" }
            playback.props = { node.target = "" }
        }
    }
]
```

### 4. Pair both speakers

```bash
bluetoothctl
> scan on
> pair <MAC_SPEAKER_1>
> connect <MAC_SPEAKER_1>
> pair <MAC_SPEAKER_2>
> connect <MAC_SPEAKER_2>
```

### 5. Find device indices

```bash
python3 -c "import sounddevice; print(sounddevice.query_devices())"
```

Find "Monitor of BT_Sync_Virtual" (input) and the two speakers.

### 6. Launch

```bash
python3 bt_audio_sync.py --input <Monitor_idx> --output-a <SpeakerA_idx> --output-b <SpeakerB_idx>
```

### Linux-specific issues

| Symptom | Fix |
|---------|-----|
| Bluetooth speakers don't appear in `query_devices()` | PulseAudio managing BT instead of PipeWire. Check: `systemctl --user status pipewire`. |
| Virtual sink gone after reboot | You skipped step 3. Use the persistent config. |
| One speaker keeps dropping | Power-save kicking in: `sudo iwconfig <iface> power off` and disable BT autosuspend in `/etc/bluetooth/main.conf`. |
| `Cannot lock memory` warnings | Harmless; raise the rtprio limit if you want them gone. |
| Both speakers won't connect | Add a USB BT 5.0+ dongle; built-in chips often max at one A2DP link. |

---

## CLI Commands

Type these at the `bt-sync>` prompt while audio is playing. All take effect immediately.

### Delay & Volume

| Command | Effect |
|---------|--------|
| `da <ms>` | **Set** Speaker A delay (0–500 ms) |
| `db <ms>` | **Set** Speaker B delay (0–500 ms) |
| `va <0-100>` | Speaker A volume |
| `vb <0-100>` | Speaker B volume |
| `ma` / `mb` | Toggle mute on A / B |

> `da 50` *sets* the delay to 50ms — it does not add 50ms to the current value.

### Stereo & DSP

| Command | Effect |
|---------|--------|
| `stereo` | A=Left, B=Right with crossfeed (default) |
| `mono` | Both speakers play L+R sum |
| `full` | Both speakers play full unmodified stereo |
| `crossfeed <0-50>` | 0=hard split, 30=natural, 50=mono |
| `bass <-12..12>` | Low-shelf boost/cut at 150Hz |
| `width <0-2>` | 0=mono, 1=normal, 1.5=wide, 2=max |

### Presets

| Command | Stereo | Crossfeed | Bass | Width | Use for |
|---------|--------|-----------|------|-------|---------|
| `party` | stereo | 20% | +6 | 1.4 | Outdoor / room-fill |
| `chill` | stereo | 30% | +3 | 1.0 | Background music |
| `flat` | stereo | 30% | off | 1.0 | Accurate playback |

Presets do not change delay or volume.

### System

| Command | Effect |
|---------|--------|
| `status` | Buffer fill, underruns, drift corrections, current DSP |
| `devices` | Re-list audio devices |
| `help` | Quick reference |
| `quit` / `q` | Exit |

---

## Launch Flags

All optional. Without them you'll be prompted interactively.

| Flag | Default | Notes |
|------|---------|-------|
| `--input <n>` | — | Loopback input device index |
| `--output-a <n>` / `--output-b <n>` | — | Speaker indices |
| `--input-name <str>` | — | Match input by substring (e.g. `"BlackHole"`) |
| `--output-a-name <str>` / `--output-b-name <str>` | — | Match speaker by substring |
| `--sample-rate <Hz>` | 48000 | Try 44100 if you get errors |
| `--channels <n>` | 2 | Leave at 2 |
| `--blocksize <n>` | 512 | Raise to 1024/2048 if you get clicks |
| `--delay-a <ms>` / `--delay-b <ms>` | 0 | Starting delay |

Name-based example:

```bash
python3 bt_audio_sync.py \
  --input-name "BlackHole" \
  --output-a-name "Charge 4" \
  --output-b-name "Charge 6" \
  --delay-b 60
```

---

## Delay Tuning

Different speakers process Bluetooth audio at different speeds, so one is always slightly ahead. The delay line slows down the faster speaker.

1. Start music with a strong beat
2. Stand **equidistant** between the speakers
3. Listen for a "flam" — two distinct hits instead of one
4. If A is **early** → `da 10`, then `da 20`, etc. in 5ms steps
5. If B is **early** → `db 10`, `db 20`, etc.
6. Stop when the hits merge into one

Charge 6 usually has lower latency than Charge 4 → start with `--delay-b 60` and tune from there.

**Sine wave method:** Play a 1kHz tone, walk between the speakers. If it wobbles or sounds hollow in the overlap, they're out of phase — adjust delay until smooth.

> Only adjust **one** speaker. If A=0 and B=60, don't move both — only tweak the one that's ahead. Your ideal value depends on your adapter, distance, and room — it won't match anyone else's.

---

## Stereo & DSP

By default, **A = left channel + a little right (crossfeed); B = right + a little left**. Place A on the left, B on the right.

```
        You
        🧑
       / | \
      /  |  \
    🔊   |   🔊
   (A)       (B)
   LEFT      RIGHT
```

Aim for an equilateral triangle. Angle speakers slightly inward.

- **Speakers close together** → `crossfeed 40`
- **Speakers far apart** → `crossfeed 15` for sharper separation
- **Width** > 1 exaggerates L/R contrast on studio recordings (ineffective on podcasts)
- **Bass** +3 to +6 is usually clean; +8 to +12 thumps outdoors but may distort

### Recipes

```
# Outdoor party
party
da 0
db 60
va 100
vb 100

# Desk listening
chill
da 0
db 40
va 70
vb 70

# Podcasts
mono
bass 0
width 1.0
```

---

## Troubleshooting

### No audio at all

1. Run `status`. Are **Input callbacks** incrementing?
   - **No** → OS isn't routing to the loopback. Check OS sound output (Multi-Output on Mac, CABLE Input on Windows, virtual_out on Linux).
   - **Yes, but Fill = 0 on both speakers** → output device indices are wrong. Re-run with `devices` and check.
2. OS volume isn't muted? Speaker volume isn't 0?
3. Speakers still connected? Bluetooth menu → confirm both show "Connected".

### Clicks, pops, or stutters

- Increase blocksize: `--blocksize 1024` or `--blocksize 2048`
- Close CPU-heavy apps (browsers, video editors)
- macOS: confirm BlackHole **2ch** (not 16ch / 64ch)
- Try `--sample-rate 44100`

### One speaker much louder than the other

- Balance with `va` / `vb` (e.g. `va 85` / `vb 100`)
- Check the physical buttons on each speaker

### Drift after 30+ minutes

- The drift compensator handles it. Run `status` — a few "Drift corrections" per hour is normal.
- Rapid corrections (>10/min) → try `--sample-rate 44100`

### `PortAudio library not found`

```bash
# macOS
brew install portaudio
# Linux
sudo apt install libportaudio2 portaudio19-dev
```

### Speakers won't connect simultaneously

Most stock Bluetooth chipsets only support a single A2DP link. Use a **USB Bluetooth 5.0+** dongle.

### Bass sounds distorted

You're clipping. Lower volume (`va 70` / `vb 70`) or bass (`bass 4`).

### Click when changing delay

Normal — repositioning the ring buffer pointer causes one sample of discontinuity. Single click, not ongoing.

### Can't find my speakers in `query_devices()`

- macOS: re-pair, then re-run. Sometimes the device only registers after audio has been routed to it once.
- Windows: connect via Settings, then play any sound to "wake" the A2DP profile.
- Linux: check `pactl list sinks short` shows them. If not, PipeWire isn't managing BT.

### How do I stop?

`quit`, `q`, or `Ctrl+C`.
