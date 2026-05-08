# Using bt-audio-sync with Rekordbox

Yes, it works — but the obvious setup (Multi-Output Device containing both Bluetooth speakers) routes around the syncer entirely. If your CLI commands (`va`, `db`, etc.) don't seem to do anything, that's why.

## Why CLI commands appear to do nothing

If your Multi-Output Device contains the **two Bluetooth speakers directly**, audio flows like this:

```
Rekordbox  →  Multi-Output  →  BT Speaker A
                            →  BT Speaker B
```

Core Audio is doing the fan-out. The Python script never sees a single sample, so its volume / delay / DSP controls apply to streams that aren't playing anything.

## Correct routing

```
Rekordbox  →  Multi-Output (BlackHole 2ch + MacBook Speakers for monitoring)
                       ↓
                   BlackHole 2ch
                       ↓
                 bt_audio_sync.py
                    ↓        ↓
                Speaker A   Speaker B
```

The Bluetooth speakers must be controlled **only** by the syncer. The Multi-Output Device is just so you can monitor locally on your laptop.

## Setup steps

1. **Audio MIDI Setup** — your Multi-Output Device should contain:
   - **BlackHole 2ch** ✅
   - **MacBook Pro Speakers** (optional, for local monitoring) ✅
   - **Not** the Bluetooth speakers ❌

2. **Rekordbox → Preferences → Audio:**
   - **Audio** → select the Multi-Output Device (or BlackHole 2ch directly if you don't need monitoring)
   - **Output channels → Master Output L/R** → channels 1/2 of that device

3. Launch the syncer:

   ```bash
   python3 bt_audio_sync.py --input-name "BlackHole" \
     --output-a-name "<Speaker A>" --output-b-name "<Speaker B>"
   ```

## Rekordbox-specific gotchas

### Headphone cue

If you cue tracks through "Headphones Output", that needs a **different physical device** (built-in headphone jack, USB audio interface). BlackHole 2ch only carries 2 channels, so it can't carry master + cue.

If you want both through one virtual device: install **BlackHole 16ch** and route master to ch 1/2, cue to ch 3/4. The syncer reads ch 1/2 only, your headphones tap ch 3/4 separately.

### Sample rate mismatch

Rekordbox defaults to **44100 Hz**. The syncer defaults to **48000 Hz**. Mismatch causes resampling artifacts.

Either:
- Change Rekordbox to 48000 (Preferences → Audio → Sample Rate), or
- Launch the syncer with `--sample-rate 44100`

### Latency

Rekordbox + BlackHole + Bluetooth adds noticeable latency between the deck and what you hear. Fine for listening, **painful for live mixing on the master out**.

Cue through **wired** headphones — never through the synced BT speakers.

### Buffer size

If you get clicks during live mixing, raise both:
- Rekordbox: Preferences → Audio → Buffer Size → 512 or 1024
- Syncer: `--blocksize 1024` (or 2048)
