# Homelab Storage & Media Architecture (Ubuntu Mini PC)

## Hardware topology

- **Ubuntu Mini PC** — runs all containers via Docker. Managed by Coolify (web UI on top of Docker Compose).
- **Unraid Tower at `192.168.1.200`** — the actual mass storage (drives, parity, all the bulk media).
- Connected over the LAN through a switch.

The mini PC has **no media data of its own**. It runs the apps; Unraid holds the files.

## Storage layer

### 1. The Unraid share is mounted on the host via CIFS/SMB

In `/etc/fstab`:
```
//192.168.1.200/data  /mnt/network_share/data  cifs  username=...,password=...,vers=3.0,dir_mode=0777,file_mode=0777  0 0
```

- Mount point on the mini PC: `/mnt/network_share/data`
- Directory layout inside the share:
  ```
  /mnt/network_share/data/
  ├── books/            # Calibre library lives here (read by calibre-web-automated)
  ├── metadata.db
  ├── movies/           # Radarr root folder
  ├── tv/               # Sonarr root folder
  └── dvr/recordings/   # Channels DVR target
  ```
- Known issue: if the network blips (switch reboot), the CIFS mount drops silently and apps see an empty `/data`. fstab options should include `nofail,_netdev,reconnect=1,x-systemd.automount` to self-heal. After a remount, Docker containers must be **restarted** because bind mounts resolve at container-start time (default mount propagation = `rprivate`).
- Security caveat: credentials are inline in fstab as plaintext today; should move to a root-only `credentials=` file.

### 2. Local-only paths on the mini PC

A few paths are deliberately local (not on Unraid), used as scratch / work-in-progress areas:
```
/data/torrents/                       # qbittorrent active downloads
/data/sabnzbd_downloads/complete/     # Usenet completions before import
/data/book_ingest/                    # Calibre auto-import drop zone
/data/book_downloads/                 # Calibre library output
/docker/appdata/<app>/                # Per-app config dirs
```

The arr stack does **hardlinks/atomic moves from `/data/torrents` → `/mnt/network_share/data/movies|tv`**, which is why the local download path and the Unraid mount path are both presented to each container under the same `/data` parent inside the container — TRaSH-guides-style layout.

## Container architecture

### Network model — everything VPN-tunneled via gluetun

- **gluetun** (`qmcgaw/gluetun`) is the WireGuard VPN container. It owns a network namespace.
- All download/indexer/automation apps run with `network_mode: "service:gluetun"`, so their egress goes through the VPN and their inbound traffic is reached via ports **published on gluetun**, not on the app containers themselves.
- That's why `docker ps` shows gluetun publishing 12+ ports (80, 443, 7878, 8989, etc.) — those are really for SWAG, Radarr, Sonarr, qBittorrent, Bazarr, Pulsarr, Calibre-Web, Overseerr, etc.

Containers sharing gluetun's namespace:
```
swag, qbittorrent, radarr, sonarr, prowlarr, sabnzbd, overseerr,
bazarr, pulsarr, calibre-web-automated, watchtower
```

### Reverse proxy

- **SWAG** (`linuxserver/swag`) is the public-facing reverse proxy + Let's Encrypt. It binds host ports **80 and 443** (via gluetun's port publishes). DuckDNS handles dynamic DNS.
- **Conflict to know about:** Coolify wants to run its own Traefik on 80/443. Today SWAG wins. Either disable Coolify's proxy or move SWAG off 80/443 (would require DNS-01 cert challenges instead of HTTP-01).

### Host-network containers (not via VPN)

- **plex** — `network_mode: host` (for DLNA discovery and direct LAN streaming). Mounts only `/mnt/network_share/data/media`.
- **channels-dvr** — `network_mode: host` (needs LAN multicast for TVE tuners). Has `/dev/dri` for hardware transcoding.
- **duckdns** — bridged, no shared namespace.

### Image management

- **Watchtower** runs nightly at 04:00 (`WATCHTOWER_SCHEDULE=0 0 4 * * *`), pulls through the VPN, and prunes old images. SWAG and gluetun are excluded via `com.centurylinklabs.watchtower.enable=false`.

## How an app sees a movie

End-to-end example (Radarr finding `/data/media/movies/Foo (2023)/foo.mkv`):
1. Container path: `/data/media/movies/Foo (2023)/foo.mkv`
2. Container volume: `/mnt/network_share/data:/data` → host path is `/mnt/network_share/data/media/movies/...`
3. Host mount: `/mnt/network_share/data` is a CIFS mount of `//192.168.1.200/data`
4. Wire: SMB3 traffic over LAN to Unraid → reads off the array.

Plex maps narrower: `/mnt/network_share/data/media:/data/media`, so Plex only sees `media/` (no torrents/books).

## Permissions

- All linuxserver.io images use `PUID`/`PGID` env vars. On this host they're set to `1000:1000` (user `aaron`). The Unraid SMB share is mounted `dir_mode=0777,file_mode=0777`, so permissions don't really gate access — the share is wide-open within the LAN. Files written by the container appear as uid 1000 on the host.

## Coolify's role

- Coolify created the compose stack from `compose.yaml` in this repo. Container names get a hash suffix like `radarr-zo4gsgss4oc00sgc0wc0ckww-003629153763`.
- The `coolify` external network is referenced by gluetun (`networks: [coolify]`) so Coolify's reverse proxy *could* talk to gluetun in principle, but in practice SWAG is doing the proxying.
- Restart via Coolify UI = `docker restart`. **Redeploy** would recreate the container — avoid that for routine restarts.

---

# Porting to a Mac Mini — things that need to change

1. **CIFS mount → use SMB or NFS via macOS automount.** No `/etc/fstab`. Options:
   - `mount_smbfs` invoked from a `launchd` job at boot.
   - `/etc/auto_master` + `/etc/auto_smb` for autofs (closest equivalent to fstab+automount).
   - **Better:** turn on NFS export on Unraid and use NFS instead — POSIX semantics, no plaintext password, survives reconnects more gracefully on macOS.
2. **Docker Desktop on macOS runs containers in a Linux VM.** Bind mounts cross a virtualization boundary; CIFS-on-host bind-mounted into a Linux container is notoriously slow and sometimes flaky on macOS. Two safer patterns:
   - Mount the NAS share **inside** the Docker VM directly (Docker Desktop supports this via custom VM config) rather than on the host.
   - Or run Docker in a real Linux VM (UTM/Lima/Colima), not Docker Desktop.
3. **Coolify is Linux-only.** Either drop Coolify and use plain `docker compose`, or run Coolify inside a Linux VM on the Mac.
4. **`network_mode: host` doesn't work on Docker Desktop for Mac** the way it does on Linux (it works syntactically but it's the VM's host, not the Mac's). Plex and Channels DVR rely on host networking for LAN discovery — those need re-thinking. Options:
   - Put Plex on a NUC/Pi or keep it on the Unraid box directly.
   - Use `macvlan` from inside the Linux VM bridged to the Mac's LAN (complex).
5. **Hardware transcoding:** `channels-dvr` uses `/dev/dri` (Intel QuickSync on the current mini PC). Mac mini has no `/dev/dri`; Apple Silicon has VideoToolbox but containers can't access it. Hardware transcoding effectively becomes unavailable inside Docker on a Mac.
6. **gluetun + WireGuard kernel module:** gluetun uses userspace WireGuard inside the container, so the VPN itself ports fine. But `/dev/net/tun` access on Docker Desktop for Mac generally works (passed through the VM).
7. **PUID/PGID:** macOS default user is uid 501. Either change PUID/PGID env to 501:20, or accept the file-permission mismatch on writes (irrelevant if NAS is wide-open).
8. **Local "scratch" paths** (`/data/torrents`, `/data/sabnzbd_downloads`) — fine to keep on the Mac's internal SSD, just rename to a Mac-conventional location like `/Users/Shared/data/torrents`.

## TL;DR portability assessment

- **VPN-tunneled apps (radarr/sonarr/qbittorrent/sabnzbd/etc):** clean port, just change mount paths.
- **Plex / Channels DVR (host-networked, hardware-accelerated):** problematic on Mac mini, better left running on Unraid or a Linux box.
- **Coolify:** drop it or VM it.
- **NAS mount:** switch to NFS + autofs, don't try to recreate the CIFS-in-fstab pattern.

The cleanest Mac mini design is probably: NFS mount from Unraid via autofs → Docker Desktop (or Colima) → run only the gluetun stack there → leave Plex/DVR on dedicated hardware.
