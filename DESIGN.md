# DNSDave — Product Design Document

**Version:** 0.4.0-draft  
**Date:** 2026-04-04  
**Status:** Draft

---

## 1. Problem Statement

Pi-hole is the de facto home DNS sinkhole, but it carries a decade of design constraints:

- Configuration is file-based and requires SSH access or a purpose-built PHP UI.
- No stable, documented REST API — automation is fragile and unsupported.
- Wildcard DNS records are not natively supported.
- It is tightly coupled to a single-host systemd + dnsmasq stack, making HA or containerised deployments painful.
- Blocklist management is monolithic; adding or removing lists requires a full gravity rebuild.
- DHCP and DNS are managed separately with no automatic integration between them.
- A monolithic architecture makes adding new functionality require touching the core process.

DNSDave is a ground-up reimagining: **performance-first**, **API-first**, **event-driven**, **container-native**, with first-class blocklist support, a fully integrated DHCPv4/DHCPv6 server, authoritative local zone hosting, and compatibility with the existing Pi-hole ecosystem.

---

## 2. Goals

| # | Goal |
|---|------|
| G0 | **Performance is the top priority.** Blocked/cached queries must complete in <1ms; throughput >500K QPS on a 4-core host. The DNS hot path must be allocation-free. |
| G1 | Expose **every** configuration action through a stable, versioned REST API. |
| G2 | Support **wildcard DNS records** (`*.internal.example.com`) natively. |
| G3 | Ingest Pi-hole gravity blocklists and third-party list repos without modification. |
| G4 | Ship as a multi-container stack (`docker-compose.yml`); support Podman Compose and Kubernetes via Helm. |
| G5 | Support DNS-over-HTTPS (DoH) and DNS-over-TLS (DoT) upstream resolvers. |
| G6 | Provide rich per-client, per-domain query logs with live streaming. |
| G7 | Allow grouping of clients and applying different blocklist/allowlist policies per group. |
| G8 | Be drop-in compatible with existing Pi-hole browser extensions and mobile apps that use the Pi-hole API. |
| G9 | **Loose coupling via an event bus.** The DNS hot path offloads all side-effects asynchronously. New functionality is added by subscribing to the bus — zero changes to the DNS container. |
| G10 | **First-class DHCPv4 and DHCPv6 server** with full RFC option support, static reservations, PXE boot, DHCP relay, and automatic DNS record registration on lease assignment. |
| G11 | **Authoritative local DNS server** for any zone it owns. DNSDave answers authoritatively (AA flag set, SOA in authority section) for configured zones and never forwards those queries upstream. Supports zone transfers (AXFR/IXFR), NOTIFY to secondaries, and RFC 2136 dynamic updates. |
| G12 | **Conditional DNS forwarding.** Named forward zones route queries for specific domains to designated upstream resolvers, bypassing the global upstream pool and the blocklist. Essential for VPN/corporate split-DNS, multi-site internal networks, and ISP-delegated zones. |
| G13 | **DNSSEC support** at two levels: (a) *validation* — verify cryptographic signatures on responses from upstream resolvers and set the AD bit; (b) *signing* — sign owned zones with ZSK/KSK key pairs, serve DNSKEY/RRSIG/NSEC3 records, and expose DS records for delegation. |

### Non-Goals (v1)

- Building a full recursive resolver (we forward upstream for non-local zones).
- Automated DNSSEC key ceremony with HSM hardware (key operations are software-only via `ring`/`rcgen`).
- A bundled web UI (the API is the product; a reference SPA can come later).

---

## 3. Architecture Overview

### 3.1 Container Topology

DNSDave is decomposed into focused containers connected by an event bus. Each container has a single responsibility and can be scaled, replaced, or extended independently.

```mermaid
graph TD
    Clients["Client Network\nBrowsers · IoT · Mobile · Pi-hole apps · resolv.conf"]

    subgraph DP["Data Plane"]
        DNS["dnsdave-dns  (Rust)\n:53 UDP/TCP  ·  :853 DoT\nSO_REUSEPORT × NCPU  ·  recvmmsg/sendmmsg\nZero-alloc rule engine\nBloom + MPHF + moka cache + wildcard trie"]
    end

    subgraph EB["Event Bus"]
        NATS["NATS JetStream\nCore pub/sub  ·  JetStream  ·  Object Store  ·  KV Store"]
    end

    subgraph CP["Control Plane"]
        API["dnsdave-api  (Axum)\n:8080  ·  REST API  ·  Pi-hole shim"]
        SYNC["dnsdave-sync\nBlocklist downloader + parser"]
        LOG["dnsdave-log\nQuery log bulk writer"]
        STATS["dnsdave-stats\nIn-memory stats aggregator"]
        DHCP["dnsdave-dhcp  (Rust)\n:67 UDP DHCPv4  ·  :547 UDP DHCPv6\nDORA + SARR state machines\nPXE · relay · vendor classes"]
    end

    PG[("PostgreSQL\nconfig · lists · logs · leases")]

    Clients -->|"DNS UDP/TCP :53  ·  DoT :853"| DNS
    Clients -->|"DHCP :67  ·  DHCPv6 :547"| DHCP
    DNS -->|"query events\nfire & forget · MessagePack"| NATS
    NATS -->|"config hot-swap · blocklist diffs\nbloom filters · runtime KV\nlease events → dynamic DNS"| DNS
    API -->|"config change events"| NATS
    NATS -->|"query events"| LOG
    NATS -->|"query events"| STATS
    SYNC -->|"blocklist diffs · bloom filter\nready signals"| NATS
    DHCP -->|"lease events"| NATS
    NATS -->|"scope config · reservations"| DHCP
    API <-->|"reads / writes"| PG
    SYNC -->|"list entries delta"| PG
    LOG -->|"bulk inserts"| PG
    DHCP <-->|"leases · reservations"| PG
```

### 3.2 Container Responsibilities

| Container | Language | Role | Talks to |
|-----------|----------|------|----------|
| `dnsdave-dns` | **Rust** | DNS data plane. Serves queries. Zero DB access. | NATS only |
| `dnsdave-api` | Rust | REST API + Pi-hole shim. Owns all config writes. | Postgres + NATS |
| `dnsdave-sync` | Rust | Blocklist downloader + parser. | Postgres + NATS + HTTP (lists) |
| `dnsdave-log` | Rust | Query log consumer. Bulk-writes to Postgres. | NATS + Postgres |
| `dnsdave-stats` | Rust | Stats aggregator. Maintains in-memory counters. | NATS |
| `dnsdave-dhcp` | **Rust** | DHCPv4/DHCPv6 server. Publishes lease events. Dynamic DNS via NATS. | NATS + Postgres |
| `nats` | — | Event bus. Core pub/sub + JetStream + Object Store + KV Store. | — |
| `postgres` | — | Primary persistent storage. | — |

### 3.3 Key Design Principles

**The DNS container never touches the database.** It derives its entire runtime state (blocklist, local records, upstreams, allowlist, DHCP lease-generated records) from NATS. On startup it replays JetStream history to rebuild state without touching Postgres. `dnsdave-dns` can scale to N instances with no additional configuration.

**The DNS container offloads all side-effects.** Query log writes, stats updates, dynamic DNS from DHCP leases, and any future integrations are downstream consumers of NATS subjects. The hot path fires a non-blocking NATS publish and moves on.

**New functionality = new consumer.** Alerting, webhooks, InfluxDB export, SIEM integration — all are new containers subscribing to the bus. The DNS and API containers are never modified.

---

## 4. Event Bus Architecture

### 4.1 Why NATS JetStream

| Requirement | NATS JetStream |
|-------------|---------------|
| Publish latency | <100 µs on LAN |
| Docker image size | ~15 MB |
| Persistence / replay | JetStream (built-in) |
| Large binary payloads (bloom filter) | NATS Object Store (built-in) |
| Simple key-value config store | NATS KV Store (built-in) |
| Clustering / HA | Built-in (Raft-based) |
| Rust client | `async-nats` (official) |
| At-most-once delivery | Core NATS pub/sub |
| At-least-once delivery | JetStream consumers |

### 4.2 Subject Hierarchy

```
dnsdave.
├── query.
│   └── {client_group_id}            # DNS query log events
│                                    # Delivery: core NATS (at-most-once)
│
├── config.
│   ├── record.{action}              # action: created | updated | deleted
│   ├── list.{action}
│   ├── upstream.{action}
│   ├── client.{action}
│   ├── group.{action}
│   └── system.{action}
│                                    # Delivery: JetStream (at-least-once)
│                                    # Retention: all (replay from seq 0 on node start)
│
├── blocklist.
│   ├── diff.{list_id}               # domain add/remove diff batch
│   └── ready.{list_id}             # sync complete; bloom filter published
│                                    # Delivery: JetStream (at-least-once)
│
├── dhcp.
│   ├── lease.assigned.{scope_id}   # new or renewed lease (JetStream, at-least-once)
│   ├── lease.released.{scope_id}   # client sent RELEASE (JetStream)
│   ├── lease.expired.{scope_id}    # lease timer expired (JetStream)
│   └── discover.{scope_id}         # DISCOVER received (core, analytics only)
│                                    # Delivery: JetStream for lease.* ; core for discover
│
├── zone.
│   ├── serial.updated.{zone_name}  # SOA serial changed; triggers ArcSwap + NOTIFY
│   ├── record.created.{zone_name}  # record added to a zone (de-duplicates with config.record)
│   ├── record.deleted.{zone_name}  # record removed from a zone
│   └── transfer.requested.{zone_name}  # AXFR/IXFR request logged (core, analytics)
│                                    # Delivery: JetStream for serial.updated + record.*
│
├── config.
│   └── forwardzone.{action}        # forward zone created | updated | deleted
│                                    # Triggers ForwardZoneTable ArcSwap on all DNS nodes
│                                    # Delivery: JetStream (at-least-once); replayed on startup
│
├── dnssec.
│   ├── key.created.{zone_name}     # new ZSK or KSK generated
│   ├── key.retired.{zone_name}     # key rollover — old key retired
│   └── zone.signed.{zone_name}     # full zone re-sign complete; DNS nodes re-fetch DNSKEY+RRSIGs
│                                    # Delivery: JetStream (at-least-once)
│
└── upstream.
    └── health.{upstream_id}         # healthy | degraded | down (core)
```

### 4.3 JetStream Streams

| Stream | Subjects | Retention | Purpose |
|--------|----------|-----------|---------|
| `DNSDAVE_CONFIG` | `dnsdave.config.>` | All messages | Config replay for new/restarted DNS nodes |
| `DNSDAVE_BLOCKLIST` | `dnsdave.blocklist.>` | Last per subject | Blocklist diff replay |
| `DNSDAVE_DHCP` | `dnsdave.dhcp.lease.>` | 90 days | Lease history; replayed by DNS nodes for dynamic record state |
| `DNSDAVE_ZONES` | `dnsdave.zone.>` | All messages | Zone / SOA state replay — new DNS nodes self-bootstrap zone authority and serial state |
| `DNSDAVE_DNSSEC` | `dnsdave.dnssec.>` | All messages | DNSSEC key lifecycle events; zone re-sign notifications |
| `DNSDAVE_QUERYLOG` | `dnsdave.query.>` | 24h rolling | Audit / log writer catch-up |

### 4.4 NATS Object Store (Bloom Filters)

After a blocklist sync, `dnsdave-sync` serialises the bloom filter and uploads it to the `DNSDAVE_BLOOM` Object Store bucket. When a `dnsdave.blocklist.ready.*` event arrives, `dnsdave-dns` fetches the object, deserialises it, and atomically swaps the pointer.

### 4.5 NATS KV Store (Runtime Config)

Frequently-read, rarely-changed config (upstream list, client-to-group map, allowlist, DHCP scopes) is mirrored into `DNSDAVE_RUNTIME`. DNS and DHCP containers watch this bucket for changes; updates trigger copy-on-write rebuilds and atomic pointer swaps with sub-10ms propagation.

### 4.6 Query Event Schema

```rust
struct QueryEvent {
    ts_us:         u64,
    client_ip:     [u8; 16],   // IPv4-mapped IPv6
    client_id:     u32,
    domain_len:    u8,
    domain:        [u8; 253],
    qtype:         u16,
    response_type: u8,         // 0=allowed 1=blocked 2=cached 3=nxdomain 4=noerror
    matched_id:    u32,
    upstream_id:   u8,
    latency_us:    u32,
}
// ~300 bytes packed (MessagePack); NATS publish is a memcpy to client buffer
```

### 4.7 DNS Container Startup Sequence

On start, `dnsdave-dns` bootstraps its in-memory state entirely from NATS without touching Postgres:

```mermaid
sequenceDiagram
    participant DNS as dnsdave-dns
    participant KV as NATS KV Store
    participant JS as NATS JetStream
    participant OBJ as NATS Object Store

    DNS->>KV: Fetch DNSDAVE_RUNTIME bucket
    KV-->>DNS: upstream list · client/group map · allowlist · DHCP scope map
    Note over DNS: Build upstream pool, CIDR table, allowlist map

    DNS->>JS: Subscribe DNSDAVE_CONFIG from seq 0
    JS-->>DNS: Replay all config events
    Note over DNS: Build local records map + wildcard trie

    DNS->>JS: Subscribe DNSDAVE_BLOCKLIST from seq 0
    JS-->>DNS: Replay all blocklist diffs
    Note over DNS: Build blocklist hash map

    DNS->>JS: Subscribe DNSDAVE_DHCP from seq 0
    JS-->>DNS: Replay lease events (90-day window)
    Note over DNS: Build dynamic DNS records from DHCP leases

    loop per enabled list
        DNS->>OBJ: Fetch bloom.{list_id}
        OBJ-->>DNS: Serialised bloom filter bytes
        Note over DNS: Deserialise → ArcSwap
    end

    Note over DNS: Begin serving on :53
    DNS->>JS: Subscribe dnsdave.config.> (live updates)
    DNS->>JS: Subscribe dnsdave.blocklist.> (live updates)
    DNS->>JS: Subscribe dnsdave.dhcp.lease.> (live updates)
    Note over DNS: Publish dnsdave.query.* per query
```

### 4.8 Extensibility — Adding New Consumers

| Future feature | New consumer subscribes to |
|---------------|---------------------------|
| Malware domain alerting | `dnsdave.query.*`, filter on `response_type=blocked` |
| Slack / webhook notifications | Same as above |
| InfluxDB / Grafana metrics | `dnsdave.query.*`, aggregate counters |
| SIEM / audit log export | `DNSDAVE_QUERYLOG` JetStream with durable consumer |
| DHCP lease audit trail | `DNSDAVE_DHCP` JetStream with durable consumer |
| RPZ zone export | `dnsdave.blocklist.*` consumer that generates zone files |
| Rate limit enforcement | `dnsdave.query.*` → publishes block rules back via KV |

---

## 5. Performance Architecture

Performance is a first-class design constraint. Every decision in this section targets matching or exceeding Pi-hole FTL's query throughput while operating entirely in user-space Rust.

### 5.1 Hot Path — Query Lifecycle

Every DNS query follows this chain. Each stage must not allocate and must complete in nanoseconds:

```mermaid
flowchart TD
    A([UDP packet received\nSO_REUSEPORT · recvmmsg batch])
    A --> B["1  Decode DNS wire format\nborrow buffer from pool · parse in-place"]
    B --> C["2  Lowercase domain\nin-place 256-byte lookup table · no alloc"]
    C --> D{"3  Allowlist\nexact match?"}
    D -->|hit| R1(["Return NOERROR"])
    D -->|miss| E{"4  Local record\nexact match?"}
    E -->|hit| R2(["Return answer · AA=1 if in owned zone"])
    E -->|miss| F{"5  Wildcard trie\nmatch?"}
    F -->|hit| R3(["Return answer · AA=1 if in owned zone"])
    F -->|miss| G{"6  Zone authority check\nArcSwap suffix trie"}
    G -->|"in owned zone\nno record exists"| R6(["Encode NXDOMAIN + SOA\nAA=1 · Publish QueryEvent → NATS\nNEVER forward upstream"])
    G -->|"not in any owned zone"| FZ{"7  Forward zone check\nArcSwap table"}
    FZ -->|"in forward zone"| FZC{"7a  Answer cache\nhit?"}
    FZC -->|hit| R7a(["Return cached answer\nPublish QueryEvent → NATS"])
    FZC -->|miss| FZU["7b  Forward to zone-specific\nupstream(s) — bypass blocklist\nDNSSEC validate if enabled"]
    FZU --> R7b(["Cache answer · Return\nPublish QueryEvent → NATS"])
    FZ -->|"not in any forward zone"| H{"8  Bloom filter\nnegative?"}
    H -->|"negative\nguaranteed not blocked"| J
    H -->|positive| I{"9  Blocklist HashMap\nconfirmed blocked?"}
    I -->|blocked| R4(["Encode NXDOMAIN\nPublish QueryEvent → NATS\nReturn buffer"])
    I -->|false positive| J{"10  Answer cache\nhit?"}
    J -->|hit| R5(["Encode cached answer\nPublish QueryEvent → NATS\nReturn buffer"])
    J -->|miss| K["11  Forward global upstream\ntokio · HTTP/2 DoH pool\nDNSSEC validate if enabled · set DO bit"]
    K --> L(["Cache answer · Encode response\nPublish QueryEvent → NATS\nReturn buffer"])
```

All NATS publishes are **non-blocking writes to the async-nats client buffer** — no syscall, no network boundary on the hot path.

**Critical semantics:**
- **Step 6** — Owned zones never leak to upstream. Missing records return authoritative NXDOMAIN (AA=1, SOA in authority section).
- **Step 7** — Forward zones bypass the blocklist entirely. If a name falls in a configured forward zone, it goes to that zone's designated resolver(s), not the global pool. The answer cache is still consulted first (step 7a) to avoid redundant upstream calls.
- **Step 11** — When DNSSEC validation is enabled, the DO (DNSSEC OK) bit is set on upstream queries. Validated responses have the AD (Authentic Data) bit set in the reply; signature failures return SERVFAIL.

### 5.2 UDP I/O — Multi-Socket with SO_REUSEPORT

N Tokio tasks (one per CPU core) each own a UDP socket bound to `:53` with `SO_REUSEPORT`. The kernel distributes packets across sockets via consistent hash of source IP + port — no userspace lock on the receive path.

On Linux: `recvmmsg`/`sendmmsg` batch I/O (up to 64 packets per syscall).  
On macOS (dev): standard `recv_from` / `send_to` fallback.

### 5.3 Zero-Allocation Hot Path

All byte buffers are managed through `sync.Pool`-equivalent Tokio channels. Two tiers:

| Pool | Size | Use |
|------|------|-----|
| Small | 512 bytes | Standard DNS queries |
| Large | 4096 bytes | EDNS0 / large responses |

### 5.4 Lock-Free Data Structures (Rust)

All hot-path lookups use `ArcSwap` — readers load an Arc with a single atomic instruction. Writers do full copy-on-write in the background, then call `ArcSwap::store()`. No locks, no waits on the query path.

| Structure | Rust type | Update strategy |
|-----------|-----------|-----------------|
| Bloom filter | `ArcSwap<BloomFilter>` | Swap on NATS Object Store fetch |
| Blocklist map | `ArcSwap<BoomPHF<DomainKey>>` | Swap on blocklist diff |
| Allowlist set | `ArcSwap<HashSet<DomainKey>>` | Swap on config event |
| Local records map | `ArcSwap<HashMap<DomainKey, Vec<Record>>>` | Swap on config or DHCP lease event |
| Wildcard trie | `ArcSwap<LabelTrie>` | Swap on config event |
| Zone authority trie | `ArcSwap<ZoneTrie>` | Swap on `dnsdave.zone.*` config event — holds SOA per zone |
| Forward zone table | `ArcSwap<ForwardZoneTable>` | Swap on `dnsdave.config.forwardzone.*` event — suffix-matched; holds upstream list + DNSSEC policy per zone |

### 5.5 Blocklist In-Memory Layout

**Stage 1 — Bloom filter:** `k=10` xxHash3 functions; FPR ~0.001% at 5M domains; ~90MB. A negative is definitive — no further lookup needed.

**Stage 2 — Minimal Perfect Hash map:** `boomphf` eliminates all collisions. ~3 bits/entry; 5M domains ≈ 42MB. The map is immutable after construction — Go's map has no synchronisation overhead: reads are concurrent-safe because the map is never mutated.

### 5.6 Blocklist Parser — Streaming, Zero-Copy

- Stream-read HTTP response body; transparent gzip/zstd decompression.
- `memchr` crate SIMD-accelerated newline scanning (AVX2 on x86_64, NEON on ARM64).
- In-place lowercase via 256-byte lookup table; no regex; no per-line allocation.
- Multiple lists fetched and parsed concurrently with Tokio tasks.

| List | Size | Target |
|------|------|--------|
| StevenBlack unified | ~160K domains | <200ms |
| OISD big | ~1M domains | <1s |
| All lists combined (5M unique) | — | <3s cold; <1s incremental |

### 5.7 Answer Cache — moka

`moka` concurrent cache with Window TinyLFU policy, internally sharded. TTL-aware eviction; negative caching (NXDOMAIN/NODATA per RFC 2308); cache pre-warming from `DNSDAVE_QUERYLOG` replay on startup.

### 5.8 Performance Targets

| Metric | Target |
|--------|--------|
| Blocked query latency (p50) | <100 µs |
| Blocked query latency (p99) | <150 µs (no GC) |
| Cache hit latency (p50) | <150 µs |
| Cache miss latency (p50) | <5 ms (upstream RTT) |
| Throughput (blocked/cached) | >800K QPS on 4 cores |
| NATS query event publish | <1 µs (buffer write) |
| DNS container cold start | <10s (5M domain blocklist, from NATS) |
| RSS (5M domain blocklist, MPHF) | <300 MB |

---

## 6. DNS Feature Set

### 6.1 Record Types

| Type | Scope | Notes |
|------|-------|-------|
| `A` | Local zone | IPv4; auto-created from DHCP leases |
| `AAAA` | Local zone | IPv6; auto-created from DHCPv6 leases |
| `CNAME` | Local zone | Chained resolution |
| `MX` | Local zone | Mail routing |
| `TXT` | Local zone | Arbitrary; split-horizon Let's Encrypt |
| `PTR` | Local zone | Reverse DNS; auto-created from DHCP leases |
| `SRV` | Local zone | Service discovery |
| Wildcard `*` | Local zone | See §6.2 |

### 6.2 Wildcard DNS

Resolution priority:

```
exact match  >  wildcard match (longest suffix wins)  >  zone authority (NXDOMAIN if in owned zone)  >  blocklist check  >  upstream
```

A `recursive_wildcard` flag enables multi-label matching. The wildcard trie is keyed right-to-left by DNS label, copy-on-write, published via `ArcSwap`.

### 6.3 Split-Horizon / Views

Named views assign different responses to different client groups. View assignment is based on client IP/CIDR, resolved from the CIDR table on each query.

### 6.4 Upstream Resolver Configuration

DoH upstreams use persistent HTTP/2 connections (one per upstream, multiplexed). Health-checked continuously; state published to `dnsdave.upstream.health.*`.

### 6.5 Local Zone Management

DNSDave can be the **authoritative primary** (or secondary) DNS server for any zone, allowing it to act as a full local DNS server — not just a forwarder. This is distinct from simple local records: zones give DNSDave authority over an entire namespace with proper RFC-compliant semantics.

#### Zone Configuration

```jsonc
// POST /api/v1/dns/zones
{
  "name":    "home.arpa",          // RFC 8375 — canonical home network zone
  "type":    "primary",            // "primary" | "secondary"
  "ttl":     300,                  // default record TTL
  "soa": {
    "mname":   "ns1.home.arpa",    // primary nameserver FQDN
    "rname":   "admin.home.arpa",  // responsible mailbox (@ → .)
    "refresh": 3600,
    "retry":   900,
    "expire":  604800,
    "minimum": 60                  // negative caching TTL (RFC 2308)
  },
  "allow_transfer": ["192.168.0.0/16"],  // CIDRs allowed to request AXFR/IXFR
  "allow_update":   ["192.168.1.0/24"], // CIDRs allowed RFC 2136 DNS UPDATE
  "notify":         ["192.168.1.2"]     // secondaries to NOTIFY on serial change
}
```

Typical local zones:

| Zone | Purpose |
|------|---------|
| `home.arpa` | Home network forward records (RFC 8375) |
| `168.192.in-addr.arpa` | Reverse zone for 192.168.0.0/16 |
| `1.0.10.in-addr.arpa` | Reverse zone for 10.1.0.0/24 |
| `8.b.d.0.1.0.0.2.ip6.arpa` | IPv6 reverse zone |
| `internal.example.com` | Corporate split-horizon internal names |

#### SOA Serial Auto-Increment

Every mutation to records within a zone (via REST API, DHCP DDNS, or RFC 2136) atomically increments the zone's SOA serial in Postgres (format `YYYYMMDDnn`, wraps via RFC 1982 arithmetic). The change is published to `dnsdave.zone.serial.updated` and propagated to all DNS nodes via NATS. Each receiving node swaps its `ZoneTrie` via `ArcSwap`.

#### Zone Transfers (AXFR / IXFR)

`dnsdave-dns` serves zone transfers over TCP port 53. Transfers are restricted to `allow_transfer` CIDRs.

| Protocol | Description |
|----------|-------------|
| AXFR | Full zone transfer — streams all records in the zone |
| IXFR | Incremental transfer — streams only records changed since a given serial, sourced from `dns_zone_changes` |

#### NOTIFY

On every serial increment, `dnsdave-dns` sends RFC 1996 NOTIFY packets to all configured secondary nameservers in the zone's `notify` list. Retransmitted with exponential back-off until the secondary responds.

#### RFC 2136 Dynamic DNS Update

DNSDave accepts DNS UPDATE packets (RFC 2136) on port 53/TCP. This standard protocol is used by:

- **certbot / ACME DNS-01** — automatic certificate issuance
- **Kubernetes ExternalDNS** (webhook or RFC 2136 mode) — register service IPs
- **Kea DHCP** DDNS — register non-dnsdave clients
- **Ansible / Terraform** DNS modules

Updates are validated against `allow_update` CIDRs, translated to the same record-write path as the REST API, and published to `dnsdave.zone.serial.updated` on NATS so all nodes see the change within milliseconds.

#### mDNS Unicast Bridge *(future)*

A planned optional module bridges mDNS/Bonjour announcements (multicast `224.0.0.251:5353`) into a configured zone as unicast-accessible A/AAAA/PTR records. Devices that announce via mDNS (Apple TV, printers, smart speakers) become reachable as `device.home.arpa` from standard DNS clients.

### 6.6 Conditional DNS Forwarding

A **forward zone** routes queries for a specific domain suffix to a designated set of upstream resolvers, bypassing both the global upstream pool and the blocklist. This is distinct from an owned zone (where DNSDave is authoritative) — in a forward zone, DNSDave acts as a proxy to the named servers and caches their responses.

```jsonc
// POST /api/v1/dns/forwardzones
{
  "name":    "corp.example.com",
  "upstreams": [
    { "address": "10.1.1.10:53", "protocol": "udp" },
    { "address": "10.1.1.11:53", "protocol": "udp" }
  ],
  "dnssec_validate": true,    // validate DNSSEC signatures from these resolvers
  "enabled": true,
  "comment": "Corporate internal DNS via VPN"
}
```

Typical forward zone use cases:

| Zone | Upstream | Purpose |
|------|----------|---------|
| `corp.example.com` | `10.1.1.10` | Corporate internal names via VPN |
| `168.192.in-addr.arpa` | `10.1.1.10` | Reverse DNS for corporate subnet |
| `consul` | `127.0.0.1:8600` | Consul service discovery |
| `.` (catch-all) | Custom upstream pool | Override the global upstream for all non-local queries |

**Hot path behavior (step 7):** Forward zone lookup uses a right-to-left label trie (same structure as the zone authority trie, stored in `ArcSwap<ForwardZoneTable>`). Longest suffix wins. The answer cache is checked first (step 7a) before the network call (step 7b). Forward zone responses are cached with normal TTL.

**Blocklist bypass:** Queries routed through a forward zone skip the Bloom filter and Blocklist HashMap. If per-forward-zone blocklisting is required, it must be enforced by the target resolver.

### 6.7 DNSSEC

DNSDave provides DNSSEC at two levels, independently configurable:

#### 6.7.1 DNSSEC Validation (Resolver Mode)

When acting as a forwarder, DNSDave can validate DNSSEC signatures on responses from upstream resolvers.

| Setting | Behaviour |
|---------|-----------|
| `validate: strict` | Set DO bit on all upstream queries; return SERVFAIL if any signature fails |
| `validate: opportunistic` | Set DO bit; log failures but return the unsigned answer |
| `validate: disabled` | Do not set DO bit; pass responses through unchanged |

- Configurable globally via `DNSDAVE_DNSSEC_VALIDATE` env / API, and overridden per forward zone or per upstream.
- Trust anchors are loaded from the IANA root KSK (auto-updated via NATS KV) plus any custom anchors configured for internal CAs.
- Validated responses have the **AD (Authentic Data)** bit set in replies to clients.
- DNSSEC negative responses (NSEC/NSEC3 records) are validated and cached.

#### 6.7.2 DNSSEC Signing (Authoritative Mode)

Owned zones can be signed with ZSK/KSK key pairs. When `dnssec_signed: true` is set on a zone:

- **Key generation:** `POST /api/v1/dns/dnssec/keys` generates a ZSK (Ed25519, short-lived, 90-day default) and KSK (ECDSA P-384, long-lived, 1-year default). Private keys are stored encrypted in Postgres using the system's `DNSDAVE_SECRET_KEY`.
- **Inline signing:** Every record write within a signed zone triggers RRSIG regeneration for the affected RRset, published to `dnsdave.zone.serial.updated` so all DNS nodes swap their in-memory copy.
- **NSEC3:** Authenticated denial of existence uses NSEC3 (RFC 5155) with a random salt, preventing zone enumeration.
- **DNSKEY & DS records:** The DNSKEY RR is served automatically. The DS record (for pasting into the parent zone's delegation) is exposed at `GET /api/v1/dns/dnssec/zones/:name/ds`.
- **Key rollover:** ZSK rollover uses the pre-publish method (new ZSK published, old ZSK signing, then swap). Automated rollover schedule is configurable; NATS KV holds key lifecycle state.

```jsonc
// PATCH /api/v1/dns/zones/home.arpa  (enable signing on an existing zone)
{
  "dnssec_signed":    true,
  "dnssec_algorithm": "ed25519",   // "ed25519" | "ecdsa-p256" | "ecdsa-p384"
  "nsec3_iterations": 0,           // RFC 9276 recommends 0 iterations
  "nsec3_salt":       ""           // empty salt recommended per RFC 9276
}
```

---

## 7. Blocklist Management

### 7.1 Supported Formats

| Format | Example Source | Detection |
|--------|---------------|-----------|
| Pi-hole gravity (hosts) | StevenBlack/hosts | `0.0.0.0 example.com` |
| Domain-only list | OISD, Hagezi | `example.com` |
| Adblock Plus syntax | EasyList | `\|\|example.com^` |
| RPZ (Response Policy Zone) | ISC, enterprise feeds | zone file syntax |
| Plain IP blocklist | Threat intel feeds | One IP per line |

### 7.2 List Sources

```jsonc
// POST /api/v1/lists
{
  "name": "StevenBlack Unified",
  "url": "https://raw.githubusercontent.com/StevenBlack/hosts/master/hosts",
  "format": "auto",
  "enabled": true,
  "group_ids": ["default"],
  "sync_schedule": "0 3 * * *"
}
```

### 7.3 Sync & Distribution Flow

```mermaid
sequenceDiagram
    participant HTTP as List Source (HTTP)
    participant SYNC as dnsdave-sync
    participant PG as PostgreSQL
    participant JS as NATS JetStream
    participant OBJ as NATS Object Store
    participant DNS as dnsdave-dns (all nodes)

    SYNC->>HTTP: GET list URL (streaming)
    HTTP-->>SYNC: Response body (gzip inline)
    Note over SYNC: memchr SIMD parse<br/>Compute diff vs snapshot
    SYNC->>PG: Write list_entries delta
    SYNC->>JS: Publish domain diff batches<br/>dnsdave.blocklist.diff.{list_id}
    Note over SYNC: Build bloom filter + MPHF map
    SYNC->>OBJ: Upload bloom.{list_id}
    SYNC->>JS: Publish ready signal<br/>dnsdave.blocklist.ready.{list_id}

    JS-->>DNS: domain diff batches
    Note over DNS: Apply domain adds/removes
    JS-->>DNS: ready signal
    DNS->>OBJ: Fetch bloom.{list_id}
    OBJ-->>DNS: Serialised bloom filter
    Note over DNS: ArcSwap bloom + MPHF atomically<br/>Zero query downtime
```

### 7.4 Allowlist & Custom Rules

Allowlist entries are evaluated before the blocklist and cannot be overridden by lists. Allowlist changes publish to `dnsdave.config.list.*`; DNS nodes rebuild and swap the allowlist map inline.

### 7.5 Pi-hole API Compatibility Layer

A `/api/pihole/` shim in `dnsdave-api` exposes the Pi-hole v5/v6 admin API surface.

---

## 8. DHCP Feature Set

### 8.1 Overview

`dnsdave-dhcp` is a first-class DHCPv4 and DHCPv6 server integrated with the DNS data plane via NATS JetStream. When a lease is assigned, DNS `A`/`AAAA` and `PTR` records are automatically created in all `dnsdave-dns` instances — no manual registration, no polling. Releasing or expiring a lease removes the records.

### 8.2 Protocol Support

| Protocol | State Machine | Transport |
|----------|--------------|-----------|
| DHCPv4 | DISCOVER → OFFER → REQUEST → ACK/NAK (DORA) | UDP :67 broadcast + unicast |
| DHCPv6 | SOLICIT → ADVERTISE → REQUEST → REPLY (SARR) | UDP :547 multicast + unicast |
| DHCPv6 IA_NA | Non-temporary address assignment | — |
| DHCPv6 IA_PD | Prefix delegation | — |
| PXE (BIOS + UEFI) | Auto-detection via option 93 | — |
| DHCP Relay | RFC 3046 giaddr + option 82 | — |

### 8.3 Scopes and Pools

A scope defines a subnet with an allocatable address pool:

```jsonc
// POST /api/v1/dhcp/scopes
{
  "name": "LAN",
  "subnet": "192.168.1.0/24",
  "pool": { "start": "192.168.1.100", "end": "192.168.1.200" },
  "gateway": "192.168.1.1",
  "lease_time_s": 86400,
  "domain_name": "home.arpa",
  "domain_search": ["home.arpa", "corp.example.com"],
  "dns_servers": ["192.168.1.53"],   // auto-set to dnsdave-dns IP if omitted
  "ntp_servers": ["192.168.1.1"],
  "enabled": true
}
```

Multiple non-overlapping scopes per instance. Scope selection for relay clients uses `giaddr`.

### 8.4 DHCP Options Hierarchy

Options are evaluated in this priority order (highest wins):

```
Reservation options
    > Client Class options
        > Scope options
            > Global options
```

This allows precise targeting — Cisco switches in scope `LAN` get option 66 (TFTP server); all other clients in the same scope do not.

### 8.5 Supported DHCPv4 Options

All standard RFC options are supported. Key options with first-class handling:

| Code | Name | Notes |
|------|------|-------|
| 1 | Subnet Mask | Auto-derived from scope subnet |
| 3 | Router | Default gateway |
| 6 | DNS Servers | Auto-set to `dnsdave-dns` IP if not overridden |
| 12 | Hostname | From reservation, or client-supplied and sanitised |
| 15 | Domain Name | From scope |
| 26 | Interface MTU | Configurable per scope |
| 28 | Broadcast Address | Auto-derived |
| 33 | Static Routes | Legacy format |
| 42 | NTP Servers | Pool or specific IPs |
| 43 | Vendor-Specific | Raw bytes; applied per vendor class |
| 44 | NBNS (WINS) Servers | Legacy Windows environments |
| 51 | Lease Time | From scope or reservation |
| 58 | Renewal Time (T1) | Default 50% of lease time |
| 59 | Rebinding Time (T2) | Default 87.5% of lease time |
| 66 | TFTP Server Name | PXE boot |
| 67 | Bootfile Name | PXE; auto BIOS (`pxelinux.0`) vs UEFI (`bootx64.efi`) via option 93 |
| 93 | Client System Architecture | Auto-detect BIOS vs UEFI for PXE |
| 119 | Domain Search List | Multi-domain search |
| 121 | Classless Static Routes | RFC 3442; preferred over option 33 |
| 150 | Cisco TFTP Server | Cisco-specific PXE |
| 252 | WPAD | Web proxy auto-discovery |
| Any | Custom (raw hex) | Vendor-specific / exotic options; stored and served as-is |

### 8.6 Supported DHCPv6 Options

| Code | Name | Notes |
|------|------|-------|
| 23 | DNS Recursive Name Server | Auto-set to `dnsdave-dns` IPv6 if not overridden |
| 24 | Domain Search List | — |
| 31 | SNTP Servers | — |
| 56 | NTP Server | RFC 5908 |
| 59 | Boot File URL | PXEv6 |
| 60 | Boot File Parameters | PXEv6 arguments |

### 8.7 Static Reservations

A reservation binds a MAC address (DHCPv4) or DUID (DHCPv6) to a fixed IP and optional per-host options:

```jsonc
// POST /api/v1/dhcp/reservations
{
  "name": "NAS",
  "mac": "aa:bb:cc:dd:ee:ff",
  "ip": "192.168.1.10",
  "hostname": "nas",
  "scope_id": "scope_lan",
  "options": {
    "42": ["192.168.1.1"],      // dedicated NTP for this host
    "66": "192.168.1.10"        // TFTP (PXE) for this host only
  }
}
```

### 8.8 Client Classes and Vendor Matching

Client classes allow conditional option delivery based on DHCP option values:

```jsonc
// POST /api/v1/dhcp/classes
{
  "name": "PXEClient-UEFI",
  "match": "option[93].hex == '0007'",  // architecture = EFI BC
  "options": {
    "67": "bootx64.efi"
  }
}
```

Built-in class matchers: vendor-class-id prefix, option 93 architecture, hardware type, relay circuit ID.

### 8.9 PXE / Network Boot

PXE is configured per-scope or per-reservation with automatic BIOS/UEFI detection:

```jsonc
{
  "pxe": {
    "tftp_server": "192.168.1.10",
    "boot_file_bios": "pxelinux.0",
    "boot_file_uefi": "bootx64.efi",
    "architecture_detection": true    // reads option 93; no separate scopes needed
  }
}
```

### 8.10 DHCP Relay Support

Clients on remote subnets reach `dnsdave-dhcp` via any RFC 3046-compliant relay agent. The server reads `giaddr` to select the correct scope automatically. Option 82 (agent circuit ID) is logged with each lease for full auditability.

### 8.11 Dynamic DNS Registration

When a lease is assigned, DNS records are created automatically across all DNS nodes via NATS:

```mermaid
sequenceDiagram
    participant C as DHCP Client
    participant DHCP as dnsdave-dhcp
    participant PG as PostgreSQL
    participant JS as NATS JetStream
    participant DNS as dnsdave-dns (all nodes)

    C->>DHCP: DHCPDISCOVER
    DHCP->>PG: Allocate IP · write pending lease
    DHCP->>C: DHCPOFFER
    C->>DHCP: DHCPREQUEST
    DHCP->>PG: Confirm lease · write dhcp_leases
    DHCP->>JS: Publish dnsdave.dhcp.lease.assigned.{scope_id}
    DHCP->>C: DHCPACK (IP + all options)

    JS-->>DNS: lease.assigned event (all nodes simultaneously)
    Note over DNS: Create A: hostname.domain → IP (TTL = lease_time_s)<br/>Create PTR: reverse.in-addr.arpa → hostname.domain<br/>ArcSwap local records map
```

On lease expiry or DHCPRELEASE, a `lease.expired` / `lease.released` event causes all DNS nodes to remove the corresponding records.

### 8.12 DHCP Lease Event Schema

```rust
struct LeaseEvent {
    ts_us:       u64,
    event_type:  u8,        // 0=assigned 1=released 2=expired 3=renewed
    scope_id:    u32,
    mac:         [u8; 6],
    ip:          [u8; 16],  // IPv4-mapped IPv6
    hostname:    [u8; 64],
    domain:      [u8; 128],
    lease_end_s: u64,       // unix timestamp; 0 = released/expired
    is_v6:       bool,
    client_id:   [u8; 128], // DUID for DHCPv6
}
// ~420 bytes packed (MessagePack)
```

---

## 9. REST API Design

### 9.1 Principles

- **OpenAPI 3.1** spec is the source of truth; server stubs are generated from it.
- All endpoints are under `/api/v1/`.
- JSON in and out; errors use [RFC 9457 Problem Details](https://www.rfc-editor.org/rfc/rfc9457).
- Pagination via `?page=` + `?per_page=`; filtering via `?q=` and `?type=`.
- Bulk operations supported on all list/record endpoints.
- Every mutating operation is idempotent via `PUT`; `POST` creates-or-updates by natural key.
- After every write, `dnsdave-api` publishes the appropriate `dnsdave.config.*` event to NATS before returning 200.

### 9.2 Resource Map

```
/api/v1/
├── auth/
│   ├── POST   token
│   └── DELETE token
├── dns/
│   ├── zones/
│   │   ├── GET    /                  # list all zones
│   │   ├── POST   /                  # create zone (primary or secondary)
│   │   ├── GET    /:name             # zone detail including SOA
│   │   ├── PUT    /:name             # update zone config / SOA
│   │   ├── DELETE /:name             # remove zone (and all its records)
│   │   └── POST   /:name/notify      # manually trigger NOTIFY to secondaries
│   ├── records/
│   │   ├── GET    /
│   │   ├── POST   /
│   │   ├── GET    /:id
│   │   ├── PUT    /:id
│   │   └── DELETE /:id
│   ├── wildcards/
│   ├── upstreams/
│   ├── views/
│   ├── forwardzones/
│   │   ├── GET    /                  # list all forward zones
│   │   ├── POST   /                  # create forward zone
│   │   ├── GET    /:name
│   │   ├── PUT    /:name
│   │   └── DELETE /:name
│   └── dnssec/
│       ├── keys/
│       │   ├── GET    /              # list all DNSSEC keys (all zones)
│       │   ├── POST   /              # generate ZSK or KSK for a zone
│       │   ├── GET    /:id
│       │   └── DELETE /:id           # retire / revoke key
│       └── zones/
│           └── GET    /:name/ds      # export DS record for parent zone delegation
├── lists/
│   ├── GET    /
│   ├── POST   /
│   ├── GET    /:id
│   ├── PUT    /:id
│   ├── DELETE /:id
│   └── POST   /:id/sync
├── dhcp/
│   ├── scopes/
│   │   ├── GET    /
│   │   ├── POST   /
│   │   ├── GET    /:id
│   │   ├── PUT    /:id
│   │   └── DELETE /:id
│   ├── options/                  # global + per-scope DHCP options
│   │   ├── GET    /
│   │   ├── POST   /
│   │   ├── PUT    /:id
│   │   └── DELETE /:id
│   ├── reservations/
│   │   ├── GET    /
│   │   ├── POST   /
│   │   ├── GET    /:id
│   │   ├── PUT    /:id
│   │   └── DELETE /:id
│   ├── classes/                  # vendor / client class matching rules
│   │   ├── GET    /
│   │   ├── POST   /
│   │   ├── PUT    /:id
│   │   └── DELETE /:id
│   └── leases/
│       ├── GET    /              # active lease table (paginated)
│       ├── DELETE /:id           # force-release a lease
│       └── GET    /stream        # SSE stream of lease events
├── clients/
├── groups/
├── logs/
│   ├── GET    /                  # paginated query log
│   └── GET    /stream            # SSE live tail (NATS dnsdave.query.*)
├── stats/
│   ├── GET    /summary
│   ├── GET    /top-domains
│   ├── GET    /top-blocked
│   └── GET    /clients
└── system/
    ├── GET    /health
    ├── GET    /ready
    ├── GET    /version
    ├── POST   /flush-cache
    └── GET    /config
```

### 9.3 Authentication

| Method | Use Case |
|--------|----------|
| `X-API-Key: <key>` header | Automation, scripts, integrations |
| `Authorization: Bearer <jwt>` | Short-lived sessions, UI |
| mTLS (optional) | Kubernetes service-to-service |

### 9.4 Live Streams

`GET /api/v1/logs/stream` — SSE; subscribes to `dnsdave.query.*` in NATS. Zero Postgres reads.  
`GET /api/v1/dhcp/leases/stream` — SSE; subscribes to `dnsdave.dhcp.lease.*` in NATS. Live lease activity.

---

## 10. Data Model

### 10.1 Core Tables

```sql
CREATE TABLE dns_zones (
    name        TEXT PRIMARY KEY,         -- zone apex, e.g. "home.arpa"
    zone_type   TEXT NOT NULL DEFAULT 'primary',  -- primary | secondary
    default_ttl INTEGER NOT NULL DEFAULT 300,
    -- SOA fields
    soa_mname   TEXT NOT NULL,            -- primary nameserver FQDN
    soa_rname   TEXT NOT NULL,            -- responsible mailbox (@ → .)
    soa_serial  BIGINT NOT NULL DEFAULT 2026010101,
    soa_refresh INTEGER NOT NULL DEFAULT 3600,
    soa_retry   INTEGER NOT NULL DEFAULT 900,
    soa_expire  INTEGER NOT NULL DEFAULT 604800,
    soa_minimum INTEGER NOT NULL DEFAULT 60,
    -- Transfer / update control
    allow_transfer  JSONB NOT NULL DEFAULT '[]',  -- list of CIDRs
    allow_update    JSONB NOT NULL DEFAULT '[]',  -- CIDRs for RFC 2136 UPDATE
    notify          JSONB NOT NULL DEFAULT '[]',  -- list of secondary IPs to NOTIFY
    -- DNSSEC signing config (requires dns_dnssec_keys rows)
    dnssec_signed       BOOLEAN DEFAULT FALSE,
    dnssec_algorithm    TEXT,                     -- 'ed25519' | 'ecdsa-p256' | 'ecdsa-p384'
    nsec3_iterations    INTEGER DEFAULT 0,        -- RFC 9276: 0 recommended
    nsec3_salt          TEXT DEFAULT '',          -- RFC 9276: empty salt recommended
    enabled     BOOLEAN DEFAULT TRUE,
    comment     TEXT,
    created_at  TIMESTAMP DEFAULT NOW(),
    updated_at  TIMESTAMP DEFAULT NOW()
);

-- Records that are within a zone are linked by zone_name for AXFR/IXFR generation.
-- Records without a zone_name are "floating" local records (forward-only if not in any zone).
CREATE TABLE dns_records (
    id          TEXT PRIMARY KEY,
    zone_name   TEXT REFERENCES dns_zones(name) ON DELETE CASCADE,
    view_id     TEXT REFERENCES views(id) ON DELETE SET NULL,
    name        TEXT NOT NULL,
    type        TEXT NOT NULL,
    value       TEXT NOT NULL,
    priority    INTEGER DEFAULT 0,
    ttl         INTEGER DEFAULT 300,
    wildcard    BOOLEAN DEFAULT FALSE,
    recursive   BOOLEAN DEFAULT FALSE,
    enabled     BOOLEAN DEFAULT TRUE,
    source      TEXT DEFAULT 'manual',   -- manual | dhcp_lease | rfc2136
    lease_id    TEXT,                    -- FK to dhcp_leases if source=dhcp_lease
    comment     TEXT,
    created_at  TIMESTAMP DEFAULT NOW(),
    updated_at  TIMESTAMP DEFAULT NOW(),
    UNIQUE(zone_name, view_id, name, type)
);

-- IXFR change log: every record add/remove in a zone is appended here.
-- Allows serving IXFR (incremental zone transfer) to secondary nameservers.
CREATE TABLE dns_zone_changes (
    id          BIGSERIAL PRIMARY KEY,
    zone_name   TEXT NOT NULL REFERENCES dns_zones(name) ON DELETE CASCADE,
    serial      BIGINT NOT NULL,          -- SOA serial at time of change
    action      TEXT NOT NULL,            -- 'add' | 'remove'
    name        TEXT NOT NULL,
    type        TEXT NOT NULL,
    value       TEXT NOT NULL,
    ttl         INTEGER NOT NULL,
    changed_at  TIMESTAMP DEFAULT NOW()
);
CREATE INDEX idx_zone_changes_zone_serial ON dns_zone_changes(zone_name, serial);

-- Forward zones: queries for these suffixes bypass the blocklist and go to specific upstreams.
-- "upstreams" is a JSONB array of {address, protocol} objects.
CREATE TABLE dns_forward_zones (
    name             TEXT PRIMARY KEY,    -- zone suffix, e.g. "corp.example.com" or "." (catch-all)
    upstreams        JSONB NOT NULL,       -- [{address:"10.1.1.10:53", protocol:"udp"}, ...]
    dnssec_validate  TEXT NOT NULL DEFAULT 'disabled',  -- 'strict' | 'opportunistic' | 'disabled'
    enabled          BOOLEAN DEFAULT TRUE,
    comment          TEXT,
    created_at       TIMESTAMP DEFAULT NOW(),
    updated_at       TIMESTAMP DEFAULT NOW()
);

-- DNSSEC key store. Private keys are AES-256-GCM encrypted with DNSDAVE_SECRET_KEY.
CREATE TABLE dns_dnssec_keys (
    id               TEXT PRIMARY KEY,
    zone_name        TEXT NOT NULL REFERENCES dns_zones(name) ON DELETE CASCADE,
    key_type         TEXT NOT NULL,       -- 'ksk' | 'zsk'
    algorithm        TEXT NOT NULL,       -- 'ed25519' | 'ecdsa-p256' | 'ecdsa-p384'
    key_tag          INTEGER NOT NULL,    -- DNS key tag (RFC 4034 §B)
    public_key_b64   TEXT NOT NULL,       -- Base64 public key (DNSKEY RDATA)
    private_key_enc  TEXT NOT NULL,       -- AES-GCM encrypted private key
    is_active        BOOLEAN DEFAULT TRUE,
    created_at       TIMESTAMP DEFAULT NOW(),
    retire_at        TIMESTAMP,           -- NULL = no planned retirement
    retired_at       TIMESTAMP           -- NULL = still in use
);

CREATE TABLE lists (
    id              TEXT PRIMARY KEY,
    name            TEXT NOT NULL,
    url             TEXT NOT NULL,
    format          TEXT DEFAULT 'auto',
    list_type       TEXT DEFAULT 'block',
    enabled         BOOLEAN DEFAULT TRUE,
    sync_schedule   TEXT,
    last_synced_at  TIMESTAMP,
    last_count      INTEGER DEFAULT 0,
    comment         TEXT,
    created_at      TIMESTAMP DEFAULT NOW(),
    updated_at      TIMESTAMP DEFAULT NOW()
);

CREATE TABLE list_entries (
    id        TEXT PRIMARY KEY,
    list_id   TEXT NOT NULL REFERENCES lists(id) ON DELETE CASCADE,
    domain    TEXT NOT NULL,
    enabled   BOOLEAN DEFAULT TRUE
);
CREATE INDEX idx_list_entries_domain ON list_entries(domain);

CREATE TABLE clients (
    id         TEXT PRIMARY KEY,
    identifier TEXT NOT NULL UNIQUE,
    name       TEXT,
    group_id   TEXT REFERENCES groups(id),
    comment    TEXT,
    created_at TIMESTAMP DEFAULT NOW()
);

CREATE TABLE groups (
    id         TEXT PRIMARY KEY,
    name       TEXT NOT NULL UNIQUE,
    comment    TEXT,
    created_at TIMESTAMP DEFAULT NOW()
);

CREATE TABLE group_lists (
    group_id TEXT REFERENCES groups(id) ON DELETE CASCADE,
    list_id  TEXT REFERENCES lists(id) ON DELETE CASCADE,
    PRIMARY KEY (group_id, list_id)
);

CREATE TABLE upstreams (
    id                      TEXT PRIMARY KEY,
    name                    TEXT,
    type                    TEXT NOT NULL,
    address                 TEXT NOT NULL,
    priority                INTEGER DEFAULT 10,
    timeout_ms              INTEGER DEFAULT 2000,
    health_check_interval_s INTEGER DEFAULT 30,
    enabled                 BOOLEAN DEFAULT TRUE,
    created_at              TIMESTAMP DEFAULT NOW()
);

-- Written by dnsdave-log; partitioned by day in PostgreSQL
CREATE TABLE query_log (
    id            TEXT PRIMARY KEY,
    ts            TIMESTAMP NOT NULL,
    client_ip     TEXT NOT NULL,
    client_id     TEXT,
    domain        TEXT NOT NULL,
    qtype         TEXT NOT NULL,
    response_type TEXT NOT NULL,
    matched_list  TEXT,
    matched_rule  TEXT,
    upstream_id   TEXT,
    latency_us    INTEGER,
    answer        TEXT
) PARTITION BY RANGE (ts);

CREATE TABLE blocklist_index (
    domain    TEXT NOT NULL,
    list_id   TEXT NOT NULL REFERENCES lists(id) ON DELETE CASCADE,
    PRIMARY KEY (domain, list_id)
);
CREATE INDEX idx_blocklist_domain ON blocklist_index(domain);
```

### 10.2 DHCP Tables

```sql
CREATE TABLE dhcp_scopes (
    id           TEXT PRIMARY KEY,
    name         TEXT NOT NULL UNIQUE,
    subnet       CIDR NOT NULL,
    pool_start   INET NOT NULL,
    pool_end     INET NOT NULL,
    gateway      INET,
    lease_time_s INTEGER NOT NULL DEFAULT 86400,
    domain_name  TEXT,
    enabled      BOOLEAN DEFAULT TRUE,
    comment      TEXT,
    created_at   TIMESTAMP DEFAULT NOW(),
    updated_at   TIMESTAMP DEFAULT NOW()
);

-- Global options: scope_id IS NULL. Scope options: scope_id set.
CREATE TABLE dhcp_options (
    id       TEXT PRIMARY KEY,
    scope_id TEXT REFERENCES dhcp_scopes(id) ON DELETE CASCADE,
    code     INTEGER NOT NULL,
    value    TEXT NOT NULL,    -- JSON-encoded; interpreted by option type
    enabled  BOOLEAN DEFAULT TRUE,
    UNIQUE(scope_id, code)
);

CREATE TABLE dhcp_classes (
    id         TEXT PRIMARY KEY,
    name       TEXT NOT NULL UNIQUE,
    match_expr TEXT NOT NULL,  -- e.g. "option[93].hex == '0007'"
    comment    TEXT,
    created_at TIMESTAMP DEFAULT NOW()
);

CREATE TABLE dhcp_class_options (
    class_id TEXT NOT NULL REFERENCES dhcp_classes(id) ON DELETE CASCADE,
    code     INTEGER NOT NULL,
    value    TEXT NOT NULL,
    PRIMARY KEY (class_id, code)
);

CREATE TABLE dhcp_reservations (
    id         TEXT PRIMARY KEY,
    scope_id   TEXT NOT NULL REFERENCES dhcp_scopes(id) ON DELETE CASCADE,
    name       TEXT,
    mac        MACADDR NOT NULL,
    duid       TEXT,            -- DHCPv6 DUID (hex string)
    ip         INET NOT NULL,
    hostname   TEXT,
    enabled    BOOLEAN DEFAULT TRUE,
    comment    TEXT,
    created_at TIMESTAMP DEFAULT NOW(),
    UNIQUE(scope_id, mac)
);

CREATE TABLE dhcp_reservation_options (
    reservation_id TEXT NOT NULL REFERENCES dhcp_reservations(id) ON DELETE CASCADE,
    code           INTEGER NOT NULL,
    value          TEXT NOT NULL,
    PRIMARY KEY (reservation_id, code)
);

CREATE TABLE dhcp_leases (
    id             TEXT PRIMARY KEY,
    scope_id       TEXT NOT NULL REFERENCES dhcp_scopes(id),
    mac            MACADDR NOT NULL,
    duid           TEXT,
    ip             INET NOT NULL UNIQUE,
    hostname       TEXT,
    reservation_id TEXT REFERENCES dhcp_reservations(id),
    assigned_at    TIMESTAMP NOT NULL DEFAULT NOW(),
    expires_at     TIMESTAMP NOT NULL,
    renewed_at     TIMESTAMP,
    released_at    TIMESTAMP,
    relay_ip       INET,
    agent_circuit_id TEXT,     -- option 82 for auditing
    is_v6          BOOLEAN NOT NULL DEFAULT FALSE
);
CREATE INDEX idx_dhcp_leases_mac      ON dhcp_leases(mac);
CREATE INDEX idx_dhcp_leases_ip       ON dhcp_leases(ip);
CREATE INDEX idx_dhcp_leases_expires  ON dhcp_leases(expires_at);
CREATE INDEX idx_dhcp_leases_scope    ON dhcp_leases(scope_id);
```

### 10.3 In-Memory Structures (dnsdave-dns)

| Structure | Rust Type | Source | Includes |
|-----------|-----------|--------|---------|
| Bloom filter | `ArcSwap<BloomFilter>` | NATS Object Store | — |
| Blocklist MPHF map | `ArcSwap<BoomPHF<DomainKey>>` | NATS Object Store / diff replay | — |
| Allowlist set | `ArcSwap<HashSet<DomainKey>>` | NATS KV | — |
| Local records map | `ArcSwap<HashMap<DomainKey, Vec<Record>>>` | NATS JetStream replay | Manual records + DHCP lease records |
| Wildcard trie | `ArcSwap<LabelTrie>` | NATS JetStream replay | — |
| Client CIDR table | `ArcSwap<Vec<(IpNet, ClientId)>>` | NATS KV | — |
| Answer cache | `moka::Cache<CacheKey, CachedAnswer>` | Local upstream queries | — |
| Query event ring buffer | Lock-free MPSC | NATS fallback | — |

**Memory sizing (5M-domain blocklist + 500 DHCP leases):**

| Component | Estimate |
|-----------|----------|
| Bloom filter | ~90 MB |
| MPHF blocklist index | ~42 MB |
| moka answer cache | ~150 MB |
| Local records + trie (incl. lease records) | <10 MB |
| Zone authority trie + SOA table | <1 MB |
| **Total RSS** | **~301 MB** |

---

## 11. Deployment

### 11.1 Docker Compose (Reference Stack)

```yaml
# docker-compose.yml
services:

  nats:
    image: nats:alpine
    container_name: dnsdave-nats
    restart: unless-stopped
    command: ["-js", "-m", "8222"]
    ports:
      - "4222:4222"
      - "8222:8222"
    volumes:
      - nats-data:/data
    networks:
      - dnsdave

  postgres:
    image: postgres:16-alpine
    container_name: dnsdave-postgres
    restart: unless-stopped
    environment:
      POSTGRES_DB: dnsdave
      POSTGRES_USER: dnsdave
      POSTGRES_PASSWORD: "${POSTGRES_PASSWORD}"
    volumes:
      - pg-data:/var/lib/postgresql/data
    networks:
      - dnsdave

  dnsdave-dns:
    image: ghcr.io/dnsdave/dnsdave-dns:latest
    container_name: dnsdave-dns
    restart: unless-stopped
    depends_on: [nats]
    ports:
      - "53:53/udp"
      - "53:53/tcp"
      - "853:853/tcp"
    environment:
      DNSDAVE_NATS_URL: "nats://nats:4222"
      DNSDAVE_LOG_LEVEL: "info"
      DNSDAVE_WORKERS: "0"
    networks:
      - dnsdave
    sysctls:
      net.core.rmem_max: "26214400"
      net.core.wmem_max: "26214400"

  dnsdave-dhcp:
    image: ghcr.io/dnsdave/dnsdave-dhcp:latest
    container_name: dnsdave-dhcp
    restart: unless-stopped
    depends_on: [nats, postgres]
    ports:
      - "67:67/udp"     # DHCPv4
      - "547:547/udp"   # DHCPv6
    environment:
      DNSDAVE_NATS_URL: "nats://nats:4222"
      DNSDAVE_DB_URL: "postgres://dnsdave:${POSTGRES_PASSWORD}@postgres/dnsdave"
      DNSDAVE_LOG_LEVEL: "info"
    networks:
      - dnsdave
      - host_network    # needs access to broadcast domain

  dnsdave-api:
    image: ghcr.io/dnsdave/dnsdave-api:latest
    container_name: dnsdave-api
    restart: unless-stopped
    depends_on: [nats, postgres]
    ports:
      - "8080:8080/tcp"
    environment:
      DNSDAVE_NATS_URL: "nats://nats:4222"
      DNSDAVE_DB_URL: "postgres://dnsdave:${POSTGRES_PASSWORD}@postgres/dnsdave"
      DNSDAVE_API_KEY: "${DNSDAVE_API_KEY}"
      DNSDAVE_LOG_LEVEL: "info"
    networks:
      - dnsdave

  dnsdave-sync:
    image: ghcr.io/dnsdave/dnsdave-sync:latest
    container_name: dnsdave-sync
    restart: unless-stopped
    depends_on: [nats, postgres]
    environment:
      DNSDAVE_NATS_URL: "nats://nats:4222"
      DNSDAVE_DB_URL: "postgres://dnsdave:${POSTGRES_PASSWORD}@postgres/dnsdave"
      DNSDAVE_LOG_LEVEL: "info"
    networks:
      - dnsdave

  dnsdave-log:
    image: ghcr.io/dnsdave/dnsdave-log:latest
    container_name: dnsdave-log
    restart: unless-stopped
    depends_on: [nats, postgres]
    environment:
      DNSDAVE_NATS_URL: "nats://nats:4222"
      DNSDAVE_DB_URL: "postgres://dnsdave:${POSTGRES_PASSWORD}@postgres/dnsdave"
      DNSDAVE_BATCH_SIZE: "500"
      DNSDAVE_BATCH_INTERVAL_MS: "100"
    networks:
      - dnsdave

  dnsdave-stats:
    image: ghcr.io/dnsdave/dnsdave-stats:latest
    container_name: dnsdave-stats
    restart: unless-stopped
    depends_on: [nats]
    environment:
      DNSDAVE_NATS_URL: "nats://nats:4222"
      DNSDAVE_LOG_LEVEL: "info"
    networks:
      - dnsdave

volumes:
  nats-data:
  pg-data:

networks:
  dnsdave:
    driver: bridge
  host_network:
    driver: host   # dnsdave-dhcp needs to receive broadcast DISCOVER packets
```

> **Note on DHCP networking:** DHCP DISCOVER packets are UDP broadcast and cannot traverse Docker bridge networks by default. `dnsdave-dhcp` should use `network_mode: host` or be run with `--network host` so it receives raw broadcast traffic on port 67. In Kubernetes, use `hostNetwork: true` on the DHCP pod.

### 11.2 Minimal Stack

A `docker-compose.minimal.yml` runs only `dnsdave-dns`, `dnsdave-dhcp`, `dnsdave-api`, and NATS with SQLite. Query logging is written directly by `dnsdave-api`. Stats are disabled.

### 11.3 Podman Compose

The same compose files work with `podman-compose`. Quadlet-compatible unit files are provided for rootless Podman. `dnsdave-dhcp` requires `--cap-add=NET_BIND_SERVICE` and host network access for broadcast.

### 11.4 Kubernetes / Helm

```
deploy/helm/dnsdave/
└── templates/
    ├── daemonset-dns.yaml        # hostNetwork: true, :53
    ├── daemonset-dhcp.yaml       # hostNetwork: true, :67/:547
    ├── deployment-api.yaml       # Deployment, replicated
    ├── deployment-sync.yaml      # Deployment, single replica
    ├── deployment-log.yaml       # Deployment
    ├── deployment-stats.yaml     # Deployment, single replica
    ├── statefulset-nats.yaml     # 3-node JetStream cluster
    ├── service-dns.yaml
    ├── service-api.yaml
    ├── ingress-api.yaml
    ├── hpa-api.yaml
    └── servicemonitor.yaml
```

Both `dnsdave-dns` and `dnsdave-dhcp` run as **DaemonSets** with `hostNetwork: true`. DNS binds port 53; DHCP binds ports 67/547 to receive broadcasts directly on each node.

### 11.5 Scaling the Stack

| Container | Scale-out strategy |
|-----------|-------------------|
| `dnsdave-dns` | Add nodes; each subscribes to NATS independently. No coordination. |
| `dnsdave-dhcp` | Single active instance per broadcast domain. Leader election via NATS KV prevents duplicate offers. |
| `dnsdave-api` | Scale replicas behind a load balancer. All share Postgres + NATS. |
| `dnsdave-log` | Single replica. Scale only if Postgres write throughput saturates. |
| `dnsdave-stats` | Single replica (in-memory). HA via NATS KV leader election. |
| `dnsdave-sync` | Single replica. Leader election via NATS KV prevents duplicate syncs. |
| `nats` | 3-node JetStream cluster for HA. |
| `postgres` | Patroni HA. TimescaleDB for `query_log` at very high QPS. |

### 11.6 Configuration Priority

```
CLI flags > Environment Variables > Config file (YAML) > Defaults
```

---

## 12. Observability

### 12.1 Metrics (Prometheus)

**dnsdave-dns:**

| Metric | Type | Labels |
|--------|------|--------|
| `dnsdave_dns_queries_total` | Counter | `response_type`, `qtype` |
| `dnsdave_dns_query_duration_us` | Histogram | `response_type` |
| `dnsdave_dns_blocked_total` | Counter | — |
| `dnsdave_dns_cache_hits_total` | Counter | — |
| `dnsdave_dns_bloom_false_positives_total` | Counter | — |
| `dnsdave_dns_nats_publish_errors_total` | Counter | — |

**dnsdave-dhcp:**

| Metric | Type | Labels |
|--------|------|--------|
| `dnsdave_dhcp_leases_active` | Gauge | `scope_id` |
| `dnsdave_dhcp_leases_total` | Counter | `scope_id`, `event_type` |
| `dnsdave_dhcp_pool_utilisation` | Gauge | `scope_id` |
| `dnsdave_dhcp_offer_latency_us` | Histogram | `scope_id` |
| `dnsdave_dhcp_nak_total` | Counter | `scope_id`, `reason` |
| `dnsdave_dhcp_dynamic_dns_registrations_total` | Counter | `scope_id` |

**dnsdave-sync:**

| Metric | Type | Labels |
|--------|------|--------|
| `dnsdave_sync_domains_total` | Gauge | `list_id` |
| `dnsdave_sync_duration_s` | Histogram | `list_id` |
| `dnsdave_sync_errors_total` | Counter | `list_id` |

NATS: built-in monitoring endpoint on `:8222/metrics` via `nats-surveyor`.

### 12.2 Structured Logging

All containers emit JSON logs. Correlate across containers using a `trace_id` embedded in NATS event payloads — present in both query log entries and DHCP lease events.

### 12.3 Live Streams

| Stream | Endpoint | Source |
|--------|----------|--------|
| DNS query log | `GET /api/v1/logs/stream` (SSE) | `dnsdave.query.*` NATS |
| DHCP lease events | `GET /api/v1/dhcp/leases/stream` (SSE) | `dnsdave.dhcp.lease.*` NATS |

### 12.4 Health & Readiness

| Container | Endpoint | Ready when |
|-----------|----------|-----------|
| `dnsdave-dns` | `:9090/health`, `:9090/ready` | NATS connected, bloom filter loaded |
| `dnsdave-dhcp` | `:9093/health`, `:9093/ready` | NATS + Postgres connected, ≥1 scope configured |
| `dnsdave-api` | `:8080/api/v1/system/health` | NATS + Postgres connected |
| `dnsdave-sync` | `:9091/health` | NATS + Postgres connected |
| `dnsdave-log` | `:9092/health` | NATS + Postgres connected |
| `nats` | `:8222/healthz` | Built-in |

---

## 13. Security

| Concern | Mitigation |
|---------|-----------|
| Unauthenticated DNS | Standard; API always requires key/JWT |
| Unauthenticated DHCP | Standard protocol; rogue server protection via NATS leader election (only one active DHCP server per broadcast domain) |
| API key storage | Argon2id-hashed at rest; never logged |
| DoT/DoH TLS | ACME auto-provisioned or BYO cert |
| NATS auth | Per-container NKey credentials; deny-all default ACLs |
| DHCP option injection | Options stored as typed values; raw hex only for explicitly declared custom codes |
| Rate limiting | Per-client DNS token bucket; API token bucket; DHCP DISCOVER rate limit per MAC |
| Log retention | `query_log` and `dhcp_leases` auto-purged (default 30 days / 90 days respectively) |
| Container isolation | `distroless/static` base; no shell |
| Secrets | `.env` or Kubernetes `Secret`; never baked into images |
| CORS | Configurable allowed origins |

NATS per-container permissions:
- `dnsdave-dns`: publish `dnsdave.query.*`; subscribe `dnsdave.config.>`, `dnsdave.blocklist.>`, `dnsdave.dhcp.lease.>`
- `dnsdave-dhcp`: publish `dnsdave.dhcp.lease.*`; subscribe `dnsdave.config.>`; no publish on config subjects
- `dnsdave-api`: publish `dnsdave.config.*`; no publish on query or lease subjects

---

## 14. Milestones

### v0.1 — Core DNS + event bus foundations
- [ ] `dnsdave-dns`: Rust, SO_REUSEPORT, UDP, forward-only resolver
- [ ] `dnsdave-api`: REST API skeleton, Postgres, config CRUD
- [ ] NATS JetStream + KV store configured
- [ ] Config events published on every API write
- [ ] `dnsdave-dns` consumes config events, rebuilds in-memory state
- [ ] Basic blocklist ingest (hosts format) via sync worker
- [ ] `docker-compose.yml` reference stack
- [ ] Benchmark suite baseline

### v0.2 — Blocklist ecosystem
- [ ] All list formats (adblock, domain-only, RPZ)
- [ ] MPHF blocklist map (`boomphf`) + Bloom filter hot-swap
- [ ] Bloom filter → NATS Object Store; DNS hot-swap
- [ ] `dnsdave-log`: NATS consumer → Postgres bulk writer
- [ ] `dnsdave-stats`: in-memory counters from query events
- [ ] Groups + per-group blocklist policy
- [ ] Pi-hole API shim (v5)

### v0.3 — Advanced DNS + Local Zone Authority + Forwarding
- [ ] Wildcard records + copy-on-write label trie
- [ ] CNAME chaining, PTR, split-horizon views
- [ ] DoT + DoH upstream with HTTP/2 connection pooling
- [ ] Negative caching; cache pre-warming from JetStream replay
- [ ] Zone management (`dns_zones` table, SOA, NS records)
- [ ] Zone authority check in hot path — authoritative NXDOMAIN + SOA for owned zones
- [ ] `ZoneTrie` ArcSwap structure; `DNSDAVE_ZONES` JetStream stream
- [ ] AXFR (full zone transfer) served over TCP port 53
- [ ] IXFR (incremental zone transfer) from `dns_zone_changes` log
- [ ] RFC 1996 NOTIFY to configured secondary nameservers on serial change
- [ ] RFC 2136 DNS UPDATE ingest (ACME, ExternalDNS, Kea DDNS)
- [ ] SOA serial auto-increment on every record write within a zone
- [ ] Built-in zones: `home.arpa` template, reverse zone wizards
- [ ] Conditional DNS forwarding (`dns_forward_zones` table; `ForwardZoneTable` ArcSwap)
- [ ] Forward zone hot path step (step 7): cache-first, zone-specific upstream, blocklist bypass
- [ ] `/api/v1/dns/forwardzones/` CRUD; `dnsdave.config.forwardzone.*` NATS events
- [ ] Per-forward-zone DNSSEC validation policy (strict / opportunistic / disabled)

### v0.4 — Production readiness + DNSSEC validation
- [ ] DoT listener (:853) + DoH listener (:8080/dns-query)
- [ ] TLS auto-provisioning (ACME)
- [ ] Prometheus metrics across all containers (µs granularity)
- [ ] SSE live log stream via NATS
- [ ] Helm chart v1 (DaemonSet DNS, replicated API, NATS StatefulSet)
- [ ] `recvmmsg`/`sendmmsg` batch I/O + NATS NKey auth
- [ ] DNSSEC validation: DO bit on upstream queries; AD bit on validated replies; SERVFAIL on failure
- [ ] Trust anchor management (root KSK IANA, custom anchors for internal CAs)
- [ ] DNSSEC-aware negative caching (NSEC/NSEC3 validation)
- [ ] Global `dnssec_validate` setting + per-upstream override

### v0.4.5 — DNSSEC signing
- [ ] ZSK + KSK generation (Ed25519 / ECDSA P-384) via `ring` crate
- [ ] Private key encrypted storage in Postgres (`DNSDAVE_SECRET_KEY`)
- [ ] Inline zone signing: RRSIG regeneration on every record write
- [ ] NSEC3 records with 0 iterations, empty salt (RFC 9276)
- [ ] `GET /api/v1/dns/dnssec/zones/:name/ds` — DS record export
- [ ] ZSK pre-publish rollover (automated via NATS KV key lifecycle)
- [ ] `dnsdave.dnssec.*` NATS events; `DNSDAVE_DNSSEC` JetStream stream

### v0.5 — DHCP
- [ ] `dnsdave-dhcp`: DHCPv4 DORA state machine, scope management, static reservations
- [ ] Full DHCPv4 option suite (codes 1–252; raw hex for custom codes)
- [ ] Client class matching + vendor-specific option delivery
- [ ] PXE boot with automatic BIOS/UEFI architecture detection (option 93)
- [ ] DHCP relay support (RFC 3046 giaddr + option 82)
- [ ] Dynamic DNS: A + PTR auto-registration via NATS lease events
- [ ] `dnsdave-dns` consumes `DNSDAVE_DHCP` JetStream for automatic record creation/removal
- [ ] DHCP lease management API (`/api/v1/dhcp/*`)
- [ ] SSE lease event stream (`/api/v1/dhcp/leases/stream`)
- [ ] DHCPv6 SARR + IA_NA + IA_PD
- [ ] Dual-stack: A + AAAA + PTR dynamic registration
- [ ] DHCP Prometheus metrics + pool utilisation gauges

### v0.6 — Multi-node HA + scalability
- [ ] `dnsdave-dns` horizontal scale (N nodes, independent NATS subscribers)
- [ ] NATS JetStream 3-node cluster
- [ ] Leader election for `dnsdave-sync`, `dnsdave-stats`, `dnsdave-dhcp` via NATS KV
- [ ] Postgres Patroni HA; active-active DNS with keepalived/VRRP
- [ ] Pi-hole API shim (v6)

### v1.0 — Multi-region
- [ ] Multi-region anycast (BGP)
- [ ] Postgres logical replication cross-region
- [ ] Per-region NATS cluster with leaf node federation

---

## 15. Technology Choices

### 15.1 Per-Container Stack

| Container | Language | Key Libraries |
|-----------|----------|--------------|
| `dnsdave-dns` | **Rust** | `hickory-dns`, `tokio`, `socket2`, `nix`, `async-nats`, `arc-swap`, `hashbrown`, `boomphf`, `bloomfilter`, `moka`, `memchr`, `ring` (DNSSEC validation) |
| `dnsdave-dhcp` | **Rust** | `dhcproto`, `tokio`, `socket2`, `async-nats`, `sqlx`, `memchr` |
| `dnsdave-api` | **Rust** | `axum`, `tokio`, `async-nats`, `sqlx`, `utoipa` |
| `dnsdave-sync` | **Rust** | `tokio`, `async-nats`, `sqlx`, `reqwest`, `memchr`, `boomphf`, `bloomfilter` |
| `dnsdave-log` | **Rust** | `async-nats`, `sqlx`, `tokio` |
| `dnsdave-stats` | **Rust** | `async-nats`, `tokio`, `dashmap` |

### 15.2 Infrastructure

| Concern | Choice | Rationale |
|---------|--------|-----------|
| Event bus | **NATS JetStream** | <100µs publish; 15MB image; Object Store + KV built-in; `async-nats` Rust client |
| DHCP library | `dhcproto` | Pure Rust; DHCPv4 + DHCPv6 encode/decode; no C dependencies |
| DNS library | `hickory-dns` | Primary Rust DNS library; wire format access for zero-copy; built-in DNSSEC validation and signing support |
| DNSSEC crypto | `ring` + `rcgen` | ECDSA P-384, Ed25519 key generation and signing; no C deps on modern targets |
| Async runtime | `tokio` | De facto standard; excellent UDP + HTTP/2 |
| UDP batch I/O | `nix` (`recvmmsg`/`sendmmsg`) | Linux batch syscalls; `socket2` for SO_REUSEPORT |
| Bloom filter | `bloomfilter` | Serialisable; configurable FPR |
| Perfect hash | `boomphf` | Minimal perfect hash; ~3 bits/entry for immutable blocklist |
| Concurrent cache | `moka` | Window TinyLFU; better hit rate than LRU for DNS; internally sharded |
| Lock-free swap | `arc-swap` | ArcSwap for atomic pointer swap without unsafe |
| Concurrent map | `dashmap` | Write-heavy concurrent maps (stats counters) |
| Byte scanning | `memchr` | AVX2/NEON SIMD; fastest byte search available |
| HTTP client | `reqwest` | Async; HTTP/2; transparent gzip/zstd |
| Database | `sqlx` | Async; compile-time query checking; SQLite + PostgreSQL |
| Config | `config` crate + `clap` | Flag / env / file priority chain |
| OpenAPI | `utoipa` + `aide` | Generates spec from handler annotations |
| Serialisation (events) | `rmp-serde` (MessagePack) | ~300–420 bytes per event; no JSON parsing overhead |
| Container base | `gcr.io/distroless/cc` | No shell; minimal attack surface |
| CI | GitHub Actions | Multi-arch build (amd64 + arm64); push to GHCR |
| Benchmarking | `criterion` + `dnsperf` + `flamethrower` | Microbenchmarks + QPS + flamegraph |

### 15.3 Language Rationale

| Factor | Rust | Go |
|--------|------|----|
| GC pauses | None | <1ms, present; impacts p99/p999 |
| Blocklist lookup | `hashbrown` SIMD + MPHF | Go map (no SIMD) |
| Memory (5M blocklist) | ~300MB (MPHF) | ~600MB (map) |
| Byte scanning | `memchr` AVX2/NEON | `bytes.IndexByte` (SIMD, less tuned) |
| Lock-free swap | `ArcSwap` | `atomic.Pointer` (equivalent) |
| DNS library | `hickory-dns` (good) | `miekg/dns` (excellent) |
| DHCP library | `dhcproto` (good) | Limited options |
| Development velocity | Slower | Faster |

All containers are Rust for consistency. `dhcproto` covers DHCPv4 and DHCPv6 without CGo.

---

## 16. Open Questions

1. **Web UI scope** — API-only for v1, or ship a minimal read-only Svelte dashboard subscribing to SSE streams?
2. **RPZ as output** — Serve RPZ zones to downstream resolvers by consuming `dnsdave.blocklist.*`? (Note: DNSDave can already ingest RPZ format. Serving RPZ to downstream resolvers is a separate feature.)
3. **DoH port** — Serve DoH on `:8080/dns-query` (shared with API) or dedicated `:443`?
4. **Multi-tenancy** — First-class namespace/org concept for SaaS, or single-tenant-per-instance?
5. **`dnsdave-stats` persistence** — In-memory only (lost on restart) or checkpoint to NATS KV periodically?
6. **NATS vs Redpanda at scale** — Benchmark trigger: >500K QPS sustained for >1 hour.
7. **Bloom filter fetch at startup** — 90MB bloom filter is ~700 NATS Object Store chunks. Benchmark fetch + reassemble time vs. direct Postgres read. If Postgres is faster, use it only for startup and NATS only for live updates.
8. **DHCP HA across broadcast domains** — In a multi-site Kubernetes deployment, each node needs a DHCP server on its local broadcast domain. DaemonSet is the right primitive, but leader election must be per-node, not cluster-wide. Design the NATS KV lock key as `dhcp.leader.{node_name}`.
9. **DHCPv6 prefix delegation (IA_PD)** — For ISP-style deployments where `dnsdave-dhcp` delegates IPv6 prefixes to customer routers. Scope for v0.5 or push to v1?
10. **RFC 2136 TSIG auth** — DNS UPDATE packets can be authenticated with TSIG (HMAC-MD5 / HMAC-SHA256). Should this be required for RFC 2136 updates, or is IP-CIDR (`allow_update`) sufficient for v0.3?
11. **mDNS unicast bridge** — Is the mDNS→unicast bridge in scope for v0.6, or only if there is user demand? Requires binding a multicast socket which may conflict with host mDNS daemons (avahi, mDNSResponder).
12. **Secondary zone bootstrapping** — For `zone_type = secondary`, DNSDave must initiate an initial AXFR from the primary on zone creation. Which TCP connection pool handles this (reuse `dnsdave-dns` or a dedicated worker)?
13. **Forward zone blocklisting** — Should forward zones optionally apply the local blocklist before forwarding (for filtering ads even on corporate DNS)? Or keep the bypass unconditional for simplicity?
14. **DNSSEC validation and Pi-hole compatibility** — Pi-hole extensions and mobile apps send DNS queries directly; they don't set the DO bit. With strict DNSSEC validation enabled, should dnsdave-dns only validate its own upstream queries, or also re-validate queries that clients request validation for (CD bit)?
15. **DNSSEC key rollover automation** — ZSK rollover requires coordinated publish→activate→retire phases over days. Should this be managed by a cron-style job within `dnsdave-api`, or a dedicated `dnsdave-keymgr` sidecar subscribed to NATS?
16. **Trust anchor automatic updates** — The IANA root KSK rolls every ~5 years (next: ~2026). Should DNSDave fetch the current root trust anchor from IANA's XML feed automatically, or require manual operator action?
