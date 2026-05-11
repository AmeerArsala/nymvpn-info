# NymVPN Client — Operations Guide

---

## CLI Quick Reference

Two binaries: **`nym-vpnd`** (daemon, must run as root) and **`nym-vpnc`** (CLI client, talks to daemon over gRPC).

### Starting the Daemon

```sh
sudo nym-vpnd
```

Verbose: `sudo nym-vpnd -vv`

### Connecting / Disconnecting

```sh
nym-vpnc connect                  # fire and forget
nym-vpnc connect --wait           # block until connected
nym-vpnc disconnect
nym-vpnc disconnect --wait
nym-vpnc status                   # current state
nym-vpnc status --listen          # stream state changes
```

### Switching Between Modes

**Fast Mode (2-hop WireGuard)** — lower latency, decentralized:
```sh
nym-vpnc tunnel set --two-hop on
nym-vpnc reconnect
```

**Anonymous Mode (5-hop mixnet)** — maximal privacy, metadata protection:
```sh
nym-vpnc tunnel set --two-hop off
nym-vpnc reconnect
```

### Viewing Current Configuration

```sh
nym-vpnc tunnel get
```

### Other Commands

```sh
nym-vpnc info                     # version, network, contracts
nym-vpnc gateway list             # available gateways
nym-vpnc gateway set-entry <id>   # pick specific entry gateway
nym-vpnc gateway set-exit <id>    # pick specific exit gateway
nym-vpnc socks5 enable            # start SOCKS5 proxy (geo-exclusion)
nym-vpnc ad-block enable          # enable ad blocking
nym-vpnc dns set 1.1.1.1         # custom DNS
nym-vpnc split-tunnel add <pid>   # exclude a process from VPN
nym-vpnc diagnostic run           # run network diagnostics
```

### Minimal Working Example

```sh
# Terminal 1 — start daemon
sudo nym-vpnd

# Terminal 2 — connect in fast mode
nym-vpnc tunnel set --two-hop on
nym-vpnc connect --wait

# Verify
nym-vpnc status

# Switch to anonymous mode
nym-vpnc tunnel set --two-hop off
nym-vpnc reconnect --wait

# Tear down
nym-vpnc disconnect --wait
```

---

## Container Deployment

Use **`nym-vpnd + nym-vpnc`**, not tun2socks + nym-socks5-client. The latter is TCP-only with no kill-switch, no UDP, no fast mode, no credential management.

### Dockerfile

```dockerfile
# ---- Build Stage ----
FROM rust:latest AS builder

RUN apt-get update && apt-get install -y \
    protobuf-compiler golang-go libdbus-1-dev libmnl-dev libnftnl-dev

WORKDIR /build
COPY . .

# Build wireguard-go first (requires Go)
RUN make build-wireguard

# Build the three needed binaries
RUN cargo build -p nym-vpnd -p nym-vpnc -p nym-socks5-proxy --release

# ---- Runtime Stage ----
FROM debian:bookworm-slim

RUN apt-get update && apt-get install -y \
    libdbus-1-3 nftables ca-certificates iptables \
    && rm -rf /var/lib/apt/lists/*

COPY --from=builder /build/target/release/nym-vpnd /usr/local/bin/
COPY --from=builder /build/target/release/nym-vpnc /usr/local/bin/

ENTRYPOINT ["nym-vpnd"]
```

### Building

```sh
docker build -t nymvpn .
```

### Running

```sh
docker run --rm -it --name nymvpn \
  --device /dev/net/tun \
  --cap-add=NET_ADMIN \
  --cap-add=NET_RAW \
  --sysctl net.ipv4.conf.all.src_valid_mark=1 \
  nymvpn
```

### Interacting with a Running Container

```sh
# Connect (anonymous mode by default)
docker exec nymvpn nym-vpnc connect --wait

# Switch to fast 2-hop mode
docker exec nymvpn nym-vpnc tunnel set --two-hop on
docker exec nymvpn nym-vpnc reconnect --wait

# Status / info
docker exec nymvpn nym-vpnc status
docker exec nymvpn nym-vpnc info

# Disconnect
docker exec nymvpn nym-vpnc disconnect --wait
```

### Docker Compose

```yaml
services:
  nymvpn:
    build: .
    container_name: nymvpn
    devices:
      - /dev/net/tun
    cap_add:
      - NET_ADMIN
      - NET_RAW
    sysctls:
      net.ipv4.conf.all.src_valid_mark: 1
    restart: unless-stopped
    entrypoint: ["nym-vpnd"]
```

Then: `docker compose up -d`

---

## Environment Variables

| Variable | Effect |
|----------|--------|
| `NYM_DNS_MODULE` | Force DNS backend: `systemd`, `resolvconf`, `network-manager`, `static-file` (Linux); `iphlpapi`, `netsh`, `tcpip` (Windows) |
| `NYM_DISABLE_OFFLINE_MONITOR=1` | Don't check host connectivity |
| `NYM_FIREWALL_DEBUG=1` | Log firewall rules (nftables counters) |
| `NYM_FIREWALL_DONT_SET_SRC_VALID_MARK=1` | Skip `src_valid_mark` sysctl |
