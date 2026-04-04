# DNSDave — Product Design Document

**Version:** 0.3.0-draft  
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
- A monolithic architecture makes adding new functionality (alerting, analytics, integrations) require touching the core process.

DNSDave is a ground-up reimagining: **performance-first**, **API-first**, **event-driven**, **container-native**, with first-class blocklist support fully compatible with the existing Pi-hole / Steven Black / OISD ecosystem.

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
| G9 | **Loose coupling via an event bus.** The DNS hot path offloads all side-effects (logging, stats, notifications) asynchronously. New functionality is added by subscribing to the bus — zero changes to the DNS container. |

### Non-Goals (v1)

- Building a full recursive resolver (we forward upstream).
- DNSSEC signing of local zones.
- A built-in DHCP server (can integrate with existing DHCP via API hooks).
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
    end

    PG[("PostgreSQL\nconfig · lists · logs")]

    Clients -->|"DNS UDP/TCP :53  ·  DoT :853"| DNS
    DNS -->|"query events\nfire & forget · MessagePack"| NATS
    NATS -->|"config hot-swap · blocklist diffs\nbloom filters · runtime KV"| DNS
    API -->|"config change events"| NATS
    NATS -->|"query events"| LOG
    NATS -->|"query events"| STATS
    SYNC -->|"blocklist diffs · bloom filter\nready signals"| NATS
    API <-->|"reads / writes"| PG
    SYNC -->|"list entries delta"| PG
    LOG -->|"bulk inserts"| PG
```

### 3.2 Container Responsibilities

| Container | Language | Role | Talks to |
|-----------|----------|------|----------|
| `dnsdave-dns` | **Rust** | DNS data plane. Serves queries. Zero DB access. | NATS only |
| `dnsdave-api` | Rust or Go | REST API + Pi-hole shim. Owns all config writes. | Postgres + NATS |
| `dnsdave-sync` | Rust or Go | Blocklist downloader + parser. | Postgres + NATS + HTTP (lists) |
| `dnsdave-log` | Go or Rust | Query log consumer. Bulk-writes to Postgres. | NATS + Postgres |
| `dnsdave-stats` | Go or Rust | Stats aggregator. Maintains in-memory counters. | NATS |
| `nats` | — | Event bus. Core pub/sub + JetStream + Object Store. | — |
| `postgres` | — | Primary persistent storage. | — |

### 3.3 Key Design Principles

**The DNS container never touches the database.** It derives its entire runtime state (blocklist, local records, upstreams, allowlist) from events consumed from NATS JetStream. On startup it replays JetStream history to rebuild state from the beginning of time, without touching Postgres. This means `dnsdave-dns` can scale to N instances with no additional configuration.

**The DNS container offloads all side-effects.** Query log writes, stats updates, and any future integrations are downstream consumers of the `dnsdave.query.*` subject. The DNS hot path fires a NATS publish (a non-blocking memory write to the client buffer) and moves on. If NATS is momentarily unavailable, query events fall back to the local ring buffer.

**New functionality = new consumer.** Want to add alerting when a client queries a malware domain? Add a container that subscribes to `dnsdave.query.*` and filters. Want to push stats to InfluxDB? New container, new subscription. The DNS and API containers are never modified.

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

NATS is orders of magnitude lighter than Kafka or Redpanda for this workload. A single NATS node handles millions of messages per second with sub-millisecond latency. JetStream clustering gives you HA with 3 nodes when needed.

### 4.2 Subject Hierarchy

```
dnsdave.
├── query.                            # DNS query log events
│   └── {client_group_id}            #   e.g. dnsdave.query.default
│                                    #   Delivery: core NATS (at-most-once)
│                                    #   Volume: one message per DNS query
│
├── config.                          # Control plane config changes
│   ├── record.{action}              #   action: created | updated | deleted
│   ├── list.{action}
│   ├── upstream.{action}
│   ├── client.{action}
│   ├── group.{action}
│   └── system.{action}
│                                    #   Delivery: JetStream (at-least-once)
│                                    #   Retention: all (replay from seq 0 on node start)
│
├── blocklist.
│   ├── diff.{list_id}               #   domain add/remove diff batch
│   └── ready.{list_id}              #   sync complete; bloom filter published
│                                    #   Delivery: JetStream (at-least-once)
│                                    #   Retention: last value per list_id (KV semantics)
│
└── upstream.
    └── health.{upstream_id}         #   healthy | degraded | down
                                     #   Delivery: core NATS
```

### 4.3 JetStream Streams

| Stream | Subjects | Retention | Purpose |
|--------|----------|-----------|---------|
| `DNSDAVE_CONFIG` | `dnsdave.config.>` | All messages (no limit) | Config replay for new/restarted DNS nodes |
| `DNSDAVE_BLOCKLIST` | `dnsdave.blocklist.>` | Last per subject | Blocklist diff replay; new nodes apply all diffs since last full sync |
| `DNSDAVE_QUERYLOG` | `dnsdave.query.>` | 24h rolling window | Optional audit / replay for log writer catch-up |

### 4.4 NATS Object Store (Bloom Filters)

After a blocklist sync, `dnsdave-sync` serialises the updated bloom filter and uploads it to the **NATS Object Store** bucket `DNSDAVE_BLOOM` with key `bloom.{list_id}`. This is a binary blob (typically 10–100 MB per list) that NATS chunks and stores internally.

When a `dnsdave.blocklist.ready.{list_id}` event arrives, `dnsdave-dns` fetches the bloom filter object, deserialises it into a new `BloomFilter` struct, and atomically swaps the pointer. The old filter is dropped. No file system, no Postgres, no custom chunking code needed.

### 4.5 NATS KV Store (Runtime Config)

Frequently-read, rarely-changed config (upstream list, client-to-group mapping, allowlist) is mirrored into a **NATS KV Store** bucket `DNSDAVE_RUNTIME`. The DNS container watches this bucket for changes via a NATS watcher — when a key changes, it triggers a copy-on-write rebuild of the relevant in-memory structure and an atomic pointer swap.

This eliminates any polling and gives sub-10ms config propagation to all DNS nodes with no direct Postgres dependency.

### 4.6 Query Event Schema

Each DNS query produces one event published to `dnsdave.query.{client_group_id}`. The payload is a compact MessagePack-encoded struct (not JSON — avoids allocation overhead in the hot path):

```rust
struct QueryEvent {
    ts_us:         u64,    // unix timestamp, microseconds
    client_ip:     [u8; 16], // IPv4-mapped IPv6
    client_id:     u32,    // index into client table (0 = unknown)
    domain_len:    u8,
    domain:        [u8; 253], // DNS name, lowercased
    qtype:         u16,    // DNS QTYPE
    response_type: u8,     // 0=allowed 1=blocked 2=cached 3=nxdomain 4=noerror
    matched_id:    u32,    // list_id or record_id; 0 = none
    upstream_id:   u8,     // 0 = no upstream used
    latency_us:    u32,    // query latency in microseconds
}
// ~300 bytes packed; NATS publish is a memcpy to client buffer
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
    KV-->>DNS: upstream list · client/group map · allowlist
    Note over DNS: Build upstream pool, CIDR table, allowlist map

    DNS->>JS: Subscribe DNSDAVE_CONFIG from seq 0
    JS-->>DNS: Replay all config events
    Note over DNS: Build local records map + wildcard trie

    DNS->>JS: Subscribe DNSDAVE_BLOCKLIST from seq 0
    JS-->>DNS: Replay all blocklist diffs
    Note over DNS: Build blocklist hash map

    loop per enabled list
        DNS->>OBJ: Fetch bloom.{list_id}
        OBJ-->>DNS: Serialised bloom filter bytes
        Note over DNS: Deserialise → ArcSwap
    end

    Note over DNS: Begin serving on :53
    DNS->>JS: Subscribe dnsdave.config.> (live updates)
    DNS->>JS: Subscribe dnsdave.blocklist.> (live updates)
    Note over DNS: Publish dnsdave.query.* per query
```

Total cold-start time target: **<10 seconds** for a 5M-domain blocklist (dominated by bloom filter fetch + deserialise, ~3–5s).

### 4.8 Extensibility — Adding New Consumers

The event bus is the extension point. No existing container is modified.

| Future feature | New consumer subscribes to |
|---------------|---------------------------|
| Malware domain alerting | `dnsdave.query.*`, filters on `response_type=blocked` + threat list |
| Slack / webhook notifications | Same as above |
| InfluxDB / Grafana metrics | `dnsdave.query.*`, aggregates counters |
| SIEM / audit log export | `DNSDAVE_QUERYLOG` JetStream with durable consumer |
| DHCP lease registration | New producer publishes `dnsdave.config.record.created` |
| RPZ zone export | `dnsdave.blocklist.*` consumer that generates a zone file |
| Rate limit enforcement | `dnsdave.query.*` consumer publishes block rules back via KV |

---

## 5. Performance Architecture

Performance is a first-class design constraint, not an afterthought. Every decision in this section is motivated by the goal of matching or exceeding Pi-hole FTL's query throughput while operating entirely in user-space Rust.

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
    E -->|hit| R2(["Return answer"])
    E -->|miss| F{"5  Wildcard trie\nmatch?"}
    F -->|hit| R3(["Return answer"])
    F -->|miss| G{"6  Bloom filter\nnegative?"}
    G -->|"negative\nguaranteed not blocked"| I
    G -->|positive| H{"7  Blocklist HashMap\nconfirmed blocked?"}
    H -->|blocked| R4(["Encode NXDOMAIN\nPublish QueryEvent → NATS\nReturn buffer"])
    H -->|false positive| I{"8  Answer cache\nhit?"}
    I -->|hit| R5(["Encode cached answer\nPublish QueryEvent → NATS\nReturn buffer"])
    I -->|miss| J["9  Forward upstream\ntokio · HTTP/2 DoH pool"]
    J --> K(["Cache answer · Encode response\nPublish QueryEvent → NATS\nReturn buffer"])
```

The NATS publish in steps 7, 8, and 10 is a **non-blocking write to the async-nats client buffer**. It does not cross a network boundary on the query hot path. The async-nats client flushes the buffer to the NATS socket in a background Tokio task.

### 5.2 UDP I/O — Multi-Socket with SO_REUSEPORT

N Tokio tasks (one per CPU core) each own a UDP socket bound to `:53` with `SO_REUSEPORT`. The kernel distributes packets across sockets via a consistent hash of source IP + port. No userspace lock on the receive path.

On Linux: `recvmmsg`/`sendmmsg` batch I/O via the `nix` crate — up to 64 packets per syscall.  
On macOS (dev): standard `recv_from` / `send_to` fallback.

### 5.3 Zero-Allocation Hot Path

All byte buffers for DNS message parsing and encoding are managed by a pool. Two tiers:

| Pool | Size | Use |
|------|------|-----|
| Small | 512 bytes | Standard DNS queries |
| Large | 4096 bytes | EDNS0 / large responses |

Buffers are borrowed on query entry and returned on exit. No heap allocation occurs during normal query processing.

### 5.4 Lock-Free Data Structures (Rust)

All hot-path lookups use `Arc<_>` + `ArcSwap` (from the `arc-swap` crate) — the Rust equivalent of `atomic.Pointer`. Writers build a new structure, swap the Arc, and drop the old one when the reference count reaches zero. Readers load the Arc with a single atomic instruction — no locks, no waits.

| Structure | Rust type | Update strategy |
|-----------|-----------|-----------------|
| Bloom filter | `ArcSwap<BloomFilter>` | Swap on NATS Object Store fetch |
| Blocklist map | `ArcSwap<HashMap<DomainKey, u64>>` | Swap on blocklist diff apply |
| Allowlist set | `ArcSwap<HashSet<DomainKey>>` | Swap on config event |
| Local records map | `ArcSwap<HashMap<DomainKey, Vec<Record>>>` | Swap on config event |
| Wildcard trie | `ArcSwap<LabelTrie>` | Swap on config event |

### 5.5 Blocklist In-Memory Layout

**Stage 1 — Bloom filter:** `hashbrown`-backed counting Bloom filter with `k=10` xxHash3 functions. FPR of ~0.001% at 5M domains, ~90MB. A negative is definitive — no further lookup. Held behind `ArcSwap`.

**Stage 2 — Perfect hash map:** For the confirmed-positive path, the blocklist map is built with `boomphf` (Backyard Hashing — Minimal Perfect Hashing). Since the map is immutable between syncs, a minimal perfect hash eliminates all collisions and gives O(1) lookup with ~3 bits/entry storage. For 5M domains: ~2MB for the MPHF index + ~40MB for the value array. Total: ~42MB vs ~300MB for a standard HashMap.

### 5.6 Blocklist Parser — Streaming, Zero-Copy

- Stream-read HTTP response body; transparent gzip/zstd decompression inline.
- `memchr` crate for SIMD-accelerated newline scanning (AVX2 on x86_64, NEON on ARM64).
- In-place lowercase via a `[u8; 256]` lookup table.
- Domain validation via direct byte scan — no regex, no allocation per line.
- Multiple list sources fetched and parsed concurrently with Tokio tasks; results fanned into a shared concurrent dedup set.

**Target parse rates:**

| List | Size | Target |
|------|------|--------|
| StevenBlack unified | ~160K domains | <200ms |
| OISD big | ~1M domains | <1s |
| Hagezi multi pro++ | ~700K domains | <800ms |
| All lists combined (5M unique) | — | <3s cold; <1s incremental |

### 5.7 Answer Cache — moka

The answer cache uses `moka` — a high-performance concurrent cache for Rust modelled on Caffeine (Java). It uses a window TinyLFU policy that outperforms LRU for DNS access patterns (power-law domain distribution). Internally sharded; no global lock.

- TTL-aware eviction; negative caching (NXDOMAIN/NODATA per RFC 2308).
- Cache pre-warming on startup by replaying recent `DNSDAVE_QUERYLOG` events from JetStream.

### 5.8 NATS Publish — Impact on Hot Path

The `async-nats` client maintains an internal send buffer (a Tokio MPSC channel). Publishing a query event is a non-blocking channel send — equivalent to a single atomic write. The actual network send to the NATS socket happens in a background Tokio task. This means:

- **No syscall** on the DNS hot path for query logging.
- **No DNS latency added** by the NATS publish.
- **Backpressure handling**: if the NATS client buffer fills (NATS unreachable), the DNS container falls back to the local ring buffer. Query serving never blocks.

### 5.9 Performance Targets

| Metric | Target | Condition |
|--------|--------|-----------|
| Blocked query latency (p50) | <100 µs | Bloom filter negative |
| Blocked query latency (p99) | <150 µs | No GC pauses (Rust) |
| Local record latency (p50) | <80 µs | HashMap hit |
| Cache hit latency (p50) | <150 µs | moka hit |
| Cache miss latency (p50) | <5 ms | DoH upstream, LAN |
| Throughput (blocked/cached) | >800K QPS | 4-core host, UDP LAN |
| Throughput (upstream) | >50K QPS | Network RTT bound |
| Blocklist parse (1M domains) | <1s | Cold sync |
| Bloom filter hot-swap (5M) | <5s | Background; zero downtime |
| NATS query event publish | <1 µs | Non-blocking buffer write |
| DNS container cold start | <10s | 5M domain blocklist, from NATS |
| RSS (5M domain blocklist, MPHF) | <250MB | vs ~650MB with Go + HashMap |

### 5.10 Benchmarking & Profiling

- `cargo bench` — criterion-based microbenchmarks for each hot-path stage.
- `make bench-dns` — spins up the stack and runs `flamethrower`/`dnsperf` for QPS + latency distribution.
- `cargo flamegraph` / `perf` — CPU profiling.
- pprof-compatible endpoints available in `debug` build via `tokio-console`.
- Benchmark results recorded in `benchmarks/` and committed; CI fails on >5% regression.

---

## 6. DNS Feature Set

### 6.1 Record Types

| Type | Scope | Notes |
|------|-------|-------|
| `A` | Local zone | IPv4 override or custom record |
| `AAAA` | Local zone | IPv6 override |
| `CNAME` | Local zone | Chained resolution |
| `MX` | Local zone | Mail routing |
| `TXT` | Local zone | Arbitrary; useful for split-horizon Let's Encrypt |
| `PTR` | Local zone | Reverse DNS for local hosts |
| `SRV` | Local zone | Service discovery (Kubernetes-style) |
| Wildcard `*` | Local zone | See §6.2 |

### 6.2 Wildcard DNS

A record with a name of `*.internal.example.com` matches any single label prefix that does not have a more-specific record. Resolution priority:

```
exact match  >  wildcard match (longest suffix wins)  >  blocklist check  >  upstream
```

Examples (given `*.internal.example.com → 10.0.0.50`):

| Query | Result |
|-------|--------|
| `api.internal.example.com` | `10.0.0.50` (wildcard) |
| `db.internal.example.com` | `10.0.0.50` (wildcard) |
| `db.internal.example.com` with explicit `A` record `10.0.0.51` | `10.0.0.51` (exact wins) |
| `nested.api.internal.example.com` | forwarded upstream (single-label by default) |

A `recursive_wildcard` flag enables multi-label matching (`**.internal.example.com` semantics).

**Wildcard trie:** Keyed right-to-left by DNS label. Copy-on-write; published via `ArcSwap`. Updated by consuming `dnsdave.config.record.*` events from NATS.

### 6.3 Split-Horizon / Views

Named views allow different responses for different client groups. View assignment is based on client IP/CIDR, looked up in the CIDR table on each query. The CIDR table is a small `Vec<(IpNet, ViewId)>` loaded from the NATS KV Store.

### 6.4 Upstream Resolver Configuration

```jsonc
// POST /api/v1/upstreams
{
  "name": "cloudflare-doh",
  "type": "doh",
  "address": "https://1.1.1.1/dns-query",
  "priority": 1,
  "timeout_ms": 2000,
  "health_check_interval_s": 30
}
```

DoH upstreams use persistent HTTP/2 connections (one connection per upstream, multiplexed via Tokio). Health is checked continuously; state is published to `dnsdave.upstream.health.*` for all consumers to observe.

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

A `/api/pihole/` shim in `dnsdave-api` exposes the Pi-hole v5/v6 admin API surface so that existing clients (browser extensions, iOS/Android apps, Grafana datasources) work without modification.

---

## 8. REST API Design

### 8.1 Principles

- **OpenAPI 3.1** spec is the source of truth; server stubs are generated from it.
- All endpoints are under `/api/v1/`.
- JSON in and out; errors use [RFC 9457 Problem Details](https://www.rfc-editor.org/rfc/rfc9457).
- Pagination via `?page=` + `?per_page=`; filtering via `?q=` and `?type=`.
- Bulk operations supported on all list/record endpoints.
- Every mutating operation is idempotent via `PUT`; `POST` creates-or-updates by natural key.
- After every write, `dnsdave-api` publishes the appropriate `dnsdave.config.*` event to NATS before returning 200 to the client.

### 8.2 Resource Map

```
/api/v1/
├── auth/
│   ├── POST   token
│   └── DELETE token
├── dns/
│   ├── records/
│   │   ├── GET    /
│   │   ├── POST   /
│   │   ├── GET    /:id
│   │   ├── PUT    /:id
│   │   └── DELETE /:id
│   ├── wildcards/
│   ├── upstreams/
│   └── views/
├── lists/
│   ├── GET    /
│   ├── POST   /
│   ├── GET    /:id
│   ├── PUT    /:id
│   ├── DELETE /:id
│   └── POST   /:id/sync
├── clients/
├── groups/
├── logs/
│   ├── GET    /               # paginated (reads Postgres via dnsdave-log)
│   └── GET    /stream         # SSE live tail (reads NATS dnsdave.query.*)
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

### 8.3 Authentication

| Method | Use Case |
|--------|----------|
| `X-API-Key: <key>` header | Automation, scripts, integrations |
| `Authorization: Bearer <jwt>` | Short-lived sessions, UI |
| mTLS (optional) | Kubernetes service-to-service |

### 8.4 Live Log Stream

`GET /api/v1/logs/stream` is a Server-Sent Events endpoint that subscribes directly to `dnsdave.query.*` in NATS and streams decoded events to the client. This bypasses Postgres entirely — zero storage overhead for the live tail path. Supports `?client=`, `?domain=`, `?type=blocked` query filters applied server-side before forwarding to the SSE client.

---

## 9. Data Model

### 9.1 Core Tables

```sql
CREATE TABLE dns_records (
    id          TEXT PRIMARY KEY,
    view_id     TEXT REFERENCES views(id) ON DELETE SET NULL,
    name        TEXT NOT NULL,
    type        TEXT NOT NULL,       -- A, AAAA, CNAME, MX, TXT, PTR, SRV
    value       TEXT NOT NULL,
    priority    INTEGER DEFAULT 0,
    ttl         INTEGER DEFAULT 300,
    wildcard    BOOLEAN DEFAULT FALSE,
    recursive   BOOLEAN DEFAULT FALSE,
    enabled     BOOLEAN DEFAULT TRUE,
    comment     TEXT,
    created_at  TIMESTAMP DEFAULT NOW(),
    updated_at  TIMESTAMP DEFAULT NOW(),
    UNIQUE(view_id, name, type)
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
    identifier TEXT NOT NULL UNIQUE,   -- IP, CIDR, or MAC
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
    type                    TEXT NOT NULL,     -- plain | dot | doh
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

### 9.2 In-Memory Structures (dnsdave-dns)

All structures are derived from NATS — never from direct DB access.

| Structure | Rust Type | Source |
|-----------|-----------|--------|
| Bloom filter | `ArcSwap<BloomFilter>` | NATS Object Store |
| Blocklist MPHF map | `ArcSwap<BoomPHF<DomainKey>>` | NATS Object Store / diff replay |
| Allowlist set | `ArcSwap<HashSet<DomainKey>>` | NATS KV Store |
| Local records map | `ArcSwap<HashMap<DomainKey, Vec<Record>>>` | NATS JetStream replay |
| Wildcard trie | `ArcSwap<LabelTrie>` | NATS JetStream replay |
| Client CIDR table | `ArcSwap<Vec<(IpNet, ClientId)>>` | NATS KV Store |
| Answer cache | `moka::Cache<CacheKey, CachedAnswer>` | Local; upstream queries |
| Query event buffer | Lock-free MPSC ring buffer | NATS flush fallback |
| Upstream conn pool | `Arc<Mutex<HashMap<UpstreamId, Client>>>` | On startup + config events |

**Memory sizing (5M-domain blocklist, MPHF):**

| Component | Estimate |
|-----------|----------|
| Bloom filter (FPR 0.001%) | ~90 MB |
| MPHF blocklist index | ~42 MB |
| moka answer cache (1M entries) | ~150 MB |
| Local records + wildcard trie | <10 MB |
| Query event ring buffer | ~5 MB |
| **Total target RSS** | **~300 MB** |

---

## 10. Deployment

### 10.1 Docker Compose (Reference Stack)

```yaml
# docker-compose.yml
services:

  nats:
    image: nats:alpine
    container_name: dnsdave-nats
    restart: unless-stopped
    command: ["-js", "-m", "8222"]   # JetStream enabled + monitoring
    ports:
      - "4222:4222"    # client
      - "8222:8222"    # monitoring (internal only in prod)
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
    depends_on:
      - nats
    ports:
      - "53:53/udp"
      - "53:53/tcp"
      - "853:853/tcp"
    environment:
      DNSDAVE_NATS_URL: "nats://nats:4222"
      DNSDAVE_LOG_LEVEL: "info"
      DNSDAVE_WORKERS: "0"           # 0 = num_cpus
    networks:
      - dnsdave
    sysctls:
      net.core.rmem_max: "26214400"
      net.core.wmem_max: "26214400"

  dnsdave-api:
    image: ghcr.io/dnsdave/dnsdave-api:latest
    container_name: dnsdave-api
    restart: unless-stopped
    depends_on:
      - nats
      - postgres
    ports:
      - "8080:8080/tcp"    # REST API + DoH
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
    depends_on:
      - nats
      - postgres
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
    depends_on:
      - nats
      - postgres
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
    depends_on:
      - nats
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
```

### 10.2 Minimal Stack (dns + api only)

For resource-constrained environments (Raspberry Pi, home lab), a `docker-compose.minimal.yml` runs only `dnsdave-dns`, `dnsdave-api`, NATS, and uses SQLite via the API container. Query logging is written directly by `dnsdave-api` (sacrificing throughput for simplicity). Stats are disabled.

### 10.3 Podman Compose

The same compose files work with `podman-compose` without modification. A quadlet-compatible systemd unit file is provided for rootless Podman with socket activation on port 53 (`CAP_NET_BIND_SERVICE`).

### 10.4 Kubernetes / Helm

```
deploy/helm/dnsdave/
├── Chart.yaml
├── values.yaml
└── templates/
    ├── deployment-dns.yaml       # DaemonSet (host network, :53)
    ├── deployment-api.yaml       # Deployment (replicated)
    ├── deployment-sync.yaml      # Deployment (single replica)
    ├── deployment-log.yaml       # Deployment (single or replicated)
    ├── deployment-stats.yaml     # Deployment (single replica)
    ├── statefulset-nats.yaml     # NATS cluster (3 replicas for JetStream HA)
    ├── service-dns.yaml          # LoadBalancer :53 or hostPort DaemonSet
    ├── service-api.yaml          # ClusterIP :8080
    ├── ingress-api.yaml          # HTTPS ingress for REST API + DoH
    ├── pvc-nats.yaml
    ├── pvc-postgres.yaml
    ├── configmap.yaml
    ├── secret.yaml
    ├── hpa-api.yaml              # HPA for API replicas
    └── servicemonitor.yaml       # Prometheus ServiceMonitor
```

The DNS container runs as a **DaemonSet** with `hostNetwork: true` so it binds the node's port 53 directly. The API container runs as a standard **Deployment** (replicated, behind a ClusterIP service). NATS runs as a 3-node **StatefulSet** with JetStream clustering for HA.

### 10.5 Scaling the Stack

| Tier | Scale-out strategy |
|------|--------------------|
| `dnsdave-dns` | Add nodes; each subscribes to NATS independently. No coordination needed. |
| `dnsdave-api` | Scale Deployment replicas behind a load balancer. All replicas share Postgres + NATS. |
| `dnsdave-log` | Single replica is sufficient. Scale only if Postgres write throughput is saturated. |
| `dnsdave-stats` | Single replica (in-memory counters). HA via leader election via NATS KV. |
| `dnsdave-sync` | Single replica (one sync job per list). Leader election via NATS KV prevents duplicate syncs. |
| `nats` | 3-node JetStream cluster for HA. Scales to 5+ nodes for very high message volume. |
| `postgres` | Patroni HA (primary + standby). TimescaleDB for `query_log` at very high QPS. |

### 10.6 Configuration Priority

```
CLI flags > Environment Variables > Config file (YAML) > Defaults
```

---

## 11. Observability

### 11.1 Metrics (Prometheus)

Each container exposes a `/metrics` endpoint.

**dnsdave-dns:**

| Metric | Type | Labels |
|--------|------|--------|
| `dnsdave_dns_queries_total` | Counter | `response_type`, `qtype` |
| `dnsdave_dns_query_duration_us` | Histogram | `response_type` |
| `dnsdave_dns_blocked_total` | Counter | — |
| `dnsdave_dns_cache_hits_total` | Counter | — |
| `dnsdave_dns_cache_misses_total` | Counter | — |
| `dnsdave_dns_bloom_false_positives_total` | Counter | — |
| `dnsdave_dns_nats_publish_errors_total` | Counter | — |
| `dnsdave_dns_nats_buffer_size` | Gauge | — |

**dnsdave-sync:**

| Metric | Type | Labels |
|--------|------|--------|
| `dnsdave_sync_domains_total` | Gauge | `list_id` |
| `dnsdave_sync_duration_s` | Histogram | `list_id` |
| `dnsdave_sync_errors_total` | Counter | `list_id` |

**nats:**

NATS exposes a Prometheus-compatible `/metrics` endpoint on port 8222 via the `nats-surveyor` sidecar or the built-in monitoring endpoint.

### 11.2 Structured Logging

All containers emit JSON logs. Level: `debug | info | warn | error`. Correlate across containers using a `trace_id` that is generated per DNS query and embedded in the NATS query event payload.

### 11.3 Live Query Stream

`GET /api/v1/logs/stream` (SSE) subscribes to `dnsdave.query.*` in NATS and streams decoded events. Zero Postgres reads. Sub-100ms latency from query to stream delivery.

### 11.4 Health & Readiness

| Container | Endpoint | Ready when |
|-----------|----------|-----------|
| `dnsdave-dns` | `:9090/health`, `:9090/ready` | NATS connected, bloom filter loaded |
| `dnsdave-api` | `:8080/api/v1/system/health`, `/ready` | NATS + Postgres connected |
| `dnsdave-sync` | `:9091/health` | NATS + Postgres connected |
| `dnsdave-log` | `:9092/health` | NATS + Postgres connected |
| `nats` | `:8222/healthz` | Built-in |

---

## 12. Security

| Concern | Mitigation |
|---------|-----------|
| Unauthenticated DNS | Standard; API always requires key/JWT |
| API key storage | Argon2id-hashed at rest; never logged |
| DoT/DoH TLS | ACME auto-provisioned or BYO cert |
| NATS auth | Username/password or NKey per container; deny-all default |
| Rate limiting | Per-client DNS token bucket; API token bucket |
| Log retention | `query_log` auto-partitioned and purged (default 30 days) |
| Container isolation | `distroless/static` base; no shell, no package manager |
| CORS | Configurable allowed origins |
| Secrets | `.env` file or Kubernetes `Secret`; never baked into images |

NATS authentication uses per-container NKey credentials. `dnsdave-dns` has publish permission on `dnsdave.query.*` and subscribe permission on `dnsdave.config.>`, `dnsdave.blocklist.>`. It has no publish permission on config subjects — it cannot modify its own config.

---

## 13. Milestones

### v0.1 — Core DNS + event bus foundations
- [ ] `dnsdave-dns`: Rust, SO_REUSEPORT, UDP, forward-only resolver
- [ ] `dnsdave-api`: REST API skeleton, Postgres, config CRUD
- [ ] `nats`: JetStream + KV store configured
- [ ] Config events published on every API write
- [ ] `dnsdave-dns` consumes config events, rebuilds in-memory state
- [ ] Basic blocklist ingest (hosts format) via sync worker
- [ ] `docker-compose.yml` reference stack
- [ ] Benchmark suite baseline

### v0.2 — Blocklist ecosystem + event-driven distribution
- [ ] All list formats (adblock, domain-only, RPZ)
- [ ] MPHF blocklist map (`boomphf`)
- [ ] Bloom filter serialisation → NATS Object Store
- [ ] `dnsdave-dns` hot-swap via Object Store fetch
- [ ] `dnsdave-log`: NATS consumer → Postgres bulk writer
- [ ] `dnsdave-stats`: in-memory counters from query events
- [ ] Groups + per-group policy
- [ ] Pi-hole API shim (v5)

### v0.3 — Advanced DNS
- [ ] Wildcard records + copy-on-write label trie
- [ ] CNAME chaining
- [ ] PTR (reverse DNS)
- [ ] Split-horizon views
- [ ] DoT + DoH upstream with HTTP/2 connection pooling
- [ ] Negative caching (NXDOMAIN/NODATA)
- [ ] Cache pre-warming from JetStream querylog replay

### v0.4 — Production readiness
- [ ] DoT listener (port 853) in `dnsdave-dns`
- [ ] DoH listener in `dnsdave-api`
- [ ] TLS auto-provisioning (ACME)
- [ ] Prometheus metrics across all containers
- [ ] SSE live log stream via NATS
- [ ] Helm chart v1 (DaemonSet DNS, replicated API, NATS StatefulSet)
- [ ] `recvmmsg`/`sendmmsg` batch I/O
- [ ] NATS NKey per-container auth

### v0.5 — Multi-node HA + scalability
- [ ] `dnsdave-dns` horizontal scale (N nodes, all subscribe independently)
- [ ] NATS JetStream 3-node cluster
- [ ] Leader election for `dnsdave-sync` and `dnsdave-stats` via NATS KV
- [ ] Postgres Patroni HA integration
- [ ] Active-active DNS with keepalived/VRRP (single site)
- [ ] Pi-hole API shim (v6)

### v1.0 — Multi-region
- [ ] Multi-region anycast (BGP)
- [ ] Postgres logical replication cross-region
- [ ] Per-region NATS cluster with leaf node federation

---

## 14. Technology Choices

### 14.1 Per-Container Stack

| Container | Language | Key Libraries |
|-----------|----------|--------------|
| `dnsdave-dns` | **Rust** | `hickory-dns`, `tokio`, `socket2`, `nix` (recvmmsg), `async-nats`, `arc-swap`, `hashbrown`, `boomphf`, `bloomfilter`, `moka`, `memchr` |
| `dnsdave-api` | **Rust** | `axum`, `tokio`, `async-nats`, `sqlx`, `utoipa` (OpenAPI) |
| `dnsdave-sync` | **Rust** | `tokio`, `async-nats`, `sqlx`, `reqwest`, `memchr`, `boomphf`, `bloomfilter` |
| `dnsdave-log` | **Rust** | `async-nats`, `sqlx`, `tokio` |
| `dnsdave-stats` | **Rust** | `async-nats`, `tokio`, `dashmap` |

### 14.2 Infrastructure

| Concern | Choice | Rationale |
|---------|--------|-----------|
| Event bus | **NATS JetStream** | <100µs publish latency; 15MB image; built-in Object Store + KV; Rust `async-nats` client; JetStream clustering for HA |
| DNS library | `hickory-dns` | Primary Rust DNS library; exposes wire format for zero-copy |
| Async runtime | `tokio` | De facto standard; excellent UDP + HTTP/2 support |
| UDP batch I/O | `nix` (`recvmmsg`/`sendmmsg`) | Batch syscalls on Linux; `socket2` for SO_REUSEPORT |
| Bloom filter | `bloomfilter` | Serialisable; configurable FPR |
| Perfect hash | `boomphf` | Minimal perfect hash for immutable blocklist; ~3 bits/entry |
| Concurrent cache | `moka` | Window TinyLFU; better hit rate than LRU for DNS; internally sharded |
| Lock-free swap | `arc-swap` | ArcSwap for atomic pointer swap without unsafe code |
| Concurrent map | `dashmap` | For write-heavy concurrent maps (stats counters) |
| Byte scanning | `memchr` | AVX2/NEON SIMD; fastest byte search available |
| HTTP client | `reqwest` | Async; HTTP/2; transparent gzip/zstd |
| Database | `sqlx` | Async; compile-time query checking; SQLite + PostgreSQL |
| SQLite | `sqlx` with `libsqlite3` | Bundled; no separate install |
| ORM | None — `sqlx` raw queries | No magic; full control |
| Config | `config` crate + `clap` | Flag / env / file priority chain |
| OpenAPI | `utoipa` + `aide` | Macro-based; generates spec from handler annotations |
| Serialisation (events) | `rmp-serde` (MessagePack) | ~300 bytes per query event; no JSON parsing overhead |
| Container base | `gcr.io/distroless/cc` | No shell; minimal attack surface; glibc for Rust linkage |
| CI | GitHub Actions | Multi-arch build (amd64 + arm64); push to GHCR |
| Benchmarking | `criterion` + `dnsperf` + `flamethrower` | Microbenchmarks + QPS + flamegraph |

### 14.3 Language Rationale

Rust is chosen over Go for the following concrete reasons:

| Factor | Rust | Go |
|--------|------|----|
| GC pauses | None | <1ms, but present; impacts p99/p999 |
| Blocklist lookup | `hashbrown` SIMD probing + MPHF | Go map (no SIMD) |
| Memory (5M blocklist) | ~300MB (MPHF) | ~600MB (map) |
| Byte scanning | `memchr` AVX2/NEON | `bytes.IndexByte` (SIMD, but less tuned) |
| Lock-free swap | `ArcSwap` (no unsafe) | `atomic.Pointer` (equivalent) |
| Async I/O | `tokio` (equivalent) | goroutines (equivalent) |
| DNS library maturity | `hickory-dns` (good) | `miekg/dns` (excellent) |
| Development velocity | Slower | Faster |

All containers are Rust for consistency. If `hickory-dns` proves insufficient, the wire-format layer can be replaced with a custom implementation using the raw `tokio` UDP socket — nothing else changes.

---

## 15. Open Questions

1. **Web UI scope** — API-only for v1, or ship a minimal read-only Svelte dashboard that subscribes to the SSE stream?
2. **DHCP integration** — Expose a webhook so Kea/dnsmasq can register leases by publishing `dnsdave.config.record.created` to NATS?
3. **RPZ as output** — Serve RPZ zones to downstream resolvers by consuming `dnsdave.blocklist.*` and generating zone files?
4. **DoH port** — Serve DoH on `:8080/dns-query` (shared with API) or dedicated `:443`?
5. **Multi-tenancy** — First-class namespace/org concept for SaaS, or single-tenant-per-instance?
6. **`dnsdave-stats` persistence** — Keep counters in-memory only (lost on restart) or checkpoint to NATS KV periodically?
7. **NATS vs Redpanda at scale** — At what sustained QPS should we evaluate replacing NATS with Redpanda? Benchmark trigger: >500K queries/sec sustained for >1 hour.
8. **Blocklist NATS Object Store limits** — NATS Object Store chunks objects into 128KB parts. For a 90MB bloom filter this is ~700 chunks. Benchmark the fetch + reassemble time at startup vs. fetching from Postgres directly.
