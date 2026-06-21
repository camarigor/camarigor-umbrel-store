# Stream — Umbrel media stack

Single app: Jellyfin + Jellyseerr + Sonarr + Radarr + Bazarr + Prowlarr +
qBittorrent + Byparr + cross-seed + Unpackerr + gluetun (ProtonVPN).

> qBittorrent runs behind ProtonVPN (gluetun). Its WebUI is still on host
> port `8082` (published by gluetun); inside the stack reach it at
> `gluetun:8082`, not `qbittorrent:8082`.

## UIs (host `umbrel`)

| App | URL | Role |
|---|---|---|
| Jellyfin | `http://<umbrel-ip>:8096` | Watch (the app icon on Umbrel opens this) |
| Jellyseerr | `http://<umbrel-ip>:5055` | Request content |
| Sonarr | `http://<umbrel-ip>:8989` | TV shows config |
| Radarr | `http://<umbrel-ip>:7878` | Movies config |
| Bazarr | `http://<umbrel-ip>:6767` | Subtitles config |
| Prowlarr | `http://<umbrel-ip>:9696` | Indexers config |
| qBittorrent | `http://<umbrel-ip>:8082` | Torrent client |

Byparr and cross-seed have no exposed UI (internal only).

## Post-install setup (order matters)

### 1. qBittorrent (`:8082`)

1. Temporary password: `sudo docker logs $(sudo docker ps --format '{{.Names}}' | grep qbittorrent) 2>&1 | grep -i 'temporary password'`
2. Log in as `admin` + temporary password → Tools → Options → WebUI →
   change username/password.
3. Options → Downloads → Default Save Path: `/data/torrents`.
4. Create categories (right-click Categories in the sidebar):
   - `movies` → Save Path `/data/torrents/movies`
   - `tv` → Save Path `/data/torrents/tv`

### 2. Prowlarr (`:9696`)

1. Create user/password on first access (Authentication: Forms).
2. Settings → Indexers → Add Indexer Proxy → **FlareSolverr** (Byparr is
   API-compatible, so the proxy type stays "FlareSolverr"): Host
   `http://byparr:8191`, Tag `byparr`.
3. Indexers → Add: register your public indexers (apply the
   `byparr` tag on Cloudflare-protected ones) and private trackers
   (tracker API key).
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
4. Confirm in Settings → Media Management → Importing: **Use Hardlinks
   instead of Copy** = ON (default).

### 4. Bazarr (`:6767`)

1. Settings → Languages: Language Filter `Portuguese (Brazil)`; create a
   Language Profile `pt-BR` (Subtitle = Portuguese (Brazil)) and set it as
   Default for Series and Movies.
2. Settings → Providers: add OpenSubtitles.com (free account) and/or
   others.
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
4. Validate: play a video forcing a lower quality (player gear icon) and
   check Dashboard → Active Devices shows `(hw)` on the transcode.

### 6. Jellyseerr (`:5055`)

1. Sign in with Jellyfin: URL `http://jellyfin:8096`, Jellyfin admin.
2. Settings → Radarr: Add Server — Hostname `radarr`, Port `7878`, API
   Key, Quality Profile and Root Folder `/data/media/movies`, set Default.
3. Settings → Sonarr: same — `sonarr`/`8989`, Root `/data/media/tv`.

### 7. cross-seed (no UI)

1. The first start generates
   `~/umbrel/app-data/camarigor-stream/config/cross-seed/config.js` and
   the container restart-loops until it is filled in (expected). Edit
   **only these three fields** in the generated file — the remaining
   defaults (v6: `useClientTorrents: true`, `linkType: "hardlink"`,
   `matchMode: "flexible"`, `action: "inject"`, `port: 2468`) are already
   correct:

   ```js
   torrentClients: ["qbittorrent:http://USER:PASSWORD@gluetun:8082"],

   torznab: [
     // Prowlarr → Indexers → copy the Torznab feed URL of each indexer
     // "http://prowlarr:9696/1/api?apikey=YOUR_PROWLARR_API_KEY",
   ],

   linkDirs: ["/data/torrents/cross-seed"],
   ```

2. qBittorrent → Options → Downloads → Run external program on torrent
   finished:
   `curl -s -XPOST http://cross-seed:2468/api/webhook --data-urlencode "infoHash=%I"`
3. Restart the app from the Umbrel UI.
4. Logs: `sudo docker logs -f $(sudo docker ps --format '{{.Names}}' | grep cross-seed)`.

### 8. Unpackerr (no UI)

Auto-extracts RAR/scene releases (e.g. iNVANDRAREN BluRay packs that arrive
as `.r00`-`.rNN`) so Sonarr/Radarr can import them instead of stalling on
"Found archive file".

1. The first start writes
   `~/umbrel/app-data/camarigor-stream/config/unpackerr/unpackerr.conf`.
   Edit it and, under the `[[sonarr]]` and `[[radarr]]` blocks, uncomment and
   set `url`, `api_key` and `paths`:

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

All qBittorrent traffic goes through ProtonVPN (WireGuard) with a kill-switch
and port forwarding, so torrents stay connectable without leaking your IP.
**Requires a ProtonVPN paid plan** (Plus/Unlimited — they include P2P + port
forwarding). gluetun restart-loops until the key files below exist (expected).

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

3. qBittorrent → Options → WebUI → enable **"Bypass authentication for
   clients on localhost"** (lets the qbit-port-sync sidecar set the forwarded
   port without credentials).
4. Restart the app from the Umbrel UI.
5. Verify:
   - `sudo docker logs -f $(sudo docker ps --format '{{.Names}}' | grep gluetun)`
     should show it connected to a Brazil server and a line like
     `port forwarding: opened port XXXXX`. If it reports no matching server,
     change `SERVER_COUNTRIES` in the compose (e.g. to `Netherlands`) or remove
     it to auto-pick any port-forwarding server.
   - Confirm the egress IP is ProtonVPN's:
     `sudo docker exec $(sudo docker ps --format '{{.Names}}' | grep gluetun) wget -qO- https://ipinfo.io/ip`
   - `qbit-port-sync` logs should show `listen_port=XXXXX` matching the
     forwarded port; qBittorrent → Options → Connection port should equal it.

> Notes: the BitTorrent port is NOT published on the host — incoming peers
> arrive through the VPN's forwarded port. If you ever stop using the VPN,
> revert qBittorrent to its own `network_mode`/ports and point Sonarr/Radarr/
> cross-seed back to `qbittorrent:8082`.

## Remote access

Through Tailscale (separate Umbrel app): use the tailnet IP in the
Jellyfin mobile/TV app and in the browser for Jellyseerr.
