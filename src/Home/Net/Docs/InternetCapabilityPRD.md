# PRD: ZealOS Internet Capability (v1)

## Summary
ZealOS already contains an in-progress network stack and NIC drivers under `src/Home/Net/`. This PRD defines a v1 milestone that turns that existing code into a **reliable, VirtualBox-on-macOS usable** “internet capability” feature set, with **plain HTTP (no TLS)** as the first real-world proof.

## Goals
1. **Reliable VirtualBox networking** (macOS host) using NAT.
2. **DHCP** obtains a lease (IP, router, subnet mask, DNS resolver).
3. **DNS** resolves hostnames (A records) reliably.
4. **TCP** works well enough for outbound client connections to real servers.
5. Provide a user-visible “it works” demo: **HTTP GET** over plain TCP.

## Non-goals (v1)
- TLS / HTTPS
- IPv6
- Wi-Fi
- POSIX compliance for sockets
- High-performance TCP (congestion control, window scaling correctness)

## Target environment
- VirtualBox (macOS host), x86_64 guest
- Network mode: NAT
- NIC model (primary): **Intel E1000 family**
  - Must support at least:
    - 82540EM (PCI device id `0x100E`) — commonly used by VirtualBox “Intel PRO/1000 MT Desktop”
    - 82545EM (PCI device id `0x100F`)

## Current repo reality (what exists today)
Location: `src/Home/Net/`

- Drivers: `Drivers/{E1000, PCNet, RTL8139, VirtIONet}.ZC`
- Stack: Ethernet, ARP, IPv4, ICMP, UDP, TCP, DNS, DHCP
- Entry points:
  - `Start.ZC` → loads stack and runs `NetConfigure` (DHCP) + `NetRep`
  - `Drivers/Run.ZC` → PCI probe + autoload driver

Notes in `Docs/NetworkingNotes.DD` indicate parts are WIP (especially TCP, ICMP send, some DNS/DHCP cleanup).

## User stories
- As a user, I can boot ZealOS in VirtualBox and run one command to bring up networking.
- As a user, I can fetch `http://example.com/` and see a valid HTTP response.
- As a developer, I can run a smoke test that validates DHCP → DNS → TCP → HTTP.

## Acceptance criteria
In VirtualBox NAT:
1. `C:/Home/Net/Start.ZC` succeeds (non-zero `local_ip`).
2. DNS resolves `example.com`.
3. TCP connects to `example.com:80`.
4. HTTP GET returns a response starting with `HTTP/`.
5. Smoke test can be repeated multiple times without crashing to Debug.

## Implementation plan

### Phase 1: VirtualBox E1000 compatibility (must-have)
- Expand PCI ID matching so the E1000 driver loads in VirtualBox:
  - `src/Home/Net/Drivers/Run.ZC`
  - `src/Home/Net/Drivers/E1000.ZC`

Deliverable: driver autoload works for 0x100E + 0x100F.

### Phase 2: DHCP hardening
- Make `NetConfigure` stable under NAT (retries, timeouts, option parsing).

Files:
- `src/Home/Net/Protocols/DHCP.ZC`

### Phase 3: DNS hardening
- Make DNS parsing, retries, and caching safe (no leaks/crashes under repeated queries).

Files:
- `src/Home/Net/Protocols/DNS.ZC`

### Phase 4: TCP hardening for client workloads
- Focus on correctness for outbound client connections:
  - handshake
  - seq/ack tracking
  - retransmit basics
  - close handling

Files:
- `src/Home/Net/Protocols/TCP/*`

### Phase 5: User-facing proof + regression suite
Add:
- `src/Home/Net/Programs/HttpGet.ZC` — minimal HTTP GET tool (no TLS)
- `src/Home/Net/Tests/NetSmokeTest.ZC` — DHCP → DNS → TCP → HTTP

## VirtualBox setup (macOS)
Recommended for v1 testing:
- Adapter type: **Intel PRO/1000 MT Desktop (82540EM)**
- Network: NAT
- “Cable connected”: enabled

## Future phases (post-v1)
- HTTPS/TLS (likely requires: crypto, certificate store, timekeeping, SNI)
- ICMP ping send completion
- IPv6
- More drivers / bridged mode polishing
