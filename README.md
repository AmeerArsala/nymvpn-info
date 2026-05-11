# NymVPN Client — Architecture Overview

This document covers what runs under the hood in the [nym-vpn-client](https://github.com/nymtech/nym-vpn-client) monorepo.

---

## Quick Answers

**Does it use tun2socks?** No. Zero usage anywhere in the codebase.

**Does it use nym-socks5-client?** No. The only reference is `nym-socks5-client-core` as an indirect git dependency name in `Cargo.lock` (for license attribution in mobile apps). Not actually used.

**How does it create the VPN interface?** TUN devices via the `tun` crate (Linux `/dev/net/tun`, macOS `utun`, Windows Wintun). No userspace TCP/IP stack for the main VPN path.

**How does 2-hop vs 5-hop work?** Two mutually exclusive `TunnelType` modes — `Mixnet` (5-hop Sphinx mixnet) and `Wireguard` (2-hop nested WireGuard tunnels).

**How do you operate it?** Client-server (daemon) architecture: `nym-vpnd` (background daemon with gRPC API) + `nym-vpnc` (CLI client).

---

## Binaries

| Binary | Role |
|--------|------|
| `nym-vpnd` | VPN daemon — runs as root/background service, owns the tunnel state machine, exposes gRPC API |
| `nym-vpnc` | CLI client — talks to daemon over gRPC (tonic/protobuf) |
| `nym-socks5-proxy` | Standalone SOCKS5 proxy (child process of daemon, for geo-exclusion only) |
| `nym-diagnostic` | Network diagnostic tool |

---

## Two Tunnel Modes

### 1. Mixnet Mode (5-hop "Anonymous Mode")

- Runs through the Nym **mixnet** — 5 Sphinx-encrypted hops with cover traffic, packet padding, timing obfuscation
- Each IP packet from the TUN is encapsulated as an `IpPacketRequest` and sent to an **IP Packet Router (IPR)** at the exit gateway
- The IPR unwraps packets and forwards to the internet
- Return traffic comes back through the mixnet, received by `MixnetListener`, written to the TUN
- Single TUN device
- Provides metadata protection (defeats traffic correlation)

**Traffic flow:**
```
App → TUN (tun0) → MixnetProcessor → nym-sdk MixnetClient → [5 mixnet hops] → Exit Gateway IPR → Internet
Internet → Exit Gateway IPR → [5 mixnet hops] → MixnetListener → TUN → App
```

### 2. WireGuard Mode (2-hop "Fast Mode")

- Two nested WireGuard tunnels: **Client → Entry Gateway → Exit Gateway → Internet**
- Two sub-implementations:

  **a) TunTun (desktop: Linux, macOS, Windows):**
  - Two real TUN devices (one for entry, one for exit)
  - Entry WG: `AllowedIPs = [exit_gateway_ip, metadata_endpoint]`
  - Exit WG: `AllowedIPs = 0.0.0.0/0, ::/0`
  - Exit route goes through the entry tunnel (tunnel-in-tunnel)

  **b) Netstack (mobile: Android, iOS; also testable on desktop):**
  - One real TUN device (exit)
  - Entry tunnel is virtual via wireguard-go's netstack (userspace TCP/IP stack)
  - UDP forwarder bridges the two tunnels

**Traffic flow (TunTun):**
```
App → TUN (wg-exit) → WireGuard → TUN (wg-entry) → WireGuard → Entry Gateway → Exit Gateway → Internet
```

---

## VPN Interface Creation

Cross-platform TUN via the `nym-ifconfig` crate (wrapping `tun = "0.8.7"`):

| Platform | Mechanism |
|----------|-----------|
| Linux | `/dev/net/tun` |
| macOS | `utun` devices via sysctl |
| Windows | Wintun adapters (WireGuard TUN driver) |
| Android | `VpnService` TUN fd from OS |
| iOS | `NEPacketTunnelProvider` TUN fd from OS |

MTU: 1500 (desktop), 1280 (mobile) for mixnet; adjusted lower for entry/exit to account for WG overhead in 2-hop mode (exit MTU = 1340, entry MTU = 1420).

---

## Daemon + CLI Architecture

```
┌──────────────────────────────────────────────────────────────────────┐
│                          nym-vpnd (daemon)                           │
│                                                                      │
│  ┌─────────────────┐    ┌────────────────────┐    ┌───────────────┐  │
│  │ CommandInterface │───▶│  NymVpnService     │───▶│  TunnelState  │  │
│  │ (gRPC over Unix) │    │  (main orchestrator)│   │  Machine      │  │
│  └─────────────────┘    └────────────────────┘    └───────┬───────┘  │
│                                                            │         │
│                    ┌───────────────────────────────────────▼──────┐  │
│                    │            TunnelMonitor                    │  │
│                    │  (gateway selection → registration → start) │  │
│                    └───┬────────────┬──────────────┬─────────────┘  │
│                        │            │              │               │
│                   ┌────▼───┐  ┌────▼───┐   ┌──────▼──────┐        │
│                   │Firewall│  │Routing │   │  DNS        │        │
│                   │(nft/pf)│  │(tables)│   │(resolved..) │        │
│                   └────────┘  └────────┘   └─────────────┘        │
│                                                                      │
│  Child process: nym-socks5-proxy (geo-exclusion SOCKS5 proxy)       │
└──────────────────────────────────────────────────────────────────────┘
         ▲                    │
         │ gRPC (tonic)       │ events (broadcast)
         ▼                    │
┌──────────────────────────────────────────────────────────────────────┐
│                          nym-vpnc (CLI)                              │
│  connect / disconnect / status / tunnel set / gateway / account ...  │
└──────────────────────────────────────────────────────────────────────┘
```

### nym-vpnd Daemon

Run as root/privileged (needs `NET_ADMIN`, `NET_RAW`, TUN device access). Starts:
1. A gRPC server (Unix socket or TCP) — the `CommandInterface`
2. The `NymVpnService` — handles all commands, owns the tunnel state machine
3. Optional child processes: `nym-socks5-proxy` (SOCKS5 proxy for geo-exclusion)

### nym-vpnc CLI

| Command | Purpose |
|---------|---------|
| `connect [--wait]` | Connect tunnel (block with `-w`) |
| `disconnect [--wait]` | Disconnect tunnel |
| `reconnect` | Reconnect to a suitable gateway |
| `status [--listen]` | Show/stream tunnel state |
| `info` | Daemon version, network, contracts |
| `tunnel get` | Show tunnel configuration |
| `tunnel set --two-hop <on/off>` | Toggle 2-hop WG vs 5-hop mixnet |
| `tunnel set --ipv6 <on/off>` | Toggle IPv6 |
| `gateway list` / `gateway set-entry` / `set-exit` | Gateway management |
| `socks5 enable/disable/status` | SOCKS5 proxy control |
| `account ...` | Account/credential management |
| `split-tunnel ...` | App/process exclusion |
| `ad-block enable/disable` | Ad blocking |
| `dns ...` | DNS configuration |
| `network ...` | Nym network selection |
| `diagnostic ...` | Network diagnostics |

---

## Key Crates

| Crate | Purpose |
|-------|---------|
| `nym-vpn-lib` | Core VPN engine — tunnel state machine, routing, firewall, DNS, ad-block, SOCKS5, accounts |
| `nym-vpn-lib-types` | Shared types: `TunnelType`, `TunnelState`, `ConnectionData`, `EntryPoint`, `ExitPoint` |
| `nym-vpn-proto` | Protobuf + gRPC definitions for daemon↔client communication |
| `nym-vpnd` | Daemon binary |
| `nym-vpnc` | CLI binary |
| `nym-ifconfig` | Cross-platform TUN device creation |
| `nym-routing` | Route management (rtnetlink, routing socket, IP helper API) |
| `nym-firewall` | Firewall (nftables, pf, WinFw); kill-switch |
| `nym-dns` | DNS configuration (systemd-resolved, resolvconf, NetworkManager, netsh, iphlpapi, SCDynamicStore) |
| `nym-wg-go` | Rust wrapper around `wireguard-go` (CGo) — includes AmneziaWG support |
| `nym-socks5-proxy` | Custom SOCKS5 proxy (TCP CONNECT, geo-IP routing) |
| `nym-socks5-proxy-ipc` | IPC protocol between daemon and SOCKS5 child process |
| `nym-gateway-directory` | Gateway discovery and client |
| `nym-vpn-account-controller` | zk-credential management (ticketbooks, bandwidth) |
| `nym-connection-monitor` | Connection health (ICMP/TCP probes) |
| `nym-offline-monitor` | Host network connectivity state |
| `nym-cgroup` | Linux cgroup v1/v2 for split tunneling |
| `nym-split-tunnel` | Cross-platform split tunneling (cgroup, Endpoint Security, WinFw driver) |
| `nym-statistics` | Anonymous network statistics |

---

## Gateway Selection

The `GatewayProvider` (in `gateway_provider/` module) selects entry and exit gateways based on:
- Performance tier (min mixnode/gateway performance thresholds)
- Geo-location proximity (optional)
- User preferences (entry point, exit point, residential-only)
- Automatic vs manual selection algorithm

The selection algorithm can be set via `nym-vpnc tunnel set --gateway-selection-algorithm <LEVEL>`:
- 0: Fully manual entry + exit
- 1: Auto entry, manual exit
- 2+: Auto entry + exit

---

## Firewall / Kill-Switch

The firewall is set in three phases:
1. **Blocked** (initial state before connecting) — blocks all non-LAN traffic
2. **Connecting** — allows only gateway endpoints, DNS servers, API endpoints
3. **Connected** — allows all traffic through the tunnel interface only

Linux: nftables, dedicated routing table `0x14d = 333`, fwmark `0x100`.
macOS: pf, routing sockets for interface-scoped routes.
Windows: WinFw (Windows Filtering Platform), IP helper API.

---

## Container Installation

For running in a container, use `nym-vpnd + nym-vpnc` from this repo — NOT tun2socks + nym-socks5-client.

**Why this is better than a DIY approach:**
- Full Layer 3 VPN (not just TCP SOCKS5)
- Both 2-hop fast and 5-hop anonymous modes
- Built-in kill-switch / firewall
- DNS handling (systemd-resolved, resolvconf, etc.)
- zk-credential system for private authentication
- Split tunneling, ad blocking, IPv6
- Account management and bandwidth control
- Cross-platform support

**Container requirements:**
- `--device /dev/net/tun` (TUN device access)
- `--cap-add=NET_ADMIN` (routing, nftables)
- Go runtime (for wireguard-go CGo library)
- Protobuf compiler, D-Bus library, libmnl, libnftnl (build-time)

**Build:**
```sh
make build-wireguard           # Build wireguard-go first
cargo build -p nym-vpnd -p nym-vpnc -p nym-socks5-proxy --release
```
