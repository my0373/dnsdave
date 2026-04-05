# DNSDave – Limitations

This document is a frank account of what DNSDave does not do, cannot do in its initial release, or does differently from competing products. It is written for anyone evaluating DNSDave against an existing tool or deciding whether to adopt it.

Limitations fall into three categories, clearly labelled throughout:

| Label | Meaning |
|-------|---------|
| **Never** | Intentional design decision; not planned at any point |
| **Not v1** | Acknowledged gap; on the roadmap for a future milestone |
| **Different** | The capability exists but works differently than you may expect |

---

## Contents

1. [Architectural Constraints](#1-architectural-constraints)
2. [DNS Protocol Coverage](#2-dns-protocol-coverage)
3. [DHCP Coverage](#3-dhcp-coverage)
4. [Security and Key Management](#4-security-and-key-management)
5. [Deployment and Infrastructure](#5-deployment-and-infrastructure)
6. [Hardware and Platform Support](#6-hardware-and-platform-support)
7. [Legacy and Enterprise Integrations](#7-legacy-and-enterprise-integrations)
8. [Compared With Pi-hole](#8-compared-with-pi-hole)
9. [Compared With AdGuard Home](#9-compared-with-adguard-home)
10. [Compared With Technitium DNS Server](#10-compared-with-technitium-dns-server)
11. [Compared With dnsmasq](#11-compared-with-dnsmasq)
12. [Compared With BIND9](#12-compared-with-bind9)
13. [Compared With PowerDNS Suite](#13-compared-with-powerdns-suite)
14. [Compared With CoreDNS](#14-compared-with-corednss)
15. [Compared With Infoblox DDI](#15-compared-with-infoblox-ddi)
16. [Maturity and Ecosystem](#16-maturity-and-ecosystem)

---

## 1. Architectural Constraints

### No Native IPAM — **Different**

DNSDave handles the **D** (DNS) and **D** (DHCP) in DDI. It does not have a built-in IPAM module: no subnet planning UI, no address utilisation dashboards, no network topology maps, and no automated network discovery engine.

This is an intentional design boundary, not an oversight. IPAM is a large, distinct problem domain; building it inside DNSDave would roughly double the project's scope and dilute focus.

**What DNSDave does instead:** the optional `dnsdave-netbox` container (v0.7) automatically pushes IPAM-relevant data to [NetBox](https://netbox.dev) 4.5+ in near-real-time via either:

- **Diode** (recommended) – NetBox Labs' gRPC ingestion agent; handles idempotent reconciliation.
- **NetBox REST API** – direct HTTP pushes for operators who prefer not to run a Diode server.

The following DNSDave objects are pushed automatically:

| DNSDave object | NetBox object pushed |
|---|---|
| DHCP scope (network + mask) | Prefix |
| DHCP pool (start – end) | IP Range |
| DHCP lease (MAC → IP → hostname) | IP Address (status=dhcp) |
| Static DHCP reservation | IP Address (status=dhcp, reserved=true) |
| DNS A/AAAA record | IP Address with `dns_name` set |
| DHCP client (optional, off by default) | Device + Interface |

The result is a continuously reconciled NetBox instance that reflects the live state of your network without manual data entry.

**What still requires NetBox directly:**
- Subnet planning and prefix utilisation visualisation.
- Network discovery (scanning, ARP/MAC table collection).
- Rack and cable management.
- IP assignment workflows that originate in NetBox and are pushed *to* DNSDave (this is planned as a future feature: NetBox webhook → DNSDave API).

**What to do if you don't use NetBox:** [phpIPAM](https://phpipam.net) is an alternative; DNSDave's REST API makes integration straightforward via its webhook backend. An HTTP webhook export from `dnsdave-export` can populate any IPAM system with a JSON receiver.

---

### Not a Full Recursive Resolver — **Never**

DNSDave is a **forwarding resolver**. It does not walk the DNS tree from root hints. When a query cannot be answered locally (from zones, cache, or the blocklist-allow list), DNSDave forwards it to a configured upstream resolver.

This means:

- DNSDave does not contact root nameservers directly.
- It cannot be used as a standalone internet-facing recursive resolver.
- It is dependent on at least one upstream resolver being reachable.

**What to do instead:** Use [Unbound](https://nlnetlabs.nl/projects/unbound/) as the upstream resolver if you want full recursion with local DNSSEC validation all the way to the root. DNSDave can forward to Unbound.

---

### Not a Public Authoritative DNS Host — **Never**

DNSDave's authoritative DNS is designed for **private network zones** (`home.arpa`, `corp.example`, `192.168.in-addr.arpa`, etc.). It is not designed to serve as a public-internet authoritative nameserver for a domain where external resolvers will query it.

Specific gaps for public hosting:

- No Anycast routing support (required for resilient public DNS).
- No built-in DDoS mitigations (rate limiting is per-client, not internet-scale).
- Zone transfer ACLs use IP-CIDR in v1, not TSIG (see §4).
- No traffic management, GeoDNS, or EDNS client subnet for split responses.

**What to do instead:** Use Cloudflare DNS, AWS Route 53, PowerDNS with dnsdist, or a dedicated authoritative hosting provider for public zones.

---

### Not a Kubernetes Cluster DNS Replacement — **Never**

DNSDave coexists with CoreDNS in Kubernetes; it does not replace it. CoreDNS handles `cluster.local` service discovery because it has deep Kubernetes API integration via the `kubernetes` plugin. DNSDave handles the **network perimeter**: ad blocking, DHCP, and local zone authority for non-cluster names.

The recommended topology is: client DNS → DNSDave → CoreDNS (for `cluster.local`) → upstream.

---

## 2. DNS Protocol Coverage

| Feature | Status | Notes |
|---------|--------|-------|
| Forward (stub) resolver | ✅ v1 | Core feature |
| Full recursive resolver | ❌ Never | See §1 |
| Authoritative server (local zones) | ✅ v0.3 | SOA, NS, A, AAAA, CNAME, MX, TXT, PTR, SRV |
| Wildcard records | ✅ v0.3 | Label trie; longest match wins |
| Split-horizon DNS (views) | **Different** | Achieved via client groups + conditional forwarding; not the BIND `view` model |
| AXFR (full zone transfer) | ✅ v0.3 | Outbound only (DNSDave as primary) |
| IXFR (incremental zone transfer) | ✅ v0.3 | From `dns_zone_changes` log |
| RFC 1996 NOTIFY | ✅ v0.3 | DNSDave notifies configured secondaries |
| RFC 2136 DNS UPDATE | ✅ v0.3 | Accepts updates from ACME, ExternalDNS, Kea |
| TSIG authentication | ❌ Not v1 | IP-CIDR `allow_transfer` / `allow_update` lists used in v1; TSIG planned post-v1 |
| Secondary zone hosting | ❌ Not v1 | DNSDave as a secondary (receiving AXFR) planned post-v1 |
| DNS-over-HTTPS (DoH) upstream | ✅ v0.3 | HTTP/2 connection pool |
| DNS-over-TLS (DoT) upstream | ✅ v0.3 | |
| DoH server (inbound) | ✅ v0.4 | ACME or self-signed cert |
| DoT server (inbound) | ✅ v0.4 | |
| DNS-over-QUIC (DoQ) | ❌ Not v1 | Planned post-v1 |
| DNSSEC validation | ✅ v0.4 | DO bit; AD bit; SERVFAIL on failure |
| DNSSEC signing (own zones) | ✅ v0.4.5 | ZSK + KSK; NSEC3; DS export |
| HSM-backed DNSSEC keys | ❌ Never | Keys are AES-GCM encrypted in Postgres; no HSM integration planned |
| mDNS (`.local`) bridge | ❌ Not v1 | Bridging multicast `.local` to unicast planned post-v1 |
| EDNS Client Subnet (ECS) | ❌ Not v1 | Useful for CDN-aware resolution; planned post-v1 |
| RPZ (Response Policy Zones) as consumer | ✅ v0.2 | Ingests RPZ-format blocklists |
| RPZ as server (serving RPZ to other resolvers) | ❌ Not v1 | Consuming RPZ vs serving it are separate features |
| Lua/scripting for custom policy | ❌ Never | Rule engine is configured via API; no embedded scripting language |
| GeoDNS / traffic steering | ❌ Never | Use dnsdist or a cloud DNS provider |
| BIND zone file import | ❌ Not v1 | Planned for v0.3 |
| BIND zone file export | ❌ Not v1 | Planned for v0.3 |

---

## 3. DHCP Coverage

| Feature | Status | Notes |
|---------|--------|-------|
| DHCPv4 | ✅ v0.6 | Full DORA state machine |
| DHCPv6 SARR + IA_NA | ✅ v0.6 | Stateful address assignment |
| DHCPv6 Prefix Delegation (IA_PD) | ❌ Not v1 | ISP-style prefix delegation for CPE routers; planned post-v1 |
| DHCP relay (RFC 3046) | ✅ v0.6 | giaddr + option 82 |
| Static reservations | ✅ v0.6 | MAC → IP binding |
| Client classes | ✅ v0.6 | Vendor class, user class |
| PXE boot | ✅ v0.6 | BIOS/UEFI auto-detection via option 93 |
| Failover / HA DHCP | **Different** | NATS leader election per broadcast domain; not RFC 2131 failover protocol |
| OMAPI / ISC DHCP config compatibility | ❌ Never | Not wire-compatible with ISC DHCP or Kea |
| Dynamic DNS (DDNS) | ✅ v0.6 | Via NATS: lease events → A + PTR records |

---

## 4. Security and Key Management

### TSIG is Not Supported in v1 — **Not v1**

TSIG (Transaction SIGnature, RFC 2845) provides keyed authentication for DNS messages – primarily zone transfers (AXFR/IXFR) and RFC 2136 dynamic updates. Without TSIG, DNSDave uses IP-CIDR access control lists (`allow_transfer`, `allow_update`).

This is adequate for most private deployments where the network boundary is trusted. It is not adequate for any deployment where zone transfer partners are reachable across an untrusted network.

TSIG is on the roadmap but not in v1.

### DNSSEC Signing Requires Postgres — **Different**

Private DNSSEC keys are stored AES-GCM encrypted in Postgres (encryption key from `DNSDAVE_SECRET_KEY` environment variable). There is no option to store keys in an HSM (Hardware Security Module), external KMS (AWS KMS, HashiCorp Vault), or on-disk key files.

For home labs and SMBs this is acceptable. For organisations with compliance requirements around cryptographic key custody (PCI-DSS, FIPS 140-2), this is a gap.

### No RBAC — **Not v1**

All API keys have the same permission level. There is no role-based access control in v1: no read-only API keys, no zone-scoped keys, no user/group model.

This means you cannot, for example, give a developer read access to the query log without also giving them the ability to modify zone records.

Scoped API keys (read-only, zone-restricted, log-only) are planned post-v1.

### No Audit Log UI — **Not v1**

Configuration changes (record additions, zone edits, blocklist modifications, etc.) are recorded as NATS events and stored in Postgres, but there is no dedicated audit log page in the UI in v1. Changes are visible via the `dnsdave-export` observability pipeline (syslog, Elasticsearch, etc.) and via the API (`GET /api/v1/audit/`), but not in a dedicated UI view.

---

## 5. Deployment and Infrastructure

### Requires NATS and Postgres — **Different from single-binary tools**

The full DNSDave stack requires:

- **NATS JetStream** – event bus (mandatory; the DNS container will not start without it)
- **PostgreSQL** – configuration and log persistence (mandatory in production)
- **Container runtime** – Docker, Podman, or Kubernetes

This is a higher barrier to entry than Pi-hole, AdGuard Home, or Technitium, which ship as a single binary or single container. A DNSDave `docker-compose.yml` starts five to seven containers.

A **minimal profile** (`docker-compose.minimal.yml`) uses SQLite instead of Postgres and omits `dnsdave-log`, `dnsdave-stats`, and `dnsdave-ui` to fit on a Raspberry Pi 3 with 1 GB RAM. In this profile, the infrastructure is lighter, but NATS is still required.

**What this means in practice:**
- First-start time is 30–60 seconds (NATS cluster formation, Postgres migrations, bloom filter load) rather than ~3 seconds for a single-binary tool.
- You need to manage Postgres backups; Pi-hole backs up its data in a single SQLite file.
- Upgrades touch multiple container images, not one binary.

### No Windows Native Support — **Never**

DNSDave containers run on Linux (and therefore work on Windows via WSL2 or Docker Desktop), but there are no native Windows binaries and no Windows Service integration. The control plane could theoretically compile on Windows (Rust is cross-platform), but this is not a supported or tested configuration.

If you need a native Windows DNS + DHCP server, Technitium DNS Server or Windows Server DNS/DHCP are better choices.

### No Installer / Package Manager Packages — **Not v1**

DNSDave ships as container images only in v1. There are no `.deb`, `.rpm`, `.apk`, or Homebrew packages, and no OS-native service installer. You must use Docker Compose, Podman Compose, or Kubernetes to deploy it.

System-level packages (installable via `apt`, `dnf`, etc.) are considered for post-v1.

---

## 6. Hardware and Platform Support

| Platform | Status | Notes |
|----------|--------|-------|
| `linux/amd64` | ✅ | Primary target; full feature set |
| `linux/arm64` | ✅ | Raspberry Pi 3+ (64-bit OS), Pi 4/5, AWS Graviton |
| `linux/arm/v7` | ✅ | Raspberry Pi 3+ (32-bit OS); SIMD disabled |
| `macOS/arm64` | ✅ | Apple Silicon; development builds; no `recvmmsg` |
| `macOS/x86_64` | ✅ | Development builds only |
| `linux/arm/v6` | ❌ Never | Raspberry Pi 1 and Zero; insufficient RAM for the bloom filter data structures; no hardware FPU for the MPHF computation |
| `linux/386` (32-bit x86) | ❌ Never | No modern server use case; not tested or built |
| `windows/amd64` | ❌ Never | See §5 |
| FreeBSD | ❌ Not v1 | Rust code may compile; containers are Linux-only; `recvmmsg` not available on FreeBSD |
| OpenWrt / embedded Linux | ❌ Never | Requires container runtime; too heavyweight for embedded targets |

**Raspberry Pi 3 (1 GB RAM) specifics:**

The minimal compose profile (`docker-compose.minimal.yml`) omits `dnsdave-log`, `dnsdave-stats`, `dnsdave-ui`, and uses SQLite. The remaining containers (dnsdave-dns, dnsdave-api, dnsdave-sync, NATS) fit within ~600 MB working set, leaving headroom for the OS. The bloom filter is limited to ~50 MB on this profile, supporting blocklists of up to ~5 million domains.

Raspberry Pi 3 is a supported but **not optimal** platform. Pi 4 (4 GB) or any amd64 machine is strongly preferred for the full stack.

---

## 7. Legacy and Enterprise Integrations

| Integration | Status | Notes |
|-------------|--------|-------|
| Syslog / rsyslog output | ✅ v0.5 | Via `dnsdave-export` container; RFC 5424 UDP/TCP |
| SNMP (MIB, traps) | ❌ Never | Prometheus metrics cover modern monitoring stacks; SNMP is not planned |
| Active Directory DNS integration | ❌ Never | AD-integrated zones use proprietary Microsoft replication; not in scope. Use a conditional forward zone pointing to AD DNS |
| LDAP / Active Directory authentication | ❌ Never | API key authentication only in v1; SSO/LDAP planned post-v1 |
| Windows DHCP failover protocol | ❌ Never | Not wire-compatible |
| ISC DHCP / Kea config import | ❌ Never | API-driven configuration only |
| BIND zone file import | ❌ Not v1 | v0.3 target |
| Solarwinds / ManageEngine integration | ❌ Not v1 | Achievable via `dnsdave-export` webhook backend |
| Cisco Network Registrar | ❌ Never | Not in scope |
| Infoblox WAPI compatibility | ❌ Never | REST API is DNSDave-native; no Infoblox shim planned |
| Pi-hole API compatibility | ✅ v0.2 | v5 shim; Pi-hole browser extensions and mobile apps work unchanged |
| Pi-hole v6 API compatibility | 🔮 v0.6 | Pi-hole v6 changed its API significantly |

---

## 8. Compared With Pi-hole

Pi-hole is DNSDave's primary inspiration. Both target home networks and SMBs with DNS-level ad blocking and a local resolver. The gaps run in both directions.

### What Pi-hole Does That DNSDave Does Not (v1)

| Pi-hole feature | DNSDave status | Detail |
|-----------------|---------------|--------|
| **Gravity list community ecosystem** | Different | DNSDave ingests the same list sources (StevenBlack, oisd, etc.) in the same formats; it does not have Pi-hole's `gravity.db` management tooling or the `pihole -g` update command |
| **One-command installation** | Different | `curl … \| bash` Pi-hole installer vs Docker Compose with multiple containers |
| **Built-in recursive resolver option** (Unbound integration) | Not v1 | Pi-hole's documented Unbound integration works out-of-the-box; DNSDave requires you to stand up your own Unbound container and configure it as an upstream |
| **Pausing ad blocking** | Different | Pi-hole's "Disable for X minutes" is a well-known feature; DNSDave has per-group policy that achieves the same effect but with a different (more explicit) workflow |
| **Teleporter (backup/restore)** | Not v1 | Pi-hole exports its entire configuration as a `.tar.gz`; DNSDave will have API-driven config export but no equivalent single-file backup in v1 |
| **FTL (the Pi-hole DNS engine)** | Different | FTL is Pi-hole's fork of dnsmasq; DNSDave's DNS engine is written in Rust and performs comparably or faster, but they are architecturally very different |
| **Community and ecosystem** | Never | Pi-hole has millions of users, a large forum, curated list repositories, integrations with every home automation platform, and years of community documentation. DNSDave is new and has none of this yet |
| **DHCP via dnsmasq** | Different | Pi-hole's DHCP is dnsmasq-based (DHCPv4 only, limited options). DNSDave's DHCP is a purpose-built DHCPv4+v6 server with full option support — this is a DNSDave advantage, not a gap |
| **Regex blocklist matching** | ✅ Both | Pi-hole supports regex-based block/allow rules; DNSDave supports the same |
| **Wildcard blocklist entries** | ✅ Both | Both support wildcard domain matching |
| **FTL query database** | Different | Pi-hole stores queries in FTL's SQLite database; DNSDave stores them in Postgres (production) or SQLite (minimal profile), with the same queryable history |
| **Group management** | ✅ Both | Pi-hole has group-based blocklist assignment; DNSDave has client groups with per-group policy |

### What DNSDave Does That Pi-hole Does Not

- Full authoritative DNS zones (SOA, NS, AXFR, RFC 2136).
- Full DHCPv6, PXE boot, DHCP relay, vendor classes.
- REST API with OpenAPI spec, scoped keys, bulk operations, and SSE streams.
- Horizontal scaling (multiple DNS nodes, NATS cluster, Postgres HA).
- Kubernetes-native deployment with Helm charts.
- DNSSEC validation and signing.
- Conditional DNS forwarding per zone.
- Pluggable observability export (syslog, Grafana, ELK, Splunk, etc.).
- TLS certificate management with ACME.

---

## 9. Compared With AdGuard Home

AdGuard Home is the closest open-source alternative to DNSDave in the "single-product ad blocker + local DNS" space.

| AdGuard feature | DNSDave status |
|-----------------|---------------|
| **DNS-over-QUIC server** (inbound DoQ) | Not v1 – planned post-v1 |
| **Single binary / single container** | Never – DNSDave requires NATS + Postgres |
| **Parental controls** (category-based blocking) | Not v1 – DNSDave's blocklist system covers the same use case via curated lists; per-category toggle is not a v1 feature |
| **DNS rewrites with wildcard** | ✅ Both – DNSDave's wildcard trie |
| **Statistics retention** | Different – AdGuard stores stats in memory (resets on restart) or BBolt; DNSDave stores in Postgres with configurable retention |
| **Encrypted DNS server** (DoH/DoT inbound) | ✅ Both (v0.4) |
| **Safe Search enforcement** | Not v1 – can be approximated by DNS rewrites for Google/Bing/etc., but not a first-class feature |
| **Commercial backing / cloud offering** | Never – DNSDave is AGPL-3.0 community software |

---

## 10. Compared With Technitium DNS Server

Technitium is the most feature-complete open-source DNS server and the closest single-product comparison to what DNSDave aims to be.

| Technitium feature | DNSDave status |
|--------------------|---------------|
| **DNSSEC signing** | Not v1 – Technitium ships with DNSSEC signing now; DNSDave targets v0.4.5 |
| **Single binary** | Never – DNSDave requires NATS + Postgres in production |
| **Windows native** | Never – see §5 |
| **.NET runtime** | N/A – DNSDave is Rust; no runtime dependency |
| **DNS-over-QUIC** | Not v1 |
| **App system / plugins** | Never – DNSDave's equivalent is the NATS subscriber model for new functionality |
| **Built-in DNS caching proxy** | ✅ Both |
| **DNS blocking with regex** | ✅ Both |
| **Authoritative zones** | ✅ Both (DNSDave v0.3) |
| **DHCP server** | ✅ Both – DNSDave has deeper DHCPv6 and PXE support |
| **API** | Both – Technitium has an HTTP API; DNSDave has a richer REST + OpenAPI spec |

---

## 11. Compared With dnsmasq

dnsmasq is the gold standard for embedded and constrained deployments. It underpins Pi-hole, OpenWrt, and many Linux distributions.

| dnsmasq feature | DNSDave status |
|-----------------|---------------|
| **Sub-1 MB memory footprint** | Never – DNSDave's minimum working set is ~200 MB (minimal profile) |
| **No external dependencies** | Never – DNSDave requires NATS |
| **Built into router firmware** | Never – requires a container runtime |
| **DHCP relay** | ✅ Both (DNSDave v0.6) |
| **TFTP server** (for PXE) | Not v1 – DNSDave delivers PXE boot DHCP options; you still need a TFTP server (or HTTP boot) for the actual firmware |
| **dnsmasq config file syntax** | Never – DNSDave is API-driven; no dnsmasq.conf compatibility |

**Positioning:** dnsmasq is not a migration target for DNSDave. If you are running dnsmasq on an OpenWrt router with 64 MB of RAM, DNSDave cannot replace it. If you are running Pi-hole (which uses dnsmasq under the hood) on a Pi 4 or a VM, DNSDave is a viable migration path.

---

## 12. Compared With BIND9

BIND9 is the reference implementation of DNS, implementing every DNS RFC with decades of production hardening.

| BIND9 feature | DNSDave status |
|---------------|---------------|
| **Full recursive resolver** | Never – see §1 |
| **TSIG** | Not v1 |
| **Views** (split-horizon) | Different – client groups + conditional forwarding approximate this; not a BIND `view` block |
| **DLZ** (Dynamic Loadable Zones) | Never – not relevant to DNSDave's architecture |
| **BIND zone file syntax** | Not v1 – import/export planned for v0.3 |
| **`rndc` management** | Never – REST API replaces rndc |
| **Full RFC coverage** | Different – DNSDave implements the RFCs required for private-network DNS; not every DNS RFC |
| **Response Rate Limiting** (RRL) | Not v1 – token-bucket rate limiting per client is in v1; RRL for amplification protection is post-v1 |
| **DNS64** | Not v1 |
| **Catalog zones** | Not v1 |
| **Commercial support** | Never – community-supported AGPL-3.0 project |
| **ACL complexity** | Different – BIND's ACL system is very powerful for public DNS; DNSDave's client group model is more appropriate for private network use |

---

## 13. Compared With PowerDNS Suite

The PowerDNS suite (Authoritative Server + Recursor + dnsdist) is the enterprise open-source DNS stack.

| PowerDNS feature | DNSDave status |
|------------------|---------------|
| **Lua scripting** for custom policy | Never – DNSDave uses the rule engine configured via API |
| **pdns-recursor** (full recursive resolver) | Never |
| **dnsdist** (DoH/DoT/DoQ frontend, DDoS mitigation) | Never – dnsdist is a separate purpose-built product; DNSDave has basic inbound DoH/DoT |
| **Multiple database backends** (MySQL, PostgreSQL, LDAP, etc.) | Different – DNSDave supports PostgreSQL and SQLite only |
| **Geographic load balancing** | Never |
| **ALIAS / ANAME record support** | Not v1 |
| **API** | Both – PowerDNS has a REST API; DNSDave's API is broader (includes DHCP, blocklists, certs, observability) |
| **Commercial support** (Open-Xchange) | Never |
| **DNSSEC maturity** | Different – PowerDNS has production-grade DNSSEC with more rollover tooling; DNSDave's DNSSEC is roadmapped for v0.4/v0.4.5 |
| **Pipe backend / remote backend** | Never – not relevant to DNSDave's architecture |
| **Zone metadata** | Different – DNSDave manages zone metadata via its REST API |

---

## 14. Compared With CoreDNS

CoreDNS is the CNCF DNS server powering Kubernetes cluster DNS.

| CoreDNS feature | DNSDave status |
|-----------------|---------------|
| **Kubernetes service discovery** (`kubernetes` plugin) | Never – this is CoreDNS's primary purpose; DNSDave does not replace it |
| **etcd backend** | Never |
| **Plug-in architecture** | Never – DNSDave extends via NATS subscribers, not Go plugins |
| **Extremely low memory footprint** | Different – CoreDNS uses ~50 MB; DNSDave minimal uses ~200 MB |
| **gRPC DNS** | Not v1 |
| **Prometheus metrics** | ✅ Both |
| **Ad blocking** | Only DNSDave – CoreDNS has no blocklist concept |
| **DHCP** | Only DNSDave |
| **DNSSEC signing** | Only DNSDave (v0.4.5) |
| **Web UI** | Only DNSDave |

**These products complement rather than compete.** The standard pattern is CoreDNS for `cluster.local`, DNSDave for everything else.

---

## 15. Compared With Infoblox DDI

Infoblox is the enterprise DDI market leader, selling hardware appliances and SaaS with pricing typically in the range of £15,000–£100,000+ per year.

### What Infoblox Has That DNSDave Does Not

| Feature | DNSDave status | Impact |
|---------|---------------|--------|
| **IPAM** – subnet tracking, utilisation, allocation, network discovery | Never | This is the largest gap. If you need the "I" in DDI, pair DNSDave with NetBox or retain Infoblox |
| **Enterprise RBAC** – granular role/permission model, delegated administration | Not v1 | Single permission level in v1; scoped keys planned post-v1 |
| **Compliance reporting** – SOC 2, PCI-DSS, HIPAA audit reports | Never | Export to SIEM via `dnsdave-export` and build reports there |
| **Threat intelligence** – RPZ feeds, DNS firewall, DGA detection | Not v1 – blocklist system covers basic DNS firewall; ML-based threat detection is not planned v1 |
| **Hardware appliances** – dedicated, hardened, guaranteed-availability hardware | Never |
| **24/7 support SLA** | Never – community support only |
| **Grid architecture** – Infoblox's proprietary HA/replication | Different – DNSDave uses NATS JetStream cluster + Postgres Patroni HA |
| **Windows IPAM integration** | Never |
| **DHCP fingerprinting** | Not v1 |
| **DNS traffic analytics** (threat-focused) | Different – DNSDave has per-client query analytics; threat-focused ML analysis is not planned v1 |
| **Reporting UI** | Different – DNSDave has operational dashboards; compliance-style reporting is out of scope |

### What DNSDave Has That Infoblox Does Not (easily)

- **Cost:** free and open source vs £15K–£100K+/yr.
- **API quality:** DNSDave's REST + OpenAPI is developer-friendly. Infoblox's WAPI is comprehensive but verbose and dated.
- **Kubernetes-native deployment:** Infoblox has a Kubernetes integration but it is not cloud-native by design.
- **Ad blocking:** Infoblox has DNS firewall but no Pi-hole-style consumer ad blocking with gravity lists.
- **Transparency:** open source; you can read, modify, and audit every line.
- **No vendor lock-in:** AGPL-3.0; your data lives in standard Postgres tables.

### Honest Assessment

If your organisation has a network team managing thousands of subnets across multiple sites with compliance requirements, Infoblox is probably the right tool. DNSDave is not trying to replace it in that context.

If your organisation is running Pi-hole (or nothing) and wants an API-first, Kubernetes-native upgrade with authoritative DNS, full DHCPv4/v6, and proper observability – without a six-figure annual licence – DNSDave is the right choice.

---

## 16. Maturity and Ecosystem

This section is the most important limitation of all.

### DNSDave is Pre-Release Software — **Currently**

At the time of writing, DNSDave does not have a single shipped line of production code. What exists is a detailed design specification. The milestones described in `DESIGN.md` represent a realistic roadmap, but software timelines are uncertain.

Pi-hole has:
- Millions of production deployments.
- A decade of community bug reports and fixes.
- Integrations with Home Assistant, Unifi, pfSense, OPNsense, and dozens of other products.
- Curated community blocklists maintained and updated continuously.
- A large forum with answers to almost every question.

DNSDave has none of this yet. Early adopters should expect:
- Breaking API changes between minor versions until v1.0.
- Bugs in edge-case DNS and DHCP behaviour that only surface in production.
- Fewer community resources when something goes wrong.
- No third-party integrations until the community builds them.

### No Third-Party Integrations — **Currently**

There are currently no DNSDave plugins, extensions, or integrations for:
- Home Assistant
- Unifi / UniFi Dream Machine
- pfSense / OPNsense
- Ansible / Terraform providers
- Prometheus exporters (DNSDave exports natively, but no third-party dashboards exist yet)
- Browser extensions
- Mobile apps (only Pi-hole compatibility shim)

These will come over time as the project matures and the community grows.

### Community Blocklists Are Inherited, Not Native — **Different**

DNSDave consumes the same blocklist sources that Pi-hole uses (StevenBlack, oisd, OISD, Hagezi, etc.). It does not have its own curated list ecosystem. The quality of blocking depends entirely on the quality of the upstream lists you configure, which is identical to Pi-hole's situation.

---

## Summary

DNSDave is deliberately scoped. The table below captures the most impactful gaps at a glance:

| Gap | Severity | Workaround / Timeline |
|-----|----------|-----------------------|
| No native IPAM | Medium – mitigated by NetBox push | `dnsdave-netbox` (v0.7) pushes leases, prefixes, IP ranges, and DNS records to NetBox 4.5+ automatically via Diode or REST API |
| Not a recursive resolver | Low for most deployments | Use Unbound upstream |
| Not suitable for public internet DNS | Medium | Use PowerDNS / cloud DNS for public zones |
| Requires NATS + Postgres | Medium | Minimal profile uses SQLite; still needs NATS |
| No TSIG | Medium for multi-site zone transfer | IP-CIDR ACLs in v1; TSIG roadmapped |
| No DNSSEC signing in v1 | Low–medium | v0.4.5 milestone |
| No RBAC in v1 | Medium for multi-team environments | Single permission level in v1 |
| No Windows support | High for Windows shops | Use Technitium DNS Server |
| No Raspberry Pi 1/Zero support | Low | Use Pi 3+ or any amd64 machine |
| Pre-release software | High | Evaluate carefully before production use |
| No ecosystem / community yet | High for fast time-to-value | Pi-hole remains the right choice for pure ad blocking with zero setup time |

---

*See also: [`COMPARISON.md`](COMPARISON.md) for a feature-by-feature table across all competitors, and [`DESIGN.md`](DESIGN.md) §14 for the full milestone roadmap.*
