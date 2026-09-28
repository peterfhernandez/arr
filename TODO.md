# TODO: ARR stack setup

Actionable checklist. See `HOWTO.md` for the full explanation of each step.

## Prep

- [x] Confirm Docker Desktop is running with the WSL2 backend enabled
      (Settings → General → "Use the WSL 2 based engine")
- [x] Confirm `D:` drive has enough free space for downloads + media (1TB+
      free)
- [x] Pick a VPN provider that supports WireGuard or OpenVPN and check it's
      on [Gluetun's supported list](https://github.com/qdm12/gluetun-wiki/tree/main/setup)
- [x] Get VPN credentials from that provider — OpenVPN username/password set
      in `.env`

## Folder structure

- [x] Create `G:\coderepo\arr\config\gluetun`
- [x] Create `G:\coderepo\arr\config\qbittorrent`
- [x] Create `G:\coderepo\arr\config\prowlarr`
- [x] Create `G:\coderepo\arr\config\sonarr`
- [x] Create `G:\coderepo\arr\config\radarr`
- [x] Create `G:\coderepo\arr\config\bazarr`
- [x] Create `G:\coderepo\arr\config\jellyfin`
- [x] Create `G:\coderepo\arr\config\caddy\data`
- [x] Create `G:\coderepo\arr\config\caddy\config`
- [x] Create `D:\arr\downloads`
- [x] Create `D:\arr\media\movies`
- [x] Create `D:\arr\media\tv`
      (PowerShell one-liner for all of the above is in `HOWTO.md` step 1)

## Configuration files

- [x] Copy `.env.example` to `.env` in `G:\coderepo\arr\`
- [x] Fill in `VPN_SERVICE_PROVIDER`, `VPN_TYPE=openvpn`, and
      `OPENVPN_USER`/`OPENVPN_PASSWORD` in `.env`
- [x] Add `.env` and `config/` to `.gitignore` (VPN credentials, API keys,
      app databases, Caddy state)
- [x] Confirm `Caddyfile` is present in `G:\coderepo\arr\` (routes
      `*.arrhost.local` to the right container) — verified: all six
      subdomains route to the correct container

## CyberGhost custom provider setup (blocking gluetun startup)

Server-mismatch theory ruled out: connecting via custom mode to the exact
IP the `.ovpn` file specifies still shows a TLS identity of
`bangkok-rack401.nodes.gen4.ninja` — same backend gluetun's built-in list
was already reaching. Routing was never the problem.

**Conclusively a CyberGhost-account problem, not gluetun or config.** The
decisive test came back negative: the exact same `custom.conf`
(inline `<ca>`, IP-resolved `remote` line) and the exact same generated
credentials, imported into OpenVPN Connect — a completely independent
client, no gluetun involved — also fail with an auth error. Every
config-level cause has now been checked and ruled out: client cert, key,
CA, IP resolution, custom vs built-in provider mode, `.env` parsing. The
only thing left in common between the two failures is the CyberGhost
account/credentials themselves. Next step is on CyberGhost's side — see
below.

- [ ] Log into CyberGhost dashboard → VPN → Configure Device → OpenVPN,
      download the config zip
- [x] ~~Extract `<cert>`/`<key>` into client.crt/client.key~~ — superseded,
      no longer needed (the `.ovpn` file's inline cert/key are used
      directly as `custom.conf` instead)
- [x] Copy the downloaded `.ovpn` file to
      `G:\coderepo\arr\config\gluetun\custom.conf` (exact filename)
- [x] `custom.conf` has `ca ca.crt` (external reference, not inline
      `<ca>`) — decided to inline it (simpler, one self-contained file)
- [x] Open `ca.crt` from the extracted zip, and in `custom.conf` replace
      the `ca ca.crt` line with a `<ca>...</ca>` block containing its
      full contents (same pattern as the existing `<cert>`/`<key>`
      blocks)
- [x] **Replace the hostname in the `remote` line with an IP address** —
      gluetun's custom mode can't resolve DNS at startup (tunnel/DNS
      isn't up yet), so `remote 87-1-th.cg-dialup.net 443` fails with
      `host is not an IP address`. Used `remote 173.239.201.17 443`
- [x] In `.env`, change `VPN_SERVICE_PROVIDER` from `cyberghost` to
      `custom`
- [x] Confirm `OPENVPN_USER`/`OPENVPN_PASSWORD` in `.env` are the
      **generated** credentials from the Configure Device page (not your
      my.cyberghostvpn.com account login)
- [x] Clear `VPN_SERVER_COUNTRIES=` (empty) in `.env` — custom mode
      errors at startup ("the country specified is not valid") if this is
      still set to `thailand` or anything else
- [x] Run `docker compose up -d --force-recreate gluetun` — connects to
      the correct IP now, but still `AUTH_FAILED`
- [x] Checked `.env` for stray whitespace/special characters — values
      confirmed `User: [set]` / `Password: [set]` in logs, custom mode
      reaches the right server (confirmed via TLS identity in logs);
      `.env` parsing and routing both ruled out
- [x] **Decisive test**: install [OpenVPN Connect](https://openvpn.net/client/),
      import the current `custom.conf` (with inline `<ca>` and the
      IP-resolved `remote` line), and try connecting with the same
      generated credentials — **failed with the same auth error outside
      gluetun entirely**, confirming this is a CyberGhost-account issue
- [ ] **Delete the existing Configure Device entry on CyberGhost's
      dashboard entirely** (VPN → Configure Device), then create a
      brand-new one — this issues a fresh cert/key/credential set
      together, ruling out any mismatch between them
- [ ] Update `custom.conf` with the newly-downloaded config (repeat the
      inline-`<ca>` and IP-resolution steps above) and `.env` with the
      new generated username/password, then
      `docker compose up -d --force-recreate gluetun` and re-test in
      OpenVPN Connect first before touching gluetun again
- [ ] If the fresh device entry *also* fails in OpenVPN Connect: contact
      CyberGhost support directly — this points to an account-side block
      (device limit, security hold, etc.) that no client-side config
      change can work around
- [ ] If the fresh device entry succeeds: credentials are correct now,
      update `custom.conf`/`.env` and retry gluetun; if gluetun *still*
      fails while OpenVPN Connect succeeds with the same fresh config,
      it's gluetun-specific — file a GitHub issue with the working
      `.ovpn` attached, or consider WireGuard as a fallback if
      CyberGhost offers a WireGuard config

## Point `arrhost.local` at this PC

- [x] Edit `C:\Windows\System32\drivers\etc\hosts` as Administrator, add
      one line per subdomain pointed at this PC's **LAN IP** (not
      `127.0.0.1`), so other devices on the home network can also resolve
      `*.arrhost.local`: `sonarr.arrhost.local`, `radarr.arrhost.local`,
      `prowlarr.arrhost.local`, `bazarr.arrhost.local`,
      `jellyfin.arrhost.local`, `qbittorrent.arrhost.local`
- [ ] Run `ipconfig /flushdns` in PowerShell after saving
- [ ] If other devices (phone, another PC) should also reach
      `*.arrhost.local`, add the same hosts entries on each of them
- [ ] Consider a DHCP reservation on your router for this PC's LAN IP, so
      the hosts entries don't go stale if the IP changes on renewal

## Start the stack

- [ ] Run `docker compose up -d` from `G:\coderepo\arr\`
- [ ] Run `docker compose ps` — confirm all 8 containers are `running`
      (7 apps + caddy)
- [ ] Check `docker compose logs -f gluetun` — confirm VPN connects
      successfully (look for a "connected" / handshake message, no repeated
      auth errors)
- [ ] Verify the VPN is actually active: run
      `docker exec gluetun wget -qO- https://am.i.mullvad.net/ip` and confirm
      it's your VPN exit IP, not your home IP
- [ ] Open each `http://*.arrhost.local` URL in a browser and confirm it
      loads the right app (not a Caddy error page)

## Configure Prowlarr (http://prowlarr.arrhost.local)

- [ ] Add your indexers (Settings → Indexers)
- [ ] Note down Sonarr's API key (Sonarr → Settings → General)
- [ ] Note down Radarr's API key (Radarr → Settings → General)
- [ ] Add Sonarr and Radarr as Apps in Prowlarr (Settings → Apps) using
      `http://sonarr:8989` / `http://radarr:7878` and their API keys
- [ ] Confirm indexers synced into Sonarr and Radarr automatically

## Configure Sonarr (http://sonarr.arrhost.local)

- [ ] Settings → Download Clients → add qBittorrent (host `gluetun`, port
      `8080`)
- [ ] Settings → Media Management → confirm root folder is `/tv`
- [ ] Add a test TV series and confirm a search returns results

## Configure Radarr (http://radarr.arrhost.local)

- [ ] Settings → Download Clients → add qBittorrent (host `gluetun`, port
      `8080`)
- [ ] Settings → Media Management → confirm root folder is `/movies`
- [ ] Add a test movie and confirm a search returns results

## Configure Bazarr (http://bazarr.arrhost.local)

- [ ] Settings → Sonarr → connect using `http://sonarr:8989` + API key
- [ ] Settings → Radarr → connect using `http://radarr:7878` + API key
- [ ] Settings → Languages → set desired subtitle languages

## Configure Jellyfin (http://jellyfin.arrhost.local)

- [ ] Run first-time setup wizard, create admin account
- [ ] Add library: Movies → `/data/movies`
- [ ] Add library: TV Shows → `/data/tvshows`
- [ ] Confirm libraries scan and show content once Sonarr/Radarr have
      downloaded something

## End-to-end test

- [ ] Search for and download one real TV episode via Sonarr, confirm it
      lands in `D:\arr\media\tv` and plays in Jellyfin
- [ ] Search for and download one real movie via Radarr, confirm it lands in
      `D:\arr\media\movies` and plays in Jellyfin
- [ ] Confirm Bazarr pulled a subtitle for at least one of the above

## Nice-to-haves (later, not required for a working setup)

- [ ] Caddy already reverse-proxies everything on the LAN via
      `*.arrhost.local` — if you also want access from *outside* your home
      network, that needs port forwarding, a real domain, and TLS, which is
      a separate step from what's set up here
- [ ] Set Docker Desktop to start on Windows login (Settings → General) so
      the stack survives a reboot
- [ ] Schedule/verify `docker compose pull && docker compose up -d` runs
      periodically to keep images updated
