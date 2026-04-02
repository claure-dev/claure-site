---
title: "Building an AI Video Pipeline on a Home GPU"
description: "Voice cloning, image generation, and automated assembly — all local, all free after hardware."
pubDate: 2026-03-20
tags: ["AI", "python", "GPU", "video"]
readTime: 7
---

I wanted to make YouTube Shorts without paying per-video API costs. The cloud pricing for AI video is designed for people who make 10 videos. I wanted to make hundreds.

So I built a pipeline that runs entirely on my RTX 3080. Script to finished video in about a minute. The marginal cost per video is my electricity bill.

## The pipeline

Six stages, run in sequence:

**Script** — Claude generates a structured JSON script with narration text and visual prompts per section. The visual prompts matter — they're what the image generator sees, so the script writer needs to know what kind of images will look good.

**Voice** — Chatterbox TTS from Resemble AI. Open-source, runs on 3.2GB VRAM, and the voice quality is genuinely good. You give it a 15-second reference clip and it clones the voice. I tuned the settings toward "expressive" rather than "accurate" — higher exaggeration, lower cfg weight, slight temperature.

**Image** — This is where it gets interesting. I've used three backends: SDXL Turbo (fast but mediocre), Flux via ComfyUI (slow but great), and Pollinations API (fast, good, free tier). The pipeline supports mixing backends — generate what you can with the API, fill the rest locally.

**Subtitles** — Whisper transcribes the audio and aligns words to timestamps. The subtitles get burned into the video because that's what the YouTube algorithm rewards.

**Music** — MusicGen generates a background track from a text prompt. Optional — you can also use a static music bed.

**Assembly** — FFmpeg composites everything. Each image gets a DepthFlow parallax effect (depth estimation + 3D camera motion), voice and music get mixed, subtitles get overlaid. The result is a vertical 1080x1920 MP4.

## What actually works

The voice cloning is the most impressive piece. Chatterbox produces output that sounds natural enough that people don't immediately clock it as AI. The trick is the reference audio — clean recording, no background noise, natural pacing.

The image generation is the weakest link. SDXL Turbo is fast but produces soft, generic images. Flux is better but takes 3+ minutes per image on my 3080. Pollinations is the sweet spot right now — good quality, fast, but rate-limited on the free tier.

The assembly step with DepthFlow is what makes the videos look professional. A static image with subtle parallax motion looks dramatically better than a static image or a basic zoom-pan. The depth estimation model (Depth Anything V2) runs in about a second per image.

## What doesn't work

Resume. If the pipeline fails on image 4 of 5, restarting means figuring out which steps completed, which outputs are salvageable, and how to skip ahead. I've added skip-existing checks for most steps, but the experience is still rough. This is the next thing to fix.

API rate limits. Pollinations gives you a free tier that's generous until it isn't. Running out of credits mid-batch means switching backends for the remaining images, which means different visual styles in the same video. I'm working on getting to their next tier.

Quality consistency. AI-generated images don't always match the mood or style you want. One image in five will look off — wrong lighting, wrong composition, doesn't match the narration. The fix is regeneration, but that adds friction.

## The code

The orchestrator chains everything:

```bash
python orchestrator.py --brand rogue-ai --topic "your topic here"
```

Brands define the full config — which voice profile, which image style, which effects, which music. The brand system means I can create different "shows" with different visual identities without changing code.

The workbench UI (React + FastAPI) lets you preview and iterate on individual runs — edit scripts, regenerate specific images, swap voice settings, re-assemble. It's the difference between "run the pipeline and hope" and "sculpt the video."

## Economics

Hardware: RTX 3080 (already owned). Models: free (Hugging Face). API costs: $0 on free tiers, ~$10/month if you need more.

Compare to: ElevenLabs TTS ($5-22/month) + Midjourney ($10-30/month) + any video assembly tool ($20+/month). The local pipeline pays for itself in a month if you're producing regularly.

The real cost is time. Getting the pipeline working took weeks of development. Maintaining it takes ongoing effort. This is a tool for someone who wants to understand and control every stage, not for someone who wants to upload a topic and get a video back.
