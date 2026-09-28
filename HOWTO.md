# HOWTO: ARR stack on Docker Desktop (Windows 11 Home)

This sets up the "*arr" media automation stack on your Windows 11 Home PC using
Docker Desktop, with torrent traffic routed through a VPN.

## What's in the stack

| Service | Purpose | Web UI |
|---|---|---|
| Gluetun | VPN gateway — qBittorrent's traffic goes through this | n/a |
| qBittorrent | Torrent client | http://qbittorrent.arrhost.local |
| Prowlarr | Indexer manager — feeds indexers to Sonarr/Radarr | http://prowlarr.arrhost.local |
| Sonarr | TV show automation | http://sonarr.arrhost.local |
| Radarr | Movie automation | http://radarr.arrhost.local |
| Bazarr | Subtitle automation for Sonarr/Radarr libraries | http://bazarr.arrhost.local |
| Jellyfin | Media server — browse/stream what gets downloaded | http://jellyfin.arrhost.local |
| Caddy | Reverse proxy — routes each `*.arrhost.local` subdomain to the right container | n/a |

All containers use [LinuxServer.io](https://www.linuxserver.io/) images, which
is the de facto standard for this stack and has consistent config conventions
(`PUID`/`PGID`/`TZ` env vars, `/config` volume).

### Why `arrhost.local` and not `localhost`

A single hostname can't route to seven different containers by itself —
something has to look at the request and decide where it goes. That's what
the **Caddy** container does: it listens on port 80 and, based on which
subdomain the request came in on (`sonarr.arrhost.local` vs
`radarr.arrhost.local`, etc.), forwards it to the right container on its
internal port. Your browser talks to Caddy; Caddy talks to the app
containers by their Docker container names over the internal Docker
network. This is all defined in `Caddyfile`.

This only affects **browser-facing** URLs. Traffic *between* containers
(Prowlarr talking to Sonarr, Sonarr talking to qBittorrent, Bazarr talking
to Sonarr/Radarr) keeps using Docker's internal container-name DNS
(`http://sonarr:8989`, `http://gluetun:8080`, etc.) exactly as before — it
never goes through Caddy or `arrhost.local`, and you don't need to change
anything there.

## Folder layout

Two separate locations, on purpose — one for the stack definition and its
config (small, lives with the project), one for actual media data (large,
kept on the drive with room for it):

```
G:\coderepo\arr\        <- the project: docker-compose.yml, .env, Caddyfile, HOWTO.md, TODO.md
  config\                <- per-app settings/databases (small), referenced
    gluetun\               as ./config/... (relative) in docker-compose.yml
    qbittorrent\
    prowlarr\
    sonarr\
    radarr\
    bazarr\
    jellyfin\
    caddy\
      data\                <- Caddy's internal state
      config\               <- Caddy's internal state
D:\arr\                 <- bulk data, kept on D: for the free space
  downloads\              <- qBittorrent drops everything here first
  media\
    movies\                <- Radarr's organized library
    tv\                     <- Sonarr's organized library
```

Config lives inside the project folder on `G:` (small, doesn't need the
space, and travels with the repo); `downloads` and `media` live on `D:`
together, on purpose — Sonarr/Radarr move completed downloads into `media`
by renaming the file, which is instant on the same drive/filesystem. If
`downloads` and `media` ended up on different drives, those moves would
become slow copy+delete operations instead.

## Prerequisites

1. **Docker Desktop** installed and running, with the **WSL2 backend** enabled
   (Settings → General → "Use the WSL 2 based engine"). This is the default
   on a fresh install and is required — the older Hyper-V backend is not
   supported for this compose file.
2. A **VPN subscription** that supports WireGuard or OpenVPN and is
   [supported by Gluetun](https://github.com/qdm12/gluetun-wiki/tree/main/setup)
   (Mullvad, ProtonVPN, Private Internet Access, Surfshark, etc. all work).
3. Enough free disk space on the `D:` drive for downloads + media (this
   setup assumes `D:` has plenty free — 1TB+). No extra config needed to use
   a second local drive with the WSL2 backend — Docker Desktop shares all
   local drives by default, so `D:/...` bind mounts in `docker-compose.yml`
   work the same way `G:/...` ones do.

## Setup steps

### 1. Create the folder structure

In PowerShell:

```powershell
New-Item -ItemType Directory -Force -Path `
  "G:\coderepo\arr\config\gluetun", `
  "G:\coderepo\arr\config\qbittorrent", `
  "G:\coderepo\arr\config\prowlarr", `
  "G:\coderepo\arr\config\sonarr", `
  "G:\coderepo\arr\config\radarr", `
  "G:\coderepo\arr\config\bazarr", `
  "G:\coderepo\arr\config\jellyfin", `
  "G:\coderepo\arr\config\caddy\data", `
  "G:\coderepo\arr\config\caddy\config", `
  "D:\arr\downloads", `
  "D:\arr\media\movies", `
  "D:\arr\media\tv"
```

### 2. Configure environment variables

Copy `.env.example` to `.env` in `G:\coderepo\arr\` and fill in your VPN
provider's details (provider name, WireGuard key or OpenVPN
username/password). See the
[Gluetun wiki](https://github.com/qdm12/gluetun-wiki/tree/main/setup) for the
exact values your provider needs — they vary provider to provider.

**`.env` contains secrets — do not commit it if this repo is or becomes
public.** `.gitignore` already excludes `.env`, and also excludes the whole
`config/` folder, since that holds Sonarr/Radarr/Prowlarr API keys,
qBittorrent credentials, app databases, and Caddy's internal state — none of
which belongs in git either.

### 2a. CyberGhost only: use gluetun's "custom provider" mode

CyberGhost needs more than a username/password — gluetun also needs a
**client certificate and private key** unique to your account, or it fails
to start with `ERROR VPN settings: OpenVPN settings: client certificate:
missing value`. Beyond that, gluetun's *built-in* `cyberghost` provider
picks a server from its own hardcoded list (e.g.
`bangkok-rack401.nodes.gen4.ninja`) — which is **not necessarily the same
server infrastructure your account's cert/credentials are valid for**
(CyberGhost's manual-config downloads point at a different set of hosts,
e.g. `87-1-TH.cg-dialup.net`). Connecting to the wrong one is a clean,
silent `AUTH_FAILED`: the TLS/cert layer can succeed while the
username/password get rejected, because you've effectively reached a
different backend than the one that issued them.

The fix is to skip gluetun's built-in CyberGhost server list entirely and
hand it your actual CyberGhost-provided `.ovpn` file directly, so it
connects to the exact server your credentials are valid for:

1. Log into your [CyberGhost account dashboard](https://my.cyberghostvpn.com/)
   → **VPN** section → **Configure Device**.
2. Choose **OpenVPN** (UDP or TCP — UDP is the default gluetun expects
   unless you set `OPENVPN_PROTOCOL=tcp`), pick a country/server group
   matching `VPN_SERVER_COUNTRIES` in your `.env`, and save.
3. Back on the VPN tab, click **View** next to that device, then
   **Download Configuration** — this downloads a `.zip` with one or more
   `.ovpn` files. Each one already has the certificate and private key
   embedded inline (`<cert>...</cert>`, `<key>...</key>`) — no need to
   extract them separately.
4. Extract the zip and copy one `.ovpn` file to
   `G:\coderepo\arr\config\gluetun\custom.conf` (exact filename —
   this folder is already mounted into the gluetun container as
   `/gluetun`).
4a. **If your `custom.conf` has a line like `ca ca.crt`** instead of an
    inline `<ca>...</ca>` block (CyberGhost's download can go either way):
    this points at a separate CA certificate file that wasn't embedded.
    Two ways to fix it — inlining is simpler (one self-contained file, no
    path gotchas), so prefer it unless you have a reason not to:
    - **Inline it (preferred)**: open the `ca.crt` file from the extracted
      zip in a text editor, and in `custom.conf` replace the `ca ca.crt`
      line entirely with:
      ```
      <ca>
      -----BEGIN CERTIFICATE-----
      ...paste ca.crt's full contents here...
      -----END CERTIFICATE-----
      </ca>
      ```
      Same pattern as the existing `<cert>`/`<key>` blocks.
    - **Or keep it as a separate file**: copy `ca.crt` to
      `G:\coderepo\arr\config\gluetun\ca.crt`, and change the line to
      `ca /gluetun/ca.crt` — a relative path doesn't work here because
      gluetun reads, rewrites, and re-saves your config file elsewhere at
      runtime, so any file it references needs the absolute in-container
      path (`/gluetun/...`, matching where `./config/gluetun` is mounted).
      This applies to any other relative reference in the file too (e.g.
      `up up.sh`).
4b. **Replace the hostname in the `remote` line with an IP address.**
    gluetun's custom provider mode requires this — by design, it builds
    its firewall rules and routing *before* the VPN tunnel (and therefore
    DNS) is up, so it can't resolve a hostname like
    `remote 87-1-th.cg-dialup.net 443` at startup and fails with `host is
    not an IP address`. CyberGhost's own OpenVPN client resolves this
    automatically, which is why it isn't obvious from the download.
    - On Windows, resolve it yourself: `nslookup 87-1-th.cg-dialup.net`
      in PowerShell, or use any of the IPs it returns (it's usually a
      small pool of interchangeable servers behind that hostname — any
      one works).
    - Edit the `remote` line in `custom.conf` to use that IP instead,
      keeping the port: `remote 173.239.201.17 443` (using whichever IP
      you resolved — this project's current one, at time of writing, is
      `173.239.201.17`; CyberGhost can change these over time, so if the
      VPN stops connecting later, re-run `nslookup` and update this line).
5. In `.env`, set:
   - `VPN_SERVICE_PROVIDER=custom` (not `cyberghost` — this tells gluetun
     to use your `.ovpn` file's server instead of its own list)
   - `OPENVPN_USER`/`OPENVPN_PASSWORD` to the **generated** credentials
     from the same Configure Device page — **not** your
     my.cyberghostvpn.com account login. CyberGhost's own docs are
     explicit about this for their manual setup guides: "This isn't your
     regular CyberGhost account username"/"password". (Often also
     included in a `readme.txt` inside the downloaded zip.)
   - **Clear `VPN_SERVER_COUNTRIES` to empty** (`VPN_SERVER_COUNTRIES=`).
     Custom mode has no server list to filter by country — leaving a
     value there makes gluetun fail immediately with `the country
     specified is not valid: one or more values is set but there is no
     possible value available`. Country/server selection for custom mode
     happens by picking which server when you generated the `.ovpn` file
     in step 2, not through this variable.
6. `docker-compose.yml` already has `OPENVPN_CUSTOM_CONFIG=/gluetun/custom.conf`
   set for exactly this case — no compose changes needed.
7. Run `docker compose up -d --force-recreate gluetun`.

`custom.conf` (and any `client.crt`/`client.key` if you made them under
the earlier approach — safe to delete now that `custom.conf` supersedes
them) are credentials — they're already covered by the `config/` entry in
`.gitignore`, so they won't get committed.

### 3. Point the `arrhost.local` subdomains at this PC

Windows resolves hostnames via `C:\Windows\System32\drivers\etc\hosts` before
it ever asks a DNS server. Edit that file as Administrator (e.g.
`notepad C:\Windows\System32\drivers\etc\hosts` run as admin) and add one
line per subdomain, pointed at this PC's **LAN IP** (e.g. `192.168.1.50`),
not `127.0.0.1`:

```
192.168.1.50 sonarr.arrhost.local
192.168.1.50 radarr.arrhost.local
192.168.1.50 prowlarr.arrhost.local
192.168.1.50 bazarr.arrhost.local
192.168.1.50 jellyfin.arrhost.local
192.168.1.50 qbittorrent.arrhost.local
```

Using the LAN IP instead of `127.0.0.1` means any device on your home
network can resolve `*.arrhost.local` to this PC — not just this PC itself
— as long as that device also has these hosts entries (or its own DNS
pointed at something that resolves them; there's no central DNS server
doing this automatically). If this PC's LAN IP ever changes (e.g. no DHCP
reservation set on your router), these entries — on this PC and any others
— need updating to match.

Windows hosts files don't support wildcards, so each subdomain needs its
own line — if you add another app later, add another line here and a
matching block in `Caddyfile`.

### 4. Start the stack

From `G:\coderepo\arr\` in PowerShell:

```powershell
docker compose up -d
```

Check everything is healthy:

```powershell
docker compose ps
docker compose logs -f gluetun
```

`gluetun` logs should show a successful VPN connection before qBittorrent
will be reachable — qBittorrent shares gluetun's network, so if the VPN is
down, qBittorrent's web UI won't load either.

### 5. Verify the VPN is actually protecting qBittorrent

With the stack up, check the apparent public IP from *inside* the gluetun
container and compare it to your real IP:

```powershell
docker exec gluetun wget -qO- https://am.i.mullvad.net/ip
```

(or any "what's my IP" endpoint) — it should show your VPN exit IP, not your
home IP. Do this once after every setup change to gluetun.

### 6. Configure Prowlarr (indexers)

1. Open http://prowlarr.arrhost.local
2. Add your indexers (Settings → Indexers)
3. Add Sonarr and Radarr as "Apps" (Settings → Apps) so Prowlarr can push
   indexers to them automatically — you'll need each app's URL
   (`http://sonarr:8989`, `http://radarr:7878` — container names work as
   hostnames on the shared Docker network) and API key (found in each app's
   Settings → General).

### 7. Configure Sonarr / Radarr

1. Open http://sonarr.arrhost.local (Sonarr) and http://radarr.arrhost.local (Radarr)
2. Settings → Download Clients → add qBittorrent
   - Host: `gluetun` (qBittorrent shares gluetun's network namespace, so you
     reach it via gluetun's container name)
   - Port: `8080`
3. Settings → Media Management → confirm root folders point at `/tv` (Sonarr)
   or `/movies` (Radarr) — these map to `D:\arr\media\tv` and
   `D:\arr\media\movies` per the compose file.
4. Indexers should already be populated by Prowlarr if step 5 worked.

### 8. Configure Bazarr

1. Open http://bazarr.arrhost.local
2. Settings → Sonarr / Radarr → point at `http://sonarr:8989` /
   `http://radarr:7878` with their API keys.
3. Settings → Languages → pick the subtitle languages you want.

### 9. Configure Jellyfin

1. Open http://jellyfin.arrhost.local and run through the first-time setup wizard.
2. Add a library pointing at `/data/movies` and another at `/data/tvshows`
   (these map to the same `media` folders Radarr/Sonarr populate).

## Updating the stack

```powershell
docker compose pull
docker compose up -d
```

## Troubleshooting

- **`docker compose up` fails with "port is already allocated" (e.g. for
  `0.0.0.0:8080`)**: some other process — Docker container or not — already
  has that port bound on Windows. Check `docker ps -a` for a container from
  another project publishing that port (on this PC, port 8080 is taken by
  the `wordpress-fleet` project's `k3d-wp-fleet-dev-serverlb` container) and
  either stop it or, better, remap the host side of the conflicting service
  in `docker-compose.yml` — that's exactly why qBittorrent's WebUI is
  mapped to `9090:8080` here instead of `8080:8080`. Remapping the host
  side never affects `*.arrhost.local` access, since Caddy talks to
  containers over the internal Docker network, not the published host
  port. If nothing in Docker is using the port, find the process with
  `netstat -ano | findstr :8080` (the last column is the PID), then
  `tasklist /FI "PID eq <pid>"` to identify it.
- **gluetun exits with `ERROR VPN settings: OpenVPN settings: client
  certificate: missing value`**: your provider needs a client
  certificate/key, not just a username and password — see step 2a
  (CyberGhost) above. Check `docker compose logs gluetun` after adding the
  files; a `[vpn] connection to ... succeeded` line means it's fixed.
- **gluetun's OpenVPN handshake connects but then fails with
  `AUTH_FAILED` / "Your credentials might be wrong"**: this means the
  certificate/key are fine (the connection got that far), but
  `OPENVPN_USER`/`OPENVPN_PASSWORD` in `.env` are wrong. Being able to log
  into my.cyberghostvpn.com with those credentials does **not** confirm
  they're right for OpenVPN — CyberGhost issues a separate,
  generated username/password specifically for manual OpenVPN/router
  setups, shown on the same **Configure Device** page (see step 2a). Go
  back to that page, copy the generated credentials (not your account
  login), update `.env`, and run `docker compose up -d` again.

  **Server-mismatch theory ruled out on this project.** We initially
  suspected gluetun's built-in `cyberghost` provider was connecting to
  different infrastructure than the account's cert/credentials were valid
  for (`bangkok-rack401.nodes.gen4.ninja` vs `87-1-TH.cg-dialup.net`), and
  switched to gluetun's **custom provider mode** (step 2a) with the exact
  IP the `.ovpn` file specifies to rule that out. It didn't fix it — and
  the logs proved why: the server at that IP identifies itself in the TLS
  handshake as `bangkok-rack401.nodes.gen4.ninja`, meaning both routes
  reach the *same* physical backend. So this was never a routing problem.

  With routing eliminated, `.env`-parsing double-checked (stray
  whitespace; quote the password if it contains `#`, `$`, `"`, or `\` —
  `#` truncates the rest of the line as a comment, `$` needs escaping as
  `$$`; then `docker compose up -d --force-recreate gluetun` to force new
  values to load), and the cert/key/CA/IP all confirmed correct, the
  decisive next step was to stop debugging this from logs alone and test
  the *exact same* `.ovpn` file and generated credentials with a
  standalone client, completely outside gluetun:
  1. Install [OpenVPN Connect for Windows](https://openvpn.net/client/).
  2. Import the same `.ovpn`/`custom.conf` file (File → Import → OpenVPN
     Profile) — the version with your edits (inline `<ca>`, IP-resolved
     `remote` line) works fine for this, OpenVPN Connect doesn't care
     about the extra gluetun-specific requirements.
  3. Connect using the same generated username/password.

  **Result: this also failed**, with the same auth error, in a completely
  independent client with no gluetun involved at all. That makes it
  conclusive: this is a **CyberGhost-account problem, not gluetun or
  config**. Every config-level cause has now been checked and ruled
  out — client cert, key, CA, IP resolution, custom vs built-in provider
  mode, `.env` parsing — and the one thing both failures share is the
  account/credentials themselves.

  **Next steps, in order:**
  1. Go to the CyberGhost dashboard (VPN → Configure Device) and **delete
     the existing device entry entirely**, then create a brand-new one.
     This issues a fresh cert/key/credential set together, which rules
     out any mismatch between pieces that were generated separately.
  2. Re-download the new config, redo the inline-`<ca>` and
     IP-resolution edits into `custom.conf`, and update `.env` with the
     new generated username/password.
  3. Test the new config in OpenVPN Connect *first* (same steps as
     above) before touching gluetun — cheaper to iterate on, and rules
     out gluetun-specific issues at each step.
  4. If the fresh device entry **also** fails in OpenVPN Connect: contact
     CyberGhost support directly. This points to an account-side block
     (device limit, security hold, etc.) that no client-side config
     change can work around — a config problem cannot survive a
     from-scratch cert/key/credential reissue.
  5. If the fresh device entry **succeeds** in OpenVPN Connect: update
     `docker-compose.yml`'s gluetun config with the same fresh
     `custom.conf`/`.env` values and retry
     `docker compose up -d --force-recreate gluetun`. If gluetun *still*
     fails while OpenVPN Connect succeeds with the identical fresh
     config, that's a genuine gluetun-specific incompatibility — worth a
     fresh issue on [gluetun's GitHub](https://github.com/qdm12/gluetun/issues)
     with the working `.ovpn` attached, or falling back to **WireGuard**
     instead of OpenVPN for gluetun if CyberGhost issues you a WireGuard
     config (would need `docker-compose.yml`/`.env` changes beyond what's
     documented here).
- **`*.arrhost.local` doesn't resolve / browser says it can't find the
  server**: check the hosts file entries from step 3 were saved (Notepad
  run as Administrator, or it silently fails to save), and flush DNS with
  `ipconfig /flushdns` in PowerShell.
- **`*.arrhost.local` resolves but shows a Caddy error or wrong app**: check
  `docker compose logs caddy` and confirm `Caddyfile` matches the container
  names in `docker-compose.yml`.
- **qBittorrent web UI won't load** (via `qbittorrent.arrhost.local` or
  otherwise): check `docker compose logs -f gluetun` — it almost always
  means the VPN tunnel isn't up.
- **Sonarr/Radarr can't reach qBittorrent**: make sure you used `gluetun` as
  the host, not `qbittorrent` — the container has no network of its own.
- **Downloads move slowly into the library**: confirm `downloads` and
  `media` are both under `D:\arr\` on the same drive.
- **Docker Desktop won't start / WSL errors**: run `wsl --update` in
  PowerShell (as admin), then restart Docker Desktop.
