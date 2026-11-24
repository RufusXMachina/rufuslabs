# 🏛️ Rufuslabs

[![Ubuntu](https://img.shields.io/badge/OS-Ubuntu%20Server-E95420?logo=ubuntu&logoColor=white)](https://ubuntu.com/)
[![Docker](https://img.shields.io/badge/Container-Docker%20Compose-2496ED?logo=docker&logoColor=white)](https://www.docker.com/)
[![Cloudflare](https://img.shields.io/badge/Security-Cloudflare%20Zero%20Trust-F38020?logo=cloudflare&logoColor=white)](https://www.cloudflare.com/)
[![Plex](https://img.shields.io/badge/Media-Plex%20Media%20Server-E5A00D?logo=plex&logoColor=white)](https://www.plex.tv/)
[![Intel](https://img.shields.io/badge/Transcode-Intel%20QuickSync%20(VAAPI)-0071C5?logo=intel&logoColor=white)](https://www.intel.com/)

A highly automated, containerized home infrastructure environment powering media streaming, live television broadcasting, home automation, automated content acquisition, and private cloud services.

---

## 🏗️ System Architecture

```mermaid
flowchart TD
    subgraph External["🌐 External Access & Edge"]
        UserRemote["Remote Clients / Mobile"]
        CF["Cloudflare Edge (Zero Trust)"]
        UserRemote -->|HTTPS / SSH| CF
    end

    subgraph Host["🖥️ Ubuntu Server Host (Computer)"]
        CFTunnel["Cloudflared Tunnel"]
        CF -->|Encrypted Tunnel| CFTunnel
        
        subgraph NetworkSec["Security & Routing"]
            CFTunnel --> LocalApps["Local Web UI / Stacks"]
            AdGuard["AdGuard / Pi-hole (Local DNS)"]
        end

        subgraph MediaTier["🎬 Media & Entertainment"]
            Plex["Plex Media Server (iGPU /dev/dri)"]
            Jellyfin["Jellyfin"]
            Tautulli["Tautulli (Monitoring)"]
            Kometa["Kometa (Metadata)"]
        end

        subgraph LiveTVTier["📺 Live TV Broadcast Pipeline"]
            Tvheadend["Tvheadend (Mux / Adapter Pool)"]
            Threadfin["Threadfin (Direct Stream Pass-Through)"]
            Tvheadend -->|MPEG-TS HTTP| Threadfin
            Threadfin -->|Direct Pass-Through| Plex
        end

        subgraph ArrTier["🤖 Automation & Acquisition"]
            Sonarr["Sonarr (TV)"]
            Radarr["Radarr (Movies)"]
            Prowlarr["Prowlarr (Indexers)"]
            Bazarr["Bazarr (Subtitles)"]
            Autobrr["Autobrr"]
            VPN["Gluetun VPN (Killswitch)"]
            Qbit["QBittorrent (VPN-Routed)"]
            SAB["SABnzbd (Usenet)"]
            
            Sonarr & Radarr --> Prowlarr
            Sonarr & Radarr --> Qbit & SAB
            Qbit --- VPN
        end

        subgraph CloudHome["☁️ Cloud & Home Automation"]
            HA["Home Assistant"]
            Vault["Vaultwarden"]
            Nextcloud["Nextcloud"]
            FreshRSS["FreshRSS"]
        end

        subgraph ManagementTier["⚙️ Management & Health"]
            Dockhand["Dockhand / Portainer"]
            Dozzle["Dozzle (Log Viewer)"]
            UptimeKuma["Uptime Kuma"]
            Duplicati["Duplicati / Kopia"]
        end
    end

    subgraph LocalHardware["📡 Local Hardware & Tuners"]
        HDHR["HDHomeRun FLEX 4K (OTA ATSC 1.0/3.0)"]
        HDHR -->|RF 8VSB / ATSC| Tvheadend
    end

    subgraph StorageHierarchy["💾 Multi-Tier Storage"]
        NVMe["Host NVMe SSD<br/>(OS, Docker Stacks, Transcode Cache)"]
        DAS["TerraMaster DAS (Direct Attached)<br/>(/mnt/media/data: Movies, TV, Anime, Music)"]
        NAS["Synology NAS (10.0.0.100)<br/>(Encrypted Deduplicated Backup Target)"]
        
        Plex & ArrTier --> DAS
        Plex --> NVMe
        Duplicati -->|Scheduled Snapshots| NAS
    end

    classDef primary fill:#2563eb,stroke:#1d4ed8,color:#ffffff
    classDef storage fill:#059669,stroke:#047857,color:#ffffff
    classDef hardware fill:#d97706,stroke:#b45309,color:#ffffff
    classDef external fill:#7c3aed,stroke:#6d28d9,color:#ffffff

    class Plex,Tvheadend,Threadfin primary
    class NVMe,DAS,NAS storage
    class HDHR hardware
    class CF,CFTunnel external
```

---

## 📺 Live TV & Tuner Signal Pipeline

The Live TV stack bridges over-the-air (OTA) ATSC digital antenna broadcasts into Plex clients with hardware-accelerated transcoding and zero redundant buffer latency:

```mermaid
flowchart LR
    Antenna["📡 Digital Antenna"] -->|OTA Broadcast| HDHR["SiliconDust HDHomeRun FLEX 4K<br/>(4 Tuners: ATSC 1.0 / ATSC 3.0)"]
    HDHR -->|Raw MPEG-TS Stream| TVH["Tvheadend<br/>(Adapter routing, signal locking)"]
    TVH -->|HTTP MPEG-TS Pass-thru| TF["Threadfin Proxy<br/>(Direct Pass-Through Mode)"]
    TF -->|Direct HTTP Hand-off| PMS["Plex Media Server<br/>(Intel QuickSync /dev/dri HEVC VAAPI)"]
    PMS -->|HLS Stream / Direct Stream| Client["📱 Clients<br/>(Google TV Streamer, Mobile, Web)"]
```

* **Direct Stream Pass-Through**: Threadfin is configured in direct pass-through mode (`buffer: "-"`). Instead of spawning an intermediate FFmpeg buffering process in RAM, Threadfin immediately hands the raw Tvheadend HTTP socket to Plex, shaving 3–5 seconds off channel change latency and eliminating initial keyframe packet dropouts.
* **Hardware Deinterlacing**: Intel QuickSync (`/dev/dri/renderD128`) decodes 1080i broadcast MPEG-2 and hardware-deinterlaces to progressive 60 fps HEVC on the fly.
* **Cloud EPG Integration**: Plex uses its official cloud-backed Gracenote guide for rich 14-day schedule data, sports matchups, and cover art.

---

## 💻 Hardware & Storage Hierarchy

| Tier | Hardware / Device | Purpose & Configuration |
| :--- | :--- | :--- |
| **Computer** | Ubuntu Server x86_64 | Dedicated host running Docker Compose stacks |
| **GPU / Acceleration** | Intel UHD Graphics (QuickSync) | Hardware encoding/decoding via `/dev/dri` (HEVC/H.264/MPEG-2) |
| **Active Storage** | TerraMaster DAS (Direct Attached) | Primary media repository mounted at `/mnt/media/data` (`tv`, `movies`, `music`, `anime`) |
| **Fast Workspace** | High-Speed NVMe SSD | Host OS, Docker container volumes, database storage, and high-throughput transcode workspace |
| **Backup Target** | Synology NAS (`10.0.0.100`) | Network backup target for encrypted, deduplicated system snapshots via Duplicati / Kopia |
| **Network Tuner** | HDHomeRun FLEX 4K (`10.0.0.29`) | 4-tuner OTA digital broadcast receiver (ATSC 1.0 & NextGen ATSC 3.0) |

---

## 🗂️ Service Stack Directory

All services run as modular Docker Compose stacks located in `/opt/stacks/<stack-name>/`:

| Category | Stacks & Key Services | Description |
| :--- | :--- | :--- |
| **Media & Entertainment** | `plex-stack`, `jellyfin`, `plex-companions` | Media servers, Tautulli statistics, Kometa automated artwork |
| **Live TV & Broadcasting** | `tvheadend`, `threadfin`, `webgrabplus` | Broadcast tuner management, channel proxying, EPG aggregation |
| **Automation & Downloaders** | `media-stack`, `vpn-stack`, `sabnzbd`, `autobrr`, `unpackerr` | Sonarr, Radarr, Prowlarr, Bazarr, Gluetun VPN killswitch, QBittorrent |
| **Audiobooks & Reading** | `audiobookshelf`, `kavita`, `shelfmark`, `metube`, `youtarr` | Audiobooks, comics/ebooks, YouTube/audio archival |
| **Cloud & Security** | `vaultwarden`, `nextcloud`, `freshrss`, `homeassistant`, `cloudflared` | Self-hosted password vault, personal cloud, RSS, smart home, Cloudflare tunnel |
| **System & Observability** | `dockhand`, `dozzle`, `uptime-kuma`, `duplicati`, `backup` | Container orchestration, real-time log tailing, uptime checks, snapshot backups |

---

## 🛡️ Security & Operational Principles

1. **Zero Port Forwarding**: The firewall blocks all incoming WAN ports. External ingress is secured end-to-end through Cloudflare Zero Trust tunnels with identity provider authentication.
2. **Download Isolation (VPN Killswitch)**: BitTorrent traffic is routed exclusively through a Gluetun WireGuard container. If the VPN tunnel drops, traffic instantly halts with no IP leaks.
3. **Dedicated Transcoding Mount**: Plex transcode workspace is isolated on dedicated host SSD storage (`./transcode:/transcode`) with unrestricted throughput, preventing buffer overflows during multi-hour live broadcasts and multi-tuner DVR sessions.
4. **Automated 3-2-1 Backups**: Daily encrypted snapshots of `/opt/stacks` configuration files and container databases are captured and mirrored to the Synology NAS.

---

## 📚 Technical Documentation & Incident Postmortems

Detailed operational runbooks and architectural decision records can be found in [`docs/`](docs/):
* [**Homelab Architecture Diagram**](docs/architecture.md) — Visual overview of network topology and storage mounts.
* [**Live TV Threadfin Buffer Optimization**](docs/live-tv-threadfin-buffer-optimization.md) — Incident postmortem on tuning latency, direct pass-through migration, and Plex program-boundary handling.
* [**Plex Live TV Transcode & Mux Resolution**](docs/live-tv-plex-transcode.md) — Resolution of transcode storage limits and Tvheadend stale mux cleanup.
* [**Backup & Disaster Recovery Strategy**](docs/backups.md) — Backup procedures and restore runbook.

