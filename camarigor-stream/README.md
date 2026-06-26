# Stream — Umbrel media stack

A complete, self-hosted media automation stack as a **single Umbrel app**:
request a movie or show in Jellyseerr, it gets found and downloaded
automatically (through a VPN), gets extracted if needed, subtitles are fetched,
and it shows up in Jellyfin — using **hardlinks**, so a file is never duplicated
on disk and keeps seeding while it sits in your library.

> **The whole acquisition stack runs behind the VPN (gluetun).** qBittorrent,
> Sonarr, Radarr, Prowlarr, Jellyseerr, Bazarr, Byparr, cross-seed and Threadfin
> all share gluetun's network namespace, so they egress through the WireGuard
> tunnel and have **no hostname of their own** — internally they reach each other
> over **`localhost`**, and their WebUIs are published by gluetun on the host (so
> the LAN and a reverse proxy reach them at `<host>:<port>` unchanged). Only
> **Jellyfin** and **Unpackerr** stay on Umbrel's bridge.

## Requirements

- **Umbrel** (umbrelOS) on an x86_64 host.
- **Intel iGPU** (e.g. an N100/N150 box) for Jellyfin hardware transcoding —
  exposes `/dev/dri`, uses the host `render` group (GID `991` in the compose;
  adjust if your host differs).
- **A paid VPN** with **WireGuard** and **port forwarding** support.
  qBittorrent will not start without working VPN credentials.
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
| **Sonarr** | 8989 | `linuxserver/sonarr:4.0.18.2971-ls315` | TV automation — via VPN |
| **Radarr** | 7878 | `linuxserver/radarr:6.2.1.10461-ls305` | Movie automation — via VPN |
| **Bazarr** | 6767 | `linuxserver/bazarr:v1.5.6-ls350` | Subtitles (pt-BR by default) — via VPN |
| **Prowlarr** | 9696 | `linuxserver/prowlarr:2.4.0.5397-ls149` | Indexer manager (syncs to Sonarr/Radarr) — via VPN |
| **qBittorrent** | 8082 | `linuxserver/qbittorrent:5.2.2_v2.0.13-ls462` | Download client — via VPN |
| **Threadfin** | 34400 | `fyb3roptik/threadfin` (digest-pinned) | Optional IPTV M3U/EPG proxy in front of Jellyfin Live TV — via VPN |
| **gluetun** | internal | `qmcgaw/gluetun` (digest-pinned) | VPN gateway for the whole acquisition stack (WireGuard + kill-switch + static port forwarding) |
| **Byparr** | internal `8191` | `thephaseless/byparr` (digest-pinned) | Cloudflare solver, FlareSolverr-compatible API — via VPN |
| **cross-seed** | internal `2468` | `cross-seed/cross-seed:6.13.7` | Re-seeds finished downloads on other trackers (hardlink) — via VPN |
| **Unpackerr** | internal | `golift/unpackerr:0.15.2` | Auto-extracts RAR/scene releases for the *arrs — on the bridge |
| **init** | — | `busybox:stable` | One-shot: creates the config/data directory tree and fixes ownership before anything starts |

Images are **pinned** (by version, or by digest for projects without semver tags
like gluetun and Byparr) for reproducible deploys; updates are deliberate via the
store. The UIs live at `http://<umbrel-ip>:<port>`; internal services have no
host UI.

## How it works

```
Jellyseerr (you request something)
  └─> Sonarr / Radarr  (find a release via Prowlarr's indexers)
        └─> qBittorrent (download — all traffic through the VPN)
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

- **Two network zones.** The whole acquisition stack — qBittorrent, Sonarr,
  Radarr, Prowlarr, Jellyseerr, Bazarr, Byparr, cross-seed and Threadfin — shares
  **gluetun's** network namespace, so those services have **no hostname of their
  own** and reach each other over **`localhost`** (e.g. download client
  `localhost:8082`, Byparr `localhost:8191`, Prowlarr `localhost:9696`,
  cross-seed webhook `localhost:2468`). Their WebUIs are **published by gluetun**
  on the host, so the LAN and a reverse proxy reach them at `<host>:<port>`
  unchanged. Only **Jellyfin** and **Unpackerr** stay on Umbrel's bridge: they
  reach the in-netns services at **`gluetun:<port>`** (e.g. Unpackerr →
  `gluetun:8989`), and Jellyseerr reaches Jellyfin back at the **host LAN IP**.
- **Kill-switch:** gluetun blocks all non-VPN egress. If the tunnel drops, **every
  service in its namespace** loses internet — they never leak the real IP. The
  BitTorrent listen port is **not** published on the host; peers arrive via the
  VPN's static forwarded port through the tunnel.
- **Hardening:** every service runs with `no-new-privileges`; the LinuxServer
  apps run as UID/GID `1000`; Jellyfin, Jellyseerr, cross-seed and Unpackerr run
  as non-root `1000:1000` at the Docker level. gluetun is the only service with
  an added capability (`NET_ADMIN`, required to configure the tunnel and
  iptables), so it intentionally omits `no-new-privileges`.
- **Secrets stay out of the repo:** the VPN WireGuard key/address live only
  in `config/gluetun/` on the host (see step 9). Any host-specific values (LAN
  DNS, internal hostnames) live in `exports.sh` (see *Customization*).
- **Jellyfin** uses the **official** `jellyfin/jellyfin` image (not LinuxServer):
  it ships the Intel OpenCL runtime needed for HDR→SDR tone mapping
  (`tonemap_opencl`), which the LSIO image lacks. It runs as `1000:1000` with
  `group_add: 991` (host `render`) and `/dev/dri` for Quick Sync, and its
  `JELLYFIN_*_DIR` env vars keep the existing on-disk config layout.

## VPN — gluetun

**All** acquisition-stack traffic egresses through a VPN WireGuard tunnel —
qBittorrent plus Sonarr, Radarr, Prowlarr, Jellyseerr, Bazarr, Byparr, cross-seed
and Threadfin, which all share gluetun's network namespace.

- **Protocol:** WireGuard (`VPN_TYPE=wireguard`, provider `${VPN_PROVIDER}` set in
  exports.sh), using the host's **kernel** WireGuard module when available.
- **Servers:** `SERVER_COUNTRIES=Brazil` by default; the WireGuard entry port is
  `${VPN_ENDPOINT_PORT}` (a non-standard port helps dodge ISP VPN throttling).
- **Port forwarding (STATIC):** reserve a port in your provider's panel and set it
  as `VPN_FORWARDED_PORT` in exports.sh; gluetun opens it on the tunnel via
  `FIREWALL_VPN_INPUT_PORTS`, and qBittorrent's `listen_port` is fixed to that same
  value in its own config (so it stays connectable — seeding/ratio/H&R). The port
  is static, so there is nothing dynamic to sync (no sidecar). gluetun's NAT-PMP
  is OFF.
- **WebUIs:** each in-netns service publishes its WebUI through gluetun
  (`FIREWALL_INPUT_PORTS=8082,5055,8989,7878,6767,9696,34400`), so the LAN and a
  reverse proxy reach them at `<host>:<port>` exactly as before.
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
| `VPN_OUTBOUND_SUBNETS` | `10.21.0.0/16` | Subnets gluetun may reach off-tunnel. Add your LAN if `LAN_DNS` is on it. **Never** use broad ranges like `10.0.0.0/8` — they can overlap the VPN's tunnel gateway and route tunnel traffic off-tunnel. |
| `VPN_PROVIDER` | — | gluetun VPN provider id (matches your WireGuard config). Kept out of the public compose. |
| `VPN_ENDPOINT_PORT` | gluetun default | WireGuard entry/endpoint port; a non-standard one can dodge ISP VPN throttling. |
| `VPN_FORWARDED_PORT` | (none) | Static port reserved in your provider's panel; opened on the tunnel for incoming peers (seeding). |

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
5. Options → Connection → set **"Port used for incoming connections"** to your
   reserved `VPN_FORWARDED_PORT` and turn **UPnP/NAT-PMP OFF** (the VPN forwards
   exactly that static port through the tunnel; it never changes).

### 2. Prowlarr (`:9696`)

1. Create user/password on first access (Authentication: Forms).
2. Settings → Indexers → Add Indexer Proxy → **FlareSolverr** (Byparr is
   API-compatible, so the proxy type stays "FlareSolverr"): Host
   `http://localhost:8191`, Tag `byparr`.
3. Indexers → Add: register your public indexers (apply the `byparr` tag on
   Cloudflare-protected ones) and private trackers (tracker API key).
4. Settings → Apps → Add:
   - **Sonarr**: Prowlarr Server `http://localhost:9696`, Sonarr Server
     `http://localhost:8989`, API Key (Sonarr → Settings → General → API Key).
   - **Radarr**: same with `http://localhost:7878`.
5. After saving, indexers show up automatically in Sonarr/Radarr.

### 3. Sonarr (`:8989`) and Radarr (`:7878`)

1. Create user/password (Forms).
2. Settings → Media Management → Add Root Folder:
   - Sonarr: `/data/media/tv`
   - Radarr: `/data/media/movies`
3. Settings → Download Clients → Add → qBittorrent:
   - Host `localhost` (Sonarr/Radarr share qBit's netns), Port `8082`,
     username/password from step 1, Category `tv` (Sonarr) / `movies` (Radarr).
4. Confirm Settings → Media Management → Importing: **Use Hardlinks instead of
   Copy** = ON (default).

### 4. Bazarr (`:6767`)

1. Settings → Languages: Language Filter `Portuguese (Brazil)`; create a
   Language Profile `pt-BR` (Subtitle = Portuguese (Brazil)) and set it as
   Default for Series and Movies.
2. Settings → Providers: add OpenSubtitles.com (free account) and/or others.
3. Settings → Sonarr: Host `localhost`, Port `8989`, Sonarr API Key.
4. Settings → Radarr: Host `localhost`, Port `7878`, Radarr API Key.

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

1. Sign in with Jellyfin: URL `http://<umbrel-host-ip>:8096` (Jellyfin stays on
   the bridge — use the host's LAN IP, not a container name), Jellyfin admin.
2. Settings → Radarr: Add Server — Hostname `localhost`, Port `7878`, API Key,
   Quality Profile and Root Folder `/data/media/movies`, set Default.
3. Settings → Sonarr: same — `localhost`/`8989`, Root `/data/media/tv`.

### 7. cross-seed (no UI)

1. The first start generates
   `~/umbrel/app-data/camarigor-stream/config/cross-seed/config.js` and the
   container restart-loops until it is filled in (expected). Edit **only these
   three fields** — the v6 defaults (`useClientTorrents: true`,
   `linkType: "hardlink"`, `matchMode: "flexible"`, `action: "inject"`,
   `port: 2468`) are already correct:

   ```js
   torrentClients: ["qbittorrent:http://USER:PASSWORD@localhost:8082"],

   torznab: [
     // Prowlarr → Indexers → copy each indexer's Torznab feed URL
     // "http://localhost:9696/1/api?apikey=YOUR_PROWLARR_API_KEY",
   ],

   linkDirs: ["/data/torrents/cross-seed"],
   ```

2. qBittorrent → Options → Downloads → Run external program on torrent finished:
   `curl -s -XPOST http://localhost:2468/api/webhook --data-urlencode "infoHash=%I"`
3. Restart the app from the Umbrel UI.
4. Logs: `sudo docker logs -f $(sudo docker ps --format '{{.Names}}' | grep cross-seed)`.

### 8. Unpackerr (no UI)

Auto-extracts RAR/scene releases (e.g. BluRay packs that arrive as `.r00`-`.rNN`)
so Sonarr/Radarr can import them instead of stalling on "Found archive file".

1. The first start writes
   `~/umbrel/app-data/camarigor-stream/config/unpackerr/unpackerr.conf`. Under
   the `[[sonarr]]` and `[[radarr]]` blocks, uncomment and set `url`, `api_key`
   and `paths`. Unpackerr runs on the **bridge** (not in the VPN netns), so it
   reaches the *arrs at **`gluetun:<port>`**, not `localhost`:

   ```toml
   [[sonarr]]
     url = "http://gluetun:8989"
     api_key = "YOUR_SONARR_API_KEY"
     paths = ['/data/torrents']

   [[radarr]]
     url = "http://gluetun:7878"
     api_key = "YOUR_RADARR_API_KEY"
     paths = ['/data/torrents']
   ```

   API keys: Sonarr/Radarr → Settings → General → API Key.
2. Restart the app from the Umbrel UI.
3. Logs: `sudo docker logs -f $(sudo docker ps --format '{{.Names}}' | grep unpackerr)`.

### 9. VPN — gluetun (no UI)

gluetun restart-loops until the key files below exist (expected).

1. Your VPN provider's panel: generate a **WireGuard** config and note the
   `PrivateKey`, `Address` (e.g. `10.x.x.x/32`) and `PresharedKey` (if the provider
   uses one). Reserve a forwarded port if your provider uses static port forwarding.
2. Set the provider + ports in `~/umbrel/app-data/camarigor-stream/exports.sh`, and
   drop the secrets into `config/gluetun/` (all stay out of the store repo):

   ```sh
   # exports.sh
   export VPN_PROVIDER="your-gluetun-provider-id"
   export VPN_ENDPOINT_PORT=""          # optional: WireGuard entry port
   export VPN_FORWARDED_PORT=""         # the port you reserved (static PF)
   # secrets (config/gluetun/)
   printf '%s' 'YOUR_WIREGUARD_PRIVATE_KEY' > config/gluetun/wireguard_private_key
   printf '%s' '10.x.x.x/32'                > config/gluetun/wireguard_addresses
   printf '%s' 'YOUR_PRESHARED_KEY'         > config/gluetun/wireguard_preshared_key
   ```

3. (Optional) For a custom LAN DNS resolver / internal healthcheck host, create
   `exports.sh` — see **Customization** above.
4. Restart the app from the Umbrel UI, then verify (next section).

## Verifying it works

```sh
G=$(sudo docker ps --format '{{.Names}}' | grep gluetun)

# gluetun connected + healthy:
sudo docker logs "$G" 2>&1 | grep -iE 'healthy|wireguard|connected'

# egress IP is the VPN's — must NOT be your home IP (no leak):
sudo docker exec "$G" wget -qO- https://ipinfo.io/ip

# kernel vs userspace WireGuard (should say "kernelspace"):
sudo docker logs "$G" 2>&1 | grep -i implementation

# every in-netns WebUI answers on the host (the LAN / reverse-proxy path):
for p in 8082 5055 8989 7878 6767 9696 34400; do \
  printf '%s ' "$p"; curl -fsS -o /dev/null -w '%{http_code}\n' "http://localhost:$p" || echo DOWN; done

# qBittorrent: listen port must equal your reserved VPN_FORWARDED_PORT, and the
# status must be "connected" (not "firewalled") — that confirms the static
# forwarded port works. Check qBittorrent → Options → Connection + the status bar.
```

## Troubleshooting

- **App stuck on "updating" in Umbrel.** Almost always gluetun never became
  healthy — every acquisition service `depends_on: gluetun (healthy)`, so
  they stay in `Created` and the app never converges. Check
  `sudo docker logs <gluetun>`; fix the VPN/DNS cause below, then the app
  finishes converging.
- **gluetun restart-loops on the startup healthcheck.** Either DNS can't resolve
  `HEALTH_TARGET_ADDRESSES`, or (if you set an internal `VPN_HEALTHCHECK_HOST`
  that resolves to a private IP) gluetun's DNS-rebinding protection refuses it.
  Use the public default, or set `VPN_HEALTHCHECK_HOST` (it gets auto-exempted) —
  and it must have a **valid public cert**, **not** be a raw IP.
- **qBittorrent shows "firewalled" / no downloads.** Confirm qBittorrent's listen
  port equals your reserved `VPN_FORWARDED_PORT` (step 1.5) and that the same value
  is set as `VPN_FORWARDED_PORT` in exports.sh (gluetun opens it on the tunnel via
  `FIREWALL_VPN_INPUT_PORTS`). If egress is zero too, the tunnel may be dead —
  gluetun should reconnect; a manual app restart re-announces.
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
