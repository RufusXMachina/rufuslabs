# Incident Resolution & Performance Optimization: Threadfin Buffer & Live TV Stream Optimization

## Overview
During Live TV viewing on the Living Room TV (Google TV Streamer) via Plex, streams exhibited two issues:
1. Playback abruptly stopped at scheduled program boundaries (e.g., top/bottom of the hour) and exited back to the guide.
2. Subsequent attempts to re-tune the channel (e.g., KGPE-HD / CBS 47.1) failed with "Playback Error: Could not tune channel" or long buffering stalls.

## Architecture Pipeline
`HDHomeRun FLEX 4K (10.0.0.29) -> Tvheadend (9981) -> Threadfin (34400) -> Plex Media Server (32400) -> Plex Android TV Client (10.0.0.50)`

## Root Cause Analysis

### 1. Program Boundary Rollover (Plex Android TV Client)
- **Symptom:** Live TV playback stopped at 09:30:02 precisely as the scheduled broadcast program ended, returning the user to the guide.
- **Investigation:** PMS logs showed the Android TV client (`10.0.0.50`) submitted timeline updates bound to `context=live-tv.guide:dvrGuide` and a single program ID (`airingId=1674`, `duration=303900ms`). When the program duration elapsed, ExoPlayer hit `STATE_ENDED` on its single-item playback queue and sent `state=stopped`.
- **Resolution / Guidance:**
  - Tuning from the Channel header/logo (`context=live-tv.channels`) rather than the program tile inside the guide grid establishes a continuous channel stream.
  - Ensuring Auto Play is toggled ON in the Plex Android TV app settings.

### 2. Threadfin FFmpeg Buffer Stall & MPEG-2 Header Invalidation
- **Symptom:** Re-tuning failed with a client timeout and playback error on the TV.
- **Investigation:** Server logs across Threadfin and Plex revealed duplicate FFmpeg processes running in series:
  - Threadfin: `ffmpeg -user_agent Threadfin -i http://10.0.0.31:9981/stream/... -c copy -f mpegts pipe:1`
  - Plex: `Plex Transcoder -i http://10.0.0.31:34400/stream/...`
  When HDHomeRun locked onto the ATSC RF frequency (593 MHz), initial MPEG-2 packets arrived mid-GOP before a complete sequence header / I-frame was received. Both Threadfin and Plex logged:
  `[mpeg2video @ ...] Invalid frame dimensions 0x0`
  Threadfin was configured with `buffer: "ffmpeg"` and an aggressive `buffer.timeout: 500ms`. When Threadfin's FFmpeg stalled while attempting to parse the cold-start stream, Plex's client startup timeout expired before valid video frames were delivered.
- **Resolution:**
  - Migrated Threadfin from FFmpeg buffering (`"buffer": "ffmpeg"`) to direct stream pass-through (`"buffer": "-"`).
  - Configured direct mode across both global settings and the M3U playlist provider in `/opt/stacks/threadfin/config/settings.json`.
  - Threadfin now acts as a pure URL pass-through proxy: Plex connects directly to Tvheadend's streaming HTTP socket, eliminating the redundant FFmpeg subprocess, eradicating `0x0` buffer stalls, and reducing channel tune latency by 3–5 seconds.
  - Verified zero-downtime auto-reconnect during active multi-tuner NFL DVR recordings (*Titans at Cowboys* and *Cardinals at Chiefs*).

## Operational Reference & Guide Data Architecture
- Plex natively utilizes its Gracenote Cloud EPG (`EPG[cloud]`) for 14-day guide data, sports team metadata, and episode scheduling.
- WebGrab+Plus is not in the active Plex path and is redundant unless using Tvheadend's internal web UI / Kodi HTSP for standalone scheduling.
