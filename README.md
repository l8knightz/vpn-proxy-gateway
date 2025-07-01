## Overview

This guide covers two Docker Compose setups for running a NordVPN-backed proxy gateway on your local network (`local_net`).

1. **Preferred**: SOCKS5 proxy with authentication (username & password)
2. **Alternate**: HTTP proxy (no auth) for devices like Quest 2 in Developer Mode

Both configurations leverage a `local_net` (macvlan) network to assign LAN‑routable IPs to containers, making the proxy reachable by any device on the same subnet.

---

## Prerequisites

- A Linux host (e.g., Ubuntu) with Docker & Docker Compose installed
- A pre‑created Docker macvlan network named `local_net` in the `192.168.1.0/24` range:
  ```bash
  docker network create \
    -d macvlan \
    --subnet=192.168.1.0/24 \
    --gateway=192.168.1.1 \
    -o parent=eth0 \
    local_net
  ```
  - Replace `eth0` with your LAN interface

### Benefits of `local_net`

- **LAN‑routable IPs**: Containers get real addresses (e.g., `192.168.1.10`) you can point devices at.
- **Isolation**: Keeps VPN gateway separate from host’s default network.
- **Simplicity**: No port conflicts, easy device discovery.

---

## Obtaining Your `.ovpn` File

Before you start, you need an OpenVPN configuration file (`.ovpn`) from your VPN provider:

1. Log in to your VPN provider’s dashboard (e.g., NordVPN).
2. Navigate to **Downloads** or **Manual Setup** → **OpenVPN**.
3. Choose the desired server (e.g., a country or city) and download the `.ovpn` file.
4. Place that file in your project under `./openvpn/your-server.ovpn` and reference it below.

---

## 1. Preferred: SOCKS5 Proxy with Authentication

**Use case**: Full‑featured proxy requiring username/password (compatible with most OSes).

```yaml
version: "3.8"

services:
  gluetun:
    image: qmcgaw/gluetun
    container_name: gluetun
    restart: unless-stopped
    cap_add:
      - NET_ADMIN
    devices:
      - /dev/net/tun:/dev/net/tun
    volumes:
      - ./gluetun:/gluetun
      - ./openvpn/your-server.ovpn:/gluetun/custom.ovpn:ro
    environment:
      - VPN_SERVICE_PROVIDER=nordvpn
      - OPENVPN_USER=YOUR_OPENVPN_USER
      - OPENVPN_PASSWORD=YOUR_OPENVPN_PASS
      - OPENVPN_CUSTOM_CONFIG=/gluetun/custom.ovpn
      - OPENVPN_PROTOCOL=tcp
      - TZ=America/Chicago
    networks:
      local_net:
        ipv4_address: 192.168.1.10

  socks5-proxy:
    image: serjs/go-socks5-proxy
    container_name: socks5-proxy
    depends_on:
      - gluetun
    network_mode: "service:gluetun"
    environment:
      - USERNAME=myuser
      - PASSWORD=mypass
    restart: unless-stopped

networks:
  local_net:
    external: true
```

### How it works

1. `gluetun` establishes an OpenVPN tunnel using your custom `.ovpn`.
2. `socks5-proxy` shares `gluetun`’s network namespace (`tun0` + `local_net` IP) and binds SOCKS5 on `192.168.1.10:1080`.
3. Devices configure their system or browser to use `192.168.1.10:1080` with the specified credentials.

### Device Configuration Examples

- **Linux/macOS CLI**:

  ```bash
  export ALL_PROXY="socks5h://myuser:mypass@192.168.1.10:1080"
  curl https://ifconfig.me/ip
  ```

- **Firefox**:

  1. Preferences → Network Settings → Manual proxy → SOCKS Host: `192.168.1.10`, Port: `1080`, SOCKS v5 → OK
  2. *Check* “Proxy DNS when using SOCKS v5”

  Note: May require an extension life Foxy-Proxy if using Username/Password.

- **Windows** (Command Prompt):

  ```powershell
  netsh winhttp set proxy 192.168.1.10:1080 "" bypass-list=
  ```

---

## 2. Quest 2 HTTP Proxy (No Auth)

**Use case**: Devices (like Quest 2) that only allow host+port in Wi‑Fi proxy settings; requires Developer Mode.

```yaml
version: "3.8"

services:
  gluetun:
    image: qmcgaw/gluetun
    container_name: gluetun
    restart: unless-stopped
    cap_add:
      - NET_ADMIN
    devices:
      - /dev/net/tun:/dev/net/tun
    volumes:
      - ./gluetun:/gluetun
      - ./openvpn/your-server.ovpn:/gluetun/custom.ovpn:ro
    environment:
      - VPN_SERVICE_PROVIDER=nordvpn
      - OPENVPN_USER=YOUR_OPENVPN_USER
      - OPENVPN_PASSWORD=YOUR_OPENVPN_PASS
      - OPENVPN_CUSTOM_CONFIG=/gluetun/custom.ovpn
      - OPENVPN_PROTOCOL=tcp
      - TZ=America/Chicago
      - HTTPPROXY=on
      - HTTPPROXY_LOG=on
      - HTTPPROXY_LISTENING_ADDRESS=:8888
      - FIREWALL_INPUT_PORTS=8888
    networks:
      local_net:
        ipv4_address: 192.168.1.10
    ports:
      - "8888:8888/tcp"

networks:
  local_net:
    external: true
```

### Enabling Developer Mode on Quest 2

1. In the Meta Quest mobile app → Settings → Devices → select Quest 2 → Developer Mode → toggle on
2. Reboot the headset

### Configuring Wi‑Fi Proxy on Quest 2

1. On Quest: **Settings** → **Wi‑Fi** → select your SSID → **Edit** (pencil icon)
2. **Advanced** → **Proxy** → Manual
3. **Hostname**: `192.168.1.10`
4. **Port**: `8888`
5. **Save** and reconnect

Visit [ipleak.net](https://ipleak.net) in the Quest browser to confirm your VPN IP.

---

## OPENVPN Notes & Variations

- **.ovpn placeholder**: Replace `your-server.ovpn` with the file you downloaded from your VPN dashboard.

- **Credentials**: If your VPN account uses MFA, generate *legacy* OpenVPN credentials or switch to WireGuard:

  ```yaml
  - VPN_PROTOCOL=wireguard
  - WIREGUARD_PRIVATE_KEY=<your-key>
  - WIREGUARD_ADDRESS=<assigned-ip>/32
  - WIREGUARD_PUBLIC_KEY=<peer-public-key>
  - WIREGUARD_ENDPOINT=<endpoint>:51820
  ```

- **Server selection**: You can specify `SERVER_COUNTRIES`, `SERVER_CITIES`, or mount a different `.ovpn` with `OPENVPN_CUSTOM_CONFIG`.

---

*— End of Guide —*

