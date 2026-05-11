# NymVPN Client — Technical Deep Dive

This document provides a thorough technical reference for the nym-vpn-client codebase, organized by topic.

---

## 1. Repo Structure

```
nym-vpn-client/
├── nym-vpn-core/          # Rust workspace — 43 crates (the engine)
│   ├── crates/
│   │   ├── nym-vpnd/          # Daemon binary
│   │   ├── nym-vpnc/          # CLI binary
│   │   ├── nym-vpn-lib/       # Core VPN library
│   │   ├── nym-vpn-lib-types/ # Shared types
│   │   ├── nym-vpn-proto/     # Protobuf + gRPC stubs
│   │   ├── nym-ifconfig/      # TUN device creation
│   │   ├── nym-routing/       # Route management
│   │   ├── nym-firewall/      # Firewall / kill-switch
│   │   ├── nym-dns/           # DNS configuration
│   │   ├── nym-wg-go/         # wireguard-go CGo wrapper
│   │   ├── nym-socks5-proxy/  # Custom SOCKS5 proxy
│   │   ├── nym-gateway-directory/  # Gateway discovery
│   │   ├── nym-vpn-account-controller/  # zk-credential mgmt
│   │   ├── nym-connection-monitor/      # Health probes
│   │   ├── nym-cgroup/        # Linux cgroup v1/v2
│   │   ├── nym-split-tunnel/  # Split tunneling
│   │   └── ...                # 25+ more
│   └── Cargo.toml         # Workspace definition
├── nym-vpn-app/           # Tauri desktop GUI (Linux, Windows)
├── nym-vpn-apple/         # macOS/iOS Swift wrappers
├── nym-vpn-android/       # Android Kotlin app
├── nym-vpn-windows/       # WinFw + split-tunnel driver (C)
├── wireguard/             # wireguard-go submodule
├── tests/                 # Integration/E2E tests
├── scripts/               # Build/CI scripts
└── Makefile               # Top-level build
```

---

## 2. TUN Device Creation

**Crate:** `nym-ifconfig` (`nym-vpn-core/crates/nym-ifconfig/`)

### Linux (`nym-vpn-core/crates/nym-ifconfig/src/linux/tun.rs`)
- Opens `/dev/net/tun` with `IFF_TUN | IFF_NO_PI`
- Creates a named TUN interface (e.g., `nym-tun0`)
- Assigned IP addresses: `10.250.250.1/32` (IPv4), `2001:db8::1/128` (IPv6) — these are the in-tunnel IPs
- Sets MTU (1500 desktop, 1280 mobile)
- Sets link up

### macOS (`nym-vpn-core/crates/nym-ifconfig/src/apple/utun.rs`)
- Uses `utun` devices via `sysctl` (e.g., `utun0`, `utun1`)
- Same in-tunnel IP assignment pattern as Linux

### Windows
- Uses Wintun adapters (WireGuard's TUN driver)
- Managed via `nym-vpn-lib/src/tunnel_state_machine/wintun.rs`
- Two adapters for WireGuard 2-hop: "WireGuard (entry)" + "WireGuard (exit)"

### Android / iOS
- TUN fd provided by `VpnService` (Android) or `NEPacketTunnelProvider` (iOS)
- Rust code receives a raw file descriptor from the OS
- `AndroidTunProvider` trait, `OSTunProvider` trait

---

## 3. Two-Hop WireGuard Mode (TunTun vs Netstack)

**Crate:** `nym-vpn-lib/src/tunnel_state_machine/tunnel/wireguard/`

### TunTun ("classic" desktop)
- Defined in `connected_tunnel.rs:583-625` — `TunTunTunnelOptions`
- Creates **two physical TUN devices**: one for entry, one for exit
- Entry WG tunnel: `AllowedIPs` set to only `[exit_gateway_ip, metadata_endpoint]` — the entry only routes to the exit
- Exit WG tunnel: `AllowedIPs = 0.0.0.0/0, ::/0` — all traffic routed through exit tunnel
- Exit tunnel's peer endpoint = entry tunnel's TUN IP (traffic flows: app → exit TUN → WG → entry TUN → WG → entry gateway → exit gateway)
- Implementation: `connected_tunnel.rs:134-272` — `run_using_tun_tun()`

### Netstack (mobile + optional desktop)
- Defined in `connected_tunnel.rs:628-654` — `NetstackTunnelOptions`
- **Single physical TUN device** (exit side only)
- Entry tunnel is a **virtual WireGuard tunnel** using wireguard-go's built-in netstack (userspace TCP/IP stack)
- A **UDP forwarder** bridges entry and exit:
  1. Entry tunnel creates an in-tunnel UDP proxy: `entry_tunnel.start_in_tunnel_udp_connection_proxy()`
  2. Exit tunnel's peer endpoint is set to the UDP forwarder's listen address
  3. A TCP proxy handles bandwidth metadata: `entry_tunnel.start_in_tunnel_tcp_connection_proxy()`
- Implementation: `connected_tunnel.rs:274-534` — `run_using_netstack()`

### TwoHopConfig (`two_hop_config.rs:33-120`)
- `entry: WgNodeConfig` — applied to netstack-based tunnel
- `exit: WgNodeConfig` — applied to wg-go attached to real TUN
- `forwarder: WgForwarderConfig` — UDP forwarder bridging entry↔exit
- `tun: TunConfig` — TUN addresses, DNS, MTU
- Entry MTU = 1420 (1500 - 80), Exit MTU = 1340 (1500 - 160)

### AmneziaWG
- Optional censorship-resistant WireGuard fork
- Applied to the **entry tunnel only** (connection to entry gateway)
- Adds junk packets, packet size randomization, header remapping
- `AmneziaConfig::BASE` enables minimum features compatible with plain WG peers
- Feature-gated behind `amnezia` feature in `nym-wg-go` crate

### Circumvention Transports (QUIC Bridges)
- When enabled, the connection to the entry gateway goes over QUIC instead of the default transport
- Used as an alternative to the native UDP-based WireGuard transport
- `nym-vpn-lib/src/tunnel_state_machine/tunnel/transports/` module

---

## 4. Mixnet Mode (5-Hop)

**Crate:** `nym-vpn-lib/src/mixnet/`

### Packet Processing (`processor.rs`)
- `MixnetProcessor` reads IP packets from the TUN device
- Encapsulates each packet as `IpPacketRequest` (via `nym-ip-packet-requests` crate)
- Sends through `nym-sdk::MixnetClient` — a 5-hop Sphinx-encrypted mixnet path
- Also sends periodic cover traffic (dummy packets) for metadata protection
- Handles backpressure via `MixnetBackpressureMonitor` (`backpressure.rs`)
- At shutdown, sends a disconnect request to the IPR (`IpPacketRequest::new_disconnect_request()`)

### Inbound Listener (`mixnet_listener.rs`)
- `MixnetListener` receives reconstructed messages from the mixnet client
- Uses `IprListener` (from `nym-ip-packet-client`) to decode IPR responses
- Writes IP packets back to the TUN device
- Handles `IpPackets`, `MixnetSelfPing`, `Disconnect` message types

### Traffic Flow
```
TUN device (read) → MixnetProcessor → MixnetClient → [5 mixnet hops] → Exit Gateway IPR
TUN device (write) ← MixnetListener  ← MixnetClient ← [5 mixnet hops] ← Exit Gateway IPR
```

---

## 5. Tunnel State Machine

**Crate:** `nym-vpn-lib/src/tunnel_state_machine/`
**File:** `mod.rs` — main state machine loop (1454 lines)

### States (`states/` directory)
| State | File | Description |
|-------|------|-------------|
| `DisconnectedState` | `disconnected_state.rs` | Idle. Waits for connect command. |
| `ConnectingState` | `connecting_state.rs` | Handles gateway selection, registration, tunnel setup |
| `ConnectedState` | `connected_state.rs` | Tunnel is up. Monitors health. | 
| `DisconnectingState` | `disconnecting_state.rs` | Tears down tunnel, resets firewall/routing/DNS |
| `ErrorState` | `error_state.rs` | Error recovery / reporting |
| `OfflineState` | `offline_state.rs` | Host is offline. Waits for network. |

### Transitions
```
Disconnected ↔ Offline
Disconnected → Connecting → Connected → Disconnecting → Disconnected
Disconnected → Connecting → Error → Disconnecting → Disconnected
```

### Connection Lifecycle (TunnelMonitor)
The `TunnelMonitor` (`tunnel_monitor.rs`, 1900 lines) orchestrates the full connection flow:

1. **AwaitAccountReadiness** — Wait for zk-credential account to be ready
2. **RefreshingGateways** — Fetch fresh gateway list from Nym API
3. **SelectingGateways** — Run gateway selection algorithm (entry + exit)
4. **RegisteringWithGateways** — Authenticate with entry/exit gateways:
   - WireGuard: key exchange, credential spend, bandwidth allocation
   - Mixnet: register with Nym gateway, get assigned addresses
5. **InterfaceUp** — Create TUN device(s), start WG/mixnet tunnels, configure routing/firewall/DNS
6. **Up** — Connection health monitoring begins (ICMP/TCP probes)
7. **Down** — On failure or disconnect

---

## 6. Routing

**Crate:** `nym-routing` (`nym-vpn-core/crates/nym-routing/`)

### Linux (`src/unix/linux.rs`)
- Uses `rtnetlink` via the `neli` crate
- Dedicated routing table `0x14d` (333) for VPN traffic
- fwmark `0x100` marks packets that should bypass the tunnel
- Rule: traffic NOT marked with fwmark routes through table 333 → TUN interface
- Required routes: `0.0.0.0/0` and `::/0` via TUN IP, table 333
- Also adds routes for entry/exit gateway endpoints via the default interface (so WG traffic reaches the gateways)

### macOS (`src/unix/macos/`)
- Uses BSD routing sockets (`PF_ROUTE`) 
- Interface-scoped routing (each route bound to a TUN interface name)
- Monitors default route changes and re-resolves peer IPs (for network transitions like WiFi→cellular)

### Windows (`src/windows/`)
- IP Helper API (`GetBestRoute`, `CreateIpForwardEntry`, `DeleteIpForwardEntry`, etc.)
- Monitors default route changes via `NotifyRouteChange`

### Route handling in state machine
- `route_handler.rs` in `nym-vpn-lib/src/tunnel_state_machine/` manages the full `RoutingConfig`
- Sets up required routes before tunnel starts, cleans up on disconnect

---

## 7. Firewall / Kill-Switch

**Crate:** `nym-firewall` (`nym-vpn-core/crates/nym-firewall/`)

### Policy States (`FirewallPolicy` enum in `src/lib.rs:85-132`)
- `Blocked` — Block all non-LAN traffic. Used as initial state (connect is atomic: block → connecting → connected).
- `Connecting` — Allow only specific endpoints (gateway IPs, DNS servers, Nym API endpoints, validator endpoints).
- `Connected` — Allow all traffic through tunnel interface. Block all non-tunnel traffic.

### Platform Implementations
| Platform | Mechanism |
|----------|-----------|
| Linux | `nftables` — create rulesets as `nftnl::FinalizedBatch`, apply via `nftnl` |
| macOS | `pf` (Packet Filter) — `anchor "nym"`, `rdr` rules for redirect |
| Windows | `WinFw` — custom C DLL (`nym-vpn-windows/winfw/`) for Windows Filtering Platform |
| Android | Android's `VpnService` built-in firewall |
| iOS | `NEPacketTunnelProvider`'s `NEFilterRule` / `NEFilterDataProvider` |

### Linux specifics
- `TUNNEL_FWMARK = 0x100` — marks packets that should bypass the tunnel (e.g., WG handshake traffic to entry gateway)
- `TUNNEL_TABLE_ID = 0x14d` — routing table ID
- Sets `net.ipv4.conf.all.src_valid_mark = 1` to prevent reverse path filtering from blocking relay traffic
- Sets `net.ipv4.conf.all.arp_ignore = 2` to prevent ARP-based in-tunnel IP discovery
- Debug: `NYM_FIREWALL_DEBUG=1` adds packet counters; use `sudo nft list ruleset`

---

## 8. DNS Management

**Crate:** `nym-dns` (`nym-vpn-core/crates/nym-dns/`)

### Platform Methods
| Method | Linux | macOS | Windows |
|--------|-------|-------|---------|
| `systemd-resolved` | ✓ via D-Bus | | |
| `resolvconf` | ✓ | | |
| `NetworkManager` | ✓ via D-Bus | | |
| `static-file` | ✓ (/etc/resolv.conf) | | |
| `SCDynamicStore` | | ✓ | |
| `iphlpapi` | | | ✓ |
| `netsh` | | | ✓ |
| `tcpip` | | | ✓ (registry) |

Selected via `NYM_DNS_MODULE` env var or auto-detected.

**Default DNS:** Quad9 (DoT + DoH) and Cloudflare (DoT + DoH) — configured in `nym-vpn-lib/src/lib.rs:54-60`.

---

## 9. SOCKS5 Proxy

**Crate:** `nym-socks5-proxy` (`nym-vpn-core/crates/nym-socks5-proxy/`)

This is a **custom SOCKS5 proxy** (NOT tun2socks, NOT nym-socks5-client).

- Standalone binary, spawned as a child process by the daemon
- Communication via stdin/stdout JSON lines (`nym-socks5-proxy-ipc` crate)
- TCP CONNECT only (no BIND or UDP ASSOCIATE)
- `fast-socks5` crate for SOCKS protocol handling
- Used exclusively for the **geo-exclusion** feature:
  - Traffic destined for excluded country IP ranges bypasses the VPN and goes directly via the default network interface
  - Domain-based exclusion via DNS resolution
- On Linux: uses fwmark to force bypass traffic out the default interface
- On Windows: uses interface binding
- On Android: uses VpnService `protect()` API
- Default listen: `127.0.0.1:1080`

---

## 10. Gateway Registration

**Crate:** `nym-registration-client` (from nym monorepo, git dep)

Two registration modes:
- **WireguardRegistrationResult** — Used in 2-hop mode. Exchanges WG keys with gateways, receives WG configuration (endpoint, allowed IPs, preshared key, etc.), allocates bandwidth.
- **MixnetRegistrationResult** — Used in 5-hop mode. Registers with the Nym gateway, gets assigned Nym address and IPR (IP Packet Router) address.

Both involve:
1. Gateway authentication (zk-credential spend via `nym-bandwidth-controller`)
2. Key exchange
3. Bandwidth allocation
4. Connection setup

---

## 11. Account & Credential System

**Crate:** `nym-vpn-account-controller`

- Manages **zk-nyms** (zero-knowledge credentials)
- Uses **ticketbooks** (pre-paid bandwidth bundles) for private authentication
- No link between payment identity and network activity
- `AccountCommand` / `AccountStateReceiver` channels integrated with tunnel state machine
- Credential storage: SQLite via `nym-vpn-store` crate

---

## 12. WireGuard Go Integration

**Crate:** `nym-wg-go` (`nym-vpn-core/crates/nym-wg-go/`)

- Wraps `wireguard-go` as a **CGo library** (`libwg.a`)
- Built via `make build-wireguard` (runs `wireguard/build-wireguard-go.sh`)
- Provides:
  - `Tunnel::start(config, tun_config)` — start a WG tunnel
  - `Tunnel::stop()` — stop the tunnel
  - `netstack::Tunnel` — virtual WG tunnel without a real TUN device (for mobile)
  - AmneziaWG support via `AmneziaConfig`
  - `TunnelConfig` builder pattern for specifying TUN fds (Unix) or interface names (Windows)

---

## 13. Build Process

### Prerequisites
- Rust toolchain, Go runtime (for wireguard-go CGo), protobuf compiler
- Linux: `libdbus-1-dev libmnl-dev libnftnl-dev`
- macOS: Xcode CLT
- Windows: Visual Studio + WDK

### Build commands
```sh
# 1. Build wireguard-go first (required for wg support)
make build-wireguard

# 2. Build specific binaries
cargo build -p nym-vpnd -p nym-vpnc -p nym-socks5-proxy --release

# 3. Build everything
cargo build --release
```

---

## 14. Container Deployment

For running in a container, use `nym-vpnd` directly. The `nym-vpnd + nym-vpnc` combination is the correct approach — do NOT use tun2socks + nym-socks5-client.

### What you get with nym-vpnd
- Full Layer 3 IP VPN (handles all protocols: TCP, UDP, ICMP, etc.)
- Two modes: 2-hop fast (WireGuard) and 5-hop anonymous (mixnet)
- Built-in kill-switch via nftables/pf
- DNS management (systemd-resolved, etc.)
- Account/credential management (zk-nyms)
- Bandwidth control
- Split tunneling
- Ad blocking
- SOCKS5 proxy with geo-exclusion
- Cross-platform support

### What tun2socks + nym-socks5-client gives you
- TCP-only proxy through the mixnet
- No UDP, no ICMP
- No kill-switch
- No DNS management
- No credential management

### Dockerfile requirements
```dockerfile
# Build stage
FROM rust:latest AS builder
RUN apt-get update && apt-get install -y protobuf-compiler golang-go libdbus-1-dev libmnl-dev libnftnl-dev

# Build wireguard-go + nym-vpnd + nym-vpnc + socks5-proxy
# (same as Section 13)

# Runtime stage
FROM debian:bookworm-slim
RUN apt-get update && apt-get install -y libdbus-1-3 nftables ca-certificates && rm -rf /var/lib/apt/lists/*

COPY --from=builder /build/nym-vpnd /usr/local/bin/
COPY --from=builder /build/nym-vpnc /usr/local/bin/

# Capabilities: NET_ADMIN (routing/nftables), NET_RAW (raw sockets), SYS_ADMIN (network ns)
# Device: /dev/net/tun
CMD ["nym-vpnd"]
```

### Container runtime
```sh
docker run --rm -it \
  --device /dev/net/tun \
  --cap-add=NET_ADMIN \
  --cap-add=NET_RAW \
  --sysctl net.ipv4.conf.all.src_valid_mark=1 \
  nymvpn:latest
```

For the CLI:
```sh
docker exec <container> nym-vpnc connect
docker exec <container> nym-vpnc status --listen
```

---

## 15. Key Architecture Diagrams

### Full Packet Flow: 2-Hop WireGuard (TunTun)
```
┌──────┐  IP packets   ┌────────────┐  WG encrypted   ┌────────────┐  WG encrypted   ┌─────────────┐  plain IP   ┌───────────┐
│ App  │──────────────▶│  Exit TUN  │────────────────▶│  Entry TUN │────────────────▶│Entry Gateway│───────────▶│Exit Gw    │──▶ Internet
│      │◀──────────────│ (wg-exit)  │◀────────────────│(wg-entry)  │◀────────────────│             │◀───────────│(IPR/WG)   │◀── Internet
└──────┘              └────────────┘                  └────────────┘                  └─────────────┘             └───────────┘
                         AllowedIPs:                      AllowedIPs:
                         0.0.0.0/0                        [exit_gw_ip, metadata_ip]
```

### Full Packet Flow: 5-Hop Mixnet
```
┌──────┐  IP packets   ┌────────────┐  Sphinx packets  ┌─────────────────────┐  plain IP   ┌───────────┐
│ App  │──────────────▶│  TUN (tun) │─────────────────▶│  Mixnet (5 hops)    │───────────▶│Exit Gateway│──▶ Internet
│      │◀──────────────│            │◀─────────────────│                      │◀───────────│   (IPR)   │◀── Internet
└──────┘              └────────────┘                   └─────────────────────┘             └───────────┘
```

### Daemon Internals
```
nym-vpnd
├── CommandInterface (gRPC server — tonic + prost)
│   └── Implements `NymVpnService` proto
├── NymVpnService
│   ├── VpnServiceCommand channel
│   ├── AccountController (zk-credential management)
│   ├── GatewayCache / GatewayClient
│   ├── StatisticsController
│   ├── DiscoveryRefresher
│   ├── SplitTunnelManager
│   └── Socks5Service (child process manager)
└── TunnelStateMachine
    ├── DisconnectedState
    ├── OfflineState
    ├── ConnectingState
    │   └── TunnelMonitor
    │       ├── GatewayProvider (selection algorithm)
    │       ├── RegistrationClient (auth + key exchange)
    │       ├── TUN device creation (nym-ifconfig)
    │       ├── WireGuard tunnel start (nym-wg-go)
    │       │   ├── TunTun (2 TUNs) or Netstack (1 TUN + virtual)
    │       └── Mixnet tunnel start (nym-sdk)
    ├── ConnectedState
    │   └── ConnectionMonitor (ICMP/TCP health probes)
    ├── DisconnectingState
    │   └── Firewall reset, route cleanup, DNS restore
    └── ErrorState
```

---

## 16. CLI Reference (nym-vpnc)

```
USAGE:
    nym-vpnc [OPTIONS] <COMMAND>

OPTIONS:
    --table-style <STYLE>    Table style for output
    -h, --help               Print help
    -V, --version            Print version

COMMANDS:
    connect       Connect the tunnel [aliases: connect-v2]
    disconnect    Disconnect the tunnel
    reconnect     Reconnect to any matching gateway
    status        Get current connection status
    info          Get version, network, and contract details
    tunnel        Tunnel configuration (--two-hop, --ipv6, etc.)
    gateway       Manage entry/exit gateways, list gateways
    lan           Local network policy
    dns           DNS configuration
    ad-block      Ad blocking configuration
    account       Account management (create, store, balance)
    device        Device identity management
    network       Nym network selection
    socks5        SOCKS5 proxy (enable/disable/status)
    geo-exclusion Geo-exclusion for SOCKS5 proxy
    split-tunnel  Split tunneling (app/process exclusion)
    diagnostic    Network diagnostic tool
    network-stats Anonymous statistics collection
    sentry        Sentry error monitoring

TUNNEL SET COMMANDS:
    nym-vpnc tunnel set --two-hop on          # Enable 2-hop WireGuard fast mode
    nym-vpnc tunnel set --two-hop off         # Enable 5-hop mixnet anonymous mode
    nym-vpnc tunnel set --ipv6 on             # Enable IPv6
    nym-vpnc tunnel set --ipv6 off            # Disable IPv6
    nym-vpnc tunnel set --circumvention-transports on   # Enable QUIC bridges
    nym-vpnc tunnel set --lewes-protocol on   # Enable Lewes Protocol registration
    nym-vpnc tunnel set --netstack on         # Use netstack multihop (testing)
    nym-vpnc tunnel set --gateway-selection-algorithm <LEVEL>

EXAMPLE WORKFLOW:
    # Start daemon (as root)
    sudo nym-vpnd

    # In another terminal:
    nym-vpnc connect --wait
    nym-vpnc status
    nym-vpnc tunnel get
    nym-vpnc info

    # Switch to anonymous (5-hop mixnet) mode
    nym-vpnc tunnel set --two-hop off
    nym-vpnc reconnect

    # Switch to fast (2-hop WireGuard) mode
    nym-vpnc tunnel set --two-hop on
    nym-vpnc reconnect

    # Disconnect
    nym-vpnc disconnect --wait
```

---

## 17. Environment Variables

| Variable | Description | Platform |
|----------|-------------|----------|
| `NYM_FIREWALL_DEBUG` | Logging level for firewall (1=counters, pass/drop/all) | Linux, macOS |
| `NYM_FIREWALL_DONT_SET_SRC_VALID_MARK` | Skip setting `src_valid_mark=1` | Linux |
| `NYM_FIREWALL_DONT_SET_ARP_IGNORE` | Skip setting `arp_ignore=2` | Linux |
| `NYM_DNS_MODULE` | Force DNS module: `static-file`, `resolvconf`, `systemd`, `network-manager`, `iphlpapi`, `netsh`, `tcpip` | Linux, Windows |
| `NYM_DISABLE_LOCAL_DNS_RESOLVER` | Disable local DNS resolver | macOS |
| `NYM_DISABLE_OFFLINE_MONITOR` | Force daemon to always assume online | All |
| `NYM_USE_PATH_MONITOR` | Use Apple Network framework for offline monitoring | macOS |
| `NYM_CGROUP2_FS` | Custom cgroup2 filesystem path | Linux |
| `NYM_NET_CLS_MOUNT_DIR` | Custom net_cls controller mount dir | Linux |
