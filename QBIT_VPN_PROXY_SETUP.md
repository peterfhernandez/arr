# qBittorrent VPN Proxy Setup

This guide explains how to configure qBittorrent to optionally use gluetun's SOCKS5 proxy when running in VPN mode.

## Key Points

- **Sonarr/Radarr always point to:** `qbittorrent:8080` (no changes needed when toggling VPN)
- **qBittorrent without VPN:** Connects directly (no proxy)
- **qBittorrent with VPN:** Routes through gluetun's SOCKS5 proxy on `gluetun:1080`

## Configuration

### Without VPN Mode
No additional configuration needed. qBittorrent runs normally and connects directly.

```bash
docker compose up -d
```

### With VPN Mode

1. Start the stack with VPN profile:
   ```bash
   docker compose --profile vpn up -d
   ```

2. Open qBittorrent WebUI: `http://qbittorrent.arrhost.local` (or `http://localhost:9090`)

3. Go to **Settings → Connection** and configure the proxy:
   - **Proxy Settings**
     - Proxy type: `SOCKS5`
     - Server address: `gluetun`
     - Port: `1080`
     - Enable **PeerExchange** if you want peer connections to also use the proxy
     - Enable **UTP** (uTorrent Protocol) if supported

4. Save and restart qBittorrent or let it reconnect

5. Verify by checking your IP on a torrent — it should show your VPN's exit IP

### Toggling Between Modes

**When switching from VPN to no-VPN:**
1. Stop: `docker compose down`
2. Go to qBittorrent Settings → Connection and disable the SOCKS5 proxy
3. Start without VPN: `docker compose up -d`

**When switching from no-VPN to VPN:**
1. Stop: `docker compose down`
2. Start with VPN: `docker compose --profile vpn up -d`
3. Go to qBittorrent Settings → Connection and enable SOCKS5 proxy (`gluetun:1080`)

## Configuration Persistence

qBittorrent stores proxy settings in `./config/qbittorrent/config/qBittorrent.conf`. You could create two profiles and swap them if you frequently toggle, but for most cases manually enabling/disabling the proxy is fine.

## Troubleshooting

**Can't connect to proxy:**
- Make sure gluetun is running: `docker compose --profile vpn ps`
- Check gluetun logs: `docker compose --profile vpn logs gluetun`
- Verify SOCKS5 is listening: `docker compose --profile vpn exec gluetun curl localhost:1080`

**Downloads failing in VPN mode:**
- Some indexers block SOCKS5 traffic — you may need to whitelist qBittorrent's proxy IP in qBittorrent settings
- Try disabling PeerExchange if you see connection issues
- Check that your VPN provider's port forwarding is configured (if using port forwarding)