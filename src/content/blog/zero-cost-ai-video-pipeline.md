---
title: "The $0/month AI Video Pipeline"
description: "Voice cloning, AI image generation, and automated video assembly on a homelab GPU. 45 seconds per video, no external API costs."
pubDate: 2026-03-18
tags: ["youtube", "AI", "automation", "python", "GPU"]
readTime: 10
---

The pitch sounds implausible: generate YouTube-quality videos in 45 seconds, on local hardware, at zero marginal cost per video.

It works. Here's the architecture.

## The problem with cloud-based video tools

Most AI video tools charge per minute of output. ElevenLabs charges for TTS. Midjourney charges for images. RunwayML charges for video clips. Stack them together and you're paying $0.50-2.00 per video — which seems reasonable until you want to post daily.

A YouTube channel with daily uploads is 365 videos per year. At $1/video: $365/year minimum, before thumbnails, editing, or any quality upgrades.

The alternative: run everything locally on hardware you already own.

## Hardware requirements

I'm using an RTX 3080 (10GB VRAM). The realistic minimums for this pipeline:

| Component | Minimum | Why |
|-----------|---------|-----|
| GPU VRAM | 8GB | SDXL Turbo + Chatterbox can share if not running simultaneously |
| RAM | 16GB | Models stay loaded between generations |
| Storage | 100GB free | Models are large; cache grows |
| CPU | Anything modern | Assembly, subtitle generation are CPU tasks |

The pipeline runs in sequence (not parallel), so you don't need to fit everything in VRAM at once.

## The pipeline components

### 1. Script generation (Claude via `claude -p`)

Every video starts with a structured script prompt. The output is JSON:

```json
{
  "title": "Your homelab woke me up at 3am. Here's what happened.",
  "hook": "It was 2:47 AM when the alert fired...",
  "sections": [
    {
      "text": "...",
      "visual_prompt": "dark server room, single blinking red light, atmospheric",
      "is_terminal": false
    }
  ],
  "call_to_action": "...",
  "description": "...",
  "tags": ["homelab", "self-hosting", "AI"],
  "thumbnail_prompt": "..."
}
```

Key design decision: the script includes `visual_prompt` for each section. The script writer and the image generator are the same prompt chain. This matters — the images actually match the narration.

### 2. Voice cloning (Chatterbox TTS)

[Chatterbox](https://github.com/resemble-ai/chatterbox) from Resemble AI is the best open-source TTS I've tested. It beats ElevenLabs in blind tests on naturalness, and it runs on 3.2GB VRAM.

Voice cloning setup:
1. Record 15 seconds of clean audio (no background noise, natural pacing)
2. Clean it: `ffmpeg -i input.wav -af 'highpass=f=100,lowpass=f=7000,afftdn=nf=-20' -ar 24000 reference.wav`
3. Set `audio_prompt_path` to the cleaned sample

The settings that create the "angry butler" character voice:
- `exaggeration`: 0.8 (more expressive than default)
- `cfg_weight`: 0.3 (low = allows drift from reference toward learned affect)
- `temperature`: 0.9 (slight randomness keeps it from sounding robotic)

Generation time: ~10 seconds for a 60-second script on RTX 3080.

### 3. Image generation (SDXL Turbo)

SDXL Turbo is the speed/quality tradeoff that works for this pipeline. Full SDXL gives better images but takes 3-4 minutes. Turbo takes 3-5 seconds.

For a 60-second video with 4-6 sections, that's 15-30 seconds of image generation total.

The quality is sufficient for the terminal-aesthetic visual style. Dark backgrounds, atmospheric lighting, abstract tech imagery — SDXL Turbo handles these well. Where it struggles (faces, text in images) I avoid.

Resolution: 512x512 generation, upscaled to 1080x1920 (Shorts format) with `latent-diffusion` upscaler. Full 1080p generation exceeds VRAM.

### 4. Terminal frame animation (PIL)

The most distinctive visual element: fake terminal output that appears character by character, with timestamps, colors, and macOS window chrome.

```python
def render_terminal_frame(text_lines, frame_idx, chars_per_frame=2):
    """Render a terminal frame with text appearing progressively."""
    img = Image.new('RGB', (1080, 1920), color=(13, 17, 23))
    draw = ImageDraw.Draw(img)

    # macOS window chrome
    draw_window_chrome(draw, img.width)

    # Render text up to current character position
    total_chars = frame_idx * chars_per_frame
    current_pos = 0
    for line in text_lines:
        visible = line[:max(0, total_chars - current_pos)]
        draw.text((60, y_pos), visible, font=mono_font, fill=GREEN)
        current_pos += len(line)
        y_pos += LINE_HEIGHT

    return img
```

Generate 30 frames per "screen", export as image sequence, FFmpeg assembles into video. The result looks like a real terminal, typing in real time.

### 5. Ken Burns effect (FFmpeg zoompan)

Static images become dynamic with slow zoom/pan:

```bash
ffmpeg -loop 1 -i background.png \
  -vf "zoompan=z='1.2':x='iw/2-(iw/zoom/2)':y='ih/2-(ih/zoom/2)':d=300:fps=30" \
  -t 10 background_animated.mp4
```

The `z='1.2'` creates a 20% zoom over the duration. `x` and `y` center the zoom. For pan effects, make `x` or `y` a function of time.

### 6. Assembly (FFmpeg)

The final FFmpeg command assembles everything:

```bash
ffmpeg \
  -i background_animated.mp4 \
  -i terminal_animation.mp4 \
  -i voiceover.wav \
  -i music_bed.mp3 \
  -filter_complex "
    [0:v][1:v]overlay=0:0[vid];
    [2:a]volume=1.0[voice];
    [3:a]volume=0.15[music];
    [voice][music]amix=inputs=2:duration=first[audio]
  " \
  -map "[vid]" -map "[audio]" \
  -c:v libx264 -crf 18 -preset fast \
  -c:a aac -b:a 192k \
  output.mp4
```

Music bed at 15% volume means it's present but not competing with the voice. The `duration=first` ensures the mix ends with the voiceover, not the music loop.

### 7. Subtitle generation (Whisper)

Whisper transcribes the voiceover and generates `.srt` files with word-level timestamps. These become the burned-in subtitles (required for Shorts algorithm performance).

```python
import whisper

model = whisper.load_model("base")
result = model.transcribe("voiceover.wav", word_timestamps=True)
# Export as SRT
```

### The full run

On RTX 3080, a 60-second Short:

| Step | Time |
|------|------|
| Script generation (Claude) | 8-12s |
| TTS (Chatterbox) | 10s |
| Image generation (SDXL × 5) | 20s |
| Terminal animation (PIL) | 3s |
| Subtitle generation (Whisper) | 5s |
| Assembly (FFmpeg) | 3s |
| **Total** | **~50s** |

45-50 seconds from topic to finished MP4.

## The economics

One-time costs:
- GPU: already owned (or $200-400 used RTX 3070/3080)
- Models: free to download (Hugging Face)

Ongoing costs:
- Electricity: roughly $0.05/video at typical residential rates
- Nothing else

Compare to ElevenLabs + Midjourney + any cloud video tool: $50-100/month minimum for daily uploads.

## What's next

The pipeline generates the video. The missing piece is upload automation — YouTube Data API OAuth, scheduling, metadata management. That's the next post.

The character and channel are also still being defined. One AI character, one channel, three content streams. More on that soon.

---

*The pipeline scripts are available in the [AI Video Pipeline product](/products#ai-video-pipeline). Includes voice settings, SDXL prompts, and the full FFmpeg assembly chain.*
