# Stream — Umbrel media stack

A complete, self-hosted media automation stack as a **single Umbrel app**:
request a movie or show in Jellyseerr, it gets found and downloaded
automatically (through a VPN), gets extracted if needed, subtitles are fetched,
and it shows up in Jellyfin — using **hardlinks**, so a file is never duplicated
on disk and keeps seeding while it sits in your library.

> **qBittorrent runs behind ProtonVPN (gluetun).** Its WebUI is still on host
> port `8082` (published by gluetun); **inside the stack reach it at
> `gluetun:8082`, not `qbittorrent:8082`** — qBittorrent shares gluetun's network
> namespace and has no hostname of its own.

## Requirements

- **Umbrel** (umbrelOS) on an x86_64 host.
- **Intel iGPU** (e.g. an N100/N150 box) for Jellyfin hardware transcoding —
  exposes `/dev/dri`, uses the host `render` group (GID `991` in the compose;
  adjust if your host differs).
- **A ProtonVPN paid plan** (Plus/Unlimited — they include P2P + port
  forwarding). qBittorrent will not start without working VPN credentials.
- **Kernel WireGuard** on the host (modern kernels have it) — gluetun uses the
  kernel module automatically and falls back to a slower userspace impl. if it's
  missing.
- Enough disk for your library plus in-flight downloads (downloads and library
  share one volume; see *Data layout*).

## What's inside

| Service | Port | Image (pinned) | Role |
|---|---|---|---|
| **Jellyfin** | 8096 | `jellyfin/jellyfin:10.11.11` | Media server / player (Intel QSV transcoding) |
| **Jellyseerr** | 5055 | `seerr/seerr:v3.3.0` | Request UI (logs in with Jellyfin accounts) |
| **Sonarr** | 8989 | `linuxserver/sonarr:4.0.17.2952-ls313` | TV automation |
| **Radarr** | 7878 | `linuxserver/radarr:6.2.1.10461-ls305` | Movie automation |
| **Bazarr** | 6767 | `linuxserver/bazarr:v1.5.6-ls350` | Subtitles (pt-BR by default) |
| **Prowlarr** | 9696 | `linuxserver/prowlarr:2.4.0.5397-ls149` | Indexer manager (syncs to Sonarr/Radarr) |
| **qBittorrent** | 8082 | `linuxserver/qbittorrent:5.2.2_v2.0.13-ls462` | Download client — behind the VPN, at `gluetun:8082` |
| **gluetun** | internal | `qmcgaw/gluetun` (digest-pinned) | ProtonVPN gateway (WireGuard + kill-switch + port forwarding) |
| **Byparr** | internal `8191` | `thephaseless/byparr` (digest-pinned) | Cloudflare solver, FlareSolverr-compatible API |
| **cross-seed** | internal `2468` | `cross-seed/cross-seed:6.13.7` | Re-seeds finished downloads on other trackers (hardlink) |
| **Unpackerr** | internal | `golift/unpackerr:0.15.2` | Auto-extracts RAR/scene releases for the *arrs |
| **qbit-port-sync** | internal | `curlimages/curl:8.11.1` | Keeps qBittorrent's listen port = ProtonVPN's forwarded port |
| **init** | — | `busybox:stable` | One-shot: creates the config/data directory tree and fixes ownership before anything starts |

Images are **pinned** (by version, or by digest for projects without semver tags
like gluetun and Byparr) for reproducible deploys; updates are deliberate via the
store. The UIs live at `http://<umbrel-ip>:<port>`; internal services have no
host UI.

## How it works

```
Jellyseerr (you request something)
  └─> Sonarr / Radarr  (find a release via Prowlarr's indexers)
        └─> qBittorrent (download — all traffic through ProtonVPN)
              └─> Unpackerr (extract, if it's a .rNN scene pack)
                    └─> Sonarr/Radarr import by HARDLINK into the library
                          └─> Bazarr fetches subtitles
                                └─> Jellyfin plays it (HW transcode if needed)
  cross-seed re-seeds finished downloads on other trackers (hardlink, no re-download)
```

### Data layout (one volume, hardlinks, zero duplication)

Downloads and the library share **one filesystem**, so imports are hardlinks: a
file exists once on disk but appears both in the torrent folder (still seeding)
and in the library. The `init` container creates this tree (owned by `1000:1000`)
on first start:

```
data/
├── torrents/
│   ├── movies/     # Radarr downloads
│   ├── tv/         # Sonarr downloads
│   └── cross-seed/ # cross-seed hardlinks
└── media/
    ├── movies/     # Radarr library  ─┐ Jellyfin reads /media (read-only)
    └── tv/         # Sonarr library  ─┘
```

The *arrs, qBittorrent, cross-seed and Unpackerr all mount this volume at
`/data` (so hardlinks work); Jellyfin mounts `data/media` at `/media:ro`.
Per-service config lives under `~/umbrel/app-data/camarigor-stream/config/<service>/`.

## Networking & security

- **Internal DNS names:** services reach each other by service name on Umbrel's
  Docker network — `http://sonarr:8989`, `http://radarr:7878`,
  `http://prowlarr:9696`, `http://byparr:8191`, `http://cross-seed:2468`, etc.
  qBittorrent is the exception: it shares gluetun's network namespace, so it has
  **no name of its own** — reach it at **`gluetun:8082`**.
- **Kill-switch:** gluetun blocks all non-VPN egress. If the tunnel drops,
  qBittorrent (and the port-sync sidecar) lose internet — they never leak the
  real IP. The BitTorrent listen port is **not** published on the host; peers
  arrive via ProtonVPN's forwarded port through the tunnel.
- **Hardening:** every service runs with `no-new-privileges`; the LinuxServer
  apps run as UID/GID `1000`; Jellyfin, Jellyseerr, cross-seed and Unpackerr run
  as non-root `1000:1000` at the Docker level. gluetun is the only service with
  an added capability (`NET_ADMIN`, required to configure the tunnel and
  iptables), so it intentionally omits `no-new-privileges`.
- **Secrets stay out of the repo:** the ProtonVPN WireGuard key/address live only
  in `config/gluetun/` on the host (see step 9). Any host-specific values (LAN
  DNS, internal hostnames) live in `exports.sh` (see *Customization*).
- **Jellyfin** uses the **official** `jellyfin/jellyfin` image (not LinuxServer):
  it ships the Intel OpenCL runtime needed for HDR→SDR tone mapping
  (`tonemap_opencl`), which the LSIO image lacks. It runs as `1000:1000` with
  `group_add: 991` (host `render`) and `/dev/dri` for Quick Sync, and its
  `JELLYFIN_*_DIR` env vars keep the existing on-disk config layout.

## VPN — gluetun + ProtonVPN

**All** qBittorrent traffic egresses through a ProtonVPN WireGuard tunnel.

- **Protocol:** WireGuard (`VPN_TYPE=wireguard`, provider `protonvpn`), using the
  host's **kernel** WireGuard module when available.
- **Servers:** `PORT_FORWARD_ONLY=on` connects only to ProtonVPN's P2P /
  port-forwarding servers; `SERVER_COUNTRIES=Brazil` by default.
- **Port forwarding:** gluetun requests a port via NAT-PMP and writes it to
  `/gluetun/forwarded_port`; the **qbit-port-sync** sidecar pushes it into
  qBittorrent's `listen_port` and re-announces all torrents whenever it changes,
  so seeding/ratio/H&R survive reconnects.
- **Healthcheck:** a startup TLS check (`HEALTH_TARGET_ADDRESSES`, default
  `cloudflare.com:443`) plus an ongoing ICMP check **through the tunnel**
  (`HEALTH_ICMP_TARGET_IPS=1.1.1.1,8.8.8.8`) — the ICMP check is what detects a
  silently-dead tunnel and triggers a reconnect.
- **DNS:** gluetun's built-in resolver forwards every query to a plaintext
  upstream (default `1.1.1.1`; override with `LAN_DNS`).

## Customization — `exports.sh`

The compose ships **safe public defaults** (Cloudflare DNS, a public healthcheck
target, only the Umbrel Docker subnet allowed outbound), so it works out of the
box. To use **your own LAN DNS resolver**, an **internal healthcheck host**, or a
different VPN country, create `~/umbrel/app-data/camarigor-stream/exports.sh`.
Umbrel sources it before every `docker compose up`, so these values override the
defaults, stay **out of the public store repo**, and **survive app updates**:

```sh
# ~/umbrel/app-data/camarigor-stream/exports.sh
export LAN_DNS="192.168.1.53"                              # your LAN resolver (AdGuard/Pi-hole/router)
export VPN_HEALTHCHECK_HOST="health.example.com"           # internal host with a publicly-valid cert
export VPN_OUTBOUND_SUBNETS="10.21.0.0/16,192.168.1.0/24"  # Umbrel net + your LAN
```

| Variable | Default | What it does |
|---|---|---|
| `LAN_DNS` | `1.1.1.1` | Routes qBittorrent's DNS through your own resolver (exits via the LAN; torrent traffic still tunnels). |
| `VPN_HEALTHCHECK_HOST` | `cloudflare.com` | Points gluetun's **startup** check at an internal host reached over the LAN (no tunnel dependency → avoids startup flapping when DNS is on a LAN resolver). Auto-exempted from gluetun's DNS-rebinding protection. Must present a **publicly-valid cert** and **not** be a raw IP (cert verification needs a SAN match). |
| `VPN_OUTBOUND_SUBNETS` | `10.21.0.0/16` | Subnets gluetun may reach off-tunnel. Add your LAN if `LAN_DNS` is on it. **Never** use broad ranges like `10.0.0.0/8` — they overlap ProtonVPN's tunnel gateway (`10.2.0.x`) and break NAT-PMP. |

`SERVER_COUNTRIES` is set in the compose (default `Brazil`); change it there to
pick another country. After editing `exports.sh`, restart the app.

## Post-install setup (order matters)

### 1. qBittorrent (`:8082`)

1. Temporary password: `sudo docker logs $(sudo docker ps --format '{{.Names}}' | grep qbittorrent) 2>&1 | grep -i 'temporary password'`
2. Log in as `admin` + temporary password → Tools → Options → WebUI →
   change username/password.
3. Options → Downloads → Default Save Path: `/data/torrents`.
4. Create categories (right-click Categories in the sidebar):
   - `movies` → Save Path `/data/torrents/movies`
   - `tv` → Save Path `/data/torrents/tv`
5. Options → WebUI → enable **"Bypass authentication for clients on localhost"**
   (lets the qbit-port-sync sidecar set the forwarded port without credentials).

### 2. Prowlarr (`:9696`)

1. Create user/password on first access (Authentication: Forms).
2. Settings → Indexers → Add Indexer Proxy → **FlareSolverr** (Byparr is
   API-compatible, so the proxy type stays "FlareSolverr"): Host
   `http://byparr:8191`, Tag `byparr`.
3. Indexers → Add: register your public indexers (apply the `byparr` tag on
   Cloudflare-protected ones) and private trackers (tracker API key).
4. Settings → Apps → Add:
   - **Sonarr**: Prowlarr Server `http://prowlarr:9696`, Sonarr Server
     `http://sonarr:8989`, API Key (Sonarr → Settings → General → API Key).
   - **Radarr**: same with `http://radarr:7878`.
5. After saving, indexers show up automatically in Sonarr/Radarr.

### 3. Sonarr (`:8989`) and Radarr (`:7878`)

1. Create user/password (Forms).
2. Settings → Media Management → Add Root Folder:
   - Sonarr: `/data/media/tv`
   - Radarr: `/data/media/movies`
3. Settings → Download Clients → Add → qBittorrent:
   - Host `gluetun` (qBit shares gluetun's network), Port `8082`,
     username/password from step 1, Category `tv` (Sonarr) / `movies` (Radarr).
4. Confirm Settings → Media Management → Importing: **Use Hardlinks instead of
   Copy** = ON (default).

### 4. Bazarr (`:6767`)

1. Settings → Languages: Language Filter `Portuguese (Brazil)`; create a
   Language Profile `pt-BR` (Subtitle = Portuguese (Brazil)) and set it as
   Default for Series and Movies.
2. Settings → Providers: add OpenSubtitles.com (free account) and/or others.
3. Settings → Sonarr: Host `sonarr`, Port `8989`, Sonarr API Key.
4. Settings → Radarr: Host `radarr`, Port `7878`, Radarr API Key.

### 5. Jellyfin (`:8096`)

1. Initial wizard: language, admin user.
2. Add Media Library:
   - Movies → `/media/movies`
   - Shows → `/media/tv`
3. Dashboard → Playback → Transcoding:
   - Hardware acceleration: **Intel QuickSync (QSV)**
   - Enable hardware decoding: H264, HEVC, HEVC 10bit, VP9, AV1
   - Enable hardware encoding: ON
   - Enable Tone mapping: ON (HDR→SDR)
4. Validate: play a video forcing a lower quality (player gear icon) and check
   Dashboard → Active Devices shows `(hw)` on the transcode.

### 6. Jellyseerr (`:5055`)

1. Sign in with Jellyfin: URL `http://jellyfin:8096`, Jellyfin admin.
2. Settings → Radarr: Add Server — Hostname `radarr`, Port `7878`, API Key,
   Quality Profile and Root Folder `/data/media/movies`, set Default.
3. Settings → Sonarr: same — `sonarr`/`8989`, Root `/data/media/tv`.

### 7. cross-seed (no UI)

1. The first start generates
   `~/umbrel/app-data/camarigor-stream/config/cross-seed/config.js` and the
   container restart-loops until it is filled in (expected). Edit **only these
   three fields** — the v6 defaults (`useClientTorrents: true`,
   `linkType: "hardlink"`, `matchMode: "flexible"`, `action: "inject"`,
   `port: 2468`) are already correct:

   ```js
   torrentClients: ["qbittorrent:http://USER:PASSWORD@gluetun:8082"],

   torznab: [
     // Prowlarr → Indexers → copy each indexer's Torznab feed URL
     // "http://prowlarr:9696/1/api?apikey=YOUR_PROWLARR_API_KEY",
   ],

   linkDirs: ["/data/torrents/cross-seed"],
   ```

2. qBittorrent → Options → Downloads → Run external program on torrent finished:
   `curl -s -XPOST http://cross-seed:2468/api/webhook --data-urlencode "infoHash=%I"`
3. Restart the app from the Umbrel UI.
4. Logs: `sudo docker logs -f $(sudo docker ps --format '{{.Names}}' | grep cross-seed)`.

### 8. Unpackerr (no UI)

Auto-extracts RAR/scene releases (e.g. BluRay packs that arrive as `.r00`-`.rNN`)
so Sonarr/Radarr can import them instead of stalling on "Found archive file".

1. The first start writes
   `~/umbrel/app-data/camarigor-stream/config/unpackerr/unpackerr.conf`. Under
   the `[[sonarr]]` and `[[radarr]]` blocks, uncomment and set `url`, `api_key`
   and `paths`:

   ```toml
   [[sonarr]]
     url = "http://sonarr:8989"
     api_key = "YOUR_SONARR_API_KEY"
     paths = ['/data/torrents']

   [[radarr]]
     url = "http://radarr:7878"
     api_key = "YOUR_RADARR_API_KEY"
     paths = ['/data/torrents']
   ```

   API keys: Sonarr/Radarr → Settings → General → API Key.
2. Restart the app from the Umbrel UI.
3. Logs: `sudo docker logs -f $(sudo docker ps --format '{{.Names}}' | grep unpackerr)`.

### 9. VPN — gluetun + ProtonVPN (no UI)

gluetun restart-loops until the key files below exist (expected).

1. ProtonVPN dashboard → **WireGuard configuration**:
   - Platform: **GNU/Linux**; turn ON **NAT-PMP (Port Forwarding)**; leave
     Moderate NAT off; VPN Accelerator on.
   - From the generated config note the `PrivateKey` and the `Address`
     (e.g. `10.2.0.2/32`).
2. Drop the secrets into `~/umbrel/app-data/camarigor-stream/config/gluetun/`
   (these stay out of the store repo):

   ```sh
   printf '%s' 'YOUR_WIREGUARD_PRIVATE_KEY' > config/gluetun/wireguard_private_key
   printf '%s' '10.2.0.2/32'                > config/gluetun/wireguard_addresses
   ```

3. (Optional) For a custom LAN DNS resolver / internal healthcheck host, create
   `exports.sh` — see **Customization** above.
4. Restart the app from the Umbrel UI, then verify (next section).

## Verifying it works

```sh
G=$(sudo docker ps --format '{{.Names}}' | grep gluetun)

# gluetun connected + healthy + got a forwarded port:
sudo docker logs "$G" 2>&1 | grep -iE 'healthy|port forwarding|port forwarded'

# egress IP is ProtonVPN's — must NOT be your home IP (no leak):
sudo docker exec "$G" wget -qO- https://ipinfo.io/ip

# forwarded port (should equal qBittorrent → Options → Connection → listen port):
sudo docker exec "$G" cat /gluetun/forwarded_port

# kernel vs userspace WireGuard (should say "kernelspace"):
sudo docker logs "$G" 2>&1 | grep -i implementation

# qBittorrent connection status (should be "connected", not "firewalled"):
#   qBittorrent → bottom status bar, or the port-sync log:
sudo docker logs $(sudo docker ps --format '{{.Names}}' | grep qbit-port-sync) 2>&1 | tail
```

## Troubleshooting

- **App stuck on "updating" in Umbrel.** Almost always gluetun never became
  healthy — qBittorrent and qbit-port-sync `depends_on: gluetun (healthy)`, so
  they stay in `Created` and the app never converges. Check
  `sudo docker logs <gluetun>`; fix the VPN/DNS cause below, then the app
  finishes converging.
- **gluetun restart-loops on the startup healthcheck.** Either DNS can't resolve
  `HEALTH_TARGET_ADDRESSES`, or (if you set an internal `VPN_HEALTHCHECK_HOST`
  that resolves to a private IP) gluetun's DNS-rebinding protection refuses it.
  Use the public default, or set `VPN_HEALTHCHECK_HOST` (it gets auto-exempted) —
  and it must have a **valid public cert**, **not** be a raw IP.
- **qBittorrent shows "firewalled" / no downloads.** The forwarded port isn't
  synced: confirm the **localhost-bypass** is on (step 1.5) and qbit-port-sync is
  `Up`; check its log shows `listen_port=…` matching `/gluetun/forwarded_port`.
  If egress is zero too, the tunnel may be dead — gluetun should reconnect; a
  manual app restart re-syncs the port and re-announces.
- **Can't resolve trackers (DNS).** If you set `LAN_DNS`, make sure its subnet is
  in `VPN_OUTBOUND_SUBNETS`, otherwise the kill-switch blocks it.
- **Imports copy instead of hardlink (double disk usage).** Downloads and library
  must be on the same volume (they are, under `/data`); keep **Use Hardlinks** on
  and root folders under `/data/media`.
- **A scene release stalls at "Found archive file".** Unpackerr isn't configured —
  fill `unpackerr.conf` with the Sonarr/Radarr URLs + API keys (step 8).
- **A Cloudflare-protected indexer fails in Prowlarr.** Make sure the Byparr
  proxy exists and the indexer carries the `byparr` tag (step 2).
- **Bazarr provider errors / throttling.** Re-check provider credentials; some
  providers self-throttle after failed logins — wait it out or reset the provider
  rather than hammering its login.

## Updating, backup & recovery

- **Updating:** bump happens via the Umbrel store (the app shows an update when
  the store repo advances). `docker-compose.yml` and `umbrel-app.yml` are
  replaced from the store on update; your `config/`, `exports.sh` and the
  `config/gluetun/` secrets are **preserved**.
- **If an update hangs at "updating":** see *Troubleshooting* (a container that
  never turns healthy blocks convergence). Fix the cause, then re-run the
  update / restart the app from the UI.
- **Backup:** back up `~/umbrel/app-data/camarigor-stream/config/` (all app
  settings + databases), `exports.sh`, and `config/gluetun/` (your VPN secrets).
  The `data/` tree (media + torrents) is large — back it up separately if you
  want the actual files.

## Remote access

Through Tailscale (separate Umbrel app): use the tailnet IP in the Jellyfin
mobile/TV apps and in the browser for Jellyseerr.
