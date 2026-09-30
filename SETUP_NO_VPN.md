# ARR Setup: With and Without VPN

You now have two docker-compose files to choose from:

## **Without VPN (Current Setup)**
File: `docker-compose.no-vpn.yml`

**When to use:** Local use, testing, or when you don't need VPN protection for torrents.

**Command:**
```bash
docker compose -f docker-compose.no-vpn.yml up -d
docker compose -f docker-compose.no-vpn.yml logs -f qbittorrent
docker compose -f docker-compose.no-vpn.yml down
```

**Key differences:**
- No Gluetun container
- qBittorrent runs directly on ports `9090` (WebUI) and `6881` (torrent)
- All traffic is direct (not through VPN)
- Faster and simpler — fewer dependencies
- Caddy depends only on the app containers, not Gluetun

---

## **With VPN (Future Setup)**
File: `docker-compose.yml` (original)

**When to use:** When you're ready to route qBittorrent through a VPN.

**Command:**
```bash
docker compose up -d
docker compose logs -f qbittorrent
docker compose down
```

**Key differences:**
- Includes Gluetun container for VPN
- qBittorrent uses `network_mode: "service:gluetun"` to route through VPN
- Requires `.env` with VPN credentials
- Caddy depends on Gluetun

---

## Switching Between Versions

### **Step 1: Stop current stack**
```bash
# If running no-VPN version:
docker compose -f docker-compose.no-vpn.yml down

# If running VPN version:
docker compose down
```

### **Step 2: Start new version**
```bash
# To run WITHOUT VPN:
docker compose -f docker-compose.no-vpn.yml up -d

# To run WITH VPN (after setting up .env):
docker compose up -d
```

### **Step 3: Verify it's working**
```bash
docker ps  # Should see different containers depending on which version you're running
docker logs qbittorrent
```

---

## Important Notes

### Caddyfile Compatibility
Both versions use the same `Caddyfile`, but there's one difference:

- **Without VPN:** Caddy routes to `qbittorrent:8080` directly
- **With VPN:** Caddy routes to `gluetun:8080` (which passes through to qBittorrent)

The current Caddyfile routes to `gluetun:8080`. If you switch to the no-VPN version and Caddy can't reach qBittorrent, edit `Caddyfile`:
```caddy
qbittorrent.arrhost.local {
    reverse_proxy qbittorrent:8080
}
```

(Change `gluetun:8080` to `qbittorrent:8080`)

### Port Mapping
- **Without VPN:** qBittorrent listens directly on `9090` and `6881`
- **With VPN:** Gluetun publishes those ports, qBittorrent is only visible on Docker's internal network

Direct access:
- No VPN: `http://localhost:9090/`
- With VPN: `http://localhost:9090/` (still works, passes through Gluetun)

---

## .env File
The `.env` file is only needed for the VPN version. You can leave it empty or with placeholder values for the no-VPN setup.

If you ever switch to VPN, populate it with your VPN provider credentials:
```bash
VPN_SERVICE_PROVIDER=mullvad  # (or your provider)
VPN_TYPE=wireguard
WIREGUARD_PRIVATE_KEY=xxxxx
```

---

## Quick Cheatsheet

```bash
# Start without VPN
docker compose -f docker-compose.no-vpn.yml up -d

# Start with VPN (once .env is set up)
docker compose up -d

# View logs
docker compose -f docker-compose.no-vpn.yml logs -f qbittorrent
docker compose logs -f qbittorrent

# Stop
docker compose -f docker-compose.no-vpn.yml down
docker compose down

# Check what's running
docker ps
docker ps --format "table {{.Names}}\t{{.Status}}"
```
