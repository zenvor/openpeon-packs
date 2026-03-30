---
name: audio-trimming
description: Trims silence, detects dead air, and processes audio files using ffmpeg. Use when trimming trailing or leading silence, cutting audio files, checking audio quality metrics, batch processing WAV/MP3/OGG files, or when user mentions ffmpeg trim, audio has blank/empty tail, sound file too long, or remove silence from audio.
compatibility: Requires ffmpeg and ffprobe installed
metadata:
  author: zenvor
  version: "1.0"
---

# Audio Trimming with ffmpeg

## Detect trailing silence

```bash
for f in sounds/*.wav; do
  dur=$(ffprobe -v error -show_entries format=duration -of csv=p=0 "$f")
  silence=$(ffmpeg -i "$f" -af "silencedetect=noise=-40dB:d=0.5" -f null - 2>&1 \
    | grep "silence_start" | tail -1 | sed 's/.*silence_start: //' | cut -d' ' -f1)
  if [ -n "$silence" ]; then
    dead_air=$(python3 -c "print(round(($dur - $silence) * 1000))")
    echo "$f: ${dead_air}ms trailing silence"
  fi
done
```

## Trim trailing silence

Trim a file to a specific duration:

```bash
ffmpeg -y -i input.wav -t <new_duration_seconds> output.wav
```

To trim based on a known dead air amount (e.g., keep 1900ms of silence):

```python
trim_seconds = (reported_dead_air_ms - 1900) / 1000.0
new_duration = current_duration - trim_seconds
```

## Batch trim with Python

```python
import subprocess, os

files = {
    "file.wav": 2088,  # reported dead air in ms
}

for f, dead_air_ms in files.items():
    dur = float(subprocess.check_output(
        ["ffprobe", "-v", "error", "-show_entries", "format=duration",
         "-of", "csv=p=0", f]).strip())
    trim_ms = dead_air_ms - 1900
    new_dur = round(dur - trim_ms / 1000.0, 3)
    tmp = f + ".tmp.wav"
    subprocess.run(["ffmpeg", "-y", "-i", f, "-t", str(new_dur), tmp],
                   capture_output=True)
    os.replace(tmp, f)
```

## Detect leading silence

```bash
for f in sounds/*.wav; do
  leading=$(ffmpeg -i "$f" -af "silencedetect=noise=-40dB:d=0.1" -f null - 2>&1 \
    | grep "silence_end" | head -1 | sed 's/.*silence_end: //' | cut -d' ' -f1)
  if [ -n "$leading" ]; then
    echo "$f: ${leading}s leading silence"
  fi
done
```

To trim leading silence, use `-ss` to skip the silent portion:

```bash
ffmpeg -y -i input.wav -ss <leading_silence_seconds> output.wav
```

## Gotchas

- **Do not use `-c copy` for precise WAV trimming.** `-c copy` operates at packet boundaries and cannot cut at arbitrary timestamps. The file size may not change at all. Always omit `-c copy` to let ffmpeg re-encode for precise cuts.
- **Only trim what's necessary.** Do not aggressively remove all silence. A natural tail sounds better than an abrupt cutoff. How much to keep depends on context -- for peon-ping CI the block threshold is 2000ms, so keeping ~1900ms is a safe target there.
- **Detect before trim.** Always run silence detection first to know the exact silence boundaries, then calculate how much to trim. Do not guess.
