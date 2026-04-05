# DNSDave – Product Design Document

**Version:** 0.4.0-draft  
**Date:** 2026-04-04  
**Status:** Draft

---

## 1. Problem Statement

Pi-hole is the de facto home DNS sinkhole, but it carries a decade of design constraints:

- Configuration is file-based and requires SSH access or a purpose-built PHP UI.
- No stable, documented REST API – automation is fragile and unsupported.
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
| G9 | **Loose coupling via an event bus.** The DNS hot path offloads all side-effects asynchronously. New functionality is added by subscribing to the bus – zero changes to the DNS container. |
| G10 | **First-class DHCPv4 and DHCPv6 server** with full RFC option support, static reservations, PXE boot, DHCP relay, and automatic DNS record registration on lease assignment. |
| G11 | **Authoritative local DNS server** for any zone it owns. DNSDave answers authoritatively (AA flag set, SOA in authority section) for configured zones and never forwards those queries upstream. Supports zone transfers (AXFR/IXFR), NOTIFY to secondaries, and RFC 2136 dynamic updates. |
| G12 | **Conditional DNS forwarding.** Named forward zones route queries for specific domains to designated upstream resolvers, bypassing the global upstream pool and the blocklist. Essential for VPN/corporate split-DNS, multi-site internal networks, and ISP-delegated zones. |
| G13 | **DNSSEC support** at two levels: (a) *validation* – verify cryptographic signatures on responses from upstream resolvers and set the AD bit; (b) *signing* – sign owned zones with ZSK/KSK key pairs, serve DNSKEY/RRSIG/NSEC3 records, and expose DS records for delegation. |
| G14 | **Multi-architecture native binaries and container images.** Every container ships as a multi-arch manifest covering `linux/amd64`, `linux/arm64` (Raspberry Pi 3+ running 64-bit OS, Apple Silicon servers, AWS Graviton), and `linux/arm/v7` (Raspberry Pi 3+ running 32-bit OS). Development builds are supported on `macOS/arm64` (Apple Silicon) and `macOS/x86_64` without modification. |
| G15 | **Pluggable observability export.** Logs, events, and metrics can be forwarded to any external system via a dedicated `dnsdave-export` container that subscribes to NATS and speaks the target system's native protocol – syslog (rsyslog, syslog-ng), OTLP (OpenTelemetry Collector → Grafana Loki/Tempo/Prometheus, Jaeger, Splunk, Datadog, New Relic, Honeycomb), Loki push API, InfluxDB line protocol, Elasticsearch, StatsD, and generic HTTP webhooks. Zero changes to existing containers. |

### Non-Goals (v1)

- Building a full recursive resolver (we forward upstream for non-local zones).
- Automated DNSSEC key ceremony with HSM hardware (key operations are software-only via `ring`/`rcgen`).
- `linux/arm/v6` (Raspberry Pi 1/Zero) – insufficient RAM and no hardware FPU for the blocklist data structures.
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
        UI["dnsdave-ui  (SvelteKit)\n:3000  ·  Web UI  ·  SSE proxy"]
        SYNC["dnsdave-sync\nBlocklist downloader + parser"]
        LOG["dnsdave-log\nQuery log bulk writer"]
        STATS["dnsdave-stats\nIn-memory stats aggregator"]
        EXPORT["dnsdave-export\nPluggable observability exporter\nsyslog · OTLP · Loki · InfluxDB · StatsD · webhook"]
        DHCP["dnsdave-dhcp  (Rust)\n:67 UDP DHCPv4  ·  :547 UDP DHCPv6\nDORA + SARR state machines\nPXE · relay · vendor classes"]
        NETBOX["dnsdave-netbox\nNetBox 4.5+ integration\nDiode (gRPC) · REST API\nIPAM push: prefixes · IP ranges · addresses · devices"]
    end

    PG[("PostgreSQL\nconfig · lists · logs · leases")]

    Clients -->|"DNS UDP/TCP :53  ·  DoT :853"| DNS
    Clients -->|"DHCP :67  ·  DHCPv6 :547"| DHCP
    Clients -->|"HTTPS :3000  ·  browser UI"| UI
    UI -->|"REST + SSE proxy"| API
    DNS -->|"query events\nfire & forget · MessagePack"| NATS
    NATS -->|"config hot-swap · blocklist diffs\nbloom filters · runtime KV\nlease events → dynamic DNS"| DNS
    API -->|"config change events"| NATS
    NATS -->|"query events"| LOG
    NATS -->|"query events"| STATS
    NATS -->|"all events"| EXPORT
    EXPORT -->|"syslog / OTLP / Loki / InfluxDB / ..."| EXT[("External Systems\nrsyslog · Grafana · ELK\nSplunk · Datadog · InfluxDB")]
    NATS -->|"lease · scope · DNS record events"| NETBOX
    NETBOX -->|"Diode gRPC or REST API"| NB[("NetBox 4.5+\nPrefixes · IP Ranges\nIP Addresses · Devices")]
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
| `dnsdave-ui` | **TypeScript** (SvelteKit) | Web management UI. Proxies REST + SSE to `dnsdave-api`. No direct DB or NATS access. See `UI.md`. | `dnsdave-api` only |
| `dnsdave-export` | **Rust** | Pluggable observability exporter. Subscribes to all NATS event subjects and forwards to configured external backends. Zero Postgres access. Optional container – omit if no external export needed. | NATS only |
| `dnsdave-netbox` | **Rust** | NetBox 4.5+ IPAM integration. Subscribes to DHCP scope, lease, and DNS record events and pushes corresponding objects (Prefixes, IP Ranges, IP Addresses, Devices) to NetBox via either Diode (gRPC) or the NetBox REST API. Zero Postgres access. Optional container. | NATS + NetBox (Diode or REST) |
| `nats` | - | Event bus. Core pub/sub + JetStream + Object Store + KV Store. | - |
| `postgres` | - | Primary persistent storage. | - |

### 3.3 Key Design Principles

**The DNS container never touches the database.** It derives its entire runtime state (blocklist, local records, upstreams, allowlist, DHCP lease-generated records) from NATS. On startup it replays JetStream history to rebuild state without touching Postgres. `dnsdave-dns` can scale to N instances with no additional configuration.

**The DNS container offloads all side-effects.** Query log writes, stats updates, dynamic DNS from DHCP leases, and any future integrations are downstream consumers of NATS subjects. The hot path fires a non-blocking NATS publish and moves on.

**New functionality = new consumer.** Alerting, webhooks, InfluxDB export, SIEM integration – all are new containers subscribing to the bus. The DNS and API containers are never modified.

### 3.4 Supported Platforms

Every container ships as a **multi-arch manifest** so the same `docker pull` command works on any supported host with no flags.

| Platform | Docker target | Example hardware | Notes |
|----------|--------------|-----------------|-------|
| `linux/amd64` | `x86_64-unknown-linux-gnu` | x86_64 servers, desktops, VMs | Primary development target |
| `linux/arm64` | `aarch64-unknown-linux-gnu` | Raspberry Pi 3/4/5 (64-bit OS), Apple Silicon (Rosetta-free), AWS Graviton 2/3 | Preferred Pi target; 64-bit OS required |
| `linux/arm/v7` | `armv7-unknown-linux-gnueabihf` | Raspberry Pi 3+ (32-bit Raspberry Pi OS) | Supported but 64-bit OS on Pi is recommended |
| `macOS/arm64` | `aarch64-apple-darwin` | Apple Silicon (M1/M2/M3/M4) | Development only; `SO_REUSEPORT` + `recvmmsg` fallback active |
| `macOS/x86_64` | `x86_64-apple-darwin` | Intel Mac | Development only; same fallbacks as macOS/arm64 |

**Raspberry Pi minimum requirements:**

| Model | RAM | Recommended OS | Profile |
|-------|-----|---------------|---------|
| Pi 3 Model B+ | 1 GB | 64-bit Raspberry Pi OS | `docker-compose.minimal.yml` (no `dnsdave-stats`, `dnsdave-log`; SQLite) |
| Pi 4 Model B | 2–8 GB | 64-bit Raspberry Pi OS | Full stack; 4 GB+ recommended for blocklist at scale |
| Pi 5 | 4–16 GB | 64-bit Raspberry Pi OS | Full stack; `recvmmsg` supported on Linux |

**Architecture-specific behaviour:**

| Concern | x86_64 | arm64 | arm/v7 |
|---------|--------|-------|-------|
| `recvmmsg`/`sendmmsg` | ✓ (Linux) | ✓ (Linux) | ✓ (Linux) |
| SIMD byte scanning (`memchr`) | AVX2 / SSE4.2 | NEON | NEON (32-bit) |
| Bloom filter hashing | xxHash3 (hardware CRC32) | xxHash3 (hardware CRC32 via ARMv8) | xxHash3 (software fallback) |
| `ring` ECDSA/Ed25519 | ✓ | ✓ | ✓ |
| 300 MB RSS target (full blocklist) | ✓ | ✓ | ✓ (within 1 GB) |

SIMD paths are **runtime-detected** – the same binary runs on CPUs with and without AVX2/NEON extensions. No compile-time feature flags required for target-specific acceleration.

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
│   ├── discover.{scope_id}         # DISCOVER received (core, analytics only)
│   ├── scope.created.{scope_id}    # DHCP scope created via API (JetStream) → NetBox Prefix
│   ├── scope.updated.{scope_id}    # DHCP scope modified via API (JetStream) → NetBox Prefix update
│   └── scope.deleted.{scope_id}    # DHCP scope removed via API (JetStream) → NetBox Prefix remove
│                                    # Delivery: JetStream for lease.* and scope.* ; core for discover
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
│   ├── key.retired.{zone_name}     # key rollover – old key retired
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
| `DNSDAVE_DHCP` | `dnsdave.dhcp.lease.>`, `dnsdave.dhcp.scope.>` | 90 days (lease) / all (scope) | Lease history; scope lifecycle; replayed by DNS nodes and `dnsdave-netbox` |
| `DNSDAVE_ZONES` | `dnsdave.zone.>` | All messages | Zone / SOA state replay – new DNS nodes self-bootstrap zone authority and serial state |
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

### 4.8 Extensibility – Adding New Consumers

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

### 5.1 Hot Path – Query Lifecycle

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
    FZC -->|miss| FZU["7b  Forward to zone-specific\nupstream(s) – bypass blocklist\nDNSSEC validate if enabled"]
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

All NATS publishes are **non-blocking writes to the async-nats client buffer** – no syscall, no network boundary on the hot path.

**Critical semantics:**
- **Step 6** – Owned zones never leak to upstream. Missing records return authoritative NXDOMAIN (AA=1, SOA in authority section).
- **Step 7** – Forward zones bypass the blocklist entirely. If a name falls in a configured forward zone, it goes to that zone's designated resolver(s), not the global pool. The answer cache is still consulted first (step 7a) to avoid redundant upstream calls.
- **Step 11** – When DNSSEC validation is enabled, the DO (DNSSEC OK) bit is set on upstream queries. Validated responses have the AD (Authentic Data) bit set in the reply; signature failures return SERVFAIL.

### 5.2 UDP I/O – Multi-Socket with SO_REUSEPORT

N Tokio tasks (one per CPU core) each own a UDP socket bound to `:53` with `SO_REUSEPORT`. The kernel distributes packets across sockets via consistent hash of source IP + port – no userspace lock on the receive path.

| Platform | Batch I/O | `SO_REUSEPORT` |
|----------|-----------|----------------|
| Linux `x86_64` / `arm64` / `arm/v7` | `recvmmsg`/`sendmmsg` (up to 64 packets/syscall) | ✓ kernel 3.9+ (Pi 3+ kernel ≥ 4.4) |
| macOS `arm64` / `x86_64` (dev) | `recv_from` / `send_to` single-packet fallback | ✓ (macOS 10.13+) |

The I/O tier is selected at **compile time** via `#[cfg(target_os)]`; no runtime branching on the hot path.

### 5.3 Zero-Allocation Hot Path

All byte buffers are managed through `sync.Pool`-equivalent Tokio channels. Two tiers:

| Pool | Size | Use |
|------|------|-----|
| Small | 512 bytes | Standard DNS queries |
| Large | 4096 bytes | EDNS0 / large responses |

### 5.4 Lock-Free Data Structures (Rust)

All hot-path lookups use `ArcSwap` – readers load an Arc with a single atomic instruction. Writers do full copy-on-write in the background, then call `ArcSwap::store()`. No locks, no waits on the query path.

| Structure | Rust type | Update strategy |
|-----------|-----------|-----------------|
| Bloom filter | `ArcSwap<BloomFilter>` | Swap on NATS Object Store fetch |
| Blocklist map | `ArcSwap<BoomPHF<DomainKey>>` | Swap on blocklist diff |
| Allowlist set | `ArcSwap<HashSet<DomainKey>>` | Swap on config event |
| Local records map | `ArcSwap<HashMap<DomainKey, Vec<Record>>>` | Swap on config or DHCP lease event |
| Wildcard trie | `ArcSwap<LabelTrie>` | Swap on config event |
| Zone authority trie | `ArcSwap<ZoneTrie>` | Swap on `dnsdave.zone.*` config event – holds SOA per zone |
| Forward zone table | `ArcSwap<ForwardZoneTable>` | Swap on `dnsdave.config.forwardzone.*` event – suffix-matched; holds upstream list + DNSSEC policy per zone |

### 5.5 Blocklist In-Memory Layout

**Stage 1 – Bloom filter:** `k=10` xxHash3 functions; FPR ~0.001% at 5M domains; ~90MB. A negative is definitive – no further lookup needed.

**Stage 2 – Minimal Perfect Hash map:** `boomphf` eliminates all collisions. ~3 bits/entry; 5M domains ≈ 42MB. The map is immutable after construction – Go's map has no synchronisation overhead: reads are concurrent-safe because the map is never mutated.

### 5.6 Blocklist Parser – Streaming, Zero-Copy

- Stream-read HTTP response body; transparent gzip/zstd decompression.
- `memchr` crate SIMD-accelerated newline scanning – AVX2 on `x86_64`, NEON on `arm64`, NEON (32-bit) on `arm/v7`; scalar fallback auto-selected at runtime on CPUs lacking SIMD extensions.
- In-place lowercase via 256-byte lookup table; no regex; no per-line allocation.
- Multiple lists fetched and parsed concurrently with Tokio tasks.

| List | Size | Target |
|------|------|--------|
| StevenBlack unified | ~160K domains | <200ms |
| OISD big | ~1M domains | <1s |
| All lists combined (5M unique) | - | <3s cold; <1s incremental |

### 5.7 Answer Cache – moka

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

DNSDave can be the **authoritative primary** (or secondary) DNS server for any zone, allowing it to act as a full local DNS server – not just a forwarder. This is distinct from simple local records: zones give DNSDave authority over an entire namespace with proper RFC-compliant semantics.

#### Zone Configuration

```jsonc
// POST /api/v1/dns/zones
{
  "name":    "home.arpa",          // RFC 8375 – canonical home network zone
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
| AXFR | Full zone transfer – streams all records in the zone |
| IXFR | Incremental transfer – streams only records changed since a given serial, sourced from `dns_zone_changes` |

#### NOTIFY

On every serial increment, `dnsdave-dns` sends RFC 1996 NOTIFY packets to all configured secondary nameservers in the zone's `notify` list. Retransmitted with exponential back-off until the secondary responds.

#### RFC 2136 Dynamic DNS Update

DNSDave accepts DNS UPDATE packets (RFC 2136) on port 53/TCP. This standard protocol is used by:

- **certbot / ACME DNS-01** – automatic certificate issuance
- **Kubernetes ExternalDNS** (webhook or RFC 2136 mode) – register service IPs
- **Kea DHCP** DDNS – register non-dnsdave clients
- **Ansible / Terraform** DNS modules

Updates are validated against `allow_update` CIDRs, translated to the same record-write path as the REST API, and published to `dnsdave.zone.serial.updated` on NATS so all nodes see the change within milliseconds.

#### ACME DNS-01 Certificate Issuance

Because DNSDave is authoritative for its owned zones, it can fulfil ACME DNS-01 challenges for itself and for any other service in the local environment. The ACME client in `dnsdave-api` places `_acme-challenge.` TXT records via its own RFC 2136 interface and removes them automatically after validation. See §13.1 for full details.

#### mDNS Unicast Bridge *(future)*

A planned optional module bridges mDNS/Bonjour announcements (multicast `224.0.0.251:5353`) into a configured zone as unicast-accessible A/AAAA/PTR records. Devices that announce via mDNS (Apple TV, printers, smart speakers) become reachable as `device.home.arpa` from standard DNS clients.

### 6.6 Conditional DNS Forwarding

A **forward zone** routes queries for a specific domain suffix to a designated set of upstream resolvers, bypassing both the global upstream pool and the blocklist. This is distinct from an owned zone (where DNSDave is authoritative) – in a forward zone, DNSDave acts as a proxy to the named servers and caches their responses.

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

**Hot path behaviour (step 7):** Forward zone lookup uses a right-to-left label trie (same structure as the zone authority trie, stored in `ArcSwap<ForwardZoneTable>`). Longest suffix wins. The answer cache is checked first (step 7a) before the network call (step 7b). Forward zone responses are cached with normal TTL.

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

`dnsdave-dhcp` is a first-class DHCPv4 and DHCPv6 server integrated with the DNS data plane via NATS JetStream. When a lease is assigned, DNS `A`/`AAAA` and `PTR` records are automatically created in all `dnsdave-dns` instances – no manual registration, no polling. Releasing or expiring a lease removes the records.

### 8.2 Protocol Support

| Protocol | State Machine | Transport |
|----------|--------------|-----------|
| DHCPv4 | DISCOVER → OFFER → REQUEST → ACK/NAK (DORA) | UDP :67 broadcast + unicast |
| DHCPv6 | SOLICIT → ADVERTISE → REQUEST → REPLY (SARR) | UDP :547 multicast + unicast |
| DHCPv6 IA_NA | Non-temporary address assignment | - |
| DHCPv6 IA_PD | Prefix delegation | - |
| PXE (BIOS + UEFI) | Auto-detection via option 93 | - |
| DHCP Relay | RFC 3046 giaddr + option 82 | - |

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

This allows precise targeting – Cisco switches in scope `LAN` get option 66 (TFTP server); all other clients in the same scope do not.

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
| 24 | Domain Search List | - |
| 31 | SNTP Servers | - |
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
│   ├── POST   /login              # password auth → JWT + refresh cookie
│   ├── POST   /logout             # revoke refresh token
│   ├── POST   /refresh            # exchange refresh cookie → new JWT
│   ├── GET    /me                 # own profile + effective permissions
│   └── PATCH  /me/password        # change own password
│
├── iam/
│   ├── users/
│   │   ├── GET    /               # list users (paginated, filterable)
│   │   ├── POST   /               # create user
│   │   ├── GET    /:id
│   │   ├── PATCH  /:id            # update profile / active status
│   │   ├── DELETE /:id            # deactivate (soft delete)
│   │   ├── POST   /:id/reset-password
│   │   ├── POST   /:id/unlock
│   │   ├── GET    /:id/sessions   # active JWT sessions
│   │   ├── DELETE /:id/sessions   # revoke all sessions
│   │   ├── GET    /:id/permissions  # resolved effective permission set
│   │   └── api-keys/
│   │       ├── GET    /
│   │       ├── POST   /           # create key (raw key shown once)
│   │       └── DELETE /:kid
│   │
│   ├── groups/
│   │   ├── GET    /
│   │   ├── POST   /
│   │   ├── GET    /:id
│   │   ├── PATCH  /:id
│   │   ├── DELETE /:id
│   │   ├── members/
│   │   │   ├── GET    /
│   │   │   ├── POST   /           # { user_ids: [...] }
│   │   │   └── DELETE /:uid
│   │   └── roles/
│   │       ├── GET    /
│   │       ├── POST   /           # assign role to group
│   │       └── DELETE /:rid
│   │
│   ├── roles/
│   │   ├── GET    /               # list built-in + custom roles
│   │   ├── POST   /               # create custom role
│   │   ├── GET    /:id
│   │   ├── PATCH  /:id
│   │   ├── DELETE /:id            # custom only; blocked if still assigned
│   │   └── permissions/
│   │       ├── GET    /
│   │       ├── PUT    /           # replace full permission set
│   │       ├── POST   /           # add single permission
│   │       └── DELETE /:pid
│   │
│   ├── permissions/
│   │   └── GET    /               # enumerate all valid resource:action strings
│   │
│   └── audit/
│       ├── GET    /               # paginated audit log
│       └── GET    /stream         # SSE live audit stream
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
├── devices/
│   ├── GET    /               # list all devices (paginated, filterable by type/tag/zone)
│   ├── POST   /               # create device with interfaces + addresses
│   ├── GET    /:id            # device detail + interfaces + addresses + lease status
│   ├── PATCH  /:id            # update device metadata / naming strategy
│   ├── DELETE /:id            # delete device + all reservations and DNS records
│   ├── POST   /:id/interfaces            # add interface to device
│   ├── PATCH  /:id/interfaces/:iface_id  # update interface (MAC, alias, VLAN)
│   ├── DELETE /:id/interfaces/:iface_id  # remove interface + its addresses
│   ├── POST   /:id/interfaces/:iface_id/addresses        # add address to interface
│   ├── PATCH  /:id/interfaces/:iface_id/addresses/:aid   # update address
│   ├── DELETE /:id/interfaces/:iface_id/addresses/:aid   # remove address
│   ├── GET    /:id/dns        # all DNS records associated with this device
│   ├── GET    /:id/leases     # active DHCP leases for all interfaces
│   └── POST   /from-lease     # promote lease to device + reservation
│
├── integrations/
│   └── netbox/
│       ├── GET    /config
│       ├── PUT    /config
│       ├── POST   /test
│       ├── GET    /status
│       ├── GET    /status/stream   # SSE
│       ├── sync/
│       │   ├── POST   /push
│       │   ├── POST   /pull
│       │   └── GET    /preview     # dry-run diff
│       ├── POST   /webhook         # inbound NetBox event webhook
│       ├── objects/
│       │   ├── GET    /
│       │   ├── GET    /:id
│       │   ├── POST   /:id/resolve # manual conflict resolution
│       │   └── DELETE /:id
│       ├── mappings/
│       │   ├── GET    /
│       │   ├── POST   /
│       │   ├── PUT    /:id
│       │   └── DELETE /:id
│       └── log/
│           ├── GET    /
│           └── GET    /stream      # SSE live sync activity
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

`GET /api/v1/logs/stream` – SSE; subscribes to `dnsdave.query.*` in NATS. Zero Postgres reads.  
`GET /api/v1/dhcp/leases/stream` – SSE; subscribes to `dnsdave.dhcp.lease.*` in NATS. Live lease activity.

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
| Bloom filter | `ArcSwap<BloomFilter>` | NATS Object Store | - |
| Blocklist MPHF map | `ArcSwap<BoomPHF<DomainKey>>` | NATS Object Store / diff replay | - |
| Allowlist set | `ArcSwap<HashSet<DomainKey>>` | NATS KV | - |
| Local records map | `ArcSwap<HashMap<DomainKey, Vec<Record>>>` | NATS JetStream replay | Manual records + DHCP lease records |
| Wildcard trie | `ArcSwap<LabelTrie>` | NATS JetStream replay | - |
| Client CIDR table | `ArcSwap<Vec<(IpNet, ClientId)>>` | NATS KV | - |
| Answer cache | `moka::Cache<CacheKey, CachedAnswer>` | Local upstream queries | - |
| Query event ring buffer | Lock-free MPSC | NATS fallback | - |

**Memory sizing (5M-domain blocklist + 500 DHCP leases):**

| Component | Estimate |
|-----------|----------|
| Bloom filter | ~90 MB |
| MPHF blocklist index | ~42 MB |
| moka answer cache | ~150 MB |
| Local records + trie (incl. lease records) | <10 MB |
| Zone authority trie + SOA table | <1 MB |
| **Total RSS** | **~301 MB** |

### 10.4 Device Model Tables

```sql
-- Top-level device record.
CREATE TABLE devices (
    id               TEXT PRIMARY KEY,
    name             TEXT NOT NULL,
    display_name     TEXT,
    device_type      TEXT NOT NULL DEFAULT 'host',
    description      TEXT,
    tags             TEXT[] DEFAULT '{}',
    dns_zone         TEXT REFERENCES dns_zones(name) ON DELETE SET NULL,
    naming_strategy  TEXT NOT NULL DEFAULT 'affinity',
    canonical_name   TEXT,         -- FQDN; auto-derived from name + dns_zone if NULL
    netbox_id        INTEGER,
    created_at       TIMESTAMP NOT NULL DEFAULT NOW(),
    updated_at       TIMESTAMP NOT NULL DEFAULT NOW()
);
CREATE INDEX idx_devices_name     ON devices(name);
CREATE INDEX idx_devices_netbox   ON devices(netbox_id) WHERE netbox_id IS NOT NULL;

-- Network interfaces on a device.
CREATE TABLE device_interfaces (
    id              TEXT PRIMARY KEY,
    device_id       TEXT NOT NULL REFERENCES devices(id) ON DELETE CASCADE,
    name            TEXT NOT NULL,           -- 'eth0', 'em0', 'bond0', 'wlan0'
    mac_address     MACADDR,
    interface_type  TEXT NOT NULL DEFAULT 'ethernet',
    vlan_id         INTEGER,
    mtu             INTEGER,
    description     TEXT,
    dns_alias       TEXT,                    -- short name; becomes alias FQDN
    is_primary      BOOLEAN NOT NULL DEFAULT FALSE,
    netbox_iface_id INTEGER,
    created_at      TIMESTAMP NOT NULL DEFAULT NOW(),
    UNIQUE (device_id, name),
    -- Only one primary interface per device.
    EXCLUDE USING btree (device_id WITH =) WHERE (is_primary)
);
CREATE INDEX idx_device_ifaces_device ON device_interfaces(device_id);
CREATE INDEX idx_device_ifaces_mac    ON device_interfaces(mac_address)
    WHERE mac_address IS NOT NULL;

-- IP addresses on an interface. Each address may optionally link to
-- a DHCP reservation and/or a DNS record.
CREATE TABLE device_interface_addresses (
    id               TEXT PRIMARY KEY,
    interface_id     TEXT NOT NULL REFERENCES device_interfaces(id) ON DELETE CASCADE,
    address          INET NOT NULL,         -- IP with prefix length
    address_type     TEXT NOT NULL DEFAULT 'primary',  -- 'primary' | 'secondary' | 'vip' | 'anycast'
    is_primary       BOOLEAN NOT NULL DEFAULT FALSE,
    is_dhcp          BOOLEAN NOT NULL DEFAULT FALSE,    -- true = dynamically leased
    scope_id         TEXT REFERENCES dhcp_scopes(id)   ON DELETE SET NULL,
    reservation_id   TEXT REFERENCES dhcp_reservations(id) ON DELETE SET NULL,
    dns_record_id    TEXT REFERENCES dns_records(id)   ON DELETE SET NULL,
    dns_alias        TEXT,                  -- per-address alias name (e.g. 'vip')
    netbox_ip_id     INTEGER,
    created_at       TIMESTAMP NOT NULL DEFAULT NOW(),
    UNIQUE (interface_id, address)
);
CREATE INDEX idx_dia_interface  ON device_interface_addresses(interface_id);
CREATE INDEX idx_dia_scope      ON device_interface_addresses(scope_id)
    WHERE scope_id IS NOT NULL;
CREATE INDEX idx_dia_reservation ON device_interface_addresses(reservation_id)
    WHERE reservation_id IS NOT NULL;
```

### 10.5 IAM Tables

```sql
-- User accounts. Passwords hashed with Argon2id.
CREATE TABLE iam_users (
    id                TEXT PRIMARY KEY,
    username          TEXT NOT NULL UNIQUE,
    display_name      TEXT NOT NULL DEFAULT '',
    email             TEXT UNIQUE,
    password_hash     TEXT,                  -- NULL for future SSO/OIDC accounts
    is_active         BOOLEAN NOT NULL DEFAULT TRUE,
    is_superadmin     BOOLEAN NOT NULL DEFAULT FALSE,
    failed_attempts   INTEGER NOT NULL DEFAULT 0,
    locked_until      TIMESTAMP,
    password_changed  TIMESTAMP,
    session_version   INTEGER NOT NULL DEFAULT 1,   -- increment to invalidate all sessions
    last_login_at     TIMESTAMP,
    last_login_ip     TEXT,
    created_at        TIMESTAMP NOT NULL DEFAULT NOW(),
    updated_at        TIMESTAMP NOT NULL DEFAULT NOW()
);
CREATE INDEX idx_iam_users_username ON iam_users(username);

-- Groups of users.
CREATE TABLE iam_groups (
    id          TEXT PRIMARY KEY,
    name        TEXT NOT NULL UNIQUE,
    description TEXT,
    is_system   BOOLEAN NOT NULL DEFAULT FALSE,  -- built-in groups cannot be deleted
    created_at  TIMESTAMP NOT NULL DEFAULT NOW()
);

-- Group membership.
CREATE TABLE iam_group_members (
    group_id   TEXT NOT NULL REFERENCES iam_groups(id) ON DELETE CASCADE,
    user_id    TEXT NOT NULL REFERENCES iam_users(id)  ON DELETE CASCADE,
    added_at   TIMESTAMP NOT NULL DEFAULT NOW(),
    added_by   TEXT REFERENCES iam_users(id),
    PRIMARY KEY (group_id, user_id)
);

-- Roles (built-in and custom).
CREATE TABLE iam_roles (
    id          TEXT PRIMARY KEY,
    name        TEXT NOT NULL UNIQUE,
    description TEXT,
    is_builtin  BOOLEAN NOT NULL DEFAULT FALSE,  -- built-in roles cannot be deleted
    created_at  TIMESTAMP NOT NULL DEFAULT NOW(),
    created_by  TEXT REFERENCES iam_users(id)
);

-- Permissions assigned to a role.
-- resource:action[:scope] triples.
CREATE TABLE iam_role_permissions (
    id          TEXT PRIMARY KEY,
    role_id     TEXT NOT NULL REFERENCES iam_roles(id) ON DELETE CASCADE,
    resource    TEXT NOT NULL,      -- e.g. 'dns_records'
    action      TEXT NOT NULL,      -- e.g. 'create'
    scope       TEXT DEFAULT 'all', -- 'own' | 'group' | 'all'
    UNIQUE (role_id, resource, action, scope)
);
CREATE INDEX idx_iam_role_permissions_role ON iam_role_permissions(role_id);

-- Roles assigned to groups.
CREATE TABLE iam_group_roles (
    group_id   TEXT NOT NULL REFERENCES iam_groups(id) ON DELETE CASCADE,
    role_id    TEXT NOT NULL REFERENCES iam_roles(id)  ON DELETE CASCADE,
    assigned_at TIMESTAMP NOT NULL DEFAULT NOW(),
    assigned_by TEXT REFERENCES iam_users(id),
    PRIMARY KEY (group_id, role_id)
);

-- Direct permission grants/denies on individual users (override group permissions).
CREATE TABLE iam_user_permissions (
    id          TEXT PRIMARY KEY,
    user_id     TEXT NOT NULL REFERENCES iam_users(id) ON DELETE CASCADE,
    resource    TEXT NOT NULL,
    action      TEXT NOT NULL,
    scope       TEXT DEFAULT 'all',
    effect      TEXT NOT NULL DEFAULT 'allow',  -- 'allow' | 'deny'
    reason      TEXT,
    granted_by  TEXT REFERENCES iam_users(id),
    expires_at  TIMESTAMP,
    created_at  TIMESTAMP NOT NULL DEFAULT NOW(),
    UNIQUE (user_id, resource, action, scope)
);
CREATE INDEX idx_iam_user_permissions_user ON iam_user_permissions(user_id);

-- API keys linked to users; inherit a subset of the user's permissions.
CREATE TABLE iam_api_keys (
    id              TEXT PRIMARY KEY,
    user_id         TEXT NOT NULL REFERENCES iam_users(id) ON DELETE CASCADE,
    name            TEXT NOT NULL,
    key_hash        TEXT NOT NULL UNIQUE,    -- Argon2id hash of the raw key
    key_prefix      TEXT NOT NULL,           -- first 8 chars displayed in UI
    permissions     TEXT[],                  -- subset of owner's permissions; NULL = inherit all
    allowed_cidrs   TEXT[],                  -- source IP allowlist; NULL = unrestricted
    last_used_at    TIMESTAMP,
    last_used_ip    TEXT,
    expires_at      TIMESTAMP,
    is_active       BOOLEAN NOT NULL DEFAULT TRUE,
    created_at      TIMESTAMP NOT NULL DEFAULT NOW()
);
CREATE INDEX idx_iam_api_keys_user ON iam_api_keys(user_id);

-- Refresh tokens (httpOnly cookie).
CREATE TABLE iam_refresh_tokens (
    id          TEXT PRIMARY KEY,
    user_id     TEXT NOT NULL REFERENCES iam_users(id) ON DELETE CASCADE,
    token_hash  TEXT NOT NULL UNIQUE,
    issued_at   TIMESTAMP NOT NULL DEFAULT NOW(),
    expires_at  TIMESTAMP NOT NULL,
    revoked_at  TIMESTAMP,
    user_agent  TEXT,
    ip_address  TEXT
);
CREATE INDEX idx_iam_refresh_user ON iam_refresh_tokens(user_id);

-- Append-only audit log for all mutating operations.
CREATE TABLE iam_audit_log (
    id            BIGSERIAL PRIMARY KEY,
    ts            TIMESTAMP NOT NULL DEFAULT NOW(),
    user_id       TEXT,
    username      TEXT,
    ip_address    TEXT,
    session_id    TEXT,         -- JWT jti or API key id
    method        TEXT NOT NULL,
    path          TEXT NOT NULL,
    resource_type TEXT,
    resource_id   TEXT,
    action        TEXT,
    before_state  JSONB,
    after_state   JSONB,
    outcome       TEXT NOT NULL, -- 'success' | 'forbidden' | 'error'
    error_detail  TEXT
) PARTITION BY RANGE (ts);
CREATE INDEX idx_iam_audit_ts       ON iam_audit_log(ts DESC);
CREATE INDEX idx_iam_audit_user     ON iam_audit_log(user_id, ts DESC);
CREATE INDEX idx_iam_audit_resource ON iam_audit_log(resource_type, resource_id, ts DESC);
```

### 10.6 NetBox Integration Tables

```sql
-- Singleton row holding all NetBox integration configuration.
-- api_token and secrets are AES-GCM encrypted using DNSDAVE_SECRET_KEY.
CREATE TABLE netbox_config (
    id                   TEXT PRIMARY KEY DEFAULT 'singleton',
    enabled              BOOLEAN NOT NULL DEFAULT FALSE,
    netbox_url           TEXT NOT NULL DEFAULT '',
    api_token_enc        TEXT,                 -- AES-GCM encrypted
    mode                 TEXT NOT NULL DEFAULT 'api',   -- 'api' | 'diode'
    diode_url            TEXT,
    diode_key_enc        TEXT,
    webhook_secret_enc   TEXT,
    -- Push toggles
    push_prefixes        BOOLEAN DEFAULT TRUE,
    push_ip_ranges       BOOLEAN DEFAULT TRUE,
    push_ip_addresses    BOOLEAN DEFAULT TRUE,
    push_dns_records     BOOLEAN DEFAULT TRUE,
    push_devices         BOOLEAN DEFAULT FALSE,
    push_conflict        TEXT DEFAULT 'overwrite',
    on_lease_expiry      TEXT DEFAULT 'free',
    on_scope_delete      TEXT DEFAULT 'keep',
    -- Pull toggles
    pull_enabled         BOOLEAN DEFAULT FALSE,
    pull_mode            TEXT DEFAULT 'webhook',    -- 'webhook' | 'poll' | 'manual'
    poll_interval_min    INTEGER DEFAULT 15,
    pull_prefixes        BOOLEAN DEFAULT FALSE,
    pull_ip_ranges       BOOLEAN DEFAULT FALSE,
    pull_ip_addresses    BOOLEAN DEFAULT FALSE,
    pull_devices         BOOLEAN DEFAULT FALSE,
    pull_vms             BOOLEAN DEFAULT FALSE,
    pull_reservations    BOOLEAN DEFAULT FALSE,
    pull_filter_tag      TEXT DEFAULT 'dnsdave',
    pull_filter_site     TEXT,
    pull_filter_vrf      TEXT,
    pull_filter_tenant   TEXT,
    pull_conflict        TEXT DEFAULT 'skip',
    auto_create_zones    BOOLEAN DEFAULT FALSE,
    -- Context / defaults
    default_vrf          TEXT,
    default_site         TEXT,
    default_tenant       TEXT,
    default_zone         TEXT,
    device_fqdn_template TEXT DEFAULT '{name}.{site}.{default_zone}',
    tags                 TEXT[] DEFAULT '{dnsdave}',
    updated_at           TIMESTAMP DEFAULT NOW()
);

-- Maps NetBox organisational objects to DNSDave objects.
CREATE TABLE netbox_mappings (
    id              TEXT PRIMARY KEY,
    netbox_type     TEXT NOT NULL,  -- 'vrf' | 'site' | 'tenant' | 'tag'
    netbox_slug     TEXT NOT NULL,
    dnsdave_type    TEXT NOT NULL,  -- 'scope' | 'zone' | 'group'
    dnsdave_id      TEXT NOT NULL,
    description     TEXT,
    created_at      TIMESTAMP DEFAULT NOW(),
    UNIQUE (netbox_type, netbox_slug, dnsdave_type)
);

-- Tracks every object that has been pushed to or pulled from NetBox.
CREATE TABLE netbox_sync_objects (
    id               TEXT PRIMARY KEY,
    netbox_id        INTEGER NOT NULL,
    netbox_type      TEXT NOT NULL,  -- 'prefix' | 'iprange' | 'ipaddress' | 'device' | 'virtualmachine'
    netbox_url       TEXT,
    direction        TEXT NOT NULL,  -- 'push' | 'pull'
    dnsdave_type     TEXT NOT NULL,  -- 'scope' | 'ip_range' | 'ip_address' | 'dns_record' | 'reservation'
    dnsdave_id       TEXT,
    last_sync_at     TIMESTAMP,
    sync_status      TEXT NOT NULL DEFAULT 'pending',  -- 'pending' | 'syncing' | 'synced' | 'conflict' | 'error'
    retry_count      INTEGER DEFAULT 0,
    next_retry_at    TIMESTAMP,
    conflict_detail  JSONB,          -- { field, nb_value, dd_value, strategy }
    error_detail     TEXT,
    created_at       TIMESTAMP DEFAULT NOW(),
    UNIQUE (netbox_id, netbox_type, direction)
);
CREATE INDEX idx_netbox_sync_status ON netbox_sync_objects(sync_status);
CREATE INDEX idx_netbox_sync_type   ON netbox_sync_objects(netbox_type, direction);

-- Append-only audit log of every sync operation.
CREATE TABLE netbox_sync_log (
    id           BIGSERIAL PRIMARY KEY,
    ts           TIMESTAMP NOT NULL DEFAULT NOW(),
    direction    TEXT NOT NULL,   -- 'push' | 'pull' | 'webhook'
    trigger      TEXT NOT NULL,   -- 'auto' | 'manual' | 'webhook' | 'poll'
    object_type  TEXT NOT NULL,
    netbox_id    INTEGER,
    dnsdave_id   TEXT,
    action       TEXT NOT NULL,   -- 'created' | 'updated' | 'deleted' | 'skipped' | 'conflict' | 'error' | 'resolved'
    detail       JSONB,
    duration_ms  INTEGER
);
CREATE INDEX idx_netbox_log_ts        ON netbox_sync_log(ts DESC);
CREATE INDEX idx_netbox_log_direction ON netbox_sync_log(direction, ts DESC);
```

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

### 11.2 TLS in Docker Compose / Podman

All inter-container connections run over TLS. Certificates are generated on first startup by `dnsdave-api` using `rcgen` and stored in the `tls_certificates` Postgres table. Containers receive the CA cert as an environment variable or mounted file.

```yaml
  nats:
    command: [
      "-js", "-m", "8222",
      "--tls",
      "--tlscert", "/certs/nats.crt",
      "--tlskey",  "/certs/nats.key",
      "--tlscacert", "/certs/ca.crt"
    ]
    volumes:
      - nats-certs:/certs   # written by dnsdave-api on bootstrap

  postgres:
    environment:
      POSTGRES_SSL_CERT: /certs/pg.crt
      POSTGRES_SSL_KEY:  /certs/pg.key
    volumes:
      - pg-certs:/certs

  dnsdave-api:
    environment:
      DNSDAVE_NATS_TLS_CA:   "/certs/ca.crt"
      DNSDAVE_DB_URL: "postgres://dnsdave:${POSTGRES_PASSWORD}@postgres/dnsdave?sslmode=verify-full&sslrootcert=/certs/ca.crt"
      DNSDAVE_TLS_CERT: "/certs/api.crt"
      DNSDAVE_TLS_KEY:  "/certs/api.key"
      DNSDAVE_ACME_EMAIL:    "${ACME_EMAIL:-}"         # if set, attempts ACME; else self-signed
      DNSDAVE_ACME_DOMAIN:   "${DNSDAVE_DOMAIN:-}"
    volumes:
      - api-certs:/certs
```

**Bootstrap sequence:**
1. `dnsdave-api` starts first; generates a local CA + all service certs into a shared volume.
2. NATS and Postgres start with their certs in place.
3. All other containers mount the CA cert and verify peers.

### 11.3 Minimal Stack

A `docker-compose.minimal.yml` runs only `dnsdave-dns`, `dnsdave-dhcp`, `dnsdave-api`, and NATS with SQLite. Query logging is written directly by `dnsdave-api`. Stats are disabled. TLS is still enabled; self-signed cert generated on first boot.

### 11.4 Podman Compose

The same compose files work with `podman-compose`. Quadlet-compatible unit files are provided for rootless Podman. `dnsdave-dhcp` requires `--cap-add=NET_BIND_SERVICE` and host network access for broadcast.

### 11.5 Kubernetes / Helm

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
    ├── servicemonitor.yaml
    ├── certificate-api.yaml          # cert-manager Certificate resource
    └── certificate-nats.yaml         # cert-manager Certificate for NATS cluster
```

Both `dnsdave-dns` and `dnsdave-dhcp` run as **DaemonSets** with `hostNetwork: true`. DNS binds port 53; DHCP binds ports 67/547 to receive broadcasts directly on each node.

### 11.6 Scaling the Stack

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

### 11.7 Configuration Priority

```
CLI flags > Environment Variables > Config file (YAML) > Defaults
```

### 11.8 Multi-Architecture Builds

All containers are built as **multi-arch Docker manifests** and pushed to GHCR. A single `docker pull ghcr.io/dnsdave/dnsdave-dns:latest` resolves to the correct image for the host CPU automatically.

#### Rust Cross-Compilation Targets

| Docker platform | Rust target triple | Linker |
|----------------|-------------------|--------|
| `linux/amd64` | `x86_64-unknown-linux-gnu` | native |
| `linux/arm64` | `aarch64-unknown-linux-gnu` | `aarch64-linux-gnu-gcc` |
| `linux/arm/v7` | `armv7-unknown-linux-gnueabihf` | `arm-linux-gnueabihf-gcc` |

Cross-compilation uses the [`cross`](https://github.com/cross-rs/cross) tool (Docker-based) for local builds, and GitHub Actions with QEMU emulation or native ARM runners for CI.

#### GitHub Actions CI Matrix

```yaml
# .github/workflows/build.yml (excerpt)
jobs:
  build:
    strategy:
      matrix:
        include:
          - platform: linux/amd64
            runner:   ubuntu-latest
            target:   x86_64-unknown-linux-gnu
          - platform: linux/arm64
            runner:   ubuntu-latest        # QEMU; or ubuntu-latest-arm64 if available
            target:   aarch64-unknown-linux-gnu
          - platform: linux/arm/v7
            runner:   ubuntu-latest        # QEMU
            target:   armv7-unknown-linux-gnueabihf

    steps:
      - uses: actions/checkout@v4
      - uses: docker/setup-qemu-action@v3      # enables arm64/arm emulation
      - uses: docker/setup-buildx-action@v3
      - uses: dtolnay/rust-toolchain@stable
        with:
          targets: ${{ matrix.target }}

      - name: Cross-compile
        uses: cross-rs/cross@v0.2
        with:
          command: build
          args: --release --target ${{ matrix.target }}

      - name: Build and push image
        uses: docker/build-push-action@v5
        with:
          platforms: ${{ matrix.platform }}
          push: ${{ github.ref == 'refs/heads/main' }}
          tags: ghcr.io/dnsdave/${{ matrix.container }}:latest
```

The final step in CI calls `docker buildx imagetools create` to merge all per-platform images into a single multi-arch manifest.

#### macOS Development Builds

No Docker required for local development on macOS. Rust natively targets the host:

```bash
# Apple Silicon
cargo build --target aarch64-apple-darwin

# Intel Mac
cargo build --target x86_64-apple-darwin

# Run locally (uses recv_from/send_to fallback; SO_REUSEPORT active)
cargo run --bin dnsdave-dns -- --config dev.toml
```

`docker-compose.yml` works on Docker Desktop for Mac (arm64 on Apple Silicon, x86_64 on Intel). NATS, Postgres, and the UI containers pull the correct arch variant automatically.

#### Raspberry Pi Low-Memory Profile (`docker-compose.minimal.yml`)

Pi 3 (1 GB RAM) cannot comfortably run the full stack alongside an OS. A minimal compose profile omits `dnsdave-log`, `dnsdave-stats`, and `dnsdave-ui`, and switches to SQLite:

```yaml
# docker-compose.minimal.yml (key differences)
services:
  dnsdave-dns:
    environment:
      DNSDAVE_WORKERS: "4"                  # match Pi 3's 4 cores
      DNSDAVE_CACHE_SIZE_MB: "32"           # reduce moka cache
      DNSDAVE_BLOOM_MAX_DOMAINS: "1000000"  # 1M-domain blocklist cap (~20 MB)

  dnsdave-api:
    environment:
      DNSDAVE_DB_URL: "sqlite:///data/dnsdave.db"  # no Postgres needed

  nats:
    command: ["-js", "-m", "8222", "--max_memory_store=64MB", "--max_file_store=256MB"]
```

**Pi 3 memory budget at idle (minimal profile):**

| Component | RSS |
|-----------|-----|
| `dnsdave-dns` (1M blocklist, 32 MB cache) | ~130 MB |
| `dnsdave-api` + SQLite | ~30 MB |
| `dnsdave-dhcp` | ~20 MB |
| NATS | ~25 MB |
| OS + kernel | ~100 MB |
| **Total** | **~305 MB** (of 1 GB) |

Pi 4 (2 GB+) can run the full stack comfortably, including `dnsdave-log`, `dnsdave-stats`, and Postgres.

---

## 12. Observability

### 12.1 Metrics (Prometheus)

Every container exposes a Prometheus scrape endpoint. Grafana, VictoriaMetrics, and any Prometheus-compatible backend can pull metrics directly.

**dnsdave-dns** (`:9090/metrics`):

| Metric | Type | Labels |
|--------|------|--------|
| `dnsdave_dns_queries_total` | Counter | `response_type`, `qtype` |
| `dnsdave_dns_query_duration_us` | Histogram | `response_type` |
| `dnsdave_dns_blocked_total` | Counter | - |
| `dnsdave_dns_cache_hits_total` | Counter | - |
| `dnsdave_dns_bloom_false_positives_total` | Counter | - |
| `dnsdave_dns_nats_publish_errors_total` | Counter | - |
| `dnsdave_dns_forward_zone_hits_total` | Counter | `zone` |
| `dnsdave_dns_dnssec_validation_failures_total` | Counter | - |

**dnsdave-dhcp** (`:9093/metrics`):

| Metric | Type | Labels |
|--------|------|--------|
| `dnsdave_dhcp_leases_active` | Gauge | `scope_id` |
| `dnsdave_dhcp_leases_total` | Counter | `scope_id`, `event_type` |
| `dnsdave_dhcp_pool_utilisation` | Gauge | `scope_id` |
| `dnsdave_dhcp_offer_latency_us` | Histogram | `scope_id` |
| `dnsdave_dhcp_nak_total` | Counter | `scope_id`, `reason` |
| `dnsdave_dhcp_dynamic_dns_registrations_total` | Counter | `scope_id` |

**dnsdave-sync** (`:9091/metrics`):

| Metric | Type | Labels |
|--------|------|--------|
| `dnsdave_sync_domains_total` | Gauge | `list_id` |
| `dnsdave_sync_duration_s` | Histogram | `list_id` |
| `dnsdave_sync_errors_total` | Counter | `list_id` |

**dnsdave-api** (`:8080/metrics`):

| Metric | Type | Labels |
|--------|------|--------|
| `dnsdave_api_requests_total` | Counter | `method`, `path`, `status` |
| `dnsdave_api_request_duration_ms` | Histogram | `method`, `path` |
| `dnsdave_api_auth_failures_total` | Counter | `reason` |
| `dnsdave_iam_lockouts_total` | Counter | - |
| `dnsdave_api_active_sessions` | Gauge | - |
| `dnsdave_api_ws_connections` | Gauge | - (active SSE streams) |
| `dnsdave_tls_cert_expiry_seconds` | Gauge | `domain` |
| `dnsdave_dns_affinity_resolution_total` | Counter | `result` (`matched`/`fallback`) |

**dnsdave-export** (`:9096/metrics`):

| Metric | Type | Labels |
|--------|------|--------|
| `dnsdave_export_events_total` | Counter | `backend`, `status` |
| `dnsdave_export_event_latency_ms` | Histogram | `backend` |
| `dnsdave_export_queue_depth` | Gauge | `backend` |
| `dnsdave_export_backend_errors_total` | Counter | `backend`, `error` |

NATS: built-in monitoring endpoint on `:8222/metrics` via `nats-surveyor`.

> The complete metrics reference, Prometheus scrape configuration, alert rules, Grafana dashboard specifications, Loki configuration, and Kubernetes deployment instructions are documented in [`MONITORING.md`](MONITORING.md).

### 12.2 Structured Logging

All containers write **RFC 5424-compatible structured JSON** to stdout. Every log line includes:

```json
{
  "ts":        "2026-04-04T14:23:01.842Z",
  "level":     "info",
  "container": "dnsdave-dns",
  "node_id":   "dns-node-1",
  "trace_id":  "a1b2c3d4e5f6",
  "msg":       "query processed",
  "domain":    "example.com",
  "qtype":     "A",
  "result":    "upstream",
  "latency_us": 12
}
```

`trace_id` is propagated through NATS event payloads, allowing correlation of a single DNS query across `dnsdave-dns` → NATS → `dnsdave-log` → `dnsdave-export` → external system.

The container runtime's logging driver handles routing stdout to external collectors:
- **Docker**: `--log-driver=json-file` (default) / `fluentd` / `gelf` / `splunk`
- **Kubernetes**: stdout → node logging agent (Fluent Bit, Logstash, etc.)

### 12.3 Live Streams

| Stream | Endpoint | Source |
|--------|----------|--------|
| DNS query log | `GET /api/v1/logs/stream` (SSE) | `dnsdave.query.*` NATS |
| DHCP lease events | `GET /api/v1/dhcp/leases/stream` (SSE) | `dnsdave.dhcp.lease.*` NATS |
| System events | `GET /api/v1/system/events/stream` (SSE) | `dnsdave.zone.*`, `dnsdave.dnssec.*`, `dnsdave.config.*` |

### 12.4 Health & Readiness

| Container | Endpoint | Ready when |
|-----------|----------|-----------|
| `dnsdave-dns` | `:9090/health`, `:9090/ready` | NATS connected, bloom filter loaded |
| `dnsdave-dhcp` | `:9093/health`, `:9093/ready` | NATS + Postgres connected, ≥1 scope configured |
| `dnsdave-api` | `:8080/api/v1/system/health` | NATS + Postgres connected |
| `dnsdave-sync` | `:9091/health` | NATS + Postgres connected |
| `dnsdave-log` | `:9092/health` | NATS + Postgres connected |
| `dnsdave-export` | `:9094/health` | NATS connected, ≥1 backend reachable |
| `nats` | `:8222/healthz` | Built-in |

### 12.5 Export Architecture – `dnsdave-export`

`dnsdave-export` is an **optional, stateless** container that subscribes to all NATS event subjects and forwards events to one or more configured backends. It is the single seam between DNSDave's internal event bus and any external observability system.

**Design principle:** `dnsdave-export` never touches the database. All data it exports comes from NATS. Adding a new export destination requires only a new backend configuration block – zero changes to any other container.

```mermaid
flowchart LR
    NATS["NATS JetStream\ndnsdave.query.*\ndnsdave.dhcp.lease.*\ndnsdave.zone.*\ndnsdave.config.*\ndnsdave.dnssec.*"]

    NATS --> EXP["dnsdave-export\n(fan-out)"]

    EXP --> SYS["Syslog UDP/TCP\n→ rsyslog / syslog-ng"]
    EXP --> OTLP["OTLP gRPC/HTTP\n→ OpenTelemetry Collector"]
    EXP --> LOKI["Loki push API\n→ Grafana"]
    EXP --> INFL["InfluxDB line protocol"]
    EXP --> STAT["StatsD UDP\n→ Graphite / Telegraf"]
    EXP --> ES["Elasticsearch bulk API\n→ Kibana / Logstash"]
    EXP --> SPLUNK["Splunk HEC"]
    EXP --> HOOK["HTTP webhook\n(generic JSON)"]

    OTLP --> OC["OTel Collector\n(fan-out again)"]
    OC --> PROM["Prometheus\n→ Grafana dashboards"]
    OC --> LOKI2["Loki logs\n→ Grafana Explore"]
    OC --> TEMPO["Tempo traces\n→ Grafana Explore"]
    OC --> DD["Datadog"]
    OC --> NR["New Relic"]
    OC --> HON["Honeycomb"]
```

**Event types exported:**

| NATS subject | Export payload | Typical destination |
|-------------|---------------|-------------------|
| `dnsdave.query.*` | DNS query log entry (domain, client, type, result, latency) | Loki / Elasticsearch / Splunk / syslog |
| `dnsdave.dhcp.lease.*` | Lease assignment / release / expiry | Loki / InfluxDB / syslog |
| `dnsdave.zone.*` | Zone serial change, record add/delete | syslog / Elasticsearch |
| `dnsdave.dnssec.*` | Key lifecycle, signing events | syslog / SIEM |
| `dnsdave.config.*` | Config mutations (who changed what) | SIEM / syslog / Elasticsearch |
| `dnsdave.blocklist.*` | Sync completions, domain count changes | InfluxDB / Prometheus pushgateway |

### 12.6 Backend Configuration

`dnsdave-export` is configured via `DNSDAVE_EXPORT_CONFIG` (path to a TOML/YAML file) or environment variables. Multiple backends can be active simultaneously.

```toml
# export.toml – example: syslog + Loki + InfluxDB all at once

[backends.syslog]
enabled  = true
protocol = "udp"           # "udp" | "tcp" | "tls"
address  = "192.168.1.5:514"
format   = "rfc5424"       # "rfc5424" | "rfc3164" | "cef"
facility = "local0"
events   = ["query", "dhcp", "config", "zone", "dnssec"]

[backends.loki]
enabled  = true
url      = "http://loki:3100/loki/api/v1/push"
labels   = { app = "dnsdave", env = "home" }
batch_size = 100
batch_wait_ms = 500
events   = ["query", "dhcp"]

[backends.influxdb]
enabled       = true
url           = "http://influxdb:8086"
token         = "${INFLUXDB_TOKEN}"
org           = "home"
bucket        = "dnsdave"
protocol      = "line"     # InfluxDB line protocol over HTTP
events        = ["query", "dhcp"]   # writes as time-series measurements

[backends.otlp]
enabled   = true
endpoint  = "http://otel-collector:4317"   # gRPC
protocol  = "grpc"         # "grpc" | "http"
events    = ["query", "dhcp", "zone", "dnssec", "config"]

[backends.webhook]
enabled    = true
url        = "https://my-siem.example.com/api/events"
method     = "POST"
headers    = { "Authorization" = "Bearer ${SIEM_TOKEN}" }
format     = "json"        # "json" | "ndjson" | "cef"
batch_size = 50
events     = ["config", "dnssec"]  # config changes + key events only

[backends.statsd]
enabled  = true
address  = "telegraf:8125"
protocol = "udp"
prefix   = "dnsdave"
# Metrics only – emits counters and gauges from query/dhcp events
```

### 12.7 Supported Destinations

#### rsyslog / syslog-ng

DNSDave events are forwarded as **RFC 5424** structured syslog messages over UDP or TCP. Structured data elements carry the DNSDave-specific fields (domain, client, result, etc.).

```
<134>1 2026-04-04T14:23:01.842Z dns-node-1 dnsdave - query [dnsdave@12345
    domain="example.com" qtype="A" result="upstream" client="192.168.1.50"
    latency_us="12"] query processed
```

**rsyslog receiver config:**
```
# /etc/rsyslog.d/dnsdave.conf
module(load="imudp")
input(type="imudp" port="514")

template(name="dnsdaveJSON" type="string"
  string="%rawmsg%\n")

if $programname == 'dnsdave' then {
  action(type="omfile" file="/var/log/dnsdave/events.log"
    template="dnsdaveJSON")
  stop
}
```

**CEF format** (for SIEM compatibility – ArcSight, QRadar) is also supported by setting `format = "cef"`:
```
CEF:0|DNSDave|dnsdave-dns|0.1|DNS_QUERY|DNS Query Processed|3|
  src=192.168.1.50 dhost=example.com cat=allowed rt=1712238181842
```

#### Logstash (ELK Stack)

Two integration paths:

**Path 1 – Docker log driver → Logstash** (simplest, no `dnsdave-export` needed):
```yaml
# docker-compose.yml
services:
  dnsdave-dns:
    logging:
      driver: gelf
      options:
        gelf-address: "udp://logstash:12201"
        tag: "dnsdave-dns"
```

**Path 2 – `dnsdave-export` → Elasticsearch bulk API** (richer, structured):
```toml
[backends.elasticsearch]
enabled  = true
url      = "http://elasticsearch:9200"
index    = "dnsdave-{yyyy.MM.dd}"
username = "dnsdave"
password = "${ES_PASSWORD}"
events   = ["query", "dhcp", "zone", "config"]
```

Kibana dashboards: a pre-built `kibana-dashboards.ndjson` export is shipped in the `deploy/kibana/` directory covering query volume, block rate, top clients, DHCP lease map, and zone change audit log.

#### Grafana (Loki + Prometheus + Tempo)

Grafana is the recommended full-stack observability destination:

| Grafana component | DNSDave source | Integration |
|------------------|---------------|-------------|
| **Prometheus** (metrics) | `/metrics` scrape endpoints on all containers | Prometheus scrapes → Grafana datasource |
| **Loki** (logs/events) | `dnsdave-export` → Loki push API | Direct HTTP push, or OTel Collector |
| **Tempo** (traces) | `dnsdave-export` → OTLP | OTel Collector → Tempo |

Pre-built Grafana dashboards are shipped in `deploy/grafana/dashboards/`:
- `dns-overview.json` – QPS, block rate, cache hit ratio, p99 latency
- `dhcp-leases.json` – pool utilisation gauges, lease events, active count
- `dns-query-explorer.json` – Loki-powered query log with client and domain filters
- `cluster-health.json` – per-node QPS sparklines, NATS health, Postgres lag
- `blocklist-sync.json` – list sizes over time, sync durations, error rates
- `dnssec-keys.json` – key expiry countdown, signing events

**Docker Compose with full Grafana stack:**
```yaml
# docker-compose.grafana.yml (include alongside docker-compose.yml)
services:
  prometheus:
    image: prom/prometheus:latest
    volumes:
      - ./deploy/prometheus/prometheus.yml:/etc/prometheus/prometheus.yml
    ports: ["9090:9090"]

  loki:
    image: grafana/loki:latest
    ports: ["3100:3100"]

  grafana:
    image: grafana/grafana:latest
    ports: ["3001:3000"]
    environment:
      GF_DATASOURCE_PROMETHEUS_URL: "http://prometheus:9090"
      GF_DATASOURCE_LOKI_URL: "http://loki:3100"
    volumes:
      - ./deploy/grafana/dashboards:/etc/grafana/provisioning/dashboards
      - ./deploy/grafana/datasources:/etc/grafana/provisioning/datasources

  otel-collector:
    image: otel/opentelemetry-collector-contrib:latest
    volumes:
      - ./deploy/otel/collector.yaml:/etc/otelcol/config.yaml
    ports: ["4317:4317", "4318:4318"]

  dnsdave-export:
    image: ghcr.io/dnsdave/dnsdave-export:latest
    environment:
      DNSDAVE_NATS_URL: "nats://nats:4222"
      DNSDAVE_EXPORT_CONFIG: "/config/export.toml"
    volumes:
      - ./deploy/export/grafana.toml:/config/export.toml
```

#### OpenTelemetry Collector

The OTel Collector is the **recommended universal adapter** when exporting to more than one backend, or when the destination is not directly supported by `dnsdave-export`. `dnsdave-export` sends OTLP (gRPC or HTTP) to the collector, which handles fan-out.

```yaml
# deploy/otel/collector.yaml
receivers:
  otlp:
    protocols:
      grpc:  { endpoint: "0.0.0.0:4317" }
      http:  { endpoint: "0.0.0.0:4318" }

processors:
  batch:
    timeout: 1s
  resource:
    attributes:
      - { action: insert, key: "service.name", value: "dnsdave" }

exporters:
  prometheus:
    endpoint: "0.0.0.0:8889"
  loki:
    endpoint: "http://loki:3100/loki/api/v1/push"
  otlp/tempo:
    endpoint: "http://tempo:4317"
  datadog:
    api:
      key: "${DD_API_KEY}"
  splunk_hec:
    token: "${SPLUNK_HEC_TOKEN}"
    endpoint: "https://splunk:8088/services/collector"
  elasticsearch:
    endpoints: ["http://elasticsearch:9200"]
    index: "dnsdave"

service:
  pipelines:
    logs:
      receivers:  [otlp]
      processors: [batch, resource]
      exporters:  [loki, elasticsearch]
    metrics:
      receivers:  [otlp]
      processors: [batch]
      exporters:  [prometheus, datadog]
    traces:
      receivers:  [otlp]
      processors: [batch]
      exporters:  [otlp/tempo]
```

#### InfluxDB

`dnsdave-export` translates query and DHCP events into InfluxDB **line protocol** measurements, enabling time-series analysis in InfluxDB 2.x and dashboards in Grafana (using the InfluxDB datasource) or Chronograf.

```
# Line protocol examples
dns_query,host=dns-node-1,result=upstream,qtype=A,client_group=default latency_us=12i 1712238181842000000
dns_query,host=dns-node-1,result=blocked,qtype=A,client_group=iot latency_us=0i 1712238181900000000
dhcp_lease,scope=lan,event=assigned ip="192.168.1.50",hostname="mylaptop",mac="aa:bb:cc:dd:ee:ff" 1712238182000000000
```

#### Splunk

`dnsdave-export` sends events to the **Splunk HTTP Event Collector (HEC)**. Events are formatted as Splunk JSON events with a `sourcetype` of `dnsdave:query`, `dnsdave:dhcp`, or `dnsdave:audit`.

Pre-built Splunk apps (Technology Add-On + Dashboard) will be shipped in `deploy/splunk/` covering the same dashboard set as Grafana.

#### Datadog, New Relic, Honeycomb, and others

These are best reached via the **OTel Collector** path: `dnsdave-export → OTLP → otel-collector → {datadog / newrelic / honeycomb}` exporter. All major vendors now publish OTel Collector exporter plugins. No custom code in DNSDave is required.

#### StatsD / Graphite / Telegraf

`dnsdave-export` emits StatsD-format counters and gauges over UDP. This is the integration path for:
- **Graphite** (via StatsD relay)
- **Telegraf** (StatsD input plugin) → any Telegraf output (InfluxDB, Prometheus, etc.)
- **AWS CloudWatch** (via `cloudwatch-agent`)

#### Generic HTTP Webhook

A configurable HTTP POST backend for any system not covered above (PagerDuty, OpsGenie, custom SIEM, home automation, etc.). Payload can be formatted as JSON, NDJSON, or CEF. Supports batching, retry with exponential back-off, and per-event-type filtering.

### 12.8 Event Schema (Exported Payloads)

All backends receive a normalised event envelope regardless of the wire format:

```rust
struct ExportEvent {
    ts:          DateTime<Utc>,
    event_type:  EventType,       // Query | DhcpLease | ZoneChange | Config | Dnssec
    node_id:     String,
    trace_id:    String,
    payload:     EventPayload,    // type-specific fields
}

enum EventPayload {
    Query {
        client_ip:    IpAddr,
        client_group: String,
        domain:       String,
        qtype:        String,
        result:       QueryResult,  // Allowed | Blocked | Cached | Local | Forward
        source:       String,       // upstream name, forward zone, "blocklist", etc.
        latency_us:   u64,
        dnssec_ad:    bool,
    },
    DhcpLease {
        event:    LeaseEvent,  // Assigned | Renewed | Released | Expired
        scope_id: String,
        mac:      [u8; 6],
        ip:       IpAddr,
        hostname: String,
        duration_s: u32,
    },
    ZoneChange {
        zone:   String,
        serial: u64,
        action: String,  // "record_added" | "record_deleted" | "serial_bumped"
        name:   String,
        rtype:  String,
    },
    Config {
        action:   String,  // "created" | "updated" | "deleted"
        resource: String,  // "dns_record" | "zone" | "forward_zone" | ...
        id:       String,
        actor:    String,  // API key ID that made the change
    },
    Dnssec {
        zone:     String,
        key_type: String,  // "ksk" | "zsk"
        action:   String,  // "generated" | "rolled" | "retired"
        key_tag:  u16,
    },
}
```

---

## 13. Security

| Concern | Mitigation |
|---------|-----------|
| Unauthenticated DNS | Standard; API always requires key/JWT |
| Unauthenticated DHCP | Standard protocol; rogue server protection via NATS leader election (only one active DHCP server per broadcast domain) |
| API key storage | Argon2id-hashed at rest; never logged |
| DoT/DoH TLS | ACME auto-provisioned (public domains) or local CA (private deployments) – see §13.1 |
| NATS auth | Per-container NKey credentials; deny-all default ACLs; TLS on all NATS connections |
| DHCP option injection | Options stored as typed values; raw hex only for explicitly declared custom codes |
| Rate limiting | Per-client DNS token bucket; API token bucket; DHCP DISCOVER rate limit per MAC |
| Log retention | `query_log` and `dhcp_leases` auto-purged (default 30 days / 90 days respectively) |
| Container isolation | `distroless/static` base; no shell |
| Secrets | `.env` or Kubernetes `Secret`; never baked into images |
| CORS | Configurable allowed origins |
| Postgres TLS | All container→Postgres connections require `sslmode=verify-full`; CA cert mounted as a volume |

NATS per-container permissions:
- `dnsdave-dns`: publish `dnsdave.query.*`; subscribe `dnsdave.config.>`, `dnsdave.blocklist.>`, `dnsdave.dhcp.lease.>`, `dnsdave.zone.>`, `dnsdave.dnssec.>`
- `dnsdave-dhcp`: publish `dnsdave.dhcp.lease.*`; subscribe `dnsdave.config.>`; no publish on config subjects
- `dnsdave-api`: publish `dnsdave.config.*`, `dnsdave.zone.*`, `dnsdave.dnssec.*`; no publish on query or lease subjects
- `dnsdave-export`: subscribe `dnsdave.query.*`, `dnsdave.dhcp.lease.*`, `dnsdave.zone.*`, `dnsdave.config.*`, `dnsdave.dnssec.*`, `dnsdave.blocklist.*`; publish nothing; no Postgres access; deny-all on any publish subject
- `dnsdave-netbox`: subscribe `dnsdave.dhcp.lease.*`, `dnsdave.dhcp.scope.*`, `dnsdave.zone.record.*`, `dnsdave.config.dhcp.*`; publish nothing; no Postgres access; deny-all on any publish subject

### 13.1 Certificate Management

Certificates are needed in three places: **DNSDave's own services**, **inter-container transport**, and **ACME challenge delegation to other services**.

#### DNSDave's Own Services

All three TLS endpoints (REST API, DoT :853, DoH) can share a single certificate or use separate certs:

| Endpoint | Port | Notes |
|----------|------|-------|
| REST API (HTTPS) | 8080 / 443 | Clients, browser UI, Pi-hole shim |
| DNS-over-TLS | 853 | Mobile resolvers (iOS, Android, Quad9-compatible) |
| DNS-over-HTTPS | 8080/dns-query | DoH clients, browsers |

**Deployment mode determines the certificate source:**

| Deployment | Certificate source | Notes |
|------------|-------------------|-------|
| Public domain | ACME (Let's Encrypt / ZeroSSL) | DNS-01 challenge via RFC 2136 to own authoritative zone (no port 80 needed) |
| Private / home lab | Local CA or self-signed | `rcgen` generates a self-signed cert on first boot; or mount a cert from a local CA (Step CA, XCA, cfssl) |
| Kubernetes | cert-manager | Annotate the `Ingress` / `Certificate` resource; cert-manager handles renewal |

**ACME challenge strategy for DNSDave itself:** Because DNSDave is authoritative for its own zone, it can satisfy DNS-01 challenges by writing `_acme-challenge.` TXT records via its own RFC 2136 interface. The `dnsdave-api` container includes an ACME client (`instant-acme` crate) that:

1. Requests a certificate from the ACME CA.
2. Places the DNS-01 challenge TXT record via RFC 2136 to `dnsdave-dns`.
3. Notifies the CA to validate.
4. Removes the TXT record on success.
5. Stores the issued cert + key in the `tls_certificates` table (encrypted) and publishes `dnsdave.config.tls.renewed` to NATS.
6. All DNS and API containers reload the cert in-memory within seconds of the NATS event.

Self-signed bootstrap: on first start without a configured cert, `dnsdave-api` generates a self-signed cert via `rcgen` (valid 90 days, auto-regenerated). A warning is emitted and surfaced in the API health endpoint.

#### Certificate Storage

```sql
CREATE TABLE tls_certificates (
    id           TEXT PRIMARY KEY,
    domain       TEXT NOT NULL,
    source       TEXT NOT NULL,   -- 'acme' | 'self_signed' | 'manual'
    cert_pem_enc TEXT NOT NULL,   -- AES-GCM encrypted PEM certificate chain
    key_pem_enc  TEXT NOT NULL,   -- AES-GCM encrypted PEM private key
    issued_at    TIMESTAMP NOT NULL,
    expires_at   TIMESTAMP NOT NULL,
    renewed_at   TIMESTAMP,
    created_at   TIMESTAMP DEFAULT NOW()
);
```

Renewal is triggered at 30 days before expiry (for ACME certs) or 7 days (self-signed) by a scheduled task in `dnsdave-api`. On renewal, `dnsdave.config.tls.renewed` is published; all DNS nodes hot-reload the new cert from Postgres without restarting.

#### Inter-Container TLS (NATS + Postgres)

| Connection | TLS |
|-----------|-----|
| Container → NATS | TLS with per-container client cert (NKey + mTLS); CA cert is the NATS deployment's CA |
| Container → Postgres | `sslmode=verify-full`; server cert verified against a mounted CA bundle |
| API → DNS (health check) | HTTPS with the API cert |

In Docker Compose, the NATS server is started with `--tls` and certs generated by `dnsdave-api` on first boot or by a mounted volume. In Kubernetes, cert-manager issues NATS and Postgres TLS certs automatically.

#### DNSDave as ACME DNS-01 Helper for Other Services

Because DNSDave is an authoritative server for local zones, it can satisfy ACME DNS-01 challenges for **any other service** in the environment that needs a certificate. This is one of the most practical benefits of running a local authoritative DNS server.

```bash
# Example: issue a cert for a local Nginx instance using certbot + RFC 2136
certbot certonly \
  --dns-rfc2136 \
  --dns-rfc2136-credentials /etc/certbot/dnsdave-rfc2136.ini \
  -d nginx.home.arpa

# /etc/certbot/dnsdave-rfc2136.ini
dns_rfc2136_server  = 192.168.1.1
dns_rfc2136_port    = 53
dns_rfc2136_name    = nginx.home.arpa
dns_rfc2136_secret  = <TSIG key or leave empty if IP-based allow_update>
dns_rfc2136_algorithm = HMAC-SHA256
```

This works for any service running in the local environment: Nginx, Proxmox, Home Assistant, Kubernetes Ingress, etc. The cert is issued by Let's Encrypt (real, browser-trusted) because the ACME CA only validates that the TXT record exists at the DNS level – it doesn't care whether the server is publicly reachable.

**Requirement:** The zone (`home.arpa`) must be publicly delegated to DNSDave, OR the ACME CA must be an internal CA that trusts it. For truly private zones (`home.arpa` is never publicly delegated), only an internal ACME CA (e.g., Step CA, Smallstep, HashiCorp Vault) will work. Let's Encrypt cannot validate a zone that has no public NS delegation.

---

## 14. Milestones

### v0.1 – Core DNS + event bus foundations
- [ ] `dnsdave-dns`: Rust, SO_REUSEPORT, UDP, forward-only resolver
- [ ] `dnsdave-api`: REST API skeleton, Postgres, config CRUD
- [ ] NATS JetStream + KV store configured
- [ ] Config events published on every API write
- [ ] `dnsdave-dns` consumes config events, rebuilds in-memory state
- [ ] Basic blocklist ingest (hosts format) via sync worker
- [ ] `docker-compose.yml` reference stack
- [ ] Benchmark suite baseline

### v0.2 – Blocklist ecosystem
- [ ] All list formats (adblock, domain-only, RPZ)
- [ ] MPHF blocklist map (`boomphf`) + Bloom filter hot-swap
- [ ] Bloom filter → NATS Object Store; DNS hot-swap
- [ ] `dnsdave-log`: NATS consumer → Postgres bulk writer
- [ ] `dnsdave-stats`: in-memory counters from query events
- [ ] Groups + per-group blocklist policy
- [ ] Pi-hole API shim (v5)

### v0.3 – Advanced DNS + Local Zone Authority + Forwarding
- [ ] Wildcard records + copy-on-write label trie
- [ ] CNAME chaining, PTR, split-horizon views
- [ ] DoT + DoH upstream with HTTP/2 connection pooling
- [ ] Negative caching; cache pre-warming from JetStream replay
- [ ] Zone management (`dns_zones` table, SOA, NS records)
- [ ] Zone authority check in hot path – authoritative NXDOMAIN + SOA for owned zones
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

### v0.4 – Production readiness + TLS + DNSSEC validation
- [ ] DoT listener (:853) + DoH listener (:8080/dns-query)
- [ ] Self-signed cert bootstrap via `rcgen` on first start (no manual config required)
- [ ] ACME client (`instant-acme`) for public domain certificate issuance via DNS-01 over RFC 2136
- [ ] `tls_certificates` Postgres table; hot-reload on `dnsdave.config.tls.renewed` NATS event
- [ ] NATS TLS with per-container client certs; Postgres `sslmode=verify-full`
- [ ] Internal CA generation and cert distribution for Docker Compose / Podman stacks
- [ ] cert-manager `Certificate` resources for Kubernetes deployments
- [ ] ACME DNS-01 helper: `certbot --dns-rfc2136` works against any DNSDave-owned zone
- [ ] Prometheus metrics across all containers (µs granularity)
- [ ] SSE live log stream via NATS
- [ ] Helm chart v1 (DaemonSet DNS, replicated API, NATS StatefulSet)
- [ ] `recvmmsg`/`sendmmsg` batch I/O + NATS NKey auth
- [ ] DNSSEC validation: DO bit on upstream queries; AD bit on validated replies; SERVFAIL on failure
- [ ] Trust anchor management (root KSK IANA, custom anchors for internal CAs)
- [ ] DNSSEC-aware negative caching (NSEC/NSEC3 validation)
- [ ] Global `dnssec_validate` setting + per-upstream override

### v0.4.5 – DNSSEC signing
- [ ] ZSK + KSK generation (Ed25519 / ECDSA P-384) via `ring` crate
- [ ] Private key encrypted storage in Postgres (`DNSDAVE_SECRET_KEY`)
- [ ] Inline zone signing: RRSIG regeneration on every record write
- [ ] NSEC3 records with 0 iterations, empty salt (RFC 9276)
- [ ] `GET /api/v1/dns/dnssec/zones/:name/ds` – DS record export
- [ ] ZSK pre-publish rollover (automated via NATS KV key lifecycle)
- [ ] `dnsdave.dnssec.*` NATS events; `DNSDAVE_DNSSEC` JetStream stream

### v0.5 – Observability export + monitoring stack

- [ ] `dnsdave-export` container: NATS subscriber, fan-out to configured backends
- [ ] Syslog backend (RFC 5424 + RFC 3164 + CEF) over UDP and TCP; rsyslog/syslog-ng compatible
- [ ] Loki push API backend (batched HTTP)
- [ ] OTLP backend (gRPC + HTTP) for OpenTelemetry Collector
- [ ] InfluxDB line protocol backend (HTTP write API)
- [ ] StatsD UDP backend (counters/gauges for query and DHCP events)
- [ ] Elasticsearch bulk API backend
- [ ] Splunk HEC backend
- [ ] Generic HTTP webhook backend (JSON, NDJSON, CEF)
- [ ] `dnsdave-export` multi-arch container image (same targets as other containers)
- [ ] **Monitoring stack** (`deploy/monitoring/`) – see `MONITORING.md`:
  - [ ] `docker-compose.monitoring.yml` – Prometheus, Alertmanager, Loki, Promtail, Grafana, node-exporter, cAdvisor, postgres-exporter
  - [ ] `docker-compose.monitoring.lite.yml` – reduced-resource profile for Raspberry Pi
  - [ ] `deploy/monitoring/prometheus/prometheus.yml` – all scrape configs for every DNSDave container
  - [ ] `deploy/monitoring/prometheus/rules/` – 5 alert rule files (dns, dhcp, blocklist, infrastructure, security) with 25+ alert definitions
  - [ ] `deploy/monitoring/alertmanager/alertmanager.yml` – routing to email, Slack, PagerDuty, webhook
  - [ ] `deploy/monitoring/loki/loki.yml` – Loki storage, 90-day retention, TSDB schema
  - [ ] `deploy/monitoring/promtail/promtail.yml` – Docker log discovery + JSON pipeline
  - [ ] `deploy/monitoring/grafana/provisioning/` – auto-provisioned data sources (Prometheus, Loki, optional Tempo) and dashboard provider
  - [ ] `deploy/monitoring/grafana/dashboards/dns-overview.json` – NOC overview dashboard
  - [ ] `deploy/monitoring/grafana/dashboards/dns-performance.json` – latency heatmap, percentiles, upstream health
  - [ ] `deploy/monitoring/grafana/dashboards/dhcp.json` – pool gauges, lease lifecycle, NAK rate
  - [ ] `deploy/monitoring/grafana/dashboards/blocklists.json` – domain counts, sync health
  - [ ] `deploy/monitoring/grafana/dashboards/client-activity.json` – top clients, per-group stats
  - [ ] `deploy/monitoring/grafana/dashboards/system-resources.json` – container CPU/memory, host OS, Postgres, NATS
  - [ ] `deploy/monitoring/grafana/dashboards/cluster-health.json` – per-node QPS, NATS lag, leader indicator
  - [ ] `deploy/monitoring/grafana/dashboards/netbox-sync.json` – push/pull rates, conflicts, lag
  - [ ] `deploy/monitoring/grafana/dashboards/security-audit.json` – auth failures, lockouts, API key usage
  - [ ] `deploy/monitoring/grafana/dashboards/query-log-explorer.json` – Loki-backed interactive log search
  - [ ] Kubernetes: `ServiceMonitor` CRDs for all containers; `PrometheusRule` CRD; Grafana `ConfigMap` with sidecar import
  - [ ] `deploy/k8s/monitoring/kube-prometheus-stack-values.yml` + `loki-values.yml`
  - [ ] `deploy/monitoring/otel/otel-collector.yml` – optional OTel Collector pipeline for traces → Tempo

### v0.6 – DHCP
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

### v0.6 – Multi-node HA + scalability
- [ ] `dnsdave-dns` horizontal scale (N nodes, independent NATS subscribers)
- [ ] NATS JetStream 3-node cluster
- [ ] Leader election for `dnsdave-sync`, `dnsdave-stats`, `dnsdave-dhcp` via NATS KV
- [ ] Postgres Patroni HA; active-active DNS with keepalived/VRRP
- [ ] Pi-hole API shim (v6)

### v0.7 – NetBox Integration

- [ ] `dnsdave-netbox` container: NATS subscriber, dual-mode (Diode gRPC + REST API)
- [ ] `dhcp.scope.created/updated/deleted` NATS subjects published by `dnsdave-api`
- [ ] Prefix push: DHCP scope creation/update/deletion → NetBox Prefix lifecycle
- [ ] IP Range push: DHCP pool bounds → NetBox IP Range
- [ ] IP Address push: lease assigned/released/expired → NetBox IP Address (status management)
- [ ] IP Address push: static reservations → NetBox IP Address (status=dhcp, reserved=true)
- [ ] DNS record sync: A/AAAA records → NetBox IP Address `dns_name` field
- [ ] Merge strategy: IP Address retains lease status + DNS name when both sources present
- [ ] Conflict strategies: overwrite / skip / tag_conflict / merge
- [ ] Lifecycle config: on_lease_expiry / on_lease_release / on_scope_delete
- [ ] VRF, site, tenant, tag configuration
- [ ] NetBox custom field bootstrap script (`deploy/netbox/bootstrap_custom_fields.py`)
- [ ] Diode mode: gRPC push via `tonic` + `prost`; auto-reconnect on Diode server restart
- [ ] REST API mode: create-or-update with conflict detection; token-bucket rate limiting
- [ ] Device auto-creation (off by default): MAC lookup → assign IP to interface; create Device if not found; guarded by per-hour cap, dry-run mode, MAC-prefix allowlist
- [ ] Health endpoint `:9095/health`; Prometheus metrics `:9095/metrics`
- [ ] NATS lag alerting: `max_lag_seconds` threshold → health endpoint `503`
- [ ] Reference `docker-compose.netbox.yml` (Diode + dnsdave-netbox)
- [ ] Reference `docker-compose.netbox-api.yml` (REST API only, no Diode)
- [ ] Helm chart additions: `Deployment/dnsdave-netbox`, `ConfigMap`, `Secret`
- [ ] `dnsdave-netbox` multi-arch container image (amd64, arm64, arm/v7)

### v0.8 – RBAC and Identity Management

- [ ] `iam_users`, `iam_groups`, `iam_roles`, `iam_role_permissions`, `iam_group_members`, `iam_group_roles`, `iam_user_permissions`, `iam_api_keys`, `iam_refresh_tokens`, `iam_audit_log` Postgres tables
- [ ] Argon2id password hashing (memory: 64 MB, time: 3, parallelism: 4)
- [ ] JWT issuance (Ed25519 signing); 15-minute access token + 30-day refresh cookie
- [ ] Effective permission computation on login (union of all group roles minus direct denies)
- [ ] Session invalidation via `session_version` + NATS `dnsdave.iam.session_invalidated.{user_id}`
- [ ] API key creation: Argon2id hash, prefix stored, permission subset enforced
- [ ] API key source-IP CIDR restriction + expiry
- [ ] In-process LRU permission cache for API key lookups (no per-request Postgres read)
- [ ] `POST /api/v1/auth/login` with lockout after N failed attempts + unlock endpoint
- [ ] `GET /api/v1/auth/me` returning effective permission set in JWT format
- [ ] Full `/api/v1/iam/users`, `/groups`, `/roles`, `/permissions`, `/audit` REST API
- [ ] Authorisation middleware: permission check on every handler; scope injection for `own`/`group`
- [ ] `iam_audit_log` writer: every mutating call logged with before/after JSON diff
- [ ] NATS `dnsdave.iam.audit` event on every auditable action (consumed by `dnsdave-log`, `dnsdave-export`)
- [ ] Seven built-in roles pre-seeded (super_admin, admin, dns_admin, dhcp_admin, read_only, operator, netbox_sync)
- [ ] Breached-password k-anonymity check (optional; enabled via config)
- [ ] Direct permission grant/deny API (expiring, with reason field)
- [ ] Audit log partitioned by day; configurable retention (default: 365 days)
- [ ] UI: login page, permission store, `<Guard>` / `<GuardPage>` / `<GuardDisable>` components
- [ ] UI: Users / Groups / Roles / Audit Log tabs with full matrix editor

### v0.9 – Multi-Homed Devices and Networks

- [ ] `devices`, `device_interfaces`, `device_interface_addresses` Postgres tables
- [ ] `POST /api/v1/devices` – create device with N interfaces and M addresses per interface; atomically creates DHCP reservations and DNS records
- [ ] `POST /api/v1/devices/from-lease` – promote an existing dynamic lease to a static device + reservation
- [ ] `PATCH /api/v1/devices/:id/interfaces/:iface_id` – update interface MAC, alias, VLAN; hot-patches in-memory affinity table via NATS
- [ ] Network-affinity DNS resolution in `dnsdave-dns` hot path: `ArcSwap<Vec<(IpNet, Vec<RData>)>>` affinity table; ≤ 200 ns overhead per query
- [ ] Naming strategies: `round_robin`, `affinity`, `primary_only`, `per_interface`, `explicit`
- [ ] Per-interface DNS alias generation (auto and manual)
- [ ] PTR record auto-creation for all device interface addresses
- [ ] RFC 2136 DNS UPDATE scoped to individual interface records
- [ ] Multi-reservation device linking: `device_interfaces.mac_address` → `dhcp_reservations`
- [ ] Lease correlation: `dnsdave.dhcp.lease.*` events carry `device_id` + `interface_id`
- [ ] Option 82 (agent circuit ID + remote ID) parsed, logged, available as scope-selection hint
- [ ] VLAN ID field on scopes; Option 82 circuit-ID → VLAN → scope auto-mapping (configurable)
- [ ] VRF-aware scope isolation: split-horizon view per VRF keyed on client group CIDR
- [ ] `listen` array on DNS and DHCP containers for per-interface binding
- [ ] NATS subjects: `dnsdave.device.*` and `dnsdave.device.interface.*` and `dnsdave.device.address.*`
- [ ] NetBox pull: import multi-interface devices; create device + all interface addresses
- [ ] NetBox push: push device record with all interfaces and IPs to `dcim.Device` / `dcim.Interface` / `ipam.IPAddress`
- [ ] UI: Device Management page (§6.15) with interface topology cards, lease status, DNS record summary

### v1.0 – Multi-region
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
| `dnsdave-export` | **Rust** | `async-nats`, `tokio`, `opentelemetry`, `opentelemetry-otlp`, `tonic` (gRPC), `reqwest`, `serde_json`, `serde`, `toml` (config), `bytes` (line protocol), `tracing` |
| `dnsdave-netbox` | **Rust** | `async-nats`, `tokio`, `tonic` (gRPC for Diode), `prost` (protobuf), `reqwest` (REST API mode), `serde_json`, `serde`, `toml` (config), `tracing`, `governor` (rate-limit token bucket) |

### 15.2 Infrastructure

| Concern | Choice | Rationale |
|---------|--------|-----------|
| Event bus | **NATS JetStream** | <100µs publish; 15MB image; Object Store + KV built-in; `async-nats` Rust client |
| DHCP library | `dhcproto` | Pure Rust; DHCPv4 + DHCPv6 encode/decode; no C dependencies |
| DNS library | `hickory-dns` | Primary Rust DNS library; wire format access for zero-copy; built-in DNSSEC validation and signing support |
| DNSSEC crypto | `ring` + `rcgen` | ECDSA P-384, Ed25519 key generation and signing; no C deps on modern targets |
| TLS cert generation | `rcgen` | Self-signed and CA cert bootstrap without OpenSSL; pure Rust |
| ACME client | `instant-acme` | Async ACME v2 (RFC 8555); DNS-01 challenge via RFC 2136 self-update |
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
| CI | GitHub Actions | Multi-arch build (`linux/amd64`, `linux/arm64`, `linux/arm/v7`); QEMU via `docker/setup-qemu-action`; `cross` for Rust cross-compilation; multi-arch manifests pushed to GHCR |
| Cross-compilation | `cross` (cross-rs) | Docker-based Rust cross-compilation; same binary produced locally and in CI |
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

1. **Web UI scope** – API-only for v1, or ship a minimal read-only Svelte dashboard subscribing to SSE streams?
2. **RPZ as output** – Serve RPZ zones to downstream resolvers by consuming `dnsdave.blocklist.*`? (Note: DNSDave can already ingest RPZ format. Serving RPZ to downstream resolvers is a separate feature.)
3. **DoH port** – Serve DoH on `:8080/dns-query` (shared with API) or dedicated `:443`?
4. **Multi-tenancy** – First-class namespace/org concept for SaaS, or single-tenant-per-instance?
5. **`dnsdave-stats` persistence** – In-memory only (lost on restart) or checkpoint to NATS KV periodically?
6. **NATS vs Redpanda at scale** – Benchmark trigger: >500K QPS sustained for >1 hour.
7. **Bloom filter fetch at startup** – 90MB bloom filter is ~700 NATS Object Store chunks. Benchmark fetch + reassemble time vs. direct Postgres read. If Postgres is faster, use it only for startup and NATS only for live updates.
8. **DHCP HA across broadcast domains** – In a multi-site Kubernetes deployment, each node needs a DHCP server on its local broadcast domain. DaemonSet is the right primitive, but leader election must be per-node, not cluster-wide. Design the NATS KV lock key as `dhcp.leader.{node_name}`.
9. **DHCPv6 prefix delegation (IA_PD)** – For ISP-style deployments where `dnsdave-dhcp` delegates IPv6 prefixes to customer routers. Scope for v0.5 or push to v1?
10. **RFC 2136 TSIG auth** – DNS UPDATE packets can be authenticated with TSIG (HMAC-MD5 / HMAC-SHA256). Should this be required for RFC 2136 updates, or is IP-CIDR (`allow_update`) sufficient for v0.3?
11. **mDNS unicast bridge** – Is the mDNS→unicast bridge in scope for v0.6, or only if there is user demand? Requires binding a multicast socket which may conflict with host mDNS daemons (avahi, mDNSResponder).
12. **Secondary zone bootstrapping** – For `zone_type = secondary`, DNSDave must initiate an initial AXFR from the primary on zone creation. Which TCP connection pool handles this (reuse `dnsdave-dns` or a dedicated worker)?
13. **Forward zone blocklisting** – Should forward zones optionally apply the local blocklist before forwarding (for filtering ads even on corporate DNS)? Or keep the bypass unconditional for simplicity?
14. **DNSSEC validation and Pi-hole compatibility** – Pi-hole extensions and mobile apps send DNS queries directly; they don't set the DO bit. With strict DNSSEC validation enabled, should dnsdave-dns only validate its own upstream queries, or also re-validate queries that clients request validation for (CD bit)?
15. **DNSSEC key rollover automation** – ZSK rollover requires coordinated publish→activate→retire phases over days. Should this be managed by a cron-style job within `dnsdave-api`, or a dedicated `dnsdave-keymgr` sidecar subscribed to NATS?
16. **Trust anchor automatic updates** – The IANA root KSK rolls every ~5 years (next: ~2026). Should DNSDave fetch the current root trust anchor from IANA's XML feed automatically, or require manual operator action?
17. **Local CA vs external CA for private deployments** – For home labs with no public NS delegation, the ACME DNS-01 flow requires a private ACME CA (Step CA, Vault PKI, etc.). Should DNSDave ship with an embedded minimal ACME CA, or document how to integrate with an external one? Shipping one adds significant scope but removes a dependency.
18. **Certificate SAN coverage** – DoT/DoH clients validate the server certificate hostname. In multi-node deployments (multiple `dnsdave-dns` instances), each node needs a cert that covers its individual hostname AND the shared DNS service name. Should certs use a wildcard SAN, or per-node issuance?
19. **ACME DNS-01 for split-horizon zones** – If the zone exists only locally (no public NS delegation), Let's Encrypt cannot validate DNS-01 challenges. Should DNSDave detect this case and warn the operator, or automatically fall back to self-signed?
20. **`dnsdave-export` back-pressure** – If a backend (e.g., Logstash, remote Loki) is slow or down, NATS JetStream consumer ACK delay will build up. Should `dnsdave-export` use a bounded in-memory queue per backend with drop-oldest semantics, or block ACK and let JetStream naturally slow the consumer? The latter risks head-of-line blocking for other subjects on the same consumer.
21. **Export schema versioning** – As new event types or fields are added, how should backward compatibility be maintained for external consumers that have dashboards or parser rules built against the schema? Versioned `event_schema_version` field in every envelope, with a compatibility matrix doc?
22. **Syslog TLS (RELP/RFC 5425)** – For production deployments where syslog traffic must be encrypted (PCI-DSS, HIPAA). `dnsdave-export` should support TLS-wrapped TCP syslog (RFC 5425) and optionally RELP. Is this in scope for v0.5 or a follow-up?
23. **NetBox device auto-creation** – When `dnsdave-netbox` sees a new DHCP lease for an unknown MAC address, should it be allowed to auto-create a Device/VM record in NetBox? Powerful but risky: misconfigured auto-creation can pollute the NetBox DCIM database. Default should be off; what guard-rails prevent runaway creation (rate limit, max-per-hour cap, dry-run mode)?
24. **NetBox VRF mapping** – In a multi-VRF environment, how should `dnsdave-netbox` determine which NetBox VRF a DHCP scope belongs to? Per-scope config annotation, or automatic mapping from the scope's gateway interface?
25. **Diode vs REST API consistency** – Diode performs asynchronous reconciliation; a REST API push is synchronous. If `dnsdave-netbox` is in Diode mode and the Diode server is down, events queue in NATS JetStream. In REST API mode, a NetBox outage blocks the ACK. Should both modes expose a configurable `max_lag_seconds` threshold that alerts via the health endpoint?

---

## 17. NetBox Integration

DNSDave integrates with [NetBox](https://netbox.dev) 4.5+ to automatically push IPAM data – IP addresses, prefixes, IP ranges, and optionally device records – as DHCP leases are assigned, scopes are created, and DNS records are updated. This closes the "I" in DDI without requiring NetBox to poll DNSDave; data flows from NATS events in near-real-time.

### 17.1 Architecture

`dnsdave-netbox` is an optional, stateless container that subscribes to a subset of NATS subjects and pushes corresponding objects to NetBox. Like `dnsdave-export`, it makes zero changes to any existing container and has no database access – all its data comes from the NATS event bus.

```mermaid
flowchart LR
    subgraph DNSDave
        DHCP["dnsdave-dhcp"]
        API["dnsdave-api"]
        NATS["NATS JetStream\ndhcp.lease.*\ndhcp.scope.*\nzone.record.*"]
        NB["dnsdave-netbox"]
    end

    DHCP -->|"lease events"| NATS
    API  -->|"scope / record events"| NATS
    NATS --> NB

    subgraph Integration Layer
        DIODE["Diode Server\n(gRPC · async reconciliation)"]
        REST["NetBox REST API\n(HTTP · synchronous)"]
    end

    NB -->|"mode = diode"| DIODE
    NB -->|"mode = api"| REST

    subgraph NetBox 4.5+
        PFX["Prefixes"]
        IPR["IP Ranges"]
        IPA["IP Addresses"]
        DEV["Devices / Interfaces"]
        VMS["Virtual Machines"]
    end

    DIODE --> PFX & IPR & IPA & DEV & VMS
    REST  --> PFX & IPR & IPA & DEV & VMS
```

### 17.2 Integration Modes

#### Mode A – Diode (recommended for continuous / high-frequency updates)

[Diode](https://github.com/netboxlabs/diode) is NetBox Labs' open-source gRPC ingestion agent. It accepts a stream of entity objects, handles idempotent create-or-update reconciliation internally, and pushes changes to NetBox in batches. The key properties that make it the recommended mode for DNSDave:

- **Stateless push model:** `dnsdave-netbox` sends every event without needing to track whether the object already exists in NetBox – Diode handles "does this IP already exist? update or create" transparently.
- **High throughput:** gRPC + protobuf is significantly more efficient than HTTP JSON for the volume of events at busy networks (thousands of lease events per hour).
- **NATS back-pressure handling:** if the Diode server is temporarily unavailable, NATS JetStream holds the unacked events and delivers them when Diode reconnects. No data loss.
- **Decoupled from NetBox availability:** the Diode server can buffer events and replay them even if NetBox itself is in maintenance mode.

`dnsdave-netbox` communicates with a `diode-server` container via gRPC using the Diode proto schema. The Diode server is a separate container added to the compose stack.

#### Mode B – NetBox REST API (for fine-grained control or existing deployments)

For operators who already have a NetBox deployment and prefer not to run a Diode server, `dnsdave-netbox` can push directly to the NetBox 4.5 REST API (`/api/ipam/`, `/api/dcim/`).

- **Synchronous:** each NATS event triggers one or more HTTP calls to NetBox.
- **Stateful conflict handling:** `dnsdave-netbox` must check whether an object exists (HTTP `GET`) before deciding to `POST` (create) or `PATCH` (update). This requires two round-trips per event.
- **Rate limited:** NetBox's API enforces rate limiting; `dnsdave-netbox` implements token-bucket throttling on outgoing requests.
- **NetBox outage risk:** if NetBox is unreachable, the NATS consumer ACK is delayed, causing NATS JetStream back-pressure. A configurable `max_lag` threshold triggers an alert via the health endpoint if unacked events accumulate.

### 17.3 NATS Events Consumed

| NATS Subject | Trigger | NetBox Object(s) Affected |
|---|---|---|
| `dnsdave.dhcp.scope.created.{scope_id}` | DHCP scope created via API | Create Prefix (network/mask); create IP Range (pool start–end) |
| `dnsdave.dhcp.scope.updated.{scope_id}` | DHCP scope modified | Update Prefix description, tags; update IP Range bounds |
| `dnsdave.dhcp.scope.deleted.{scope_id}` | DHCP scope removed | Prefix → status `container`; IP Range removed (configurable) |
| `dnsdave.dhcp.lease.assigned.{scope_id}` | Lease assigned or renewed | Create or update IP Address (status=`dhcp`); optionally update Device interface |
| `dnsdave.dhcp.lease.released.{scope_id}` | Client sent RELEASE | IP Address status → `available` or remove (configurable) |
| `dnsdave.dhcp.lease.expired.{scope_id}` | Lease timer expired | IP Address status → `available` or remove (configurable) |
| `dnsdave.zone.record.created.{zone}` | DNS A or AAAA record added | Create or update IP Address with `dns_name` set |
| `dnsdave.zone.record.deleted.{zone}` | DNS A or AAAA record removed | Clear `dns_name` from IP Address, or remove if no lease |

### 17.4 NetBox Object Mapping

#### DHCP Scope → Prefix + IP Range

```
dnsdave_scope {                     NetBox Prefix {
  id:         "scope_lan"             prefix:      "192.168.1.0/24"
  name:       "LAN"         →         description: "LAN (dnsdave scope_lan)"
  network:    "192.168.1.0"           status:      "active"
  prefix_len: 24                      tags:        ["dnsdave", "dhcp"]
  gateway:    "192.168.1.1"           custom_fields: {
  pool_start: "192.168.1.100"           dnsdave_scope_id: "scope_lan"
  pool_end:   "192.168.1.200"           dnsdave_gateway:  "192.168.1.1"
}                                     }
                                    }

                                    NetBox IP Range {
                                      start_address: "192.168.1.100/24"
                                      end_address:   "192.168.1.200/24"
                                      status:        "dhcp"
                                      description:   "DHCP pool (scope_lan)"
                                      tags:          ["dnsdave", "dhcp"]
                                    }
```

#### DHCP Lease → IP Address

```
dnsdave_lease {                     NetBox IP Address {
  ip:         "192.168.1.105"         address:     "192.168.1.105/24"
  mac:        "aa:bb:cc:dd:ee:ff"  →  status:      "dhcp"
  hostname:   "mylaptop"              dns_name:    "mylaptop.home.arpa"
  scope_id:   "scope_lan"            description: "DHCP lease (aa:bb:cc:dd:ee:ff)"
  expires_at: 1712330000             tags:        ["dnsdave", "dhcp-lease"]
}                                     custom_fields: {
                                        dnsdave_mac:        "aa:bb:cc:dd:ee:ff"
                                        dnsdave_scope_id:   "scope_lan"
                                        dnsdave_expires_at: "2026-04-05T10:00:00Z"
                                      }
                                    }
```

#### Static DHCP Reservation → IP Address

```
dnsdave_reservation {               NetBox IP Address {
  ip:       "192.168.1.10"            address:  "192.168.1.10/24"
  mac:      "11:22:33:44:55:66"  →    status:   "dhcp"
  name:     "nas"                     dns_name: "nas.home.arpa"
  scope_id: "scope_lan"              description: "Static reservation (nas)"
}                                     custom_fields: {
                                        dnsdave_mac:      "11:22:33:44:55:66"
                                        dnsdave_reserved: true
                                      }
                                    }
```

#### DNS A/AAAA Record → IP Address

```
dnsdave_record {                    NetBox IP Address {
  name:  "server.home.arpa"           address:  "192.168.1.1/24"
  type:  "A"                  →       status:   "active"
  value: "192.168.1.1"                dns_name: "server.home.arpa"
  zone:  "home.arpa"                  tags:     ["dnsdave", "dns-record"]
}                                   }
```

When both a DHCP lease and a DNS record exist for the same IP, the objects are merged: the IP Address retains `status=dhcp` from the lease and gains `dns_name` from the record.

#### Optional: Device / Interface Auto-creation

When `sync.devices = true` (disabled by default), on each lease assignment `dnsdave-netbox` queries NetBox for an existing Device or Virtual Machine with a matching MAC address:

- **Found:** the IP Address is assigned to that interface in NetBox.
- **Not found and `auto_create_devices = true`:** a new Device is created with a configurable role, device type, site, and tenant. One interface (`eth0` or derived from DHCP option 61/client-id) is created with the MAC address and the leased IP assigned to it.

Auto-creation is gated by:
- A per-hour cap (`auto_create_max_per_hour`, default 10) to prevent runaway creation from DHCP floods.
- A dry-run mode (`auto_create_dry_run = true`) that logs what would be created without writing to NetBox.
- An explicit allowlist (`auto_create_mac_prefix`) to restrict creation to specific OUI prefixes.

### 17.5 Configuration

`dnsdave-netbox` is configured via `netbox.toml` (path set by `DNSDAVE_NETBOX_CONFIG`) or environment variables with the prefix `DNSDAVE_NETBOX_`.

```toml
# netbox.toml — full example

[connection]
mode = "diode"          # "diode" | "api"
nats_url = "nats://nats:4222"
nats_creds = "/run/secrets/netbox.creds"

# ── Mode A: Diode ────────────────────────────────────────────────
[diode]
target  = "grpc://diode-server:8080"   # Diode server gRPC endpoint
api_key = "${DIODE_API_KEY}"
tls     = true
ingester = "dnsdave"                   # Diode ingester label

# ── Mode B: NetBox REST API ──────────────────────────────────────
[api]
url     = "https://netbox.example.com"
token   = "${NETBOX_TOKEN}"
tls_verify = true
timeout_s  = 10
rate_limit_per_s = 20       # stay well below NetBox's 1,000 req/min default

# ── What to push ─────────────────────────────────────────────────
[sync]
prefixes     = true          # DHCP scopes → NetBox Prefixes
ip_ranges    = true          # DHCP pool bounds → NetBox IP Ranges
ip_addresses = true          # Leases + DNS records → NetBox IP Addresses
devices      = false         # Device / interface assignment (see [devices])

vrf          = ""            # VRF slug to assign all objects to ("" = global table)
site         = ""            # NetBox site slug (optional; blank = no site)
tenant       = ""            # NetBox tenant slug (optional)
tags         = ["dnsdave"]   # Tags applied to every object written by DNSDave

# Custom field names in NetBox (must exist before starting dnsdave-netbox)
[sync.custom_fields]
scope_id   = "dnsdave_scope_id"
mac        = "dnsdave_mac"
expires_at = "dnsdave_expires_at"
reserved   = "dnsdave_reserved"
gateway    = "dnsdave_gateway"

# ── On lease expiry / release ────────────────────────────────────
[sync.lifecycle]
on_lease_expiry   = "free"        # "free" (status→available) | "remove" | "keep"
on_lease_release  = "free"
on_scope_delete   = "keep"        # "keep" (Prefix status→container) | "remove"

# ── Conflict strategy ────────────────────────────────────────────
[sync.conflict]
strategy = "overwrite"    # "overwrite" | "skip" | "tag_conflict" | "merge"
# overwrite:     DNSDave data always wins
# skip:          if the object exists with a different value, leave it unchanged
# tag_conflict:  add a "dnsdave-conflict" tag and log; do not overwrite
# merge:         update only fields that NetBox has empty/null

# ── Device auto-creation (requires sync.devices = true) ──────────
[devices]
auto_create           = false       # create new Device records for unknown MACs
auto_create_dry_run   = true        # log what would be created; do not write
auto_create_max_per_hour = 10       # safety cap
auto_create_mac_prefix   = []       # restrict to OUI prefixes e.g. ["aa:bb:cc"]
default_role          = "server"    # NetBox device role slug
default_device_type   = "generic"   # NetBox device type slug
default_site          = ""
default_tenant        = ""
interface_name        = "eth0"      # interface name used for IP assignment
```

### 17.6 NetBox Custom Fields

Before starting `dnsdave-netbox`, create the following custom fields in NetBox (Admin → Customisation → Custom Fields):

| Field name | Type | Object types | Required | Purpose |
|---|---|---|---|---|
| `dnsdave_scope_id` | Text | IP Address, Prefix, IP Range | No | DNSDave scope identifier |
| `dnsdave_mac` | Text | IP Address | No | MAC address of the leased device |
| `dnsdave_expires_at` | Date/Time | IP Address | No | Lease expiry timestamp |
| `dnsdave_reserved` | Boolean | IP Address | No | True if this is a static reservation |
| `dnsdave_gateway` | Text | Prefix | No | DHCP gateway IP |

A Bootstrap script (`deploy/netbox/bootstrap_custom_fields.py`) is provided that creates these fields automatically using the NetBox API.

### 17.7 Docker Compose Integration

#### With Diode

```yaml
# docker-compose.netbox.yml (include alongside docker-compose.yml)
services:
  diode-server:
    image: netboxlabs/diode:latest
    environment:
      DIODE_NETBOX_URL:         "http://netbox:8080"
      DIODE_NETBOX_API_TOKEN:   "${NETBOX_TOKEN}"
      DIODE_SECRET_KEY:         "${DIODE_SECRET_KEY}"
    ports: ["8080:8080"]          # gRPC
    depends_on: [netbox]

  dnsdave-netbox:
    image: ghcr.io/dnsdave/dnsdave-netbox:latest
    environment:
      DNSDAVE_NATS_URL:          "nats://nats:4222"
      DNSDAVE_NETBOX_CONFIG:     "/config/netbox.toml"
    volumes:
      - ./deploy/netbox/diode.toml:/config/netbox.toml:ro
      - ./secrets/netbox.creds:/run/secrets/netbox.creds:ro
    depends_on: [nats, diode-server]
```

#### With REST API Only (no Diode)

```yaml
  dnsdave-netbox:
    image: ghcr.io/dnsdave/dnsdave-netbox:latest
    environment:
      DNSDAVE_NATS_URL:          "nats://nats:4222"
      DNSDAVE_NETBOX_CONFIG:     "/config/netbox.toml"
      DNSDAVE_NETBOX_API_TOKEN:  "${NETBOX_TOKEN}"   # override toml token
    volumes:
      - ./deploy/netbox/api.toml:/config/netbox.toml:ro
    depends_on: [nats]
```

### 17.8 Kubernetes Deployment

In Kubernetes, `dnsdave-netbox` deploys as a single-replica `Deployment` (not a DaemonSet – it is a control-plane component, not a data-plane one):

```yaml
# templates/deployment-netbox.yaml (Helm)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: dnsdave-netbox
spec:
  replicas: 1
  template:
    spec:
      containers:
        - name: dnsdave-netbox
          image: ghcr.io/dnsdave/dnsdave-netbox:{{ .Values.image.tag }}
          envFrom:
            - secretRef:
                name: dnsdave-netbox-secrets   # NETBOX_TOKEN, DIODE_API_KEY
          volumeMounts:
            - name: config
              mountPath: /config
      volumes:
        - name: config
          configMap:
            name: dnsdave-netbox-config
```

Leader election is not required – `dnsdave-netbox` is idempotent (all pushes are create-or-update) so running a second replica during a rolling update causes harmless duplicate pushes that Diode or the REST API deduplicates.

### 17.9 Health and Observability

`dnsdave-netbox` exposes a health endpoint on `:9095/health` and Prometheus metrics on `:9095/metrics`:

| Metric | Type | Description |
|---|---|---|
| `dnsdave_netbox_pushes_total` | Counter | `mode`, `object_type`, `status` (success/conflict/error) |
| `dnsdave_netbox_push_duration_ms` | Histogram | Round-trip latency to Diode / NetBox API |
| `dnsdave_netbox_nats_lag` | Gauge | Unacked NATS messages (back-pressure indicator) |
| `dnsdave_netbox_devices_created_total` | Counter | Auto-created device records |
| `dnsdave_netbox_conflicts_total` | Counter | `strategy`, `object_type` |

The health endpoint returns `503` if the NATS lag exceeds the configured `max_lag_seconds` threshold, enabling Kubernetes liveness probes to surface NetBox outages as a container health issue.

---

### 17.10 Pull – NetBox → DNSDave

`dnsdave-netbox` is fully bidirectional. In addition to pushing DNSDave state into NetBox, it can pull authoritative data from NetBox to create or update DNS records, DHCP scopes, DHCP reservations, and other objects in DNSDave. This makes NetBox the source of truth for IP addressing whilst DNSDave remains the runtime enforcement layer.

#### 17.10.1 What Can Be Pulled

| NetBox object | Pull condition | DNSDave object created / updated |
|---|---|---|
| **Prefix** (status=active) | Tagged, or matching site/VRF filter | DHCP scope (network, prefix length, description) |
| **IP Range** (status=dhcp) | Within a managed/pulled Prefix | DHCP scope pool bounds (start–end) |
| **IP Address** (status=active, dns_name set) | Tagged or in managed Prefix | DNS A or AAAA record (zone inferred from dns_name or default_zone) |
| **IP Address** (status=dhcp, `dnsdave_mac` custom field set) | Tagged or in managed Prefix | DHCP static reservation (MAC → IP) |
| **IP Address** (status=reserved) | Tagged or in managed Prefix | DNS record + DHCP reservation |
| **Device** (primary_ip4 or primary_ip6 set) | Tagged, or matching site filter | DNS A + PTR record (name from device.name, zone from default_zone or mapping) |
| **Virtual Machine** (primary_ip4 or primary_ip6 set) | Tagged or matching site filter | DNS A + PTR record |
| **VRF** | Always (for mapping table) | Registered in mapping table; used to route objects to correct DHCP scope |

#### 17.10.2 Pull Trigger Modes

Three modes are available and can be combined:

**Mode 1 – Webhook (event-driven, recommended)**

NetBox 4.x supports event rules that fire HTTP webhooks on object changes. Configure NetBox to POST to:

```
POST https://dnsdave.example.com/api/v1/integrations/netbox/webhook
X-NetBox-Signature: sha256=<hmac>
Content-Type: application/json
```

`dnsdave-api` validates the HMAC-SHA256 signature against the configured `webhook_secret`, parses the NetBox event payload, determines the affected DNSDave object(s), and applies the change immediately (subject to conflict strategy).

NetBox event rules to create (one per object type):

| Event Rule | Object Types | Actions | Destination |
|---|---|---|---|
| `dnsdave-prefix-sync` | IPAM → Prefix | Created, Updated, Deleted | `POST /api/v1/integrations/netbox/webhook` |
| `dnsdave-iprange-sync` | IPAM → IP Range | Created, Updated, Deleted | — |
| `dnsdave-ipaddress-sync` | IPAM → IP Address | Created, Updated, Deleted | — |
| `dnsdave-device-sync` | DCIM → Device | Created, Updated, Deleted | — |
| `dnsdave-vm-sync` | Virtualization → VM | Created, Updated, Deleted | — |

A setup wizard in the DNSDave UI (`/integrations/netbox` → "Configure NetBox Webhooks") generates the event rule configuration as a NetBox API script, optionally applying it automatically if the supplied token has permissions.

**Mode 2 – Scheduled Poll**

`dnsdave-netbox` polls the NetBox REST API on a configurable schedule (default: every 15 minutes) and performs a full reconciliation of all managed objects. This is less reactive than webhooks but requires no NetBox configuration beyond the API token.

Poll behaviour:
- Fetches all Prefixes, IP Ranges, IP Addresses, Devices, and VMs matching the active filters.
- Computes a diff against the `netbox_sync_objects` tracking table.
- Applies creates, updates, and deletes according to the configured conflict strategy.
- Records results in `netbox_sync_log`.

The poll interval is configurable per object type to allow fine-grained control (e.g., Prefixes polled hourly, IP Addresses polled every 5 minutes).

**Mode 3 – Manual**

Triggered from the DNSDave UI or via `POST /api/v1/integrations/netbox/sync/pull`. Runs a full poll immediately and streams progress to the UI via SSE.

#### 17.10.3 Pull Data Mapping

**Prefix → DHCP Scope**

```
NetBox Prefix {                     DNSDave DHCP Scope {
  prefix:  "192.168.10.0/24"          id:          "nb-prefix-42"
  status:  "active"          →        network:     "192.168.10.0"
  description: "Office LAN"           prefix_len:  24
  tags:    ["dnsdave"]                name:        "Office LAN"
  site:    { slug: "london" }         enabled:     true
  vrf:     { slug: "corporate" }      source:      "netbox"
  custom_fields: {                    netbox_id:   42
    gateway: "192.168.10.1"         }
  }
}
```

If the Prefix has no pool bounds, DNSDave creates the scope but does not enable DHCP serving (status=`planned`). Pool bounds are set when the associated IP Range is pulled.

**IP Range → DHCP Pool**

```
NetBox IP Range {                   DNSDave Scope pool bounds {
  start_address: "192.168.10.100"     pool_start: "192.168.10.100"
  end_address:   "192.168.10.200"  →  pool_end:   "192.168.10.200"
  status:        "dhcp"               parent_scope: "nb-prefix-42"
}                                   }
```

**IP Address (active, with dns_name) → DNS Record**

```
NetBox IP Address {                 DNSDave DNS Record {
  address:  "192.168.10.5/24"         name:   "fileserver.office.example"
  status:   "active"          →       type:   "A"
  dns_name: "fileserver.office.example"  value: "192.168.10.5"
  tags:     ["dnsdave"]               zone:   "office.example"
}                                     source: "netbox"
                                      netbox_id: 317
                                    }
```

DNSDave infers the DNS zone from `dns_name`: it strips the record label and checks whether a matching zone exists. If not, and `auto_create_zones = true`, it creates the zone.

**Device → DNS Records**

```
NetBox Device {                     DNSDave DNS Records {
  name:        "router-01"            { name: "router-01.london.corp",
  primary_ip4: { address: "10.0.0.1" }  type: "A", value: "10.0.0.1" }
  site:        { slug: "london" }  →  { name: "1.0.0.10.in-addr.arpa",
  tags:        ["dnsdave"]              type: "PTR",
  device_role: { slug: "router" }       value: "router-01.london.corp." }
}                                   }
```

The FQDN is constructed from `{device.name}.{site.slug}.{default_domain}` (configurable template). PTR records are created in the corresponding reverse zone if it exists in DNSDave.

**IP Address (DHCP + MAC) → Static Reservation**

```
NetBox IP Address {                 DNSDave DHCP Reservation {
  address: "192.168.10.50/24"         scope_id: "nb-prefix-42"
  status:  "dhcp"                →    ip:       "192.168.10.50"
  custom_fields: {                    mac:      "aa:bb:cc:dd:ee:ff"
    dnsdave_mac: "aa:bb:cc:dd:ee:ff"  name:     "printer-01"
  }                                   source:   "netbox"
  dns_name: "printer-01.office.example"  }
}
```

#### 17.10.4 Conflict Strategy on Pull

| Strategy | Behaviour |
|---|---|
| `skip` (default) | If a matching object exists in DNSDave, leave it unchanged |
| `overwrite` | NetBox data always wins; DNSDave object is updated to match |
| `merge` | Update only fields that DNSDave has null/empty; leave populated fields alone |
| `tag_conflict` | Mark both the DNSDave object and the NetBox object with a conflict tag; alert in the UI; do not write |
| `ask` | Queue the conflict for manual resolution in the UI conflict review panel |

Conflict strategy can be set globally or overridden per object type.

---

### 17.11 Bidirectional Sync State Machine

Every managed object has a sync state tracked in `netbox_sync_objects`:

```mermaid
stateDiagram-v2
    [*]         --> pending     : event received (push or pull)
    pending     --> syncing     : worker picks up task
    syncing     --> synced      : NetBox / DNSDave accepted change
    syncing     --> conflict    : object exists with different value
    syncing     --> error       : network / auth failure
    conflict    --> synced      : operator resolves in UI
    conflict    --> synced      : auto-resolved by strategy
    error       --> pending     : retry after back-off (max 5 attempts)
    synced      --> pending     : source object changed again
    synced      --> [*]         : object deleted in source (removed from tracking)
```

**Retry logic:**
- Errors back off exponentially: 30s, 2m, 8m, 32m, 2h.
- After 5 failures the object is marked `error` and requires manual intervention.
- A failed push does not block subsequent events for other objects.

---

### 17.12 Object Mapping Configuration

Mappings control how NetBox organisational objects (VRF, site, tenant, tag) are associated with DNSDave objects (DHCP scope, DNS zone, client group).

| NetBox concept | Maps to | Effect |
|---|---|---|
| VRF | DHCP scope | Pulled Prefixes/IP Ranges in this VRF are associated with the mapped scope |
| Site | DNS zone suffix | Devices in this site get DNS names under the mapped zone (e.g., `.london.corp.example`) |
| Tenant | Client group | Devices/VMs belonging to this tenant are placed in the corresponding client group |
| Tag | Sync filter | Only objects with this tag are included in push/pull operations |

---

### 17.13 REST API for the Integration

These endpoints are served by `dnsdave-api` and are the primary interface for the UI page and automation.

```
/api/v1/integrations/
└── netbox/
    ├── GET    /config              # full config (token masked to last 4 chars)
    ├── PUT    /config              # update config (token, URL, mode, push/pull toggles)
    ├── POST   /test                # test connection → { version, latency_ms, status }
    ├── GET    /status              # last push/pull timestamps, object counts, error counts
    ├── GET    /status/stream       # SSE stream of sync activity (live progress)
    │
    ├── sync/
    │   ├── POST   /push            # trigger manual push (all objects or filtered)
    │   ├── POST   /pull            # trigger manual pull (all objects or filtered)
    │   └── GET    /preview         # dry-run: returns what a pull would create/update/delete
    │
    ├── webhook/
    │   └── POST   /               # receive NetBox event webhooks (HMAC-SHA256 validated)
    │
    ├── objects/
    │   ├── GET    /               # paginated list of all tracked objects + sync status
    │   ├── GET    /:id            # individual object detail (source data + DNSDave data + conflicts)
    │   ├── POST   /:id/resolve    # manually resolve a conflict
    │   └── DELETE /:id           # remove from tracking (does not delete the underlying record)
    │
    ├── mappings/
    │   ├── GET    /               # list all VRF/site/tenant/tag mappings
    │   ├── POST   /               # create mapping
    │   ├── PUT    /:id            # update mapping
    │   └── DELETE /:id           # remove mapping
    │
    └── log/
        ├── GET    /               # paginated sync activity log
        └── GET    /stream         # SSE live sync activity stream
```

**Key request/response examples:**

`PUT /api/v1/integrations/netbox/config`:
```json
{
  "enabled": true,
  "netbox_url": "https://netbox.example.com",
  "api_token": "abc123def456...",
  "mode": "api",
  "diode_url": null,
  "diode_api_key": null,
  "push": {
    "prefixes": true,
    "ip_ranges": true,
    "ip_addresses": true,
    "dns_records": true,
    "devices": false,
    "conflict": "overwrite",
    "on_lease_expiry": "free",
    "on_scope_delete": "keep"
  },
  "pull": {
    "enabled": true,
    "mode": "webhook",
    "poll_interval_minutes": 15,
    "prefixes": true,
    "ip_ranges": true,
    "ip_addresses": true,
    "devices": true,
    "virtual_machines": true,
    "reservations": true,
    "filter_tag": "dnsdave",
    "filter_site": "",
    "filter_vrf": "",
    "filter_tenant": "",
    "conflict": "skip",
    "auto_create_zones": false
  },
  "context": {
    "default_vrf": "",
    "default_site": "",
    "default_tenant": "",
    "default_zone": "home.arpa",
    "device_fqdn_template": "{name}.{site}.{default_zone}",
    "tags": ["dnsdave"]
  }
}
```

`GET /api/v1/integrations/netbox/sync/preview`:
```json
{
  "would_create": [
    { "type": "dns_record", "name": "fileserver.office.example", "value": "192.168.10.5", "source_nb_id": 317 }
  ],
  "would_update": [
    { "type": "dhcp_scope", "id": "nb-prefix-42", "field": "name", "from": "Office", "to": "Office LAN" }
  ],
  "would_delete": [],
  "conflicts": [
    { "type": "dns_record", "name": "printer.home.arpa", "nb_value": "192.168.10.50", "dd_value": "192.168.1.50", "strategy": "skip" }
  ],
  "total_nb_objects": 247,
  "scanned_in_ms": 1842
}
```

`POST /api/v1/integrations/netbox/webhook` (received from NetBox):
```json
{
  "event": "updated",
  "timestamp": "2026-04-05T10:23:01Z",
  "model": "ipaddress",
  "username": "admin",
  "data": {
    "id": 317,
    "url": "https://netbox.example.com/api/ipam/ip-addresses/317/",
    "address": "192.168.10.5/24",
    "status": { "value": "active" },
    "dns_name": "fileserver.office.example",
    "tags": [{ "slug": "dnsdave" }]
  }
}
```

---

## 19. Multi-Homed Devices and Networks

Many real networks contain devices with more than one network interface – servers with a management NIC and a data NIC, routers with LAN/WAN ports, hypervisors with bonded uplinks, access points bridging wired and wireless. DNSDave treats this as a first-class topology and provides:

1. **A device model** – devices and their interfaces are first-class objects that can own DHCP reservations and DNS records across multiple scopes and zones.
2. **Network-affinity DNS resolution** – a query for a multi-homed device's canonical name returns the IP that is topologically closest to the querying client, rather than a random round-robin answer.
3. **Per-interface naming** – each interface may carry its own DNS alias in addition to the shared canonical name.
4. **Unified DHCP reservation management** – all reservations for a device's interfaces are linked, so a single device record gives a complete inventory of all its network identities.
5. **Multi-segment network awareness** – DHCP relay (RFC 3046), VLAN-tagged scopes, and VRF-aware scope assignment allow a single DNSDave instance to serve dozens of isolated L2 segments.

---

### 19.1 Device Model

A **device** is any logical endpoint that may appear on more than one network segment. Each device has one or more **interfaces**, and each interface may hold one or more **IP addresses** (primary, secondary, VIP, anycast). Addresses are linked back to DHCP scopes (for dynamic addresses) or DNS records (for static addresses).

```
Device "fileserver-01"
├── Interface "eth0"   MAC aa:bb:cc:dd:ee:ff   VLAN 10   primary
│   ├── 192.168.10.5/24     scope: LAN     reservation: res-abc
│   │   DNS: fileserver-01.home.arpa A 192.168.10.5
│   │        fileserver-01-lan.home.arpa A 192.168.10.5    (interface alias)
│   └── (DHCPv6 FD00::5/64  scope: LAN-v6  reservation: res-xyz)
│
└── Interface "eth1"   MAC 11:22:33:44:55:66   VLAN 20
    ├── 10.20.0.5/24        scope: SERVERS  reservation: res-def
    │   DNS: fileserver-01.home.arpa A 10.20.0.5          (second A record)
    │        fileserver-01-srv.home.arpa A 10.20.0.5      (interface alias)
    └── 10.20.0.6/24        secondary (VIP)
        DNS: fileserver-01-vip.home.arpa A 10.20.0.6
```

#### Device types

| Type | Examples |
|---|---|
| `host` | Workstations, laptops, desktop servers |
| `server` | Rack servers, hypervisors, NAS |
| `router` | Hardware routers, firewalls (pfSense, VyOS) |
| `switch` | Managed L2/L3 switches |
| `ap` | Wireless access points |
| `printer` | Network printers, MFDs |
| `vm` | Virtual machines |
| `container` | Docker / Kubernetes pods with fixed IPs |
| `iot` | Smart home, industrial sensors |
| `other` | Anything not categorised above |

#### Interface types

| Type | Notes |
|---|---|
| `ethernet` | Standard wired NIC |
| `wifi` | 802.11 wireless |
| `bond` | Linux bonding (active-backup, LACP) |
| `vlan` | 802.1Q tagged sub-interface |
| `bridge` | Linux/BSD bridge |
| `virtual` | VMware VMXNET, VirtIO, loopback-like |
| `loopback` | Lo0 / lo – used for stable anycast VIPs on routers |
| `tunnel` | WireGuard, OpenVPN, GRE, VXLAN |
| `infiniband` | HPC / storage fabrics |

---

### 19.2 DNS for Multi-Homed Devices

#### 19.2.1 Naming strategy (per device, configurable)

| Strategy | Behaviour |
|---|---|
| `round_robin` | All interface IPs returned under the canonical name; answer order randomised per query |
| `affinity` | Only the IP on the same subnet as the querying client is returned; round-robin fallback if no affinity match (default) |
| `primary_only` | Only the primary interface IP is returned; secondary IPs are reachable only via interface-specific alias names |
| `per_interface` | No shared canonical name; only interface-specific aliases are created |
| `explicit` | Operator manually assigns which IPs appear in which DNS records |

#### 19.2.2 Network-affinity resolution

This is the most powerful strategy and the default for devices with more than one interface. When `dnsdave-dns` receives a query for a multi-homed device's canonical name:

```
Client 192.168.10.42 queries A fileserver-01.home.arpa

Hot path:
  1. Look up "fileserver-01.home.arpa" in local records map.
  2. Fetch all A records (affinity-tagged): [192.168.10.5, 10.20.0.5]
  3. Find the record whose subnet contains the querying client IP:
       192.168.10.5/24 ⊇ 192.168.10.42  ✓  → select this record
  4. Return: A 192.168.10.5  (TTL = record TTL)
  5. If no match: return all records in round-robin order
```

The affinity lookup is a prefix-match against the in-memory CIDR table (`ArcSwap<Vec<(IpNet, Vec<RData>)>>`), added on the zero-allocation hot path after the main record lookup. It adds approximately 50–200 ns per query on a 5M-record DNS instance.

Affinity data is encoded in the record metadata stored in Postgres and propagated to the DNS data plane via NATS on every device address change. No Postgres reads occur in the hot path.

#### 19.2.3 Per-interface alias names

Each interface may be given a **DNS alias** in addition to (or instead of) the device canonical name. Aliases are generated automatically or set manually:

| Pattern | Example | Notes |
|---|---|---|
| Automatic (interface label) | `{device}.{iface}.{zone}` → `fileserver-01-eth0.home.arpa` | Sanitised: slashes, dots, underscores stripped |
| Manual | `mgmt.fileserver-01.home.arpa` | Operator-defined |
| VIP | `{device}-vip.{zone}` | For secondary/VIP addresses |

#### 19.2.4 PTR records for multi-homed

Each interface IP that has a DNS name generates a PTR record in the appropriate reverse zone:

```
5.10.168.192.in-addr.arpa  PTR  fileserver-01.home.arpa.
5.0.20.10.in-addr.arpa     PTR  fileserver-01.home.arpa.
6.0.20.10.in-addr.arpa     PTR  fileserver-01-vip.home.arpa.
```

PTR records are published via NATS `dnsdave.zone.record.*` events when device interface addresses change.

#### 19.2.5 Dynamic DNS updates from multi-homed clients

When a multi-homed client sends RFC 2136 DNS UPDATE packets:

- The update is matched against the **device record** by MAC address (looked up from the DHCP lease).
- Only the record corresponding to the updating interface is modified.
- Other interfaces' records are left unchanged.
- A NATS `dnsdave.zone.record.updated` event is published for the specific record only.

---

### 19.3 DHCP for Multi-Homed Devices

#### 19.3.1 Multi-reservation device linking

A single device entity can hold reservations in multiple scopes (one per interface per scope). This replaces the previous pattern of managing N unrelated reservations:

```jsonc
// Device with two interfaces, each reserved in a different scope
{
  "id": "dev-01",
  "name": "fileserver-01",
  "type": "server",
  "interfaces": [
    {
      "name": "eth0",
      "mac": "aa:bb:cc:dd:ee:ff",
      "scope_id": "scope-lan",
      "reserved_ip": "192.168.10.5",
      "dns_alias": "fileserver-01-lan",
      "is_primary": true
    },
    {
      "name": "eth1",
      "mac": "11:22:33:44:55:66",
      "scope_id": "scope-servers",
      "reserved_ip": "10.20.0.5",
      "dns_alias": "fileserver-01-srv",
      "is_primary": false
    }
  ],
  "canonical_name": "fileserver-01",
  "dns_zone": "home.arpa",
  "naming_strategy": "affinity"
}
```

When this device record is saved, `dnsdave-api` creates or updates:
- Two DHCP reservations (one per interface × scope).
- DNS A records for each interface IP under both the canonical name and any aliases.
- PTR records in the relevant reverse zones.
- All linked via foreign keys through `device_interface_addresses`.

#### 19.3.2 Lease correlation across interfaces

`dnsdave-dhcp` correlates leases from multiple MACs back to a single device:

- When `aa:bb:cc:dd:ee:ff` acquires a lease, the DISCOVER is matched against `device_interfaces` by MAC. If the device has other interfaces currently active, the lease event payload includes the full device ID and sibling interface states.
- NATS event `dnsdave.dhcp.lease.acquired` carries `device_id` and `interface_id` in addition to the standard MAC/IP fields.
- The UI can display device online/offline status based on whether **any** interface has an active lease.

#### 19.3.3 Option 82 (Relay Agent Information) for multi-segment DHCP

When `dnsdave-dhcp` receives a DISCOVER via a relay agent, it reads:
- **giaddr** (RFC 3046) to select the correct scope.
- **Option 82 sub-option 1** (Agent Circuit ID) to identify the physical port/VLAN the client arrived on.
- **Option 82 sub-option 2** (Agent Remote ID) to identify the relay device.

Circuit ID and Remote ID are logged with each lease and can be used as scope-selection hints when multiple scopes share the same `giaddr` subnet (e.g., VLAN-separated networks on a shared relay subnet).

#### 19.3.4 DHCP failover for multi-homed devices

In HA deployments, multi-homed device reservations are replicated to all `dnsdave-dhcp` instances via NATS JetStream. If one interface's lease is renewed on a secondary node due to primary failure, the lease state is reconciled back to Postgres and the device record is kept consistent.

---

### 19.4 Multi-Segment Network Topologies

#### 19.4.1 VLAN-tagged scopes

Each scope can be annotated with a VLAN ID for documentation, NetBox sync, and Option 82 matching:

```toml
[scopes.lan]
name        = "Office LAN"
subnet      = "192.168.10.0/24"
vlan_id     = 10
vrf         = "corporate"

[scopes.servers]
name        = "Server VLAN"
subnet      = "10.20.0.0/24"
vlan_id     = 20
vrf         = "corporate"
```

VLAN IDs are used:
- To disambiguate DHCP DISCOVER packets arriving via an Option 82-capable relay.
- For NetBox Prefix/VLAN synchronisation (pushed as VLAN association to the NetBox Prefix).
- As a visual tag in the UI scope list.

#### 19.4.2 VRF-aware scope isolation

When multiple scopes share overlapping RFC 1918 address space (e.g., two separate `192.168.1.0/24` networks in different VRFs), DNSDave uses the `vrf` field on the scope to ensure:
- DHCP leases and reservations are never cross-contaminated between VRFs.
- DNS A records in one VRF do not resolve for clients in another VRF (enforced via split-horizon views keyed on VRF name, using client group CIDR membership).
- NetBox sync pushes each prefix into the correct VRF.

#### 19.4.3 DNSDave host multi-homing (listener binding)

When the host running `dnsdave-dns` and `dnsdave-dhcp` is itself multi-homed, listener binding can be configured per-interface:

```toml
[dns]
listen = [
  "192.168.10.53:53",     # LAN interface
  "10.20.0.53:53",        # Server VLAN interface
  "127.0.0.1:53",         # Loopback (internal)
]

[dhcp]
listen = [
  "192.168.10.1:67",      # Serve LAN directly
  "0.0.0.0:67",           # Or listen on all interfaces for relay traffic
]
```

`SO_REUSEPORT` is used to allow multiple worker threads to share each listener socket. UDP DHCP relay traffic from `0.0.0.0:67` is disambiguated by `giaddr`.

#### 19.4.4 Active-active DNS across multi-homed nodes

In Kubernetes, `dnsdave-dns` pods are deployed as a DaemonSet – one pod per node. Each pod binds to the node's local IP for the relevant network. Clients on each network query their local pod. Consistent answers are guaranteed because all pods load the same `ArcSwap`-backed in-memory state from NATS JetStream.

For multi-homed nodes where a single pod must serve multiple network segments, the pod's `listen` list is populated from the node's interface addresses via the Kubernetes Downward API.

---

### 19.5 Configuring Multi-Homed Devices via the API

#### Create a multi-homed device with two interfaces

```http
POST /api/v1/devices
Content-Type: application/json

{
  "name":            "fileserver-01",
  "type":            "server",
  "description":     "Primary file server",
  "tags":            ["production"],
  "dns_zone":        "home.arpa",
  "naming_strategy": "affinity",
  "interfaces": [
    {
      "name":          "eth0",
      "mac":           "aa:bb:cc:dd:ee:ff",
      "interface_type": "ethernet",
      "vlan_id":       10,
      "is_primary":    true,
      "dns_alias":     "fileserver-01-lan",
      "addresses": [
        {
          "address":      "192.168.10.5/24",
          "scope_id":     "scope-lan",
          "address_type": "primary"
        }
      ]
    },
    {
      "name":          "eth1",
      "mac":           "11:22:33:44:55:66",
      "interface_type": "ethernet",
      "vlan_id":       20,
      "is_primary":    false,
      "dns_alias":     "fileserver-01-srv",
      "addresses": [
        {
          "address":      "10.20.0.5/24",
          "scope_id":     "scope-servers",
          "address_type": "primary"
        },
        {
          "address":      "10.20.0.6/24",
          "scope_id":     "scope-servers",
          "address_type": "vip",
          "dns_alias":    "fileserver-01-vip"
        }
      ]
    }
  ]
}
```

Response includes the automatically-created reservation IDs and DNS record IDs for all addresses.

#### Promote a lease to a device record

```http
POST /api/v1/devices/from-lease
Content-Type: application/json

{ "lease_id": "lease-abc", "device_name": "fileserver-01" }
```

Creates a device with one interface populated from the lease (MAC, IP, scope). The existing lease is converted to a static reservation. Additional interfaces can be added via `POST /api/v1/devices/:id/interfaces`.

---

### 19.6 NATS Events for Device Changes

| Subject | Published by | Description |
|---|---|---|
| `dnsdave.device.created.{id}` | `dnsdave-api` | New device record saved; all reservations and records created |
| `dnsdave.device.updated.{id}` | `dnsdave-api` | Device name/strategy/tags changed |
| `dnsdave.device.interface.added.{dev_id}.{iface_id}` | `dnsdave-api` | New interface added to device; reservation + DNS records created |
| `dnsdave.device.interface.removed.{dev_id}.{iface_id}` | `dnsdave-api` | Interface removed; associated reservation + DNS records deleted |
| `dnsdave.device.address.changed.{dev_id}.{iface_id}` | `dnsdave-api` or `dnsdave-dhcp` | Address on an interface changed (lease renewal to different IP) |
| `dnsdave.device.deleted.{id}` | `dnsdave-api` | Device deleted; all reservations and DNS records removed |

`dnsdave-dns` subscribes to these subjects to update its in-memory affinity CIDR table without restarting. `dnsdave-netbox` subscribes to push changes to NetBox device/interface/IP objects.

---

### 19.7 NetBox Integration for Multi-Homed Devices

Multi-homed devices map cleanly to NetBox's `dcim.Device` + `dcim.Interface` + `ipam.IPAddress` hierarchy:

| DNSDave object | NetBox object |
|---|---|
| `Device` | `dcim.Device` |
| `DeviceInterface` | `dcim.Interface` (with type, MAC, VLAN) |
| `DeviceInterfaceAddress` (primary) | `ipam.IPAddress` assigned to interface; `primary_ip4`/`primary_ip6` on the device |
| `DeviceInterfaceAddress` (secondary/VIP) | `ipam.IPAddress` assigned to interface; `is_primary=False` |
| `DeviceInterfaceAddress` (DHCP) | `ipam.IPAddress` with status=dhcp |

On pull from NetBox (§17.10), devices with multiple interfaces are imported and create the corresponding multi-interface device record in DNSDave automatically, applying the default or configured naming strategy.

## 18. Identity, Access Management and RBAC

DNSDave implements a first-class, fine-grained Role-Based Access Control (RBAC) system. Every UI action, every API call, and every automated integration is subject to permission evaluation. The system supports local users, groups, custom roles, and per-resource/per-action permission grants, with a complete audit trail.

---

### 18.1 Identity Model

```
User  ──────────────────────────────────────────┐
  │  belongs to (0..n)                           │
  ▼                                              │
Group ──── assigned (0..n) ────► Role            │
                                  │              │
                                  │ consists of  │
                                  ▼              │
                             Permission          │
                      { resource × action × scope } │
                                                 │
User ──── also holds direct (0..n) ────► Permission
```

- A **User** may belong to multiple **Groups**.
- A **Group** may hold multiple **Roles**.
- A **Role** is a named, reusable bundle of **Permissions**.
- A **User** may also hold **Permissions** directly (grants or explicit denies), which take precedence over group-inherited permissions.
- **Effective permissions** = union of all role permissions from all groups + direct grants, minus any direct denies.

---

### 18.2 Permission Model

Each permission is a triple of **resource**, **action**, and **scope**.

#### Resources

| Resource key | Description |
|---|---|
| `dns_records` | Individual DNS resource records |
| `dns_zones` | DNS zone management (SOA, NS, AXFR) |
| `dns_upstreams` | Upstream resolver configuration |
| `dns_forwardzones` | Conditional forward zones |
| `dns_dnssec` | DNSSEC keys and signing |
| `blocklists` | Blocklist sources and allow/block rules |
| `dhcp_scopes` | DHCP scopes and options |
| `dhcp_leases` | DHCP lease table (view only; leases are protocol-generated) |
| `dhcp_reservations` | Static DHCP reservations |
| `dhcp_options` | DHCP option sets |
| `clients` | Client identity records |
| `client_groups` | Client group membership and policies |
| `query_log` | Query log read and export |
| `analytics` | Statistics and dashboards |
| `certificates` | TLS certificate management |
| `cluster` | Cluster node management and NATS monitoring |
| `settings` | Global system settings |
| `integrations_netbox` | NetBox integration configuration and sync control |
| `integrations_export` | Observability export configuration |
| `iam_users` | User account management |
| `iam_groups` | Group management |
| `iam_roles` | Role definition and permission assignment |
| `iam_audit` | Audit log read access |
| `api_keys` | API key lifecycle for own or all accounts |

#### Actions

| Action | Description |
|---|---|
| `view` | Read any object in the resource |
| `create` | Create new objects |
| `edit` | Modify existing objects |
| `delete` | Remove objects |
| `bulk` | Bulk import / export operations |
| `sync` | Trigger a sync or refresh (blocklists, NetBox, etc.) |
| `flush` | Flush caches or force-rebuild indexes |
| `export` | Download data (CSV, JSON, zone file, etc.) |
| `rotate` | Rotate credentials (API keys, DNSSEC keys, TLS certs) |
| `manage_members` | Add/remove members in a group |
| `impersonate` | Act as another user (super-admin only) |

#### Scopes

| Scope | Description |
|---|---|
| `own` | Only objects the user created or owns |
| `group` | Objects belonging to the user's groups |
| `all` | All objects (unrestricted) |

Scope is optional and defaults to `all` when not specified.

#### Permission notation

Permissions are expressed as `resource:action[:scope]` strings, e.g.:

```
dns_records:view                  # view all DNS records
dns_records:create                # create DNS records
dns_records:delete:own            # delete only records the user created
dhcp_leases:view                  # read-only lease table
iam_users:edit                    # manage other user accounts
analytics:view                    # view dashboards
query_log:export                  # download query log
```

---

### 18.3 Built-in Roles

Five built-in roles are pre-seeded and cannot be deleted (only customised by creating new roles):

| Role | Description | Key permissions |
|---|---|---|
| `super_admin` | Unrestricted access to everything | `*:*` wildcard |
| `admin` | Full operational access; cannot manage users/roles | All except `iam_*` |
| `dns_admin` | Full DNS and blocklist management | `dns_*:*`, `blocklists:*` |
| `dhcp_admin` | Full DHCP management, view DNS | `dhcp_*:*`, `dns_records:view`, `dns_zones:view` |
| `read_only` | View-everything, change-nothing | `*:view` |
| `operator` | Read + flush + sync, no config changes | `*:view`, `*.flush`, `*.sync`, `dhcp_leases:view` |
| `netbox_sync` | Dedicated role for the NetBox service account | `integrations_netbox:*`, `dns_records:edit`, `dhcp_scopes:view` |

---

### 18.4 Authentication

#### Local Users

Local user accounts are stored in Postgres. Passwords are hashed with **Argon2id** (memory: 64 MB, time: 3, parallelism: 4). Password policy is configurable:

- Minimum length (default: 12)
- Complexity requirements (configurable per deployment)
- Maximum age (default: never; configurable)
- Lockout after N failed attempts (default: 10, 15-minute window)
- Breached-password check via k-anonymity against HIBP (optional)

#### Sessions and JWT

On login (`POST /api/v1/auth/login`), the API issues:

- A **short-lived JWT** (default: 15-minute expiry) signed with the server's Ed25519 private key, containing the user's identity and resolved effective permissions.
- A **refresh token** (opaque, 30-day expiry, stored in an `HttpOnly; Secure; SameSite=Strict` cookie).

The UI stores the JWT in memory only (never `localStorage`). On expiry, the SvelteKit SSR layer transparently exchanges the refresh cookie for a new JWT. The JWT payload:

```json
{
  "sub": "user_01j...",
  "username": "alice",
  "display_name": "Alice Smith",
  "groups": ["grp_ops", "grp_dns"],
  "roles": ["dns_admin", "operator"],
  "permissions": [
    "dns_records:view",
    "dns_records:create",
    "dns_records:edit",
    "dns_records:delete:own",
    "blocklists:view",
    "analytics:view"
  ],
  "iat": 1743840000,
  "exp": 1743840900,
  "iss": "dnsdave",
  "jti": "tok_01j..."
}
```

The `permissions` array is the **computed effective permission set** at login time. Any change to a user's roles or group membership invalidates all active sessions (via a `session_version` counter in Postgres and NATS `dnsdave.iam.session_invalidated.{user_id}` event that causes token re-validation on next request).

#### API Keys

API keys are linked to a user account and inherit a **subset** of that user's permissions (configured at key creation time). A key cannot be granted more permissions than its owner currently holds. Keys are:

- Hashed with Argon2id at rest (the raw key is shown exactly once on creation).
- Scoped to a named permission subset.
- Optionally restricted to source IP CIDR.
- Optionally time-limited (expiry date).
- Individually revocable without affecting the parent user account.

---

### 18.5 Authorisation Enforcement

Every `dnsdave-api` handler is protected by a composable middleware chain:

```
Request
  │
  ├─ 1. TLS termination
  ├─ 2. Rate limit (per IP / per key)
  ├─ 3. Authenticate (JWT or API key)
  ├─ 4. Load effective permissions from JWT claims (or cached from Postgres if API key)
  ├─ 5. Authorise: check permission(s) required by the handler
  │       If denied → 403 Forbidden with body:
  │       { "error": "forbidden", "required": "dns_records:create", "resource_id": "..." }
  ├─ 6. Scope filter: if permission has scope=own, inject `WHERE created_by = user_id`
  └─ 7. Handler logic
```

No handler bypasses the authorisation middleware. The permission check is a pure function on the JWT claims — no Postgres read per request (except for API key lookups, which are cached in a short-lived in-process LRU).

---

### 18.6 Audit Log

Every mutating API call (POST, PUT, PATCH, DELETE) is written to the `iam_audit_log` table and published to the `dnsdave.iam.audit` NATS subject. Each entry records:

- Timestamp, user ID, username, user IP
- HTTP method, path, resource type, resource ID
- Before-state and after-state (JSON diff for updates)
- Outcome: `success` / `forbidden` / `error`
- Session/key ID

The audit log is append-only (no UPDATE or DELETE on the table). Retention is configurable (default: 365 days, enforced by a scheduled partition drop).

---

### 18.7 REST API for IAM

```
/api/v1/
├── auth/
│   ├── POST   /login              # password auth → JWT + refresh cookie
│   ├── POST   /logout             # revoke refresh token
│   ├── POST   /refresh            # exchange refresh cookie → new JWT
│   ├── GET    /me                 # current user's profile + effective permissions
│   └── PATCH  /me/password        # change own password
│
└── iam/
    ├── users/
    │   ├── GET    /               # paginated user list
    │   ├── POST   /               # create user
    │   ├── GET    /:id            # user detail
    │   ├── PATCH  /:id            # update profile / status
    │   ├── DELETE /:id            # deactivate (soft delete)
    │   ├── POST   /:id/reset-password   # admin-initiated password reset
    │   ├── POST   /:id/unlock           # unlock after lockout
    │   ├── GET    /:id/sessions         # active sessions
    │   ├── DELETE /:id/sessions         # revoke all sessions
    │   ├── GET    /:id/permissions      # resolved effective permissions
    │   └── api-keys/
    │       ├── GET    /           # list user's API keys
    │       ├── POST   /           # create API key (raw key shown once)
    │       └── DELETE /:kid       # revoke API key
    │
    ├── groups/
    │   ├── GET    /               # list groups
    │   ├── POST   /               # create group
    │   ├── GET    /:id            # group detail + member list
    │   ├── PATCH  /:id            # update name / description
    │   ├── DELETE /:id            # delete group (cannot delete system groups)
    │   ├── GET    /:id/members    # list members
    │   ├── POST   /:id/members    # add member(s) { user_ids: [...] }
    │   ├── DELETE /:id/members/:uid    # remove member
    │   ├── GET    /:id/roles      # roles assigned to group
    │   ├── POST   /:id/roles      # assign role to group
    │   └── DELETE /:id/roles/:rid      # remove role from group
    │
    ├── roles/
    │   ├── GET    /               # list roles (built-in + custom)
    │   ├── POST   /               # create custom role
    │   ├── GET    /:id            # role detail + permission list
    │   ├── PATCH  /:id            # update name / description (custom only)
    │   ├── DELETE /:id            # delete role (custom only; blocked if assigned)
    │   ├── GET    /:id/permissions     # list permissions
    │   ├── PUT    /:id/permissions     # replace full permission set
    │   ├── POST   /:id/permissions     # add individual permission
    │   └── DELETE /:id/permissions/:pid   # remove permission
    │
    ├── permissions/
    │   └── GET    /               # enumerate all valid resource:action combinations
    │
    └── audit/
        ├── GET    /               # paginated audit log (filterable)
        └── GET    /stream         # SSE live audit stream
```

---

### 18.8 NATS IAM Events

| Subject | Published by | Consumed by | Description |
|---|---|---|---|
| `dnsdave.iam.session_invalidated.{user_id}` | `dnsdave-api` | All API instances | Invalidate in-process permission cache on role/group change |
| `dnsdave.iam.user_locked.{user_id}` | `dnsdave-api` | `dnsdave-ui` SSE | Notify admin console of lockout |
| `dnsdave.iam.audit` | `dnsdave-api` | `dnsdave-log`, `dnsdave-export` | Stream audit events to log store and export backends |
