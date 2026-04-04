# DNSDave – Test Specification

**Version:** 0.1.0-draft  
**Date:** 2026-04-04  
**Status:** Draft  
**Tracks:** DESIGN.md v0.4.0

Every test in this document must pass before the corresponding milestone is considered complete. Tests are labelled by type and component and are written to be directly translatable into Rust test code (`#[test]`, `#[tokio::test]`, integration harness, or `cargo bench`).

---

## Test ID Scheme

```
{COMPONENT}-{TYPE}-{NNN}

COMPONENT : DNS | BL | DHCP | DDNS | ZONE | FWD | DNSSEC | API | BUS | PERF | SEC | COMPAT
TYPE      : U (unit) | I (integration) | L (load)
NNN       : zero-padded sequence within component+type
```

---

## Test Types

| Type | Scope | External deps | Run on |
|------|-------|--------------|--------|
| **Unit** | Single function or module | None – all I/O mocked | `cargo test` |
| **Integration** | One or more containers, full stack | Docker Compose test stack | CI, `make test-integration` |
| **Load** | Full stack under sustained traffic | Docker Compose + traffic generator | Dedicated runner, `make test-load` |

---

## 1. DNS – Unit Tests

### DNS-U-001 · Wire format decode – valid query

**Goal:** Parser correctly decodes a well-formed DNS query packet.

| Field | Value |
|-------|-------|
| Input | Raw bytes: single A-record query for `example.com`, QTYPE=A, QCLASS=IN |
| Expected | `name = "example.com"`, `qtype = A`, `id` matches packet ID |
| Pass | Zero allocations on hot path; parsed fields equal expected values |

---

### DNS-U-002 · Wire format decode – truncated packet

**Goal:** Parser returns an error, does not panic, on a truncated UDP payload.

| Field | Value |
|-------|-------|
| Input | First 6 bytes of a valid query packet (header cut mid-field) |
| Expected | `Err(ParseError::Truncated)` |
| Pass | No panic; no partial state leak |

---

### DNS-U-003 · Domain lowercasing – in-place lookup table

**Goal:** Domain lowercasing uses the 256-byte lookup table and produces no heap allocation.

| Field | Value |
|-------|-------|
| Input | `"AdS.EXaMPLE.CoM"` as wire-format bytes |
| Expected | `"ads.example.com"` |
| Pass | No allocation measured by Rust allocator instrumentation; output matches |

---

### DNS-U-004 · Rule engine – allowlist takes priority over blocklist

**Goal:** A domain present in both the allowlist and the blocklist resolves as allowed.

| Field | Value |
|-------|-------|
| Setup | Allowlist contains `safe.example.com`; blocklist contains `safe.example.com` |
| Input | Query for `safe.example.com` |
| Expected | Rule engine returns `RuleResult::Allowed` before consulting blocklist |
| Pass | `RuleResult` variant is `Allowed`; blocklist lookup function is never called |

---

### DNS-U-005 · Rule engine – exact local record takes priority over wildcard

**Goal:** An exact record match beats a wildcard match for the same name.

| Field | Value |
|-------|-------|
| Setup | Local records map: `api.home.arpa A 10.0.0.10`; wildcard trie: `*.home.arpa A 10.0.0.1` |
| Input | Query for `api.home.arpa` |
| Expected | Answer is `10.0.0.10` |
| Pass | Returned IP is `10.0.0.10`, not `10.0.0.1` |

---

### DNS-U-006 · Rule engine – priority chain exhausted → upstream

**Goal:** A domain not in allowlist, local records, or blocklist is forwarded.

| Field | Value |
|-------|-------|
| Setup | All lookups return miss; upstream mock returns `NOERROR A 1.2.3.4` |
| Input | Query for `unknown.domain` |
| Expected | `RuleResult::Forward` |
| Pass | Variant is `Forward`; upstream mock called exactly once |

---

### DNS-U-007 · Wildcard trie – single-label match

**Goal:** `*.k8s.home.arpa` matches `api.k8s.home.arpa` but not `nested.api.k8s.home.arpa`.

| Field | Value |
|-------|-------|
| Setup | Wildcard trie contains `*.k8s.home.arpa → 192.168.1.100`, `recursive = false` |
| Input A | Query for `api.k8s.home.arpa` |
| Input B | Query for `nested.api.k8s.home.arpa` |
| Expected A | Match, answer `192.168.1.100` |
| Expected B | Miss |
| Pass | Both inputs return expected result |

---

### DNS-U-008 · Wildcard trie – longest suffix wins

**Goal:** When two wildcards apply, the more specific one wins.

| Field | Value |
|-------|-------|
| Setup | Trie: `*.home.arpa → 10.0.0.1`; `*.internal.home.arpa → 10.0.0.2` |
| Input | Query for `db.internal.home.arpa` |
| Expected | Answer is `10.0.0.2` |
| Pass | Returned IP is `10.0.0.2` |

---

### DNS-U-009 · Wildcard trie – recursive flag enables multi-label match

**Goal:** `recursive = true` on `*.home.arpa` matches `deep.nested.home.arpa`.

| Field | Value |
|-------|-------|
| Setup | Trie: `*.home.arpa → 10.0.0.1`, `recursive = true` |
| Input | Query for `deep.nested.home.arpa` |
| Expected | Match, answer `10.0.0.1` |
| Pass | Returned IP is `10.0.0.1` |

---

### DNS-U-010 · Bloom filter – known blocked domain returns positive

**Goal:** A domain inserted into the Bloom filter is detected on lookup.

| Field | Value |
|-------|-------|
| Setup | Bloom filter populated with 1,000 known ad domains |
| Input | Each of the 1,000 domains |
| Expected | All 1,000 return positive |
| Pass | Zero false negatives (Bloom filter must never miss a true member) |

---

### DNS-U-011 · Bloom filter – empirical false positive rate within spec

**Goal:** FPR for 5M-domain filter is ≤0.001% against a disjoint probe set.

| Field | Value |
|-------|-------|
| Setup | Bloom filter built from 5,000,000 unique domains |
| Input | 10,000,000 domains guaranteed not in the filter |
| Expected | ≤100 false positives (FPR ≤0.001%) |
| Pass | `false_positive_count / 10_000_000 <= 0.00001` |

---

### DNS-U-012 · MPHF map – all inserted domains resolve

**Goal:** Every domain inserted into the MPHF blocklist map is found on lookup.

| Field | Value |
|-------|-------|
| Setup | MPHF built from 100,000 random domains |
| Input | All 100,000 domains |
| Expected | Each returns the associated `list_id` bitmask |
| Pass | Zero misses |

---

### DNS-U-013 · Answer cache – TTL-based expiry

**Goal:** A cached entry is not returned after its TTL elapses.

| Field | Value |
|-------|-------|
| Setup | Insert entry with TTL = 1 second into cache |
| Input A | Lookup before TTL expires |
| Input B | Lookup 1100ms after insertion |
| Expected A | Cache hit |
| Expected B | Cache miss |
| Pass | Both expectations met without flakiness (use mock clock) |

---

### DNS-U-014 · Answer cache – negative cache (NXDOMAIN)

**Goal:** An NXDOMAIN response is cached and served from cache on repeat queries.

| Field | Value |
|-------|-------|
| Setup | Upstream mock returns NXDOMAIN with SOA minimum TTL = 60s |
| Input | Same domain queried twice within 60 seconds |
| Expected | Upstream called once; second query served from cache as NXDOMAIN |
| Pass | Upstream mock call count = 1 |

---

### DNS-U-015 · ArcSwap – concurrent readers see new value after swap

**Goal:** After an ArcSwap, all readers load the new value; no reader ever gets a partially-updated state.

| Field | Value |
|-------|-------|
| Setup | 16 reader threads spin-reading from ArcSwap; writer swaps to new value |
| Expected | Every read returns either the old complete value or the new complete value; never a mix |
| Pass | 10,000 iterations with Loom or miri; zero races detected |

---

### DNS-U-016 · Buffer pool – no heap allocation on hot path

**Goal:** Processing a DNS query borrows from the pool and returns to it; heap allocator is not called.

| Field | Value |
|-------|-------|
| Setup | Instrument allocator to count allocations |
| Input | 1,000 back-to-back query-response cycles through the rule engine (all mocked) |
| Expected | Zero heap allocations per cycle after first warm-up |
| Pass | `allocation_count_after_warmup == 0` |

---

## 2. Blocklist – Unit Tests

### BL-U-001 · Pi-hole hosts format – valid lines parsed

**Goal:** Parser extracts domain from `0.0.0.0 example.com` lines.

| Input lines | Expected domains |
|-------------|-----------------|
| `0.0.0.0 ads.example.com` | `ads.example.com` |
| `127.0.0.1 tracking.io` | `tracking.io` |
| `# comment line` | *(skipped)* |
| *(blank line)* | *(skipped)* |
| `0.0.0.0 localhost` | *(skipped – reserved)* |
| `0.0.0.0 0.0.0.0` | *(skipped – invalid)* |

**Pass:** Extracted domain set matches expected; no panic on any input line.

---

### BL-U-002 · Domain-only format – valid and invalid lines

| Input | Expected |
|-------|----------|
| `ads.example.com` | `ads.example.com` extracted |
| `example.com` | `example.com` extracted |
| `not a domain` | skipped |
| `too-long-label-exceeding-63-chars.example.com` | skipped |
| `例え.jp` (IDN) | accepted as punycode `xn--r8jz45g.jp` |

**Pass:** Correct domains extracted; invalid lines skipped without error.

---

### BL-U-003 · Adblock Plus format – `||domain^` extraction

| Input | Expected |
|-------|----------|
| `\|\|ads.example.com^` | `ads.example.com` |
| `\|\|tracker.io^$third-party` | `tracker.io` |
| `##.ad-banner` (element hiding) | skipped |
| `@@\|\|safe.example.com^` (exception) | skipped |
| `\|\|example.com^/path` | skipped (path present) |

**Pass:** Extracted domains match expected; cosmetic filters and exceptions skipped.

---

### BL-U-004 · RPZ format – QNAME policy extraction

| Input | Expected |
|-------|----------|
| `ads.example.com CNAME .` | `ads.example.com` |
| `tracker.io A 127.0.0.1` | `tracker.io` |
| `$TTL 3600` (directive) | skipped |
| `@ SOA ...` (SOA record) | skipped |

**Pass:** Domain extracted for QNAME policy records; directives/SOA skipped.

---

### BL-U-005 · Incremental diff – add and remove domains

**Goal:** Diff between old and new domain sets produces correct add/remove batches.

| Field | Value |
|-------|-------|
| Old set | `{a.com, b.com, c.com}` |
| New set | `{b.com, c.com, d.com}` |
| Expected adds | `{d.com}` |
| Expected removes | `{a.com}` |
| Expected unchanged | `{b.com, c.com}` |
| Pass | All three sets match exactly |

---

### BL-U-006 · Bloom filter serialise / deserialise round-trip

**Goal:** A serialised Bloom filter deserialises to an identical filter.

| Field | Value |
|-------|-------|
| Setup | Build filter from 1M domains; serialise to bytes |
| Input | Deserialise bytes; probe all 1M domains + 1M non-member domains |
| Expected | Same positive/negative results as original filter |
| Pass | Zero divergences between original and deserialised |

---

### BL-U-007 · MPHF construction – no false negatives, minimal false positives

| Field | Value |
|-------|-------|
| Input | 500,000 unique domain strings |
| Expected | Every inserted domain found; domains not in set not found |
| Pass | True positive rate = 100%; false positive rate = 0% (MPHF is exact) |

---

### BL-U-008 · Parser throughput – 1M domains in under 1 second

**Goal:** The streaming parser sustains >1M domains/second on a single core.

| Field | Value |
|-------|-------|
| Input | In-memory buffer containing 1,000,000 domain-only format lines |
| Expected | Parse completes in <1000ms |
| Pass | `criterion` benchmark `parse_1m_domains` mean < 1000ms on CI runner |

---

## 3. DHCP – Unit Tests

### DHCP-U-001 · DHCPv4 DISCOVER packet decode

**Goal:** DISCOVER packet decoded to correct fields.

| Field | Value |
|-------|-------|
| Input | Valid DISCOVER packet bytes (BOOTP op=1, htype=1, xid=0xDEADBEEF) |
| Expected | `op = BootRequest`, `xid = 0xDEADBEEF`, `chaddr = client MAC`, option 53 = DISCOVER |
| Pass | All decoded fields match |

---

### DHCP-U-002 · DHCPv4 OFFER packet encode

**Goal:** OFFER packet encodes all required fields.

| Field | Value |
|-------|-------|
| Input | Lease struct: `yiaddr = 192.168.1.100`, `xid = 0xDEADBEEF`, options 1/3/6/51/54 |
| Expected | Encoded bytes decode back to same fields; magic cookie = `0x63825363` |
| Pass | Round-trip equality; magic cookie present |

---

### DHCP-U-003 · Option encoding – option 121 (classless static routes)

**Goal:** Multiple classless routes encoded correctly per RFC 3442.

| Field | Value |
|-------|-------|
| Input | Routes: `10.0.0.0/8 → 192.168.1.2`, `172.16.0.0/12 → 192.168.1.3` |
| Expected | Option 121 bytes follow `{prefix_len}{masked_octets}{gateway}` format |
| Pass | Decoded routes match input routes exactly |

---

### DHCP-U-004 · Option hierarchy – reservation overrides scope

**Goal:** When the same option code exists at scope and reservation level, reservation value wins.

| Field | Value |
|-------|-------|
| Setup | Scope option 6 = `[8.8.8.8]`; reservation option 6 = `[192.168.1.53]` |
| Input | Resolve options for a client matching the reservation |
| Expected | Option 6 value = `[192.168.1.53]` |
| Pass | Returned option 6 matches reservation value |

---

### DHCP-U-005 · Option hierarchy – global fallback when scope not set

**Goal:** A global option is used when neither scope nor reservation define it.

| Field | Value |
|-------|-------|
| Setup | Global option 42 = `[0.ntp.pool.org]`; scope has no option 42 |
| Input | Resolve options for a client in the scope |
| Expected | Option 42 = `[0.ntp.pool.org]` |
| Pass | Returned option 42 matches global value |

---

### DHCP-U-006 · PXE architecture detection – option 93

| Option 93 value | Expected bootfile |
|-----------------|------------------|
| `0x0000` | `pxelinux.0` (BIOS) |
| `0x0006` | `bootia32.efi` (EFI IA32) |
| `0x0007` | `bootx64.efi` (EFI BC) |
| `0x0009` | `bootx64.efi` (EFI x86-64) |
| `0x000B` | `bootaa64.efi` (EFI ARM64) |

**Pass:** Each input value maps to the correct bootfile string.

---

### DHCP-U-007 · IP pool allocation – sequential assignment

**Goal:** Pool allocates IPs sequentially; no duplicate assignments.

| Field | Value |
|-------|-------|
| Setup | Pool: `192.168.1.100 – 192.168.1.110` (11 addresses) |
| Input | 11 sequential DISCOVER requests from unique MACs |
| Expected | Each gets a unique IP in the range |
| Pass | All 11 IPs distinct; all in range; no error |

---

### DHCP-U-008 · IP pool allocation – exhaustion returns NAK

**Goal:** When pool is full, the 12th request receives a NAK.

| Field | Value |
|-------|-------|
| Setup | Pool: `192.168.1.100 – 192.168.1.110`; 11 leases already active |
| Input | 12th DISCOVER from a new MAC |
| Expected | NAK response |
| Pass | Response message type = NAK (53 = 0x06) |

---

### DHCP-U-009 · IP pool – released IP returned to pool

**Goal:** After a client releases a lease, its IP is available for reassignment.

| Field | Value |
|-------|-------|
| Setup | Pool of 1; IP `192.168.1.100` assigned to MAC A |
| Step 1 | RELEASE from MAC A |
| Step 2 | DISCOVER from MAC B |
| Expected | MAC B gets `192.168.1.100` |
| Pass | `ack.yiaddr == 192.168.1.100` for MAC B |

---

### DHCP-U-010 · Client class match – vendor-class-id prefix

| Field | Value |
|-------|-------|
| Setup | Class rule: `option[60].text.startswith("PXEClient")` |
| Input A | DISCOVER with option 60 = `"PXEClient:Arch:00000"` |
| Input B | DISCOVER with option 60 = `"MSFT 5.0"` |
| Expected A | Class matches |
| Expected B | Class does not match |
| Pass | Both match expectations |

---

### DHCP-U-011 · Lease T1/T2 calculation

| Lease time | Expected T1 | Expected T2 |
|-----------|------------|------------|
| 86400s (24h) | 43200s (12h) | 75600s (21h) |
| 3600s (1h) | 1800s (30m) | 3150s (52.5m) |
| 7200s (2h) | 3600s (1h) | 6300s (1h45m) |

**Pass:** All T1/T2 values within 1 second of expected (integer truncation acceptable).

---

### DHCP-U-012 · DHCPv6 SOLICIT decode

| Field | Value |
|-------|-------|
| Input | Valid SOLICIT packet: msg-type=1, transaction-id=3 bytes, option IA_NA |
| Expected | `msg_type = Solicit`, `transaction_id` matches, IA_NA option present |
| Pass | All decoded fields match |

---

### DHCP-U-013 · DHCP relay – giaddr scope selection

**Goal:** `giaddr` matching a scope's subnet selects that scope.

| Field | Value |
|-------|-------|
| Setup | Scope A: `10.0.0.0/24`; Scope B: `10.0.1.0/24` |
| Input | DISCOVER with `giaddr = 10.0.1.1` |
| Expected | Scope B selected |
| Pass | Selected scope ID matches Scope B |

---

## 4. Blocklist – Integration Tests

### BL-I-001 · StevenBlack unified list sync and blocking

**Goal:** After syncing the StevenBlack unified hosts list, known ad domains are blocked.

| Field | Value |
|-------|-------|
| Setup | Stack running; POST StevenBlack list URL; trigger sync |
| Input | DNS queries for 50 domains known to be in the StevenBlack list |
| Expected | All 50 return NXDOMAIN |
| Pass | Zero non-NXDOMAIN responses |

---

### BL-I-002 · Adblock format list sync and blocking

**Goal:** After syncing an Adblock Plus format list, extracted domains are blocked.

| Field | Value |
|-------|-------|
| Setup | POST Adblock format list (EasyList subset); trigger sync |
| Input | 20 spot-check domains known to be in the list |
| Expected | All 20 return NXDOMAIN |
| Pass | All 20 blocked |

---

### BL-I-003 · Incremental sync – only delta applied

**Goal:** Adding 1,000 new domains to an existing list only blocks those new domains.

| Field | Value |
|-------|-------|
| Setup | List with 10,000 domains synced; list updated with 1,000 additions |
| Step | Trigger resync |
| Input A | 1,000 new domains |
| Input B | 100 domains not in the list (never were) |
| Expected A | All 1,000 blocked after sync |
| Expected B | All 100 return NOERROR / upstream response |
| Pass | All expectations met; sync completed in <5s |

---

### BL-I-004 · Blocklist hot-swap – zero dropped queries during rebuild

**Goal:** DNS continues serving during a full blocklist rebuild; no queries dropped or error.

| Field | Value |
|-------|-------|
| Setup | `flamethrower` sending 50K QPS of mixed blocked/clean queries |
| Step | Trigger full blocklist resync of 1M-domain list |
| Expected | Zero DNS errors (SERVFAIL) during sync; blocked domains remain blocked throughout |
| Pass | Error rate = 0%; blocked rate unchanged before/after swap |

---

### BL-I-005 · Per-group policy – group A blocked, group B allowed

**Goal:** The same domain is blocked for one client group and allowed for another.

| Field | Value |
|-------|-------|
| Setup | Blocklist assigned to group `restricted` only; domain `ads.example.com` in list |
| Client A | IP in group `restricted` |
| Client B | IP in group `default` (no blocklist) |
| Input A | Client A queries `ads.example.com` |
| Input B | Client B queries `ads.example.com` |
| Expected A | NXDOMAIN |
| Expected B | NOERROR (forwarded upstream) |
| Pass | Both responses match expectations |

---

### BL-I-006 · Allowlist overrides blocklist

**Goal:** A domain in both the blocklist and allowlist resolves correctly.

| Field | Value |
|-------|-------|
| Setup | `safe.internal.com` in blocklist via list; `safe.internal.com` in allowlist |
| Input | DNS query for `safe.internal.com` |
| Expected | NOERROR; forwarded upstream or local record answer |
| Pass | Response is not NXDOMAIN |

---

## 5. DNS – Integration Tests

### DNS-I-001 · End-to-end A record resolution

**Goal:** A manually created A record resolves correctly end-to-end over UDP port 53.

| Field | Value |
|-------|-------|
| Setup | POST `{ name: "nas.home.arpa", type: "A", value: "192.168.1.10" }` |
| Input | DNS query for `nas.home.arpa` A over UDP :53 |
| Expected | Answer: `192.168.1.10`, NOERROR |
| Pass | Response IP matches; response code NOERROR; received within 200ms |

---

### DNS-I-002 · End-to-end AAAA record

| Field | Value |
|-------|-------|
| Setup | POST AAAA record `{ name: "nas.home.arpa", type: "AAAA", value: "fd00::10" }` |
| Input | DNS query for `nas.home.arpa` AAAA |
| Expected | Answer: `fd00::10`, NOERROR |
| Pass | Response matches |

---

### DNS-I-003 · End-to-end CNAME chain

| Field | Value |
|-------|-------|
| Setup | Records: `alias.home.arpa CNAME target.home.arpa`; `target.home.arpa A 10.0.0.1` |
| Input | Query for `alias.home.arpa` A |
| Expected | Response contains CNAME and A records; final answer `10.0.0.1` |
| Pass | RCODE NOERROR; A record present in additional/answer section |

---

### DNS-I-004 · End-to-end PTR record

| Field | Value |
|-------|-------|
| Setup | POST `{ name: "10.1.168.192.in-addr.arpa", type: "PTR", value: "nas.home.arpa." }` |
| Input | Query for `10.1.168.192.in-addr.arpa` PTR |
| Expected | Answer: `nas.home.arpa.` |
| Pass | RCODE NOERROR; PTR value matches |

---

### DNS-I-005 · End-to-end wildcard resolution

| Field | Value |
|-------|-------|
| Setup | Wildcard record `*.k8s.home.arpa A 192.168.1.200` |
| Input A | Query for `api.k8s.home.arpa` |
| Input B | Query for `web.k8s.home.arpa` |
| Input C | Query for `k8s.home.arpa` (no label prefix) |
| Expected A | `192.168.1.200` |
| Expected B | `192.168.1.200` |
| Expected C | Miss (forwarded) |
| Pass | A and B return wildcard IP; C is forwarded |

---

### DNS-I-006 · Exact record beats wildcard

| Field | Value |
|-------|-------|
| Setup | Wildcard `*.k8s.home.arpa A 192.168.1.200`; exact `api.k8s.home.arpa A 192.168.1.201` |
| Input | Query for `api.k8s.home.arpa` |
| Expected | `192.168.1.201` |
| Pass | Response IP is `192.168.1.201` |

---

### DNS-I-007 · Blocked domain returns NXDOMAIN

| Field | Value |
|-------|-------|
| Setup | Domain `ads.doubleclick.net` in active blocklist |
| Input | Query for `ads.doubleclick.net` A |
| Expected | NXDOMAIN |
| Pass | RCODE = 3 (NXDOMAIN); no answer section |

---

### DNS-I-008 · DoH upstream resolution

| Field | Value |
|-------|-------|
| Setup | Upstream configured as DoH `https://1.1.1.1/dns-query`; no local record for query domain |
| Input | Query for `github.com` A |
| Expected | NOERROR; valid IP in answer |
| Pass | Response received within 5000ms; RCODE NOERROR |

---

### DNS-I-009 · Upstream failover

**Goal:** When the primary upstream is down, the secondary responds without error to the client.

| Field | Value |
|-------|-------|
| Setup | Primary upstream: unreachable mock; secondary: valid DoH |
| Input | Query for `github.com` A |
| Expected | NOERROR from secondary |
| Pass | Response received within `2 × timeout_ms + 500ms`; RCODE NOERROR |

---

### DNS-I-010 · Split-horizon view – internal vs guest

| Field | Value |
|-------|-------|
| Setup | View `internal` (CIDR 192.168.1.0/24): record `server.home A 10.0.0.1`. View `guest` (CIDR 192.168.10.0/24): no local records |
| Input A | Query from `192.168.1.50` for `server.home` |
| Input B | Query from `192.168.10.50` for `server.home` |
| Expected A | `10.0.0.1` (local record) |
| Expected B | Forwarded upstream (no local match in guest view) |
| Pass | Both expectations met |

---

### DNS-I-011 · Config change propagates – new record resolves within 5 seconds

| Field | Value |
|-------|-------|
| Setup | DNS server running |
| Step 1 | POST new A record via API |
| Step 2 | Poll DNS every 100ms for up to 5 seconds |
| Expected | Record resolves with correct IP within 5 seconds |
| Pass | Resolution succeeds within 5000ms of API response |

---

### DNS-I-012 · Record deletion propagates – record stops resolving within 5 seconds

| Field | Value |
|-------|-------|
| Setup | A record exists and resolves |
| Step 1 | DELETE record via API |
| Step 2 | Poll DNS every 100ms for up to 5 seconds |
| Expected | Record returns NXDOMAIN or is forwarded within 5 seconds |
| Pass | No local answer returned after 5000ms |

---

## 6. Local Zone Authority – Unit Tests

### ZONE-U-001 · Zone authority trie – exact zone match

| Field | Value |
|-------|-------|
| Input | Zone `home.arpa` configured; query name `server.home.arpa` |
| Expected | `ZoneTrie::find("server.home.arpa")` returns `Some(&ZoneEntry{name: "home.arpa", ...})` |
| Pass | Trie correctly identifies name as within the zone |

---

### ZONE-U-002 · Zone authority trie – non-member query

| Field | Value |
|-------|-------|
| Input | Only `home.arpa` zone configured; query name `example.com` |
| Expected | `ZoneTrie::find("example.com")` returns `None` |
| Pass | Trie correctly identifies name is NOT in any owned zone |

---

### ZONE-U-003 · Zone authority trie – subdomain of non-owned apex

| Field | Value |
|-------|-------|
| Input | Zone `home.arpa` configured; query `sub.notmine.arpa` |
| Expected | `ZoneTrie::find` returns `None` |
| Pass | No false positive match on shared TLD suffix |

---

### ZONE-U-004 · SOA wire encoding

| Field | Value |
|-------|-------|
| Input | SOA record: mname `ns1.home.arpa`, serial `2026040101`, refresh `3600`, retry `900`, expire `604800`, minimum `60` |
| Expected | Encoded wire bytes parse back via `hickory-dns` SOA parser to identical field values |
| Pass | Round-trip encode/decode is lossless |

---

### ZONE-U-005 · SOA serial auto-increment – format and wrap

| Field | Value |
|-------|-------|
| Input A | Serial `2026040101`; increment today (2026-04-01) |
| Input B | Serial `2026040199`; increment (should overflow nn counter, bump date or increment by 1 per RFC 1982) |
| Expected A | Serial becomes `2026040102` |
| Expected B | Serial becomes `2026040200` (or `2026040201` if date bumped) |
| Pass | Serial always increases monotonically per RFC 1982 |

---

### ZONE-U-006 · NXDOMAIN response includes SOA in authority section

| Field | Value |
|-------|-------|
| Input | Zone `home.arpa` configured with SOA; query for `missing.home.arpa` (no record exists) |
| Expected | Response: RCODE `NXDOMAIN`, AA bit set, authority section contains one SOA record matching zone SOA |
| Pass | Response is RFC-2308-compliant negative response |

---

### ZONE-U-007 · AA flag set for records in owned zone

| Field | Value |
|-------|-------|
| Input | Zone `home.arpa` owns `server.home.arpa A 10.0.0.1`; query for `server.home.arpa A` |
| Expected | Response: RCODE `NOERROR`, AA bit = 1, answer = `10.0.0.1` |
| Pass | AA flag is set on authoritative positive answers |

---

### ZONE-U-008 · AA flag NOT set for forwarded/cached answers

| Field | Value |
|-------|-------|
| Input | Query for `google.com A` (not in any owned zone); upstream returns answer |
| Expected | Response: RCODE `NOERROR`, AA bit = 0 |
| Pass | Non-authoritative forwarded answers have AA=0 |

---

## 7. Local Zone Authority – Integration Tests

### ZONE-I-001 · Create zone and resolve authoritative NXDOMAIN

| Field | Value |
|-------|-------|
| Setup | DNSDave running; no zones configured |
| Step 1 | POST `/api/v1/dns/zones` to create zone `home.arpa` |
| Step 2 | Query DNS for `missing.home.arpa A` |
| Expected | NXDOMAIN, AA=1, SOA in authority section; upstream is NOT queried |
| Pass | Authoritative NXDOMAIN returned within 1 second of zone creation |

---

### ZONE-I-002 · Zone creation publishes NATS event and all nodes pick it up

| Field | Value |
|-------|-------|
| Setup | 2 `dnsdave-dns` nodes running |
| Step 1 | POST zone `home.arpa` via API |
| Step 2 | Subscribe to `DNSDAVE_ZONES` JetStream; confirm `zone.serial.updated.home.arpa` published |
| Step 3 | Query both DNS nodes for `missing.home.arpa` |
| Expected | Both nodes return NXDOMAIN with AA=1 within 2 seconds |
| Pass | Zone state propagated to all nodes via NATS |

---

### ZONE-I-003 · Record added to zone resolves with AA=1

| Field | Value |
|-------|-------|
| Setup | Zone `home.arpa` configured |
| Step 1 | POST A record `server.home.arpa → 10.0.0.1` via API |
| Step 2 | Query `server.home.arpa A` |
| Expected | Answer `10.0.0.1`, AA=1, RCODE NOERROR |
| Pass | Record resolves authoritatively within 2 seconds |

---

### ZONE-I-004 · SOA serial increments on record write

| Field | Value |
|-------|-------|
| Setup | Zone `home.arpa` with SOA serial `S0` |
| Step 1 | POST new A record in zone via API |
| Step 2 | Query `home.arpa SOA` |
| Expected | SOA serial in response > `S0` |
| Pass | Serial incremented after any write within the zone |

---

### ZONE-I-005 · AXFR returns all zone records

| Field | Value |
|-------|-------|
| Setup | Zone `home.arpa` with `allow_transfer: ["127.0.0.1/32"]`; 5 A records created |
| Step 1 | Send AXFR request over TCP from `127.0.0.1` |
| Expected | Transfer begins with SOA, contains all 5 records, ends with SOA. Record count and values match database. |
| Pass | AXFR transfer complete and correct; connection closes cleanly |

---

### ZONE-I-006 · AXFR rejected from unauthorised IP

| Field | Value |
|-------|-------|
| Setup | Zone `home.arpa` with `allow_transfer: ["192.168.1.0/24"]` |
| Step 1 | Send AXFR request from `10.0.0.1` (outside allow_transfer) |
| Expected | RCODE `REFUSED` |
| Pass | Transfer refused; no zone data leaked |

---

### ZONE-I-007 · IXFR returns only changed records since given serial

| Field | Value |
|-------|-------|
| Setup | Zone `home.arpa` with serial `S0`; add 3 records (serial becomes `S1`); add 2 more (serial `S2`) |
| Step 1 | Send IXFR request with `SOA serial = S1` |
| Expected | Transfer contains only the 2 records added after `S1`; bounded by SOA `S1` and SOA `S2` |
| Pass | IXFR content exactly matches the records changed after the given serial |

---

### ZONE-I-008 · NOTIFY sent to configured secondary on serial change

| Field | Value |
|-------|-------|
| Setup | Zone `home.arpa` with `notify: ["127.0.0.1"]`; stand up a mock DNS NOTIFY listener on localhost |
| Step 1 | POST record to zone (triggers serial increment) |
| Expected | Mock listener receives NOTIFY packet for `home.arpa` within 2 seconds; NOTIFY OPCODE = 4; zone name = `home.arpa` |
| Pass | NOTIFY packet received and parseable |

---

### ZONE-I-009 · RFC 2136 DNS UPDATE – add record

| Field | Value |
|-------|-------|
| Setup | Zone `home.arpa` with `allow_update: ["127.0.0.1/32"]` |
| Step 1 | Send DNS UPDATE packet (RFC 2136) from `127.0.0.1`: add `dynamic.home.arpa A 10.1.2.3 TTL=60` |
| Expected | UPDATE RCODE `NOERROR`; subsequent A query for `dynamic.home.arpa` returns `10.1.2.3` with AA=1 |
| Pass | RFC 2136 dynamic update accepted and resolves within 1 second |

---

### ZONE-I-010 · RFC 2136 DNS UPDATE – delete record

| Field | Value |
|-------|-------|
| Setup | Zone `home.arpa` with `allow_update: ["127.0.0.1/32"]`; record `dynamic.home.arpa A 10.1.2.3` exists |
| Step 1 | Send DNS UPDATE: delete `dynamic.home.arpa A` |
| Expected | UPDATE RCODE `NOERROR`; subsequent query returns NXDOMAIN with AA=1 |
| Pass | RFC 2136 DELETE removes record and returns authoritative NXDOMAIN |

---

### ZONE-I-011 · RFC 2136 DNS UPDATE – rejected from unauthorised IP

| Field | Value |
|-------|-------|
| Setup | Zone `home.arpa` with `allow_update: ["192.168.1.0/24"]` |
| Step 1 | Send DNS UPDATE from `10.0.0.1` (outside allow_update) |
| Expected | RCODE `REFUSED`; no change to zone data |
| Pass | Unauthorised UPDATE refused |

---

### ZONE-I-012 · Zone deletion removes zone from authority trie

| Field | Value |
|-------|-------|
| Setup | Zone `home.arpa` configured with records |
| Step 1 | DELETE `/api/v1/dns/zones/home.arpa` |
| Step 2 | Query `missing.home.arpa` |
| Expected | Query forwarded upstream (no longer authoritative); no AA flag |
| Pass | Zone removal propagates within 2 seconds; queries forwarded |

---

### ZONE-I-013 · `home.arpa` reverse zone – PTR resolves from DHCP lease

| Field | Value |
|-------|-------|
| Setup | Zone `168.192.in-addr.arpa` configured; DHCP assigns `192.168.1.50` to hostname `mylaptop` |
| Expected | PTR query `50.1.168.192.in-addr.arpa` returns `mylaptop.home.arpa.` with AA=1 |
| Pass | Reverse zone authoritatively serves PTR records created by DHCP DDNS |

---

## 8. Conditional Forwarding – Unit Tests

### FWD-U-001 · Forward zone table – longest-suffix match

| Field | Value |
|-------|-------|
| Input | Forward zones: `corp.example.com → 10.1.1.10`, `example.com → 10.2.2.20` |
| Query | `host.corp.example.com` |
| Expected | `ForwardZoneTable::find` returns `10.1.1.10` (longest match wins) |
| Pass | Correct upstream selected; `example.com` entry not returned |

---

### FWD-U-002 · Forward zone table – catch-all (`.`)

| Field | Value |
|-------|-------|
| Input | Forward zone `.` configured with upstream `1.1.1.1`; also `home.arpa` as owned zone |
| Query A | `anything.com` (not owned, not in named forward zone) |
| Query B | `server.home.arpa` (owned zone) |
| Expected A | `ForwardZoneTable::find` returns catch-all entry `1.1.1.1` |
| Expected B | Zone authority trie match takes precedence; `ForwardZoneTable::find` not reached for this name |
| Pass | Catch-all correctly routes non-owned names; owned zone is not affected |

---

### FWD-U-003 · Forward zone bypasses blocklist

| Field | Value |
|-------|-------|
| Input | Forward zone `corp.example.com → 10.1.1.10`; domain `ads.corp.example.com` in blocklist |
| Expected | Hot path exits at step 7b (forward zone branch); blocklist check at step 8–9 is never reached |
| Pass | Blocked domain resolves via forward zone (no NXDOMAIN from blocklist) |

---

## 9. Conditional Forwarding – Integration Tests

### FWD-I-001 · Forward zone created and queries routed to correct upstream

| Field | Value |
|-------|-------|
| Setup | Mock DNS server on `127.0.0.1:5300` returns `A 192.0.2.1` for `host.corp.example.com` |
| Step 1 | POST forward zone `corp.example.com → 127.0.0.1:5300` |
| Step 2 | Query `host.corp.example.com A` |
| Expected | Answer `192.0.2.1`; AA=0; global upstream NOT queried |
| Pass | Response correct within 1 second of zone creation |

---

### FWD-I-002 · Forward zone cache – second query does not hit upstream

| Field | Value |
|-------|-------|
| Setup | Forward zone configured; mock upstream has a request counter |
| Step 1 | Query `host.corp.example.com A` (upstream hit count = 1) |
| Step 2 | Query again within TTL |
| Expected | Upstream hit count still 1; answer returned from cache |
| Pass | Answer cache serves forward zone responses on cache hit |

---

### FWD-I-003 · Forward zone deletion reverts to global upstream

| Field | Value |
|-------|-------|
| Setup | Forward zone `corp.example.com → 127.0.0.1:5300`; global upstream returns NXDOMAIN for this name |
| Step 1 | DELETE forward zone |
| Step 2 | Query `host.corp.example.com A` |
| Expected | Query reaches global upstream; NXDOMAIN returned |
| Pass | Reversion propagates within 2 seconds |

---

### FWD-I-004 · Forward zone propagation to all DNS nodes

| Field | Value |
|-------|-------|
| Setup | 2 `dnsdave-dns` nodes; mock upstream on `127.0.0.1:5300` |
| Step 1 | POST forward zone via API |
| Step 2 | Query both nodes for a name in the zone |
| Expected | Both nodes forward to mock upstream and return the correct answer within 2 seconds |
| Pass | `dnsdave.config.forwardzone.created` NATS event received by both nodes |

---

### FWD-I-005 · Consul forward zone – `.consul` routes to `127.0.0.1:8600`

| Field | Value |
|-------|-------|
| Setup | Mock Consul DNS on `127.0.0.1:8600` returns `A 10.5.0.1` for `web.service.consul` |
| Step 1 | POST forward zone `consul → 127.0.0.1:8600` |
| Step 2 | Query `web.service.consul A` |
| Expected | Answer `10.5.0.1`; query routed to port 8600 |
| Pass | Single-label TLD forward zone works correctly |

---

## 10. DNSSEC – Unit Tests

### DNSSEC-U-001 · RRSIG validation – valid signature accepted

| Field | Value |
|-------|-------|
| Input | Crafted DNS response with valid RRSIG over A RRset; correct DNSKEY provided |
| Expected | `hickory_dns` DNSSEC validator returns `ValidationResult::Secure`; AD bit set in reply |
| Pass | Valid signature accepted; no SERVFAIL |

---

### DNSSEC-U-002 · RRSIG validation – tampered record rejected (strict mode)

| Field | Value |
|-------|-------|
| Input | DNS response with valid RRSIG, but A record value altered after signing |
| Config | `dnssec_validate: strict` |
| Expected | Validator returns error; response to client is SERVFAIL |
| Pass | Tampered response caught; SERVFAIL returned |

---

### DNSSEC-U-003 · RRSIG validation – failure logged but answered (opportunistic mode)

| Field | Value |
|-------|-------|
| Input | DNS response with invalid RRSIG |
| Config | `dnssec_validate: opportunistic` |
| Expected | Validation failure logged; unsigned answer returned to client; AD bit NOT set |
| Pass | Opportunistic mode does not SERVFAIL on signature failure |

---

### DNSSEC-U-004 · NSEC3 negative proof – NXDOMAIN validated

| Field | Value |
|-------|-------|
| Input | Signed zone response with NXDOMAIN + NSEC3 proof for `missing.example.com` |
| Expected | NSEC3 proof validated; negative caching honours NSEC3 minimum TTL |
| Pass | Authenticated NXDOMAIN cached and returned correctly |

---

### DNSSEC-U-005 · DNSKEY wire encoding round-trip

| Field | Value |
|-------|-------|
| Input | Ed25519 keypair generated; DNSKEY RDATA serialised |
| Expected | Deserialised DNSKEY matches original public key bytes; key tag computed deterministically |
| Pass | DNSKEY encode/decode is lossless |

---

### DNSSEC-U-006 · RRSIG generation for A record

| Field | Value |
|-------|-------|
| Input | Zone `home.arpa`, ZSK (Ed25519), A record `server.home.arpa 300 A 10.0.0.1` |
| Expected | `sign_rrset` produces RRSIG; validating with the ZSK public key returns success |
| Pass | Generated RRSIG passes self-verification |

---

### DNSSEC-U-007 · NSEC3 chain – no zone enumeration

| Field | Value |
|-------|-------|
| Input | Zone with 5 records; NSEC3 (0 iterations, empty salt) chain generated |
| Expected | Each NSEC3 record covers a hash range; no two records have adjacent plaintext names derivable without brute-force |
| Pass | NSEC3 chain is complete and does not expose plaintext names |

---

## 11. DNSSEC – Integration Tests

### DNSSEC-I-001 · DO bit set on upstream queries when validation enabled

| Field | Value |
|-------|-------|
| Setup | Capture upstream DNS traffic (mock resolver that logs requests) |
| Config | `dnssec_validate: strict` globally |
| Step 1 | Query `example.com A` |
| Expected | Upstream query contains EDNS0 with DO bit = 1 |
| Pass | DO bit confirmed set in captured upstream query |

---

### DNSSEC-I-002 · AD bit in client reply for validated response

| Field | Value |
|-------|-------|
| Setup | Upstream mock returns fully signed, valid A record for `example.com` with RRSIG and DNSKEY |
| Config | `dnssec_validate: strict` |
| Step 1 | Query `example.com A` from client |
| Expected | Response has AD=1; RCODE NOERROR |
| Pass | Client receives Authentic Data indication |

---

### DNSSEC-I-003 · SERVFAIL on upstream signature failure (strict)

| Field | Value |
|-------|-------|
| Setup | Upstream mock returns A record with deliberately invalid RRSIG |
| Config | `dnssec_validate: strict` |
| Step 1 | Query `badchain.example.com A` |
| Expected | Client receives RCODE SERVFAIL; no answer in answer section |
| Pass | Validation failure causes SERVFAIL, not a transparent pass-through |

---

### DNSSEC-I-004 · Zone signing – DNSKEY served for signed zone

| Field | Value |
|-------|-------|
| Setup | Zone `home.arpa` created; `PATCH` to enable `dnssec_signed: true`; key generated |
| Step 1 | Query `home.arpa DNSKEY` |
| Expected | Response contains at least one DNSKEY record (KSK + ZSK); AA=1 |
| Pass | DNSKEY records served from owned zone |

---

### DNSSEC-I-005 · Zone signing – RRSIG returned for A record

| Field | Value |
|-------|-------|
| Setup | Signed zone `home.arpa`; A record `server.home.arpa 300 A 10.0.0.1` |
| Step 1 | Query `server.home.arpa A` with DO bit set |
| Expected | Answer contains both A record and RRSIG record; RRSIG validated against zone DNSKEY |
| Pass | Signed response passes external DNSSEC validation tool (e.g. `delv` or `dig +dnssec`) |

---

### DNSSEC-I-006 · Zone signing – NSEC3 returned for NXDOMAIN

| Field | Value |
|-------|-------|
| Setup | Signed zone `home.arpa` with NSEC3 enabled |
| Step 1 | Query `missing.home.arpa A` with DO bit set |
| Expected | RCODE NXDOMAIN; authority section contains SOA + NSEC3 denial proof; AA=1 |
| Pass | NSEC3 proof present and validates with external tool |

---

### DNSSEC-I-007 · DS record export API

| Field | Value |
|-------|-------|
| Setup | Signed zone `home.arpa` with KSK |
| Step 1 | `GET /api/v1/dns/dnssec/zones/home.arpa/ds` |
| Expected | Response contains DS record fields: key tag, algorithm, digest type, digest (SHA-256 and SHA-384) |
| Pass | DS record matches KSK DNSKEY via independent computation |

---

### DNSSEC-I-008 · New record in signed zone – RRSIG auto-regenerated

| Field | Value |
|-------|-------|
| Setup | Signed zone `home.arpa`; existing records already signed |
| Step 1 | POST new A record `newhost.home.arpa → 10.0.0.50` |
| Step 2 | Query `newhost.home.arpa A` with DO bit set |
| Expected | Response includes valid RRSIG for new record; SOA serial incremented |
| Pass | Inline signing complete within 2 seconds of API write |

---

## 12. DHCP – Integration Tests

### DHCP-I-001 · DORA – client receives IP from pool

**Goal:** A full DISCOVER → OFFER → REQUEST → ACK sequence assigns an IP from the configured pool.

| Field | Value |
|-------|-------|
| Setup | Scope `192.168.1.0/24`, pool `192.168.1.100–200` |
| Step | Send DISCOVER from test MAC; complete DORA handshake |
| Expected | ACK contains IP in `192.168.1.100–200`; option 3 = gateway; option 6 = DNS |
| Pass | ACK received; IP in pool range; required options present |

---

### DHCP-I-002 · Static reservation – client always gets reserved IP

| Field | Value |
|-------|-------|
| Setup | Reservation: MAC `aa:bb:cc:dd:ee:ff → 192.168.1.50` |
| Step | Complete DORA from `aa:bb:cc:dd:ee:ff` three times |
| Expected | Each ACK has `yiaddr = 192.168.1.50` |
| Pass | All three ACKs return `192.168.1.50` |

---

### DHCP-I-003 · Lease renewal – T1 elapsed, client renews

| Field | Value |
|-------|-------|
| Setup | Lease time = 60s; client has active lease |
| Step | Send REQUEST (renewal) at T=31s (after T1) |
| Expected | ACK with updated `expires_at`; same IP retained |
| Pass | ACK received; IP unchanged; `expires_at` > original |

---

### DHCP-I-004 · Lease release – IP returned to pool

| Field | Value |
|-------|-------|
| Setup | Pool of 1 IP; lease assigned to MAC A |
| Step 1 | RELEASE from MAC A |
| Step 2 | DISCOVER from MAC B |
| Expected | ACK to MAC B with previously released IP |
| Pass | MAC B receives IP; `dhcp_leases` shows MAC A `released_at` set |

---

### DHCP-I-005 · Pool exhaustion – NAK on full pool

| Field | Value |
|-------|-------|
| Setup | Pool of 5 IPs; 5 active leases |
| Input | DISCOVER from new MAC |
| Expected | NAK |
| Pass | Response type = NAK; no new lease created |

---

### DHCP-I-006 · Option delivery – DNS server set to dnsdave-dns IP

| Field | Value |
|-------|-------|
| Setup | Scope configured with `dns_servers` pointing to `dnsdave-dns` container IP |
| Input | Complete DORA |
| Expected | ACK option 6 = dnsdave-dns IP |
| Pass | Option 6 in ACK matches configured DNS IP |

---

### DHCP-I-007 · Option hierarchy – reservation overrides scope

| Field | Value |
|-------|-------|
| Setup | Scope option 42 (NTP) = `[pool.ntp.org]`; reservation for MAC A has option 42 = `[192.168.1.1]` |
| Input | DORA from MAC A |
| Expected | ACK option 42 = `[192.168.1.1]` |
| Pass | Option 42 matches reservation, not scope |

---

### DHCP-I-008 · PXE – BIOS client receives pxelinux.0

| Field | Value |
|-------|-------|
| Setup | Scope PXE config: `boot_file_bios = "pxelinux.0"`, `tftp_server = "192.168.1.10"` |
| Input | DISCOVER with option 93 = `0x0000` (BIOS) |
| Expected | ACK option 67 = `"pxelinux.0"`; option 66 = `"192.168.1.10"` |
| Pass | Both options match |

---

### DHCP-I-009 · PXE – UEFI client receives bootx64.efi

| Field | Value |
|-------|-------|
| Setup | Same PXE config as DHCP-I-008; `boot_file_uefi = "bootx64.efi"` |
| Input | DISCOVER with option 93 = `0x0007` (EFI BC) |
| Expected | ACK option 67 = `"bootx64.efi"` |
| Pass | Option 67 matches UEFI bootfile |

---

### DHCP-I-010 · DHCP relay – client behind relay assigned IP from correct scope

| Field | Value |
|-------|-------|
| Setup | Scope A: `10.0.0.0/24`; Scope B: `10.0.1.0/24`; relay agent IP `10.0.1.1` |
| Input | DISCOVER relayed with `giaddr = 10.0.1.1` |
| Expected | ACK with IP from Scope B (`10.0.1.x`) |
| Pass | `yiaddr` in Scope B range |

---

### DHCP-I-011 · Option 82 logged with lease

| Field | Value |
|-------|-------|
| Setup | Relay agent sends DISCOVER with option 82 `circuit-id = "rack-3-port-12"` |
| Step | Complete DORA |
| Expected | `dhcp_leases.agent_circuit_id = "rack-3-port-12"` |
| Pass | DB record contains correct circuit ID |

---

### DHCP-I-012 · DHCPv6 SARR – client receives IA_NA address

| Field | Value |
|-------|-------|
| Setup | DHCPv6 scope: prefix `fd00::/64`, pool `fd00::100 – fd00::200` |
| Step | Complete SOLICIT → ADVERTISE → REQUEST → REPLY |
| Expected | REPLY contains IA_NA with address in pool range; option 23 = DNS IPv6 |
| Pass | Address in pool range; option 23 present |

---

### DHCP-I-013 · Client class – vendor class match delivers class options

| Field | Value |
|-------|-------|
| Setup | Class `PXEClient` matches `option[60].text.startswith("PXEClient")`; class option 67 = `"pxelinux.0"` |
| Input | DISCOVER with option 60 = `"PXEClient:Arch:00000:UNDI:002001"` |
| Expected | ACK option 67 = `"pxelinux.0"` |
| Pass | Class option delivered |

---

## 13. Dynamic DNS (DHCP→DNS) – Integration Tests

### DDNS-I-001 · Lease assigned → A record created automatically

| Field | Value |
|-------|-------|
| Setup | DORA from MAC `aa:bb:cc:dd:ee:ff`; hostname `myhost`; scope domain `home.arpa` |
| Expected | Within 5 seconds: DNS query for `myhost.home.arpa` A returns assigned IP |
| Pass | A record resolves within 5000ms of DHCPACK |

---

### DDNS-I-002 · Lease assigned → PTR record created automatically

| Field | Value |
|-------|-------|
| Setup | Same as DDNS-I-001; assigned IP = `192.168.1.105` |
| Expected | Within 5 seconds: DNS query for `105.1.168.192.in-addr.arpa` PTR returns `myhost.home.arpa.` |
| Pass | PTR resolves within 5000ms |

---

### DDNS-I-003 · DNS TTL matches lease time

| Field | Value |
|-------|-------|
| Setup | Lease time = 3600s |
| Expected | A record TTL ≤ 3600 (within 10%) |
| Pass | `3240 <= TTL <= 3600` |

---

### DDNS-I-004 · Lease released → A and PTR records removed within 5 seconds

| Field | Value |
|-------|-------|
| Setup | Active lease; DNS records exist for hostname |
| Step | DHCPRELEASE from client |
| Expected | Within 5 seconds: A and PTR queries return NXDOMAIN or are forwarded upstream |
| Pass | Both records gone within 5000ms of RELEASE |

---

### DDNS-I-005 · Lease expired → records removed

| Field | Value |
|-------|-------|
| Setup | Lease time = 5s (test override); DNS records exist |
| Step | Wait for lease to expire (no renewal) |
| Expected | Within 10 seconds of expiry: A and PTR records removed |
| Pass | Records gone within 10000ms of `expires_at` |

---

### DDNS-I-006 · Hostname collision – new IP updates existing record

| Field | Value |
|-------|-------|
| Setup | `myhost.home.arpa A 192.168.1.100` from prior lease |
| Step | New DORA from same MAC, new IP `192.168.1.101` assigned |
| Expected | DNS resolves `myhost.home.arpa` to `192.168.1.101`; old PTR for `100` removed; new PTR for `101` created |
| Pass | A record updated; old PTR removed; new PTR present; all within 5s |

---

### DDNS-I-007 · DHCPv6 lease → AAAA and PTR6 records created

| Field | Value |
|-------|-------|
| Setup | DHCPv6 SARR; assigned `fd00::105`; hostname `myhost6`; domain `home.arpa` |
| Expected | `myhost6.home.arpa AAAA fd00::105` and corresponding `ip6.arpa` PTR within 5s |
| Pass | Both records resolve correctly |

---

### DDNS-I-008 · Dual-stack – A and AAAA for same hostname

| Field | Value |
|-------|-------|
| Setup | DHCPv4 assigns `192.168.1.105` to `myhost`; DHCPv6 assigns `fd00::105` to same host |
| Expected | `myhost.home.arpa A 192.168.1.105` AND `myhost.home.arpa AAAA fd00::105` |
| Pass | Both record types present and correct |

---

### DDNS-I-009 · DNS node restart – lease records restored from JetStream replay

| Field | Value |
|-------|-------|
| Setup | Active lease; A and PTR records present in DNS |
| Step | Restart `dnsdave-dns` container |
| Expected | Within 15 seconds of restart: A and PTR records resolve again (replayed from `DNSDAVE_DHCP`) |
| Pass | Records resolve within 15000ms of container start |

---

## 14. API – Integration Tests

### API-I-001 · Authentication – missing API key returns 401

| Field | Value |
|-------|-------|
| Input | `GET /api/v1/dns/records` with no `X-API-Key` header |
| Expected | HTTP 401; RFC 9457 Problem Details body |
| Pass | Status code 401; `Content-Type: application/problem+json` |

---

### API-I-002 · Authentication – invalid API key returns 401

| Field | Value |
|-------|-------|
| Input | `GET /api/v1/dns/records` with `X-API-Key: invalid_key` |
| Expected | HTTP 401 |
| Pass | Status code 401 |

---

### API-I-003 · Authentication – valid API key returns 200

| Field | Value |
|-------|-------|
| Input | `GET /api/v1/dns/records` with valid API key |
| Expected | HTTP 200; JSON array |
| Pass | Status code 200; valid JSON |

---

### API-I-004 · Scoped key – read-only key rejected for write operations

| Field | Value |
|-------|-------|
| Setup | Read-only API key |
| Input | `POST /api/v1/dns/records` body with valid record |
| Expected | HTTP 403 |
| Pass | Status 403; no record created in DB |

---

### API-I-005 · DNS record CRUD lifecycle

| Step | Action | Expected |
|------|--------|----------|
| 1 | POST record | 201 Created; body has `id` |
| 2 | GET record by ID | 200; fields match POST body |
| 3 | PUT record (update value) | 200; updated value returned |
| 4 | DELETE record | 204 No Content |
| 5 | GET deleted record | 404 |

**Pass:** All 5 steps succeed with correct status codes.

---

### API-I-006 · Wildcard record creation and resolution

| Field | Value |
|-------|-------|
| Input | POST `{ name: "*.test.home.arpa", type: "A", value: "10.99.0.1" }` |
| Expected | 201 Created; DNS query for `foo.test.home.arpa` returns `10.99.0.1` within 5s |
| Pass | API returns 201; DNS resolves within 5000ms |

---

### API-I-007 · Blocklist CRUD and sync trigger

| Step | Action | Expected |
|------|--------|----------|
| 1 | POST list | 201 Created |
| 2 | POST `/:id/sync` | 202 Accepted |
| 3 | GET `/:id` after sync | `last_synced_at` is set; `last_count > 0` |

**Pass:** All 3 steps succeed; `last_count` > 0 after sync.

---

### API-I-008 · DHCP scope creation

| Field | Value |
|-------|-------|
| Input | POST `/api/v1/dhcp/scopes` with valid scope body |
| Expected | 201 Created; scope ID returned |
| Pass | Status 201; GET retrieves same scope |

---

### API-I-009 · DHCP reservation creation and client receives reserved IP

| Step | Action | Expected |
|------|--------|----------|
| 1 | POST reservation for MAC | 201 Created |
| 2 | DORA from reserved MAC | ACK with reserved IP |
| 3 | DELETE reservation | 204 |
| 4 | DORA from same MAC | ACK with IP from general pool |

**Pass:** Steps 1–4 all meet expected results.

---

### API-I-010 · Force-release lease via API

| Field | Value |
|-------|-------|
| Setup | Active lease exists |
| Input | `DELETE /api/v1/dhcp/leases/:id` |
| Expected | 204; lease marked released in DB; DNS A + PTR removed within 5s |
| Pass | Status 204; DB record has `released_at`; DNS records removed |

---

### API-I-011 · SSE query log stream – event appears within 500ms

| Field | Value |
|-------|-------|
| Setup | `GET /api/v1/logs/stream` connection open |
| Step | Send DNS query for any domain |
| Expected | SSE event containing that domain received within 500ms |
| Pass | Event received within 500ms; domain field matches |

---

### API-I-012 · SSE lease stream – event appears within 500ms

| Field | Value |
|-------|-------|
| Setup | `GET /api/v1/dhcp/leases/stream` connection open |
| Step | Complete DHCP DORA |
| Expected | SSE `lease.assigned` event received within 500ms |
| Pass | Event received within 500ms; `event_type = assigned`; IP matches lease |

---

### API-I-013 · Bulk record creation

| Field | Value |
|-------|-------|
| Input | PATCH `/api/v1/dns/records` with array of 100 records |
| Expected | 200; all 100 records created |
| Pass | Status 200; GET returns 100 records |

---

### API-I-014 · Pagination – correct pages returned

| Field | Value |
|-------|-------|
| Setup | 250 DNS records in DB |
| Input | `GET /api/v1/dns/records?page=2&per_page=100` |
| Expected | 100 records; correct `page`, `total_count = 250` in response metadata |
| Pass | Exactly 100 records; `total_count = 250` |

---

### API-I-015 · RFC 9457 error body on invalid input

| Field | Value |
|-------|-------|
| Input | POST `/api/v1/dns/records` with malformed JSON |
| Expected | HTTP 400; `Content-Type: application/problem+json`; body has `type`, `title`, `status` fields |
| Pass | Status 400; all three Problem Detail fields present |

---

## 15. Event Bus – Integration Tests

### BUS-I-001 · API write publishes NATS config event

| Field | Value |
|-------|-------|
| Setup | NATS subscriber on `dnsdave.config.record.created` |
| Step | POST new DNS record via API |
| Expected | Event received on NATS subject within 200ms |
| Pass | Event received; domain field matches created record |

---

### BUS-I-002 · Config event → DNS node hot-swaps record within 1 second

| Field | Value |
|-------|-------|
| Step 1 | POST new A record via API |
| Step 2 | Poll DNS query for new record every 100ms |
| Expected | DNS resolves the record within 1000ms of NATS event publish |
| Pass | Resolution latency (API publish → DNS answer) < 1000ms |

---

### BUS-I-003 · Blocklist diff event → DNS applies within 10 seconds

| Field | Value |
|-------|-------|
| Step 1 | Add a domain to an existing blocklist; trigger sync |
| Step 2 | Poll DNS query for that domain every 500ms |
| Expected | Domain blocked within 10 seconds |
| Pass | NXDOMAIN received within 10000ms of sync trigger |

---

### BUS-I-004 · NATS unavailable – DNS continues serving from in-memory state

| Field | Value |
|-------|-------|
| Step 1 | Stop NATS container |
| Step 2 | Send 1000 DNS queries (mix of blocked/clean/local) |
| Expected | All queries answered correctly; no SERVFAIL |
| Pass | Error rate = 0%; response correctness = 100% |

---

### BUS-I-005 · NATS unavailable – query events buffered in ring buffer

| Field | Value |
|-------|-------|
| Setup | NATS stopped; send 1000 DNS queries |
| Step | Restart NATS |
| Expected | Up to 65,536 buffered events flushed to NATS after reconnect |
| Pass | NATS `DNSDAVE_QUERYLOG` stream contains events from period of unavailability |

---

### BUS-I-006 · DNS node restart – full state restored from JetStream replay

| Field | Value |
|-------|-------|
| Setup | 50 local DNS records, 2 blocklists, 20 active DHCP leases; DNS serving correctly |
| Step | Restart `dnsdave-dns` container; wait for ready probe |
| Expected | All 50 records resolve; blocked domains still blocked; lease-generated records present |
| Pass | All checks pass within 15 seconds of container start |

---

### BUS-I-007 · DHCP lease event published on assignment

| Field | Value |
|-------|-------|
| Setup | NATS subscriber on `dnsdave.dhcp.lease.assigned.>` |
| Step | Complete DHCP DORA |
| Expected | Event received within 500ms of DHCPACK |
| Pass | Event received; `ip` and `mac` fields match lease |

---

### BUS-I-008 · New DNS node joins and self-bootstraps

**Goal:** A new DNS container added to the stack bootstraps entirely from NATS with no manual intervention.

| Field | Value |
|-------|-------|
| Setup | Stack running with records, blocklists, and DHCP leases |
| Step | Start a second `dnsdave-dns` container pointing to the same NATS |
| Expected | Within 15 seconds: second container resolves all records, blocks all blocked domains |
| Pass | Identical query results from both containers |

---

## 16. Performance / Load Tests

All load tests run on a **dedicated 4-core host** (not CI). Results are recorded in `benchmarks/` and gated in CI via stored baselines. A regression of >5% on any metric fails the build.

---

### PERF-L-001 · DNS throughput – blocked/cached queries >800K QPS

| Field | Value |
|-------|-------|
| Tool | `flamethrower` |
| Target | `dnsdave-dns` UDP :53 |
| Query mix | 70% blocked domains (from 5M-domain blocklist); 30% cached clean domains |
| Duration | 60 seconds |
| Concurrency | 4 workers |
| Pass criteria | Mean QPS ≥ 800,000; error rate = 0% |

---

### PERF-L-002 · DNS latency – p50/p99/p999 for blocked queries

| Field | Value |
|-------|-------|
| Tool | `dnsperf` with latency percentiles |
| Query type | Blocked domains only (Bloom filter negative path) |
| Rate | 100K QPS sustained |
| Pass criteria | p50 < 100µs; p99 < 150µs; p999 < 500µs |

---

### PERF-L-003 · DNS latency – p99 stable over 10 minutes (no GC spikes)

**Goal:** p99 does not degrade over time due to GC or memory pressure.

| Field | Value |
|-------|-------|
| Duration | 600 seconds at 200K QPS |
| Metric | p99 measured every 10 seconds |
| Pass criteria | All 60 p99 samples < 300µs; no single sample > 500µs |

---

### PERF-L-004 · Blocklist parse throughput – 1M domains <1 second

| Field | Value |
|-------|-------|
| Tool | `cargo bench` (`criterion`) |
| Benchmark | `parse_1m_domains` (domain-only format, in-memory buffer) |
| Pass criteria | Mean < 1000ms; p99 < 1500ms |

---

### PERF-L-005 · Bloom filter hot-swap – zero errors during rebuild

| Field | Value |
|-------|-------|
| Setup | 200K QPS load; 5M-domain blocklist |
| Step | Trigger full blocklist resync during load |
| Metric | Error count; QPS during swap |
| Pass criteria | Error count = 0; QPS during swap ≥ 95% of baseline |

---

### PERF-L-006 · Memory – RSS under 300MB at 5M-domain blocklist

| Field | Value |
|-------|-------|
| Setup | 5M-domain blocklist fully loaded; 50K entry answer cache |
| Metric | `RSS` via `/proc/{pid}/status` or `docker stats` |
| Pass criteria | RSS < 300 MB |

---

### PERF-L-007 · Memory – no growth over 1 hour of sustained load

**Goal:** RSS does not grow unboundedly (no memory leaks).

| Field | Value |
|-------|-------|
| Duration | 3600 seconds at 100K QPS |
| Metric | RSS sampled every 60 seconds |
| Pass criteria | RSS at t=3600s ≤ RSS at t=300s + 50MB (50MB tolerance for cache warm-up) |

---

### PERF-L-008 · DNS cold start – serving within 10 seconds

| Field | Value |
|-------|-------|
| Setup | NATS running with full 5M-domain blocklist state; `dnsdave-dns` stopped |
| Step | Start `dnsdave-dns`; poll `/ready` endpoint every 100ms |
| Metric | Time from container start to first successful DNS response |
| Pass criteria | First response within 10,000ms |

---

### PERF-L-009 · DHCP throughput – 1000 DISCOVER/s without loss

| Field | Value |
|-------|-------|
| Tool | Custom DHCP load generator |
| Rate | 1000 DISCOVER/s from unique MACs |
| Duration | 30 seconds |
| Pass criteria | NAK rate = 0% (pool large enough); offer latency p99 < 10ms |

---

### PERF-L-010 · Blocklist parser – 5M domains in under 5 seconds

| Field | Value |
|-------|-------|
| Tool | `cargo bench` |
| Benchmark | `parse_5m_domains` (realistic multi-format list set) |
| Pass criteria | Mean < 5000ms |

---

### PERF-L-011 · GC CPU overhead < 1% at 500K QPS

| Field | Value |
|-------|-------|
| Setup | Rust binary with allocator instrumentation |
| Duration | 300 seconds at 500K QPS |
| Metric | Total bytes allocated per second; heap growth rate |
| Pass criteria | Zero heap allocations on hot path per query after warm-up |

---

## 17. Security Tests

### SEC-001 · Unauthenticated API request – 401 on every endpoint

| Field | Value |
|-------|-------|
| Input | `GET`, `POST`, `PUT`, `DELETE` on all `/api/v1/*` paths with no auth header |
| Expected | HTTP 401 on every request |
| Pass | All unauthenticated requests return 401 |

---

### SEC-002 · Read-only key cannot mutate state

| Field | Value |
|-------|-------|
| Input | POST/PUT/DELETE requests with a read-only scoped key |
| Expected | HTTP 403 on all write operations; 200 on GET |
| Pass | Write operations all return 403; GETs return 200 |

---

### SEC-003 · NATS ACL – DNS container cannot publish config events

| Field | Value |
|-------|-------|
| Setup | DNS container NKey has no publish permission on `dnsdave.config.>` |
| Step | Attempt to publish to `dnsdave.config.record.created` using DNS container credentials |
| Expected | NATS returns permission denied error |
| Pass | Publish fails; event not received by subscribers |

---

### SEC-004 · NATS ACL – DHCP container cannot publish query events

| Field | Value |
|-------|-------|
| Setup | DHCP container NKey has no publish permission on `dnsdave.query.*` |
| Step | Attempt to publish to `dnsdave.query.default` using DHCP credentials |
| Expected | NATS returns permission denied |
| Pass | Publish fails |

---

### SEC-005 · DHCP option injection – undeclared raw custom code rejected

| Field | Value |
|-------|-------|
| Input | POST DHCP option with `code = 200` (not declared as custom) and `value = "<script>"` |
| Expected | HTTP 400; option not stored |
| Pass | Status 400; `dhcp_options` table unchanged |

---

### SEC-006 · DHCP rate limiting – excessive DISCOVERs from one MAC throttled

| Field | Value |
|-------|-------|
| Setup | Rate limit: 10 DISCOVER/s per MAC |
| Input | 100 DISCOVER/s from single MAC |
| Expected | Excess DISCOVERs silently dropped; no pool exhaustion |
| Pass | Pool not exhausted; `dnsdave_dhcp_nak_total{reason="rate_limited"}` counter increments |

---

### SEC-007 · API key stored hashed – plaintext not recoverable from DB

| Field | Value |
|-------|-------|
| Setup | API key created via API |
| Input | Direct DB query: `SELECT * FROM api_keys` |
| Expected | Only hashed value present; no plaintext |
| Pass | Raw query does not return plaintext key; hash format is Argon2id |

---

### SEC-008 · Self-signed cert generated on first boot

| Field | Value |
|-------|-------|
| Setup | Fresh deployment, no cert configured, no `DNSDAVE_ACME_DOMAIN` set |
| Step 1 | Start `dnsdave-api` |
| Expected | TLS cert present in `tls_certificates` table; `source = 'self_signed'`; API reachable over HTTPS; health endpoint reports `cert_source: self_signed` warning |
| Pass | HTTPS available within 10 seconds of startup without manual cert config |

---

### SEC-009 · ACME DNS-01 certificate issuance (staging CA)

| Field | Value |
|-------|-------|
| Setup | Zone `test.home.arpa` configured as primary; ACME staging CA URL configured; `DNSDAVE_ACME_DOMAIN=api.test.home.arpa` |
| Step 1 | Trigger ACME flow via API |
| Expected | `_acme-challenge.api.test.home.arpa TXT` record appears and is removed after validation; cert issued and stored in `tls_certificates`; `dnsdave.config.tls.renewed` NATS event published |
| Pass | Cert issued by staging CA; `source = 'acme'`; TXT record not present after issuance |

---

### SEC-010 · TLS cert hot-reload – no restart required

| Field | Value |
|-------|-------|
| Setup | HTTPS API running with certificate A |
| Step 1 | Insert a new cert (certificate B) into `tls_certificates` and publish `dnsdave.config.tls.renewed` |
| Step 2 | Open new TLS connection to API |
| Expected | New connection uses certificate B; old connections continue unaffected; no service restart |
| Pass | Certificate rotated within 5 seconds; zero dropped requests during rotation |

---

### SEC-011 · NATS connection requires TLS – plaintext rejected

| Field | Value |
|-------|-------|
| Setup | NATS server configured with TLS-only (`--no_tls=false`) |
| Input | Attempt to connect to NATS port 4222 without TLS |
| Expected | Connection refused or TLS handshake failure; no data exchanged |
| Pass | Plaintext NATS connection rejected |

---

### SEC-012 · ACME DNS-01 helper – certbot RFC 2136 places TXT record

| Field | Value |
|-------|-------|
| Setup | Zone `home.arpa` with `allow_update: ["127.0.0.1/32"]`; simulate certbot DNS-01 challenge |
| Step 1 | Send RFC 2136 UPDATE to add `_acme-challenge.svc.home.arpa TXT "token123" TTL=60` |
| Step 2 | Query `_acme-challenge.svc.home.arpa TXT` |
| Expected | TXT record resolves with value `token123`; AA=1 |
| Step 3 | Send RFC 2136 UPDATE to delete the TXT record |
| Expected | Record removed; subsequent query returns NXDOMAIN with AA=1 |
| Pass | Full ACME DNS-01 challenge cycle succeeds |

---

## 18. Pi-hole Compatibility Tests

### COMPAT-001 · Summary endpoint returns Pi-hole v5 schema

| Field | Value |
|-------|-------|
| Input | `GET /api/pihole/admin/api.php?summary` |
| Expected | JSON with keys: `domains_being_blocked`, `dns_queries_today`, `ads_blocked_today`, `ads_percentage_today`, `unique_domains`, `queries_forwarded`, `queries_cached`, `clients_ever_seen`, `unique_clients`, `status` |
| Pass | All 10 keys present; numeric fields are integers or floats |

---

### COMPAT-002 · Blocking status toggle

| Step | Input | Expected |
|------|-------|----------|
| 1 | `GET /api/pihole/admin/api.php?disable` | `status: "disabled"` |
| 2 | `GET /api/pihole/admin/api.php?enable` | `status: "enabled"` |

**Pass:** Both responses match expected `status` values.

---

### COMPAT-003 · Blacklist add via Pi-hole API

| Field | Value |
|-------|-------|
| Input | `POST /api/pihole/admin/api.php?blacklist` body: `list=newblock.example.com` |
| Expected | Domain added to blocklist; DNS query for `newblock.example.com` returns NXDOMAIN |
| Pass | DNS blocks domain within 5s of API call |

---

### COMPAT-004 · Whitelist add via Pi-hole API

| Field | Value |
|-------|-------|
| Setup | `newblock.example.com` is blocked |
| Input | `POST /api/pihole/admin/api.php?whitelist` body: `list=newblock.example.com` |
| Expected | Domain allowlisted; DNS query returns NOERROR |
| Pass | DNS does not block domain within 5s of API call |

---

### COMPAT-005 · Recent queries endpoint

| Field | Value |
|-------|-------|
| Input | `GET /api/pihole/admin/api.php?recentBlocked` |
| Expected | JSON array of recently blocked domains; each entry is a valid domain string |
| Pass | Array returned; non-empty after some blocked queries; format matches Pi-hole v5 |

---

### COMPAT-006 · Top blocked domains endpoint

| Field | Value |
|-------|-------|
| Input | `GET /api/pihole/admin/api.php?topItems` |
| Expected | JSON with `top_queries` and `top_ads` objects; keys are domain strings, values are integer counts |
| Pass | Both objects present; values are integers |

---

## Appendix A – Test Environment Requirements

| Component | Minimum spec |
|-----------|-------------|
| Unit tests (`cargo test`) | Any dev machine; no containers |
| Integration tests | Docker Compose; 4 GB RAM; 2 CPUs |
| Load tests | Dedicated bare-metal or VM; 4 physical cores; 8 GB RAM; 10 Gbps loopback |

**Integration test stack** (`docker-compose.test.yml`):
- `nats:alpine` with JetStream
- `postgres:16-alpine`
- All `dnsdave-*` containers built from current branch
- DHCP test harness container (custom; sends RFC-compliant DHCP packets)
- DNS query generator (dig, dnspython, or dnsperf)

---

## Appendix B – Milestone Test Gates

| Milestone | Must pass before merge |
|-----------|----------------------|
| v0.1 | DNS-U-001 through 016; BL-U-001 through 008; DNS-I-001 through 012; BL-I-001, 003, 006; BUS-I-001 through 003; API-I-001 through 007 |
| v0.2 | All v0.1 gates + BL-I-001 through 006; BUS-I-004 through 006; COMPAT-001 through 006 |
| v0.3 | All v0.2 gates + DNS-I-005 through 012; ZONE-U-001 through 008; ZONE-I-001 through 013; FWD-U-001 through 003; FWD-I-001 through 005 |
| v0.4 | All v0.3 gates + DNSSEC-U-001 through 004; DNSSEC-I-001 through 003; PERF-L-001 through 006; SEC-001 through 012 |
| v0.4.5 | All v0.4 gates + DNSSEC-U-005 through 007; DNSSEC-I-004 through 008 |
| v0.5 (DHCP) | All v0.4 gates + DHCP-U-001 through 013; DHCP-I-001 through 013; DDNS-I-001 through 009; API-I-008 through 014; BUS-I-007, 008 |
| v0.6 (HA) | All v0.5 gates + PERF-L-007 through 011; BUS-I-006 through 008 |

---

## Appendix C – Regression Policy

- Any test that was previously passing and now fails blocks the PR.
- Load test regressions of >5% on any metric block the PR.
- New features must add at least one unit test and one integration test before the PR can be merged.
- Load test baselines are stored in `benchmarks/baseline.json` and updated only on tagged releases.
