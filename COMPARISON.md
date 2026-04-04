# DNSDave — Competitive Comparison

**Version:** 0.1.0-draft  
**Date:** 2026-04-04  
**Status:** Draft  

This document compares DNSDave against Pi-hole, major open-source DNS/DHCP tools, and enterprise DDI (DNS, DHCP, IP Address Management) platforms. The goal is to be **honest and precise**: showing where DNSDave leads, where it matches, and where it currently falls short.

### Symbol key

| Symbol | Meaning |
|--------|---------|
| ✅ | Full support (designed and complete, or at v1) |
| 🟡 | Partial / limited support |
| 🔮 | On the roadmap (planned future milestone) |
| ❌ | Not supported / not planned |
| —  | Not applicable to this product's design scope |

> **Note on DNSDave status:** DNSDave is currently a design-complete project pre-implementation. All ✅ marks reflect features fully specified in the design; all 🔮 marks reflect documented future milestones.

---

## 1. Products at a Glance

| Product | Category | Core strength | License | Language |
|---------|----------|--------------|---------|----------|
| **DNSDave** | DNS + DHCP + filtering | API-first, event-driven, high-performance, multi-arch | OSS (TBD) | Rust |
| **Pi-hole v6** | DNS filtering + DHCP | Ad blocking, home network ease of use | EUPL-1.2 | C (FTL) / Python |
| **AdGuard Home** | DNS filtering + DHCP | Ad blocking, DoH/DoT server, easy setup | GPL-3.0 | Go |
| **Technitium DNS** | DNS + filtering + DHCP | Authoritative DNS + filtering in one, rich UI | GPL-3.0 | .NET / C# |
| **dnsmasq** | Lightweight DNS + DHCP | Minimal footprint, embedded systems | GPL-2.0 | C |
| **CoreDNS** | Cloud-native DNS | Kubernetes service discovery, plugin ecosystem | Apache-2.0 | Go |
| **BIND9** | Authoritative + recursive DNS | Battle-tested, full RFC compliance | MPL-2.0 | C |
| **PowerDNS suite** | DNS (authoritative + recursive + frontend) | Enterprise open-source DNS, DNSSEC excellence | GPL-2.0 | C++ |
| **Infoblox DDI** | Enterprise DDI | Full DNS + DHCP + IPAM, enterprise RBAC | Proprietary | — |

---

## 2. DNS Resolution & Forwarding

| Feature | DNSDave | Pi-hole | AdGuard | Technitium | dnsmasq | CoreDNS | BIND9 | PowerDNS | Infoblox |
|---------|:-------:|:-------:|:-------:|:----------:|:-------:|:-------:|:-----:|:--------:|:--------:|
| Forwarding / caching resolver | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| Full recursive resolver | ❌ | ❌ | ❌ | ✅ | ❌ | 🟡 | ✅ | ✅ | ✅ |
| DoH upstream | ✅ | ✅ | ✅ | ✅ | ❌ | 🟡 | ❌ | ✅ | ✅ |
| DoT upstream | ✅ | ✅ | ✅ | ✅ | ❌ | 🟡 | ❌ | ✅ | ✅ |
| DoH server | ✅ | ❌ | ✅ | ✅ | ❌ | ✅ | ❌ | ✅ | ✅ |
| DoT server (:853) | ✅ | ❌ | ✅ | ✅ | ❌ | ✅ | ❌ | ✅ | ✅ |
| DNS-over-QUIC (DoQ) | 🔮 | ❌ | ✅ | ❌ | ❌ | 🟡 | ❌ | ❌ | ❌ |
| Local A/AAAA/CNAME/MX/TXT records | ✅ | 🟡 | 🟡 | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| Wildcard DNS (`*.zone`) | ✅ | ❌ | 🟡 | ✅ | 🟡 | ✅ | ✅ | ✅ | ✅ |
| Conditional DNS forwarding | ✅ | 🟡 | 🟡 | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| Split-horizon / views | ✅ | ❌ | ❌ | ❌ | ❌ | 🟡 | ✅ | ✅ | ✅ |
| Response Policy Zones (RPZ) ingestion | ✅ | ❌ | ❌ | ❌ | ❌ | 🟡 | ✅ | ✅ | ✅ |
| Negative caching (RFC 2308) | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| Per-client / per-group answer policies | ✅ | ✅ | ✅ | 🟡 | ❌ | 🟡 | ✅ | 🟡 | ✅ |
| Upstream failover & health checks | ✅ | 🟡 | 🟡 | ✅ | 🟡 | ✅ | ❌ | ✅ | ✅ |

> **DNSDave notes:** Full recursive resolution is an explicit non-goal; DNSDave always forwards unknown queries to configured upstreams (DoH/DoT/plain). DoQ is on the post-v1 roadmap.

---

## 3. Authoritative DNS & Zone Management

| Feature | DNSDave | Pi-hole | AdGuard | Technitium | dnsmasq | CoreDNS | BIND9 | PowerDNS | Infoblox |
|---------|:-------:|:-------:|:-------:|:----------:|:-------:|:-------:|:-----:|:--------:|:--------:|
| Authoritative primary zones | ✅ | ❌ | ❌ | ✅ | ❌ | ✅ | ✅ | ✅ | ✅ |
| Authoritative secondary zones | ✅ | ❌ | ❌ | ✅ | ❌ | ✅ | ✅ | ✅ | ✅ |
| SOA record management | ✅ | ❌ | ❌ | ✅ | ❌ | ✅ | ✅ | ✅ | ✅ |
| SOA serial auto-increment | ✅ | — | — | ✅ | — | 🟡 | 🟡 | ✅ | ✅ |
| AXFR (full zone transfer) | ✅ | ❌ | ❌ | ✅ | ❌ | ❌ | ✅ | ✅ | ✅ |
| IXFR (incremental zone transfer) | ✅ | ❌ | ❌ | 🟡 | ❌ | ❌ | ✅ | ✅ | ✅ |
| RFC 1996 NOTIFY to secondaries | ✅ | ❌ | ❌ | ✅ | ❌ | ❌ | ✅ | ✅ | ✅ |
| RFC 2136 DNS UPDATE (dynamic) | ✅ | ❌ | ❌ | ❌ | ❌ | ❌ | ✅ | ✅ | ✅ |
| `home.arpa` zone template (RFC 8375) | ✅ | ❌ | ❌ | ❌ | ❌ | ❌ | 🟡 | 🟡 | ✅ |
| Reverse zone (PTR) management | ✅ | ❌ | ❌ | ✅ | 🟡 | ✅ | ✅ | ✅ | ✅ |
| AA flag + SOA in NXDOMAIN | ✅ | ❌ | ❌ | ✅ | ❌ | ✅ | ✅ | ✅ | ✅ |
| ExternalDNS provider (Kubernetes) | ✅ | ❌ | ❌ | ❌ | ❌ | ✅ | 🟡 | ✅ | ✅ |

> **Pi-hole / AdGuard notes:** Both support simple local A/CNAME entries but have no concept of authoritative zones, SOA records, or zone transfers. They cannot serve as a secondary nameserver or accept RFC 2136 updates.

---

## 4. Ad Blocking & DNS Filtering

| Feature | DNSDave | Pi-hole | AdGuard | Technitium | dnsmasq | CoreDNS | BIND9 | PowerDNS | Infoblox |
|---------|:-------:|:-------:|:-------:|:----------:|:-------:|:-------:|:-----:|:--------:|:--------:|
| Pi-hole gravity (hosts) format | ✅ | ✅ | ✅ | ✅ | 🟡 | ❌ | ❌ | ❌ | ❌ |
| Domain-only list format | ✅ | ✅ | ✅ | ✅ | 🟡 | ❌ | ❌ | ❌ | ❌ |
| AdBlock Plus syntax | ✅ | 🟡 | ✅ | ✅ | ❌ | ❌ | ❌ | ❌ | ❌ |
| RPZ blocklist ingestion | ✅ | ❌ | ❌ | ❌ | ❌ | 🟡 | ✅ | ✅ | ✅ |
| Scheduled list sync | ✅ | ✅ | ✅ | ✅ | ❌ | ❌ | — | — | ✅ |
| Incremental blocklist updates | ✅ | ❌ | ❌ | ❌ | — | — | — | — | ✅ |
| MPHF + Bloom filter hot-swap | ✅ | ❌ | ❌ | ❌ | — | — | — | — | — |
| Per-client / per-group blocklist policy | ✅ | ✅ | ✅ | 🟡 | ❌ | ❌ | ❌ | ❌ | ✅ |
| Allowlist overrides blocklist | ✅ | ✅ | ✅ | ✅ | 🟡 | — | — | — | ✅ |
| Live block/allow toggle (no restart) | ✅ | ✅ | ✅ | ✅ | ❌ | — | — | — | ✅ |
| Blocklist >5M domains | ✅ | 🟡 | 🟡 | ✅ | ❌ | — | — | — | ✅ |
| Blocklist query <1 µs (in-memory) | ✅ | 🟡 | 🟡 | 🟡 | 🟡 | — | — | — | — |
| Pi-hole API compatibility shim | ✅ | ✅ | ❌ | ❌ | — | — | — | — | — |

> **Performance note:** DNSDave's two-stage lookup (Bloom filter → MPHF map) is designed for sub-microsecond blocked-domain responses. Pi-hole FTL uses a binary tree; AdGuard Home uses a hash map — both are fast but allocate more on the hot path.

---

## 5. DHCP

| Feature | DNSDave | Pi-hole | AdGuard | Technitium | dnsmasq | CoreDNS | BIND9/KEA | PowerDNS/KEA | Infoblox |
|---------|:-------:|:-------:|:-------:|:----------:|:-------:|:-------:|:---------:|:------------:|:--------:|
| DHCPv4 server | ✅ | ✅ | ✅ | ✅ | ✅ | — | ✅ | ✅ | ✅ |
| DHCPv6 (IA_NA) | ✅ | ❌ | ✅ | 🟡 | ✅ | — | ✅ | ✅ | ✅ |
| DHCPv6 prefix delegation (IA_PD) | 🔮 | ❌ | ❌ | ❌ | ✅ | — | ✅ | ✅ | ✅ |
| Static MAC→IP reservations | ✅ | ✅ | ✅ | ✅ | ✅ | — | ✅ | ✅ | ✅ |
| Option hierarchy (global→scope→host) | ✅ | ❌ | ❌ | 🟡 | 🟡 | — | ✅ | ✅ | ✅ |
| Full option suite (codes 1–252) | ✅ | 🟡 | 🟡 | 🟡 | 🟡 | — | ✅ | ✅ | ✅ |
| PXE boot (BIOS + UEFI auto-detect) | ✅ | ❌ | ❌ | 🟡 | ✅ | — | ✅ | ✅ | ✅ |
| DHCP relay (RFC 3046 / option 82) | ✅ | ❌ | ❌ | ❌ | ✅ | — | ✅ | ✅ | ✅ |
| Client class / vendor matching | ✅ | ❌ | ❌ | ❌ | ✅ | — | ✅ | ✅ | ✅ |
| Dynamic DNS from leases (DDNS) | ✅ | 🟡 | 🟡 | ✅ | ✅ | — | ✅ | ✅ | ✅ |
| PTR auto-registration from leases | ✅ | 🟡 | 🟡 | ✅ | ✅ | — | ✅ | ✅ | ✅ |
| Lease event stream (real-time) | ✅ | ❌ | ❌ | ❌ | ❌ | — | ❌ | ❌ | 🟡 |
| HA DHCP (leader election, failover) | ✅ | ❌ | ❌ | ❌ | ❌ | — | ✅ | ✅ | ✅ |
| DHCP Prometheus metrics | ✅ | ❌ | ❌ | ❌ | ❌ | — | 🟡 | 🟡 | 🟡 |

> **Pi-hole note:** Pi-hole's DHCP is DHCPv4-only via dnsmasq; it has no relay support, limited option control, and no PXE without manual dnsmasq config. AdGuard Home added basic DHCPv6 in 2022 but it remains limited.

---

## 6. DNSSEC

| Feature | DNSDave | Pi-hole | AdGuard | Technitium | dnsmasq | CoreDNS | BIND9 | PowerDNS | Infoblox |
|---------|:-------:|:-------:|:-------:|:----------:|:-------:|:-------:|:-----:|:--------:|:--------:|
| DNSSEC validation (upstream) | ✅ | 🟡 | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| Strict / opportunistic / off modes | ✅ | ❌ | 🟡 | 🟡 | ❌ | ✅ | ✅ | ✅ | ✅ |
| AD bit on validated replies | ✅ | ❌ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| SERVFAIL on validation failure | ✅ | ❌ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| Custom trust anchors | ✅ | ❌ | ❌ | 🟡 | ❌ | ✅ | ✅ | ✅ | ✅ |
| Per-zone / per-forward validation policy | ✅ | ❌ | ❌ | ❌ | ❌ | 🟡 | ✅ | ✅ | ✅ |
| Zone signing (ZSK + KSK) | 🔮 | ❌ | ❌ | ✅ | ❌ | ✅ | ✅ | ✅ | ✅ |
| NSEC3 (authenticated denial) | 🔮 | ❌ | ❌ | ✅ | ❌ | ✅ | ✅ | ✅ | ✅ |
| DS record export | 🔮 | ❌ | ❌ | ✅ | ❌ | ❌ | ✅ | ✅ | ✅ |
| Automated ZSK rollover | 🔮 | ❌ | ❌ | ❌ | ❌ | ❌ | 🟡 | ✅ | ✅ |
| Ed25519 / ECDSA P-384 algorithms | 🔮 | ❌ | ❌ | ✅ | ❌ | ✅ | ✅ | ✅ | ✅ |

> **Pi-hole note:** Pi-hole enables DNSSEC validation via dnsmasq's `--dnssec` flag but does not expose granular control, cannot set trust anchors, and does not sign zones.

---

## 7. Certificates & Transport Security

| Feature | DNSDave | Pi-hole | AdGuard | Technitium | dnsmasq | CoreDNS | BIND9 | PowerDNS | Infoblox |
|---------|:-------:|:-------:|:-------:|:----------:|:-------:|:-------:|:-----:|:--------:|:--------:|
| TLS for REST API (HTTPS) | ✅ | 🟡 | ✅ | ✅ | — | — | — | ✅ | ✅ |
| Self-signed cert bootstrap (zero-config) | ✅ | ❌ | ❌ | 🟡 | — | — | — | — | — |
| ACME (Let's Encrypt) auto-provisioning | ✅ | ❌ | ✅ | ✅ | — | ✅ | — | — | ❌ |
| ACME DNS-01 via own RFC 2136 interface | ✅ | ❌ | ❌ | ❌ | — | — | — | — | ❌ |
| ACME DNS-01 helper for other services | ✅ | ❌ | ❌ | ❌ | — | — | — | — | ❌ |
| TLS cert hot-reload (no restart) | ✅ | ❌ | ✅ | 🟡 | — | ✅ | — | — | ✅ |
| NATS inter-container mTLS | ✅ | — | — | — | — | — | — | — | — |
| TSIG authentication (RFC 2845) | 🔮 | ❌ | ❌ | ❌ | ✅ | ❌ | ✅ | ✅ | ✅ |

---

## 8. API & Integration

| Feature | DNSDave | Pi-hole | AdGuard | Technitium | dnsmasq | CoreDNS | BIND9 | PowerDNS | Infoblox |
|---------|:-------:|:-------:|:-------:|:----------:|:-------:|:-------:|:-----:|:--------:|:--------:|
| REST API (first-class) | ✅ | 🟡 | 🟡 | 🟡 | ❌ | ❌ | ❌ | ✅ | ✅ |
| OpenAPI 3.1 specification | ✅ | ❌ | ❌ | ❌ | — | — | — | ✅ | ✅ |
| Pi-hole API v5 compatibility | ✅ | ✅ | ❌ | ❌ | — | — | — | — | — |
| Pi-hole API v6 compatibility | 🔮 | ✅ | ❌ | ❌ | — | — | — | — | — |
| Scoped API keys (read-only, per-resource) | ✅ | ❌ | ❌ | ❌ | — | — | — | ✅ | ✅ |
| Bulk record operations | ✅ | ❌ | ❌ | 🟡 | — | — | — | ✅ | ✅ |
| Server-Sent Events (SSE) live streams | ✅ | ❌ | ❌ | ❌ | — | — | — | — | — |
| Webhook / event push | 🔮 | ❌ | ❌ | ❌ | — | — | — | — | ✅ |
| Kubernetes ExternalDNS provider | ✅ | ❌ | ❌ | ❌ | ❌ | ✅ | 🟡 | ✅ | ✅ |
| Terraform provider | 🔮 | ❌ | ❌ | ❌ | — | — | — | 🟡 | ✅ |
| Ansible module | 🔮 | 🟡 | ❌ | ❌ | 🟡 | — | 🟡 | 🟡 | ✅ |
| gRPC / protobuf API | ❌ | ❌ | ❌ | ❌ | — | ✅ | — | — | — |

> **PowerDNS note:** pdns-api is a solid REST API but lacks live-streaming endpoints and bulk operations. Infoblox's WAPI is comprehensive but XML-heavy and expensive to maintain.

---

## 9. Performance & Architecture

| Feature | DNSDave | Pi-hole | AdGuard | Technitium | dnsmasq | CoreDNS | BIND9 | PowerDNS | Infoblox |
|---------|:-------:|:-------:|:-------:|:----------:|:-------:|:-------:|:-----:|:--------:|:--------:|
| QPS target (4-core host) | >500K | ~200K | ~150K | ~100K | ~300K | ~500K | ~400K | ~400K | >1M |
| p99 latency target (blocked/cached) | <1ms | ~1ms | ~1ms | ~2ms | <1ms | <1ms | <2ms | <1ms | <1ms |
| Zero-allocation hot path | ✅ | 🟡 | ❌ | ❌ | ✅ | ❌ | ✅ | ✅ | — |
| `recvmmsg` / `sendmmsg` batch I/O | ✅ | 🟡 | ❌ | ❌ | ❌ | ❌ | 🟡 | 🟡 | — |
| `SO_REUSEPORT` multi-socket | ✅ | ✅ | ❌ | ❌ | ❌ | ✅ | ✅ | ✅ | — |
| SIMD byte scanning (AVX2 / NEON) | ✅ | 🟡 | ❌ | ❌ | ❌ | ❌ | ❌ | ❌ | — |
| Lock-free in-memory structures | ✅ | 🟡 | ❌ | ❌ | ❌ | 🟡 | 🟡 | 🟡 | — |
| GC-free runtime | ✅ | ✅ | ❌ | ❌ | ✅ | ❌ | ✅ | ✅ | — |
| Bloom filter pre-filter for blocklist | ✅ | ❌ | ❌ | ❌ | ❌ | ❌ | — | — | — |
| MPHF blocklist map (~3 bits/entry) | ✅ | ❌ | ❌ | ❌ | — | — | — | — | — |
| Event-driven (async NATS bus) | ✅ | ❌ | ❌ | ❌ | — | ❌ | ❌ | ❌ | — |
| Stateless DNS data plane (horizontal scale) | ✅ | ❌ | ❌ | ❌ | — | ✅ | 🟡 | 🟡 | ✅ |

> **Language matters here:** Rust (DNSDave, BIND9, PowerDNS) has no garbage collector, enabling predictable low-latency operation. Go (AdGuard Home, CoreDNS) has a GC that can cause short pauses at high QPS. C# / .NET (Technitium) has a high-performance GC but still introduces latency variance.

---

## 10. Deployment & Operations

| Feature | DNSDave | Pi-hole | AdGuard | Technitium | dnsmasq | CoreDNS | BIND9 | PowerDNS | Infoblox |
|---------|:-------:|:-------:|:-------:|:----------:|:-------:|:-------:|:-----:|:--------:|:--------:|
| Docker Compose | ✅ | ✅ | ✅ | ✅ | 🟡 | ✅ | 🟡 | 🟡 | ❌ |
| Podman / Podman Compose | ✅ | 🟡 | 🟡 | 🟡 | — | ✅ | 🟡 | 🟡 | ❌ |
| Kubernetes / Helm | ✅ | 🟡 | 🟡 | ❌ | ❌ | ✅ | 🟡 | ✅ | ✅ |
| `linux/amd64` | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| `linux/arm64` (Pi 3+/4/5, Graviton) | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ❌ |
| `linux/arm/v7` (Pi 3+ 32-bit) | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | 🟡 | ❌ |
| `macOS/arm64` dev builds | ✅ | 🟡 | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ❌ |
| Horizontal DNS scaling (N nodes) | ✅ | ❌ | ❌ | ❌ | ❌ | ✅ | 🟡 | ✅ | ✅ |
| NATS JetStream HA event bus | ✅ | ❌ | ❌ | ❌ | — | — | — | — | — |
| Leader election (DHCP, sync) | ✅ | ❌ | ❌ | ❌ | — | — | ❌ | ❌ | ✅ |
| Zero-downtime config hot-swap | ✅ | 🟡 | 🟡 | 🟡 | ❌ | ✅ | ❌ | ✅ | ✅ |
| Distroless container (no shell) | ✅ | ❌ | ❌ | ❌ | 🟡 | ✅ | ❌ | ❌ | — |
| Minimal Pi 3 profile (1 GB RAM) | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | 🟡 | 🟡 | ❌ |
| IPAM (IP Address Management) | ❌ | ❌ | ❌ | ❌ | ❌ | ❌ | ❌ | ❌ | ✅ |

---

## 11. Observability

| Feature | DNSDave | Pi-hole | AdGuard | Technitium | dnsmasq | CoreDNS | BIND9 | PowerDNS | Infoblox |
|---------|:-------:|:-------:|:-------:|:----------:|:-------:|:-------:|:-----:|:--------:|:--------:|
| Prometheus metrics (native) | ✅ | ❌ | ❌ | ❌ | ❌ | ✅ | ❌ | ✅ | 🟡 |
| Per-query log (every DNS query) | ✅ | ✅ | ✅ | ✅ | 🟡 | 🟡 | ✅ | ✅ | ✅ |
| Live query stream (SSE) | ✅ | ✅ | ✅ | 🟡 | ❌ | ❌ | ❌ | ❌ | 🟡 |
| Per-client query history | ✅ | ✅ | ✅ | ✅ | ❌ | ❌ | ❌ | ❌ | ✅ |
| Per-client block rate / analytics | ✅ | ✅ | ✅ | ✅ | ❌ | ❌ | ❌ | ❌ | ✅ |
| Top-N blocked domains | ✅ | ✅ | ✅ | ✅ | ❌ | ❌ | ❌ | ❌ | ✅ |
| DHCP lease event history | ✅ | 🟡 | 🟡 | 🟡 | ❌ | — | 🟡 | 🟡 | ✅ |
| µs-granularity latency histograms | ✅ | ❌ | ❌ | ❌ | ❌ | ✅ | ❌ | ✅ | 🟡 |
| Structured JSON logs | ✅ | 🟡 | ✅ | 🟡 | ❌ | ✅ | ❌ | ✅ | ✅ |
| Query log export (CSV/JSON) | ✅ | ✅ | ✅ | ✅ | ❌ | ❌ | ❌ | ❌ | ✅ |
| **Syslog export (rsyslog, syslog-ng)** | ✅ | ❌ | ❌ | ❌ | 🟡 | 🟡 | ✅ | ✅ | ✅ |
| **OpenTelemetry (OTLP) export** | ✅ | ❌ | ❌ | ❌ | ❌ | ✅ | ❌ | 🟡 | 🟡 |
| **Grafana / Loki integration** | ✅ | 🟡 | 🟡 | ❌ | ❌ | ✅ | ❌ | 🟡 | 🟡 |
| **Elasticsearch / Logstash (ELK)** | ✅ | ❌ | ❌ | ❌ | ❌ | 🟡 | ❌ | ❌ | ✅ |
| **Splunk HEC export** | ✅ | ❌ | ❌ | ❌ | ❌ | 🟡 | ❌ | ❌ | ✅ |
| **InfluxDB line protocol export** | ✅ | ❌ | ❌ | ❌ | ❌ | 🟡 | ❌ | ❌ | 🟡 |
| **Datadog / New Relic (via OTel)** | ✅ | ❌ | ❌ | ❌ | ❌ | 🟡 | ❌ | ❌ | ✅ |
| **HTTP webhook export** | ✅ | ❌ | ❌ | ❌ | ❌ | ❌ | ❌ | ❌ | 🟡 |
| Pre-built Grafana dashboards | ✅ | 🟡 | 🟡 | ❌ | ❌ | 🟡 | ❌ | 🟡 | ✅ |
| Pre-built Kibana dashboards | ✅ | ❌ | ❌ | ❌ | ❌ | ❌ | ❌ | ❌ | ✅ |

> **Prometheus note:** Pi-hole and AdGuard Home both require third-party exporters (e.g. `pihole-exporter`) to expose Prometheus metrics. DNSDave, CoreDNS, and PowerDNS export them natively.

> **Export note:** DNSDave's `dnsdave-export` container provides native, first-party integrations with all major log aggregation and observability platforms. Other open-source DNS tools (Pi-hole, AdGuard, CoreDNS) require separate third-party agents, custom Fluent Bit pipelines, or sidecar containers to achieve the same result. Enterprise solutions (Infoblox) typically provide syslog and SIEM integrations as paid add-ons. DNSDave ships these integrations built-in.

---

## 12. User Interface

| Feature | DNSDave | Pi-hole | AdGuard | Technitium | dnsmasq | CoreDNS | BIND9 | PowerDNS | Infoblox |
|---------|:-------:|:-------:|:-------:|:----------:|:-------:|:-------:|:-----:|:--------:|:--------:|
| Web UI | ✅ | ✅ | ✅ | ✅ | ❌ | ❌ | ❌ | ❌ | ✅ |
| Mobile-responsive UI | ✅ | 🟡 | ✅ | 🟡 | — | — | — | — | 🟡 |
| Dark mode | ✅ | ✅ | ✅ | ✅ | — | — | — | — | 🟡 |
| Live query feed in UI | ✅ | ✅ | ✅ | ✅ | — | — | — | — | 🟡 |
| Command palette (Ctrl+K) | ✅ | ❌ | ❌ | ❌ | — | — | — | — | ❌ |
| Cluster / multi-node view | ✅ | ❌ | ❌ | ❌ | — | — | — | — | ✅ |
| DHCP scope pool gauge | ✅ | 🟡 | 🟡 | 🟡 | — | — | — | — | ✅ |
| DNSSEC key management UI | 🔮 | ❌ | ❌ | ✅ | — | — | — | — | ✅ |
| TLS cert management UI | ✅ | ❌ | ✅ | ✅ | — | — | — | — | ✅ |
| Analytics / time-series charts | ✅ | ✅ | ✅ | ✅ | — | — | — | — | ✅ |

---

## 13. Licensing & Cost

| Factor | DNSDave | Pi-hole | AdGuard | Technitium | dnsmasq | CoreDNS | BIND9 | PowerDNS | Infoblox |
|--------|---------|---------|---------|------------|---------|---------|-------|----------|----------|
| License | **AGPL-3.0** | EUPL-1.2 | GPL-3.0 | GPL-3.0 | GPL-2.0 | Apache-2.0 | MPL-2.0 | GPL-2.0 | Proprietary |
| Self-hosted | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| SaaS / cloud | ❌ | ❌ | ✅ | ❌ | — | — | — | — | ✅ |
| Commercial support | 🔮 | ❌ | ✅ | ❌ | ❌ | ✅ (CNCF) | ✅ (ISC) | ✅ | ✅ |
| Pricing | Free | Free | Free / ~$3/mo | Free | Free | Free | Free | Free | $15K–$100K+/yr |

---

## 14. What DNSDave Does NOT Do (v1)

Be explicit about gaps. These are features that competing products offer that DNSDave does **not plan** for v1:

| Feature | Closest alternative |
|---------|-------------------|
| **IP Address Management (IPAM)** — subnet tracking, address utilisation, network discovery | Integrate with [NetBox](https://netbox.dev) or [phpIPAM](https://phpipam.net) |
| **Full recursive resolver** — walking the DNS tree from root hints | Configure a DoH/DoT upstream like Cloudflare 1.1.1.1 or self-hosted Unbound |
| **TSIG authentication** (v1) — keyed auth for zone transfers and RFC 2136 | IP-CIDR `allow_update` / `allow_transfer` lists used instead in v1; TSIG is on the roadmap |
| **DNS-over-QUIC (DoQ)** — QUIC-based DNS transport | Planned post-v1 |
| **HSM-backed DNSSEC key storage** — hardware security module for signing keys | Signing keys are AES-GCM encrypted in Postgres; HSM is a non-goal |
| **Syslog / SNMP** — legacy monitoring integrations | Prometheus metrics + structured JSON logs cover modern stacks; SNMP not planned |
| **Active Directory / LDAP DNS integration** — Windows AD-integrated zones | Not in scope; use a Forward Zone pointing to AD DNS |
| **DHCPv6 prefix delegation (IA_PD)** (v1) | dnsmasq or ISC Kea; IA_PD is on the DNSDave roadmap |
| **Built-in mDNS bridge** — bridging `.local` multicast to unicast | Planned post-v1 |
| **BIND zone file import/export** | Planned for v0.3; not in v0.1 |

---

## 15. Product Narratives

### 15.1 DNSDave vs Pi-hole

Pi-hole is the primary inspiration for DNSDave. Both solve the same home-network and SMB problem: **block ads and telemetry at the DNS layer while providing a local resolver**. But they diverge sharply in architecture.

**Where DNSDave wins:**
- **Performance:** DNSDave's Rust + MPHF + Bloom filter hot path is designed to handle 500K+ QPS; Pi-hole FTL is fast but single-threaded DNS with Go-style allocation.
- **API quality:** DNSDave's REST API is first-class with OpenAPI spec, scoped keys, SSE streams, and bulk operations. Pi-hole's API has improved in v6 but is primarily designed for its own UI.
- **Zones and authority:** DNSDave is a full authoritative DNS server for local zones (SOA, AXFR, RFC 2136). Pi-hole has no zone concept — only a flat list of local records.
- **DHCP:** DNSDave has full DHCPv4 + DHCPv6, PXE boot, DHCP relay, vendor classes, and option hierarchy. Pi-hole's DHCP is DHCPv4-only via dnsmasq with very limited option control.
- **Kubernetes:** DNSDave is designed for Kubernetes from day one (Helm, DaemonSets, ExternalDNS). Pi-hole was designed for single-host deployment.
- **Scalability:** DNSDave scales horizontally (N stateless DNS nodes behind a VIP). Pi-hole is fundamentally single-host.
- **Pi-hole API compatibility:** DNSDave ships a compatibility shim so Pi-hole browser extensions and mobile apps continue to work unchanged.

**Where Pi-hole wins:**
- **Maturity:** Pi-hole has millions of deployments, extensive community lists, and years of production hardening.
- **Simplicity:** A one-line Docker command gives you a working Pi-hole. DNSDave requires NATS + Postgres in the full stack.
- **Ecosystem:** The Pi-hole community has built integrations, lists, and tooling that DNSDave will need time to match.

---

### 15.2 DNSDave vs AdGuard Home

AdGuard Home is the modern Pi-hole alternative. It adds DoH/DoT **server** support (not just upstream), a cleaner Go codebase, and better initial setup experience.

**Where DNSDave wins:** Zones and authority, DHCP depth (PXE, relay, classes), event-driven architecture for loose coupling, Bloom filter blocklist performance, Kubernetes-native design, open REST API.

**Where AdGuard wins:** DNS-over-QUIC server support, mature product with commercial backing and cloud offering, simpler single-binary deployment, no external NATS/Postgres dependency for basic use.

---

### 15.3 DNSDave vs Technitium DNS Server

Technitium is the most direct feature comparison in the open-source space — it has authoritative zones, DNSSEC signing, a rich UI, and ad blocking in a single product. It runs on .NET and has solid Windows support.

**Where DNSDave wins:** Performance (Rust vs .NET), event-driven architecture, horizontal scaling, Kubernetes-native, Pi-hole API compatibility, per-client SSE streams, DHCP relay and PXE, arm/v7 containers.

**Where Technitium wins:** Already ships with DNSSEC signing (DNSDave has it roadmapped for v0.4.5), Windows native support, simpler deployment (single binary, no NATS), larger feature surface in a shipping product.

---

### 15.4 DNSDave vs dnsmasq

dnsmasq is the embedded-systems and home-router staple. It does DNS forwarding and DHCPv4/v6 in under 1 MB of memory. It underlies Pi-hole, OpenWrt, and many Linux distributions.

**Where DNSDave wins:** Everything that matters beyond the basics — API, UI, zones, blocklist management, DNSSEC, observability, horizontal scaling.

**Where dnsmasq wins:** Unmatched memory footprint, battle-tested on billions of devices, DHCP relay and full option support (dnsmasq has extensive DHCP options that DNSDave matches but does not exceed), no infrastructure dependencies.

**Positioning:** dnsmasq is not a target replacement for DNSDave; it is a complement. DNSDave can use dnsmasq-style deployments on ultra-constrained hardware, or coexist with dnsmasq (e.g., DNSDave as the management plane forwarding to a dnsmasq instance on a router).

---

### 15.5 DNSDave vs CoreDNS

CoreDNS is the cloud-native DNS server powering Kubernetes cluster DNS (kube-dns). Its plugin architecture is exceptionally flexible; it can serve zones, forward queries, serve Prometheus metrics, and talk to etcd or Kubernetes APIs.

**Where DNSDave wins:** Ad blocking and blocklist management, DHCP (CoreDNS has none), Pi-hole compatibility, UI, per-client analytics, DHCP→DNS dynamic integration, ease of configuration for humans.

**Where CoreDNS wins:** Kubernetes-native service discovery (this is its primary use case), plugin ecosystem, DoH/DoT/gRPC server without configuration, extremely small footprint, trusted CNCF project.

**Positioning:** DNSDave and CoreDNS serve different needs. CoreDNS is the right tool for Kubernetes cluster DNS; DNSDave is the right tool for the network perimeter (client DNS resolution, ad blocking, DHCP). They can coexist: CoreDNS handles internal service discovery, DNSDave handles client DNS and feeds records via ExternalDNS.

---

### 15.6 DNSDave vs BIND9

BIND9 is the reference implementation of DNS and the most deployed authoritative DNS server on the internet. It implements every DNS RFC, has views, DNSSEC, TSIG, and decades of production hardening.

**Where DNSDave wins:** API (BIND has no REST API; management is via `rndc` and zone files), UI, ad blocking, DHCP integration (BIND has none; you pair it with ISC Kea separately), event-driven architecture, Kubernetes-native, developer experience.

**Where BIND9 wins:** RFC completeness, DNSSEC maturity (BIND introduced DNSSEC), full recursive resolution, TSIG, view complexity at scale, massive operational knowledge base.

**Positioning:** DNSDave is not replacing BIND9 for public-internet authoritative DNS hosting. DNSDave targets private network DNS (home, SMB, enterprise edge). For anyone running an internal DNS + DHCP stack and spending weeks on BIND zone files and ISC DHCP configs, DNSDave offers an API-driven alternative with much lower operational overhead.

---

### 15.7 DNSDave vs PowerDNS Suite

The PowerDNS suite (pdns authoritative + Recursor + dnsdist) is the enterprise open-source DNS stack. pdns has arguably the best DNSSEC implementation, a solid REST API, Lua scripting for custom policies, and a variety of database backends.

**Where DNSDave wins:** Integrated ad blocking (PowerDNS has no blocklist concept without Lua scripting), integrated DHCP with event-driven DDNS, NATS event bus for loose coupling, Pi-hole API compatibility, simpler deployment (one compose file vs configuring three separate services), web UI.

**Where PowerDNS wins:** DNSSEC maturity, Lua policy scripting, database backends (MySQL, PostgreSQL, SQLite, LDAP, etc.), commercial support (Open-Xchange), recursive resolver via pdns-recursor, dnsdist as a hardened frontend (DDoS protection, ACLs, DoH/DoT, EDNS client subnet).

**Positioning:** PowerDNS is often the right choice for enterprise DNS where a dedicated DNS ops team manages it. DNSDave targets the team that wants infrastructure-as-code DNS management without becoming DNS experts.

---

### 15.8 DNSDave vs Infoblox DDI

Infoblox is the enterprise DDI market leader. A full Infoblox deployment handles DNS + DHCP + IPAM across an entire enterprise, with hardware appliances, RBAC, reporting, threat intelligence, and SLA guarantees.

**Where DNSDave wins:** Cost (free vs $15K–$100K+/yr), open source, no vendor lock-in, API quality (Infoblox's WAPI is comprehensive but complex), cloud-native deployment, developer experience.

**Where Infoblox wins:** IPAM (DNSDave has no IPAM — this is a fundamental gap for enterprise DDI), enterprise RBAC, compliance reporting, threat intelligence feeds, hardware appliances with guaranteed availability, 24/7 support, Windows IPAM integration.

**What DNSDave explicitly lacks vs Infoblox:** IPAM. The "I" in DDI is not in scope for DNSDave v1. Organisations needing subnet management, address utilisation tracking, and network discovery should integrate DNSDave with [NetBox](https://netbox.dev) (open source) or retain Infoblox for IPAM while evaluating DNSDave for the DNS and DHCP layers.

---

## 16. Summary Positioning

```
                    FEATURE DEPTH
                         │
    Infoblox             │
    (enterprise DDI)     │
                         │         DNSDave
                         │         (target)
    PowerDNS suite       │
                         │
    Technitium DNS       │
                         │
    BIND9 + Kea          │  Pi-hole v6
                         │
    AdGuard Home         │
                         │
    dnsmasq              │
    CoreDNS              │
─────────────────────────┼─────────────────────────
    single-host          │    cluster / cloud-native
    legacy config        │    API-first
                              OPERATIONAL SIMPLICITY →
```

DNSDave targets the gap between Pi-hole (excellent filtering, limited management) and Infoblox (complete DDI, enterprise cost and complexity): a **production-grade, API-first DNS + DHCP server** that a developer or small ops team can run in Docker or Kubernetes without becoming DNS experts, while retaining the ability to scale to a multi-site HA cluster if needed.

**DNSDave is the right choice when you need:**
- Ad blocking and Pi-hole-compatible query filtering
- A local DNS server that is also authoritative for your zones
- First-class DHCPv4/v6 with DDNS integration
- A REST API you can drive from Terraform, Ansible, or CI/CD
- Horizontal scaling for DNS without external load balancers
- Multi-architecture containers that run on a Raspberry Pi and an AWS Graviton instance with the same image

**DNSDave is not the right choice when you need:**
- IPAM (subnet management, network discovery) — use Infoblox or NetBox
- A public internet authoritative DNS server at scale — use PowerDNS or Cloudflare
- A Kubernetes service discovery DNS — use CoreDNS
- A single 5 MB binary on an OpenWrt router — use dnsmasq
- Full RFC-complete recursive resolution — use BIND9 or Unbound
