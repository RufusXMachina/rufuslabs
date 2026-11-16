# Incident Resolution: Plex Live TV Buffering & Tvheadend Mux Timeout

## Overview
During extended Live TV viewing sessions (such as live sporting events), Plex playback would abruptly crash with a playback error. In addition, certain OTA channels (such as virtual channel 26.1 / FOX) experienced noticeable tuning latency or failure to lock signal.

## Root Cause Analysis

### 1. Plex Transcode Storage Exhaustion (/dev/shm)
- **Symptom:** Live TV stream crashes after approximately 90 minutes of continuous playback.
- **Investigation:** Inspection of Plex Media Server logs identified write failures: Quota exceeded (write) in the transcode directory.
- **Cause:** The transcode directory was mapped to /dev/shm (RAM disk), which was capped at 7.5 GB. While RAM transcoding reduces drive wear for short clips, prolonged Live TV sessions buffer rolling MPEG-TS segments until tmpfs space is completely exhausted.
- **Resolution:**
  - Migrated the transcode volume mount from /dev/shm to a dedicated host NVMe SSD directory (./transcode:/transcode).
  - The NVMe root volume provides >130 GB of free workspace, allowing multi-hour continuous live buffers without disk pressure.
  - Ensured /dev/dri:/dev/dri hardware acceleration (Intel QuickSync) is exposed for efficient video encoding.

### 2. Tvheadend Stale Mux Timeout (Channel 26.1 / KMPH-HD)
- **Symptom:** Up to 16-second delay when tuning virtual channel 26.1, occasionally timing out in Threadfin / Plex.
- **Investigation:** Log analysis revealed Tvheadend attempted to tune an obsolete OTA mux at 557MHz (which produced no signal / transport stream errors) before failing over to the active 509MHz mux.
- **Cause:** Duplicate, unverified muxes from older frequency scans remained in Tvheadend's OTA mux configuration.
- **Resolution:**
  - Backed up and pruned the obsolete 557MHz mux configuration.
  - Verified clean single-mux lock on 509MHz with instantaneous lock and zero transport stream packet loss.
