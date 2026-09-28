# Media Automation Stack

## Overview

This project documents the Proxmox-based media automation environment built around the shared `/media` storage pool.

The stack separates media management, downloading, anti-bot/indexer handling, extraction, and playback into dedicated services while allowing the applications to share the same media storage.

## Proxmox Layout

| LXC | Hostname | Address | Role |
|---:|---|---|---|
| 101 | `jellyfin` | Proxmox LXC | Media playback |
| 102 | `media` | `192.168.200.251` | Sonarr, Radarr, Prowlarr |
| 103 | `downloads` | `192.168.200.252` | qBittorrent, Unpackerr |
| 104 | `byparr` | `192.168.200.253` | Indexer anti-bot proxy |

```text
                         Proxmox Host
                              |
              +---------------+---------------+
              |               |               |
              v               v               v
        LXC 101           LXC 102          LXC 103
        Jellyfin           media           downloads
                              |               |
                              |               +-- qBittorrent
                              |
                    +---------+---------+
                    |         |         |
                    v         v         v
                  Sonarr    Radarr   Prowlarr
                              |
                              v
                         Indexers
                              |
                              v
                    LXC 104 / Byparr
                       192.168.200.253
                         TCP/8191
```

## Shared Media Storage

The media environment uses a shared `/media` mount across the relevant containers. The storage pool provides approximately 8.2 TB of pooled media storage.

```text
/media/
├── movies/
├── tv/
├── music/
└── downloads/
```

LXC 103 was validated with the shared mount and media-group permissions. A file create/delete test was used to verify write and cleanup behavior.

## Media Management

### Sonarr

Sonarr manages television content and works with the shared media and download paths. A monitoring issue was identified during setup where newly added series were not entering the expected monitored state. The missing monitoring setting was corrected and validated.

### Radarr

Radarr manages movie content using the shared media/download architecture.

### Prowlarr

Prowlarr provides centralized indexer management for Sonarr and Radarr. It runs natively in LXC 102 rather than in Docker.

### qBittorrent

qBittorrent runs in LXC 103 and provides the download client used by the automation stack.

### Byparr

Byparr runs in its own unprivileged Debian 13 LXC:

```text
LXC 104
192.168.200.253
TCP 8191
```

Docker runs inside this LXC with `ghcr.io/thephaseless/byparr`. Byparr provides a FlareSolverr-compatible interface for indexers affected by Cloudflare challenges.

Connectivity from LXC 102 to Byparr was validated through its health endpoint. Prowlarr was configured with a Byparr proxy and a dedicated `byparr` tag so the proxy can be applied selectively.

### Unpackerr

Unpackerr runs as a systemd service in LXC 103. Its configured completion path is `/media/downloads/complete`. The service was validated with the shared media group and create/delete permissions.

The final configuration keeps original archives by using `delete_orig=false`.

## Jellyfin

Jellyfin runs in LXC 101 and uses the shared media storage for Movies, TV Shows, and Music.

The deployment uses Intel Alder Lake-N graphics for hardware acceleration. Current validated settings include:

```text
HardwareAccelerationType=qsv
VaapiDevice=/dev/dri/renderD128
QsvDevice=blank
EncoderPreset=auto
```

Jellyfin was upgraded to version 12.0.0 and restarted successfully after configuration migration and admin-access troubleshooting. The Jellyfin service account has access to the `video` and `render` groups required for hardware acceleration.

## Operational Lessons

- Separate application roles into dedicated LXCs.
- Use shared storage rather than duplicating media between containers.
- Keep anti-bot/indexer handling isolated from the main application containers.
- Use selective proxy tagging rather than sending every indexer through the same proxy.
- Validate Linux UID/GID and group permissions before troubleshooting application behavior.
- Treat media automation as multiple cooperating services rather than one monolithic application.

## Validation

- LXC-to-LXC connectivity testing
- Shared `/media` mount validation
- File create/delete permission tests
- Byparr health checks
- Prowlarr proxy tests
- Indexer testing
- Sonarr monitoring validation
- Jellyfin service/restart validation
- Jellyfin hardware-acceleration configuration checks
- Unpackerr systemd status and extraction-path validation

## Security / Operational Notes

The media stack is a homelab service environment and should remain behind the intended LAN/VPN access boundaries. Indexer and download services should not be exposed directly to the public internet.

This document records the architecture and completed implementation work.