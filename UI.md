# DNSDave UI – Design Specification

**Version:** 0.1.0-draft  
**Date:** 2026-04-04  
**Status:** Draft  
**Tracks:** DESIGN.md v0.4.0 · Container: `dnsdave-ui`

This document specifies the design, behaviour, and implementation approach for the DNSDave web user interface. The UI is an optional container that wraps the existing REST API and SSE streams; every action it performs is achievable via the API alone.

---

## 1. Overview

DNSDave requires a management UI that can:

- Provide a **real-time view** of DNS queries, DHCP lease events, and system health.
- **Manage all configuration** objects (records, zones, blocklists, DHCP scopes, clients, certificates) with full CRUD.
- Display **cluster-wide status** when multiple `dnsdave-dns` nodes are running.
- Be **usable on any device** – from a wall-mounted monitoring display (1080p) to a phone in your pocket.
- Feel fast: **no full-page reloads**, optimistic UI updates, SSE-driven live data.

---

## 2. Design Philosophy

| Principle | Implementation |
|-----------|---------------|
| **Data density without clutter** | Tables over cards for list views; cards reserved for status summaries. Compact row heights (36px) with comfortable 48px option. |
| **Live by default** | Every list view that has an SSE stream auto-subscribes. A pause button stops the stream, not the page. |
| **Progressive disclosure** | Top-level pages show summaries. Detail panels slide in without navigation. Bulk actions revealed on selection. |
| **Dark-first** | Designed for dark mode; light mode is a full-quality inversion. Network operations happen at all hours. |
| **Zero-to-dashboard in 30 seconds** | First-run wizard detects the API URL, tests connectivity, and redirects to the dashboard. No config files. |
| **Cluster-aware** | Every status indicator, chart, and counter supports aggregation or per-node breakdown. |

---

## 3. Technology Stack

| Layer | Choice | Rationale |
|-------|--------|-----------|
| Framework | **SvelteKit 2** | First-class SSE/streaming support via `load` functions; zero-runtime reactivity; small bundles; no VDOM overhead on live-updating tables |
| Styling | **Tailwind CSS v4** | Utility-first; CSS variables for theming; dark/light via `prefers-color-scheme` + manual toggle |
| Components | **shadcn-svelte** | Headless accessible primitives (Dialog, Dropdown, Toast, Sheet, Tabs); fully styled via Tailwind |
| Charts | **LayerChart** (Svelte) + custom SVG | Reactive, SSR-compatible; custom sparklines and pool gauges in raw SVG for zero-dependency hot paths |
| Icons | **Lucide Svelte** | 1500+ consistent stroke icons; tree-shakeable |
| Tables | **svelte-virtual-list** | Virtualised rendering for query logs with >100K rows; fixed row heights |
| Forms | **Superforms** + **Zod** | Type-safe form validation mirroring backend Zod schemas; progressive enhancement |
| State | **Svelte stores** + **SvelteKit $page** | No external state manager; SSE-driven stores push into components directly |
| HTTP client | **ofetch** | Lightweight `fetch` wrapper; auto JSON; interceptors for auth headers and error normalisation |
| Time formatting | **date-fns** | Locale-aware, tree-shakeable; no moment.js |
| Container | **node:22-alpine** (SSR) or static export | Docker Compose: SSR for dynamic API proxy; Kubernetes: static export behind Nginx |

---

## 4. Design System

### 4.1 Color Palette

DNSDave uses a **slate-navy dark theme**. All colors are CSS custom properties, enabling runtime theme switching.

```css
/* Dark theme (default) */
--bg-base:       #0c0e16;   /* page background */
--bg-surface:    #131621;   /* card / panel background */
--bg-elevated:   #1c2032;   /* dropdown, popover, modal */
--bg-muted:      #252a3d;   /* table stripe, hover */
--border:        #2d3352;   /* dividers, card borders */
--border-focus:  #3b82f6;   /* focus ring */

/* Text */
--text-primary:  #e8ecf5;
--text-secondary:#8e96b0;
--text-muted:    #5a637a;
--text-mono:     #cbd5e1;   /* DNS records, log rows */

/* Semantic accents */
--dns:           #3b82f6;   /* blue-500  – DNS records, zones */
--allowed:       #10b981;   /* emerald-500 – pass / NOERROR */
--blocked:       #ef4444;   /* red-500   – NXDOMAIN from blocklist */
--cached:        #06b6d4;   /* cyan-500  – cache hit */
--local:         #8b5cf6;   /* violet-500 – local record / zone */
--dhcp:          #f59e0b;   /* amber-500 – DHCP events */
--forward:       #6366f1;   /* indigo-500 – forward zone */
--warning:       #f59e0b;
--danger:        #ef4444;
--success:       #10b981;

/* Light theme overrides */
--bg-base:       #f8fafc;
--bg-surface:    #ffffff;
--bg-elevated:   #f1f5f9;
--bg-muted:      #e2e8f0;
--border:        #cbd5e1;
--text-primary:  #0f172a;
--text-secondary:#475569;
--text-muted:    #94a3b8;
```

**Query result row colouring** (left border + faint row tint):

| Result | Colour token | Example |
|--------|-------------|---------|
| Allowed / NOERROR (upstream) | `--allowed` (green) | |
| Blocked (blocklist) | `--blocked` (red) | |
| Cache hit | `--cached` (cyan) | |
| Local record / zone | `--local` (violet) | |
| Forward zone | `--forward` (indigo) | |
| DHCP-synthesised | `--dhcp` (amber) | |
| SERVFAIL / error | `--danger` (red, solid) | |

### 4.2 Typography

```css
--font-sans: 'Inter Variable', 'Inter', system-ui, sans-serif;
--font-mono: 'JetBrains Mono Variable', 'JetBrains Mono', monospace;

/* Scale */
--text-xs:   0.75rem  / 1rem;    /* meta, badges */
--text-sm:   0.875rem / 1.25rem; /* table rows, labels */
--text-base: 1rem     / 1.5rem;  /* body, form fields */
--text-lg:   1.125rem / 1.75rem; /* section headings */
--text-xl:   1.25rem  / 1.75rem; /* page titles */
--text-2xl:  1.5rem   / 2rem;    /* dashboard stat numbers */
--text-4xl:  2.25rem  / 2.5rem;  /* hero counters */
```

Monospace font used for: domain names, IP addresses, MAC addresses, query log rows, DNS record values, certificates, keys, NATS subjects.

### 4.3 Spacing & Layout Grid

- Base unit: `4px` (`--space-1`)
- Page content max-width: `1440px`, centred
- Content padding: `24px` desktop / `16px` tablet / `12px` mobile
- Sidebar width: `240px` expanded / `56px` icon-only / `0` mobile (slide-over)
- Top bar height: `56px`
- Card border-radius: `8px`
- Input height: `36px` (compact) / `40px` (default)
- Table row height: `36px` (compact) / `48px` (comfortable); user-selectable

### 4.4 Iconography

All icons from **Lucide** unless noted. Key mappings:

| Entity | Icon |
|--------|------|
| DNS | `Globe` |
| Zone | `Map` |
| Blocklist | `ShieldOff` |
| Allowlist | `ShieldCheck` |
| Forward zone | `ArrowRightLeft` |
| DHCP | `Network` |
| Lease | `Timer` |
| Reservation | `Bookmark` |
| Certificate | `Lock` |
| DNSSEC key | `KeyRound` |
| Query log | `ScrollText` |
| Analytics | `BarChart3` |
| Cluster | `Server` |
| Settings | `Settings2` |
| Live / SSE active | `RadioTower` (pulsing) |
| Healthy | `CheckCircle2` |
| Warning | `AlertTriangle` |
| Error | `XCircle` |

### 4.5 Motion & Animation

- Transitions: `150ms ease-out` for state changes, `200ms ease-in-out` for panels
- Live feed rows: **slide-in from top** (`transform: translateY(-8px) → 0; opacity: 0 → 1; 120ms`)
- Counters: `countUp` animation on initial load and significant change (>10%)
- No decorative animations; all motion serves orientation or feedback
- `prefers-reduced-motion`: all animations disabled; transitions shortened to `1ms`

---

## 5. Application Shell

### 5.1 Global Layout

```
┌──────────────────────────────────────────────────────────────────┐
│ TOP BAR (56px)   [Logo]  [Global Search]  [Node Picker] [☾] [👤] │
├──────────┬───────────────────────────────────────────────────────┤
│          │                                                        │
│ SIDEBAR  │   PAGE CONTENT AREA                                   │
│ 240px    │   max-w-[1440px] mx-auto px-6                         │
│          │                                                        │
│ nav      │   ┌──────────────────────────────────────────────┐    │
│ items    │   │ PAGE HEADER (title + actions + breadcrumb)   │    │
│          │   ├──────────────────────────────────────────────┤    │
│          │   │                                              │    │
│          │   │   MAIN CONTENT                               │    │
│          │   │                                              │    │
│          │   └──────────────────────────────────────────────┘    │
│          │                                                        │
│ [system  │                                                        │
│  status] │                                                        │
└──────────┴───────────────────────────────────────────────────────┘
```

### 5.2 Navigation Sidebar

Sections and items:

```
  ○ OVERVIEW
    Dashboard
    Query Log    ● live
    Analytics

  ○ DNS
    Records
    Zones
    Wildcards
    Forward Zones
    Upstreams

  ○ BLOCKLIST
    Lists
    Allow / Block Rules

  ○ DHCP
    Scopes
    Leases          ● live
    Reservations
    Options
    Client Classes

  ○ CLIENTS
    Client Groups
    Client History

  ○ SECURITY
    Certificates
    DNSSEC Keys
    API Keys

  ○ INFRASTRUCTURE
    Cluster
    Event Bus
    Settings
```

- Section headers are non-clickable separators.
- Active item highlighted with `--dns` left border + faint background tint.
- `● live` badge pulses (2s animation) when SSE stream is connected.
- Collapse button at bottom toggles icon-only mode; state persisted in `localStorage`.
- **System status footer** (bottom of sidebar, always visible):
  - NATS: `● Connected` / `● Degraded` / `● Down`
  - Postgres: `● Connected` / `● Error`
  - Cert expiry: `● 87 days` / `⚠ 12 days` / `✗ Expired`

### 5.3 Top Bar

Left to right:

1. **Logo** – `dnsdave` wordmark + DNS icon; links to Dashboard
2. **Global Search** (`Ctrl+K` – opens Command Palette) – placeholder "Search records, clients, domains…"
3. **Node Picker** (visible in cluster mode) – dropdown listing all `dnsdave-dns` nodes with per-node QPS; "All nodes (aggregated)" is default
4. **Theme toggle** – sun/moon icon; cycles `system → dark → light`
5. **Notification bell** – count badge; opens notification drawer (last 50 system events)
6. **User avatar / API key** – dropdown with "Copy API key", "Manage keys", "Sign out"

### 5.4 Command Palette (`Ctrl+K`)

Full-screen overlay, fuzzy-search across:

| Category | Examples |
|----------|---------|
| Pages | "Go to DHCP Leases", "Open Analytics" |
| DNS records | "A record for server.home.arpa" |
| Zones | "Zone home.arpa" |
| DHCP leases | "Lease 192.168.1.50 (mylaptop)" |
| Clients | "Client 192.168.1.25" |
| Actions | "Add DNS record", "Sync blocklist: StevenBlack", "Revoke lease" |
| Settings | "Toggle DNSSEC validation", "Renew certificate" |

Keyboard navigation: `↑↓` to move, `Enter` to select, `Esc` to close.

### 5.5 Notification & Toast System

**Toasts** (bottom-right, stack up to 5):
- Green: success operations (record saved, lease released, cert renewed)
- Red: errors (API failure, NATS disconnected)
- Amber: warnings (cert expiry <30 days, pool >80% full)
- Auto-dismiss: 4s (success) / 8s (warning) / persistent (error)

**Notification Drawer** (right slide-over, 400px):
- Events from `DNSDAVE_QUERYLOG` and `DNSDAVE_DHCP` JetStream
- Each event shows: icon, title, description, relative time
- "Mark all read" button; per-item dismiss
- Filtered tabs: All / DNS / DHCP / System

---

## 6. Pages

### 6.1 Dashboard

**Purpose:** At-a-glance health and activity for the whole stack. The default landing page.

**Layout:**

```
┌─────────────────────────────────────────────────────────────┐
│  STAT CARDS (row of 5)                                      │
│  ┌─────────┐ ┌─────────┐ ┌─────────┐ ┌─────────┐ ┌──────┐ │
│  │ 1.24M   │ │ 18.3%   │ │ 847     │ │ 5       │ │  3   │ │
│  │Queries  │ │ Blocked │ │ Leases  │ │ Zones   │ │Nodes │ │
│  │ (24h)   │ │ (24h)   │ │ Active  │ │ Auth.   │ │  ●   │ │
│  └─────────┘ └─────────┘ └─────────┘ └─────────┘ └──────┘ │
├──────────────────────────┬──────────────────────────────────┤
│  QUERY VOLUME (24h)      │  UPSTREAM HEALTH                 │
│  [area chart]            │  [table: name, latency, status] │
│                          │                                  │
├──────────────────────────┼──────────────────────────────────┤
│  LIVE QUERY FEED         │  TOP BLOCKED DOMAINS (1h)        │
│  [last 20 rows, SSE]     │  [horizontal bar chart, top 10] │
│  [Pause] [Clear]         │                                  │
├──────────────────────────┼──────────────────────────────────┤
│  DHCP POOL UTILISATION   │  TOP CLIENTS (1h, by query count)│
│  [per-scope gauges]      │  [horizontal bar chart, top 10] │
└──────────────────────────┴──────────────────────────────────┘
```

**Stat Cards:** Each card has: large number (countUp), label, sparkline (60-minute trend), delta vs previous period (▲/▼ with %).

**Live Query Feed (mini):** 5-column abbreviated table – time (HH:mm:ss), client IP, domain (truncated to 40 chars), type, result badge. Clicking a row opens the full Query Log filtered to that client. "View all" link navigates to the Query Log page.

**Query Volume Chart:** Stacked area chart, 24 × 1-hour buckets. Stacks: Blocked (red), Cache hit (cyan), Local (violet), Upstream (green). Hover shows per-stack count + total.

**Data sources:**
- Stat cards: `GET /api/v1/stats/summary?period=24h`
- Live feed: SSE `/api/v1/dns/queries/stream`
- Query volume: `GET /api/v1/stats/queries/timeseries?bucket=1h&periods=24`
- Top blocked: `GET /api/v1/stats/domains/top?filter=blocked&limit=10&period=1h`
- Top clients: `GET /api/v1/stats/clients/top?limit=10&period=1h`
- Upstream health: `GET /api/v1/dns/upstreams`
- DHCP pools: `GET /api/v1/dhcp/scopes?include=utilisation`

---

### 6.2 Query Log

**Purpose:** Full searchable, filterable, pausable live stream of every DNS query processed.

**Layout:**

```
┌─────────────────────────────────────────────────────────────────┐
│  [● LIVE]  [⏸ Pause]  [⟳ Clear]    Showing 847 / 1,204 rows   │
│  [Search domain...]  [Client ▼]  [Type ▼]  [Result ▼]  [Time ▼]│
├──────────┬────────────────────┬──────┬────────┬──────────┬──────┤
│ Time     │ Domain             │ Type │ Result │ Src      │  ms  │
├──────────┼────────────────────┼──────┼────────┼──────────┼──────┤
│ 14:23:01 │ ads.google.com     │  A   │ BLOCK  │ blocklist│      │  ← red left border
│ 14:23:01 │ server.home.arpa   │  A   │ PASS   │ local    │   0  │  ← violet
│ 14:23:00 │ api.github.com     │  A   │ PASS   │ cache    │   0  │  ← cyan
│ 14:23:00 │ example.com        │  A   │ PASS   │ upstream │  12  │  ← green
│ 14:22:59 │ svc.corp.example   │  A   │ PASS   │ forward  │   8  │  ← indigo
└──────────┴────────────────────┴──────┴────────┴──────────┴──────┘
│ [Export CSV]  [Export JSON]                          Page 1/47 ▶ │
└─────────────────────────────────────────────────────────────────┘
```

**Full column set (configurable – drag to reorder, click eye to hide):**

| Column | Width | Notes |
|--------|-------|-------|
| Time | 90px | HH:mm:ss.SSS; hover shows full ISO timestamp |
| Client IP | 120px | Clickable → filter to client |
| Client name | 140px | Resolved from `clients` table; greyed if unknown |
| Domain | auto | Monospace; full FQDN; truncate to 2 lines |
| Type | 52px | A / AAAA / CNAME / MX / TXT / PTR … |
| Result | 80px | Coloured badge: PASS / BLOCK / NXDOMAIN / SERVFAIL |
| Source | 80px | local / cache / upstream / forward / blocklist |
| Upstream | 140px | Which upstream responded (if applicable) |
| Latency | 56px | ms; `–` for cached/local |
| DNSSEC | 48px | AD / – / FAIL |

**Filter controls:**
- **Domain search:** live regex filter (applied client-side to the in-memory buffer; server-side filter restarts the SSE subscription with `?q=` param)
- **Client dropdown:** multi-select; searches IP and name
- **Type:** multi-select checkbox list
- **Result:** checkboxes for PASS / BLOCK / NXDOMAIN / SERVFAIL / ERROR
- **Time range:** "Live" (default) / "Last 5m" / "Last 1h" / "Custom range" (date picker)

**Row detail drawer** (right slide-over, 480px, opens on row click):
- Full query metadata
- EDNS0 flags (DO, RD, CD bits)
- Response: full RRset with TTL
- RRSIG details (if DNSSEC)
- Client group assignment
- "Block this domain" / "Allow this domain" quick actions
- "View all queries from this client" link

**Performance:** Virtualised list (svelte-virtual-list); maximum in-memory buffer 10,000 rows; overflow discards oldest. Export pulls paginated API, not the buffer.

**Data source:** SSE `/api/v1/dns/queries/stream`; history `GET /api/v1/dns/queries?q=&type=&result=&client=&from=&to=&page=&per_page=100`

---

### 6.3 DNS Records

**Purpose:** Manage all DNS records (manual and DHCP-synthesised). Primary CRUD interface.

**Layout:**

```
┌──────────────────────────────────────────────────────────┐
│  DNS Records          [+ Add Record]  [↑ Import]  [▼ ⋯] │
│  Filter: [Search...] [Type ▼] [Zone ▼] [Source ▼]        │
│  3,481 records  (14 selected)  [Enable] [Disable] [Delete]│
├────┬──────────────────────────┬──────┬───────────┬───────┤
│ ☑  │ Name                     │ Type │ Value     │  TTL  │
├────┼──────────────────────────┼──────┼───────────┼───────┤
│ ☑  │ server.home.arpa         │  A   │ 10.0.0.1  │  300  │
│ ☐  │ *.apps.home.arpa         │  A   │ 10.0.0.2  │  60   │  ← wildcard badge
│ ☐  │ mail.home.arpa           │  MX  │ 10 smtp.. │  3600 │
│ ☐  │ mylaptop.home.arpa       │  A   │ 192.168.. │  60   │  ← dhcp badge
└────┴──────────────────────────┴──────┴───────────┴───────┘
```

**Inline editing:** Clicking any cell (name, value, TTL) opens an inline input. `Enter` saves; `Esc` cancels. Optimistic update shown immediately; rolled back on API error.

**Add Record form** (right slide-over):

```
Name:    [server.home.arpa                    ]
Zone:    [home.arpa ▼]  (auto-detected from name)
Type:    [A ▼]
Value:   [10.0.0.1                            ]
TTL:     [300      ]  [✓ Use zone default]
Comment: [optional                            ]
         [Cancel]  [Save Record]
```

Type-specific value helpers:
- **MX**: `priority` field appears
- **SRV**: `weight`, `port`, `target` fields
- **CAA**: `flags`, `tag`, `value` fields
- **TXT**: multi-line textarea; auto-split if >255 bytes per segment

**Import (slide-over):** Paste a BIND zone file or a CSV (`name,type,value,ttl`). Preview table shows parsed rows before commit. Conflict resolution: skip / overwrite / abort.

**Zone filter:** When a zone is selected, the table scopes to that zone and a "Zone: home.arpa" breadcrumb appears. A "View zone →" button navigates to the Zones page.

---

### 6.4 DNS Zones

**Purpose:** Manage authoritative zones, SOA records, transfer policy, and DNSSEC signing.

**Layout:** Two-column – zone list on left (280px), zone detail on right.

```
┌──────────────────┬────────────────────────────────────────────┐
│ Zones     [+ Add]│ home.arpa                        [● Primary]│
├──────────────────┤                                             │
│ ● home.arpa      │ SOA                                         │
│   6 records      │  mname:   ns1.home.arpa                     │
│   Serial 2026..  │  rname:   admin.home.arpa                   │
│                  │  Serial:  2026040302 (auto)                  │
│ ● 168.192.in-a.. │  Refresh: 3600s  Retry: 900s               │
│   PTR auto       │  Expire:  604800s  Minimum: 60s             │
│                  │                                             │
│ ○ corp.exam.. ↗  │ Transfer         Update                     │
│   (secondary)    │  192.168.0.0/16  192.168.1.0/24            │
│                  │  Notify: 192.168.1.2                        │
└──────────────────┤                                             │
                   │ DNSSEC                [Enable signing]       │
                   │  Status: Unsigned                            │
                   │                                             │
                   │ RECORDS          [+ Add] [View all →]       │
                   │  6 records – A(3) AAAA(1) MX(1) TXT(1)     │
                   │                                             │
                   │ AXFR / IXFR LOG                             │
                   │  No transfers in last 24h                   │
                   │                                             │
                   │ [Export DS Record]  [Trigger NOTIFY]        │
└───────────────────────────────────────────────────────────────┘
```

**SOA editing:** Click any SOA field to edit inline. Serial is read-only (auto-managed); a "Force increment" button bumps the serial and triggers NOTIFY.

**DNSSEC panel (when enabled):**

```
DNSSEC         [Disable signing]
 Algorithm:    Ed25519
 KSK key tag:  12345   [View DNSKEY]  [Export DS]
 ZSK key tag:  67890   [View DNSKEY]
 ZSK retires:  2026-07-01  (87 days)
 NSEC3:        0 iterations, empty salt
 Status:       ✓ All records signed
```

**Secondary zone view:** Shows primary server address, last AXFR timestamp, current serial vs primary serial (polled every 60s), "Force AXFR" button.

---

### 6.5 Blocklists

**Purpose:** Manage blocklist sources and view sync status.

**Layout:** Card grid (3 columns desktop / 2 tablet / 1 mobile) + compact list toggle.

```
┌────────────────────────────────────────────────────────────┐
│ Blocklists    [+ Add List]   [⟳ Sync All]   [List ▼ Grid] │
│ 5 lists · 3.2M domains blocked                              │
├──────────────────┬──────────────────┬──────────────────────┤
│ StevenBlack      │ OISD Big         │ Hagezi Pro           │
│ ● Enabled        │ ● Enabled        │ ○ Disabled           │
│ 163,428 domains  │ 1,012,874 domains│ 243,012 domains      │
│ Synced 3h ago    │ Synced 3h ago    │ Never synced         │
│ Next: 03:00      │ Next: 03:00      │ –                    │
│ [Sync now] [Edit]│ [Sync now] [Edit]│ [Enable] [Edit]      │
└──────────────────┴──────────────────┴──────────────────────┘
```

Each card expands to show:
- URL, format, assigned groups
- Domain count trend (sparkline over last 30 syncs)
- Last sync log (scrollable, last 20 lines of sync output)

**Sync progress:** When syncing, the card shows a progress bar, current phase (Fetching / Parsing / Diffing / Building MPHF / Uploading bloom), and domains processed.

**Allow/Block Rules** (sub-page):

```
┌────────────────────────────────────────────────────────────┐
│ Rules   [+ Add Domain]  [Search...]   [Allowlist] [Blocklist]│
├────┬──────────────────┬──────────┬────────────────────────┤
│ ✅  │ ads.example.com  │ Allowlist │ Global                 │
│ 🚫  │ telemetry.app.io │ Blocklist │ Group: work            │
└────┴──────────────────┴──────────┴────────────────────────┘
```

---

### 6.6 Forward Zones

**Purpose:** Manage conditional DNS forwarding rules.

**Layout:** Table with inline detail drawer.

```
┌──────────────────────────────────────────────────────────────┐
│ Forward Zones              [+ Add Forward Zone]              │
│ 3 zones configured                                           │
├──────────────────┬───────────────────────┬─────────┬────────┤
│ Zone             │ Upstream(s)           │ DNSSEC  │ Status │
├──────────────────┼───────────────────────┼─────────┼────────┤
│ corp.example.com │ 10.1.1.10, 10.1.1.11  │ Strict  │ ● OK   │
│ consul           │ 127.0.0.1:8600        │ Off     │ ● OK   │
│ . (catch-all)    │ 1.1.1.1, 8.8.8.8     │ Opport. │ ● OK   │
└──────────────────┴───────────────────────┴─────────┴────────┘
```

Detail drawer shows:
- Upstream list (editable; drag to reorder priority)
- Test query: type a domain and press Enter; shows which upstream was used, latency, and response
- DNSSEC policy selector (strict / opportunistic / disabled)
- Cache hit ratio for this zone (last 1h)

---

### 6.7 DHCP

#### Scopes

**Purpose:** Manage IPv4/IPv6 DHCP scopes and view pool utilisation.

```
┌──────────────────────────────────────────────────────────┐
│ DHCP Scopes    [+ Add Scope]                             │
├──────────────────────────────────────────────────────────┤
│ LAN – 192.168.1.0/24                           ● Active  │
│ Pool: 192.168.1.10 – 192.168.1.254             Router: 192.168.1.1 │
│                                                           │
│ ████████████████████████░░░░░░░░  67%  (168/250 leases)  │
│  ■ Active  ■ Reserved  □ Free                             │
│                                                           │
│  [View Leases]  [Manage Options]  [Reservations]  [Edit] │
├──────────────────────────────────────────────────────────┤
│ IOT – 192.168.10.0/24                          ● Active  │
│ ████████░░░░░░░░░░░░░░░░░░░░░░░░  24%  (60/250 leases)   │
│  [View Leases]  [Manage Options]  [Reservations]  [Edit] │
└──────────────────────────────────────────────────────────┘
```

**Pool gauge** colour: green < 70% / amber 70–90% / red > 90%.

**Options panel** (slide-over): Tree view showing global → scope → reservation option hierarchy. Each level shows which options are inherited vs overridden. Inline edit per option; type-safe inputs (e.g. IP picker for option 3, checkbox for option 46 bit flags).

#### Leases

**Purpose:** Live view of all active DHCP leases.

```
┌──────────────────────────────────────────────────────────────┐
│ Active Leases  [● LIVE]  [⏸ Pause]   848 leases              │
│ [Search IP, MAC, hostname...] [Scope ▼] [Type ▼] [Sort ▼]    │
├─────────────────┬───────────────────┬────────────┬───────────┤
│ IP Address      │ MAC / DUID        │ Hostname   │ Expires   │
├─────────────────┼───────────────────┼────────────┼───────────┤
│ 192.168.1.50   │ aa:bb:cc:dd:ee:ff │ mylaptop   │ 5h 23m    │
│ 192.168.1.51   │ 11:22:33:44:55:66 │ (unknown)  │ 11h 59m   │
│ 192.168.10.4   │ fe:dc:ba:98:76:54 │ esp32-lamp │ 2h 14m    │
└─────────────────┴───────────────────┴────────────┴───────────┘
```

Row actions (right-click or `⋯` menu): "Force Release", "Add Reservation", "View DNS records", "View query history for this client".

**Lease expiry bar:** A thin progress bar on each row shows time remaining visually (green → amber → red as expiry approaches).

**DHCP event stream** (toggle at top): switches table to a chronological event log (DISCOVER / OFFER / REQUEST / ACK / RELEASE / EXPIRE) with same colour-coding.

#### Reservations

Table of static MAC→IP reservations. Quick add: scan the Leases table for a lease and click "Reserve" to pre-fill the form.

#### Client Classes

Table of class match rules. Visual rule builder: `option 93 == 0x0006` → `[Operator ▼] [Field ▼] [Value]` row-per-condition.

---

### 6.8 Clients & Groups

**Purpose:** View per-client query history and manage group assignments for policy.

**Clients page:**

```
┌──────────────────────────────────────────────────────┐
│ Clients   [Search IP or name...]   [Group ▼]         │
│ 47 known clients                                      │
├──────────────────┬──────────────┬──────────┬─────────┤
│ Client           │ Group        │ Queries  │ Blocked │
│                  │              │ (24h)    │ rate    │
├──────────────────┼──────────────┼──────────┼─────────┤
│ 192.168.1.10     │ work         │  12,482  │  22%   │
│  (myworkpc)      │              │          │         │
│ 192.168.1.50     │ personal     │   3,901  │  18%   │
│  (mylaptop)      │              │          │         │
└──────────────────┴──────────────┴──────────┴─────────┘
```

Clicking a client opens a detail panel:
- Query volume sparkline (last 7 days)
- Top queried domains (table, 24h)
- Top blocked domains (table, 24h)
- Group assignment dropdown (inline change)
- DHCP lease info (if applicable)

**Groups page:** Table of groups. Each group shows: member count, assigned blocklists (pill badges), allowlist override count. Drag-and-drop reordering of group priority.

---

### 6.9 Analytics

**Purpose:** Historical query analysis, trend identification, and performance metrics.

**Layout:** Full-width dashboard, period selector at top (`1h / 6h / 24h / 7d / 30d / Custom`).

```
┌──────────────────────────────────────────────────────────────┐
│ Analytics          Period: [Last 24h ▼]  [↓ Export]         │
├───────────────────────────────┬──────────────────────────────┤
│ QUERY VOLUME OVER TIME        │ QUERY TYPE DISTRIBUTION      │
│ [stacked area: allow/block/   │ [donut chart: A/AAAA/MX/TXT] │
│  cache/local/forward]         │                              │
├───────────────────────────────┼──────────────────────────────┤
│ TOP BLOCKED DOMAINS           │ TOP ALLOWED DOMAINS          │
│ [horizontal bar, top 20]      │ [horizontal bar, top 20]     │
├───────────────────────────────┼──────────────────────────────┤
│ TOP CLIENTS BY QUERIES        │ UPSTREAM LATENCY PERCENTILES │
│ [horizontal bar]              │ [p50 / p95 / p99 line chart] │
├───────────────────────────────┼──────────────────────────────┤
│ BLOCK RATE OVER TIME          │ DHCP LEASE EVENTS OVER TIME  │
│ [% blocked, line chart]       │ [assign/release/expire bars] │
└───────────────────────────────┴──────────────────────────────┘
```

Each chart is interactive:
- Hover for tooltip with exact values
- Click a domain in "Top Blocked" to jump to that domain's history in the Query Log
- Click a client in "Top Clients" to open Client detail
- Chart data exports as CSV

---

### 6.10 Cluster View

**Purpose:** Monitor multiple `dnsdave-dns` nodes and the NATS/Postgres infrastructure.

```
┌──────────────────────────────────────────────────────────────┐
│ Cluster               3 DNS nodes · 3 NATS nodes · 1 PG HA  │
├──────────────────────────────────────────────────────────────┤
│ DNS NODES                                                    │
│ ┌─────────────────────┐ ┌────────────────────┐ ┌──────────┐ │
│ │ dns-node-1          │ │ dns-node-2          │ │ dns-3    │ │
│ │ ● Healthy           │ │ ● Healthy           │ │ ● OK     │ │
│ │ 142K QPS  ████▓░░░  │ │ 138K QPS  ████▓░░  │ │ 139K QPS │ │
│ │ p99: 0.4ms          │ │ p99: 0.3ms          │ │ p99: 0.4 │ │
│ │ Uptime: 14d 6h      │ │ Uptime: 14d 6h      │ │ 14d 6h   │ │
│ │ Blocklist: v1042    │ │ Blocklist: v1042    │ │ v1042    │ │
│ │ Cert: 87 days       │ │ Cert: 87 days       │ │ 87 days  │ │
│ └─────────────────────┘ └────────────────────┘ └──────────┘ │
├──────────────────────────────────────────────────────────────┤
│ NATS JETSTREAM CLUSTER                                       │
│  nats-1 ●leader  nats-2 ●follower  nats-3 ●follower         │
│  Streams: 6 · Messages: 14.2M · Storage: 2.1GB              │
│  Subject map health: ✓                                       │
├──────────────────────────────────────────────────────────────┤
│ POSTGRESQL                                                   │
│  Primary: pg-1 ●healthy  Replica: pg-2 ●streaming (0ms lag) │
│  Size: 4.2 GB · Connections: 12/100                          │
└──────────────────────────────────────────────────────────────┘
```

Each DNS node card shows a mini QPS sparkline (last 5 minutes). Clicking a node opens a detail panel with: full metrics, NATS subscription state, in-memory structure sizes (blocklist, zone trie, forward zones), and a "Drain and remove" button.

**NATS detail panel:** JetStream stream list with consumer lag per stream. Object Store bucket list with sizes.

---

### 6.11 Certificates & DNSSEC

**Certificates tab:**

```
┌────────────────────────────────────────────────────────────┐
│ TLS Certificates       [+ Request Certificate]             │
├────────────────┬───────────┬───────────┬───────────────────┤
│ Domain         │ Source    │ Expires   │ Status            │
├────────────────┼───────────┼───────────┼───────────────────┤
│ api.home.arpa  │ ACME      │ 2026-07-03│ ✓ Valid  [Renew] │
│ dns.home.arpa  │ Self-sign │ 2026-04-27│ ⚠ 23 days [Renew]│
└────────────────┴───────────┴───────────┴───────────────────┘
```

Certificate detail drawer:
- Full PEM cert chain (expandable, monospace, copy button)
- Subject / SAN / issuer fields
- Renewal history timeline
- "Download as PEM" / "Download as PKCS#12" buttons
- ACME account info (email, directory URL)

**DNSSEC Keys tab:**

```
┌────────────────────────────────────────────────────────────┐
│ DNSSEC Keys                                                │
├────────────┬───────┬──────────┬──────────────┬────────────┤
│ Zone       │ Type  │ Algorithm│ Tag   │ Retires       │
├────────────┼───────┼──────────┼───────┼───────────────┤
│ home.arpa  │ KSK   │ Ed25519  │ 12345 │ 2027-04-01    │
│ home.arpa  │ ZSK   │ Ed25519  │ 67890 │ 2026-07-01 ⚠ │
└────────────┴───────┴──────────┴───────┴───────────────┘
```

Actions: "View DNSKEY record", "Export DS record", "Initiate rollover", "Retire key".

---

### 6.12 Settings

Tab layout: `General | API Keys | Upstreams | NATS | Notifications | Danger Zone`

**General tab:**
- DNS server name, contact email
- Default TTL, negative cache TTL
- DNSSEC validation mode (global default)
- Query log retention (days slider)
- Blocklist sync schedule (cron expression builder)
- Theme preference

**API Keys tab:** Table of all keys. Create key: name, expiry (or never), permission scope (checkboxes for read/write per resource). Key is shown once on creation in a highlighted box with a copy button.

**Upstreams tab:** Ordered list of global DoH/DoT upstreams. Drag to reorder. Per-upstream: test button (shows actual resolution latency), enable/disable toggle. Add form with DoH URL or DoT hostname:port.

**NATS tab:**
- Connection URL, TLS status
- NKey identity for each container
- JetStream storage usage per stream
- "Reconnect" button (publishes test message)

**Notifications tab:** Configure which events generate notifications and toasts. Per-event type: toast (off/on), notification drawer (off/on), email (if SMTP configured).

**Danger Zone tab:** (red border section)
- Flush answer cache
- Rebuild blocklist MPHF (forces full re-parse of all lists)
- Force re-sign all DNSSEC zones
- Purge query logs older than N days
- Factory reset (requires typing `DELETE` to confirm)

---

## 7. Component Catalogue

### 7.1 `StatusBadge`

```
Props: status: 'healthy' | 'warning' | 'error' | 'offline' | 'unknown'
       label?: string (defaults to status string)
       dot?: boolean (show pulsing dot)
```

Coloured pill badge. `dot: true` adds an animated pulsing circle (3px, same colour). Used in sidebar footer, cluster node cards, upstream table.

### 7.2 `QueryRow`

```
Props: query: QueryEvent
       compact?: boolean
       onClick: () => void
```

Single row in the query log table. Left border (`4px solid`) coloured by result type. Entire row is a focus target (keyboard navigation). `compact` mode omits client name and upstream columns.

### 7.3 `SparklineChart`

```
Props: data: number[]         (up to 60 points)
       color?: string         (CSS color, defaults to --dns)
       area?: boolean         (filled area vs line only)
       height?: number        (default 32)
       tooltip?: boolean
```

Inline SVG sparkline. Used in dashboard stat cards, blocklist domain count trend, QPS per node. Zero-dependency, no canvas.

### 7.4 `PoolGauge`

```
Props: total: number
       active: number
       reserved: number
       label?: string
```

Horizontal progress bar with three segments (active / reserved / free). Colour switches at 70% and 90% total usage. Tooltip on hover shows exact counts and percentages. Used in DHCP scope cards.

### 7.5 `RecordTable`

```
Props: records: DnsRecord[]
       editable?: boolean
       selectable?: boolean
       zoneFilter?: string
       onEdit: (id, field, value) => Promise<void>
       onDelete: (ids: string[]) => Promise<void>
```

Full DNS record table with inline editing, bulk selection, and virtual scrolling. Handles optimistic updates with rollback on error.

### 7.6 `LiveFeed`

```
Props: stream: string           (SSE endpoint URL)
       maxRows?: number         (default 500)
       columns: ColumnDef[]
       rowColor: (row) => string
       onRowClick?: (row) => void
```

Reusable SSE-backed live table. Manages connection lifecycle, reconnection with exponential back-off, and pause/resume. Used for Query Log and DHCP Lease Events.

### 7.7 `NodeCard`

```
Props: node: DnsNode
       onClick?: () => void
```

DNS node status card. Contains: name, status badge, QPS sparkline, p99 latency, uptime, blocklist version, cert expiry. Used in Cluster view and Dashboard (multi-node mode).

### 7.8 `CommandPalette`

```
Props: open: boolean
       onClose: () => void
```

Full-screen overlay with debounced search. Queries multiple API endpoints in parallel (records, leases, clients). Keyboard navigable. Actions execute immediately.

### 7.9 `DetailDrawer`

```
Props: open: boolean
       title: string
       width?: number  (default 480)
       onClose: () => void
```

Right-side slide-over panel. Used universally for record detail, zone detail, client detail, cert detail, query row detail. Traps focus; closes on `Esc` or overlay click.

---

## 8. Real-time Data Layer

All SSE connections are managed by a central `LiveStore` pattern in Svelte:

```typescript
// src/lib/stores/live.ts
function createLiveStore<T>(endpoint: string, maxBuffer = 500) {
  const { subscribe, update } = writable<T[]>([]);
  let source: EventSource | null = null;
  let paused = false;

  function connect() {
    source = new EventSource(endpoint, { withCredentials: true });
    source.onmessage = (e) => {
      if (paused) return;
      const item = JSON.parse(e.data) as T;
      update(rows => [item, ...rows].slice(0, maxBuffer));
    };
    source.onerror = () => {
      source?.close();
      setTimeout(connect, exponentialBackoff());  // max 30s
    };
  }

  return { subscribe, connect, pause: () => paused = true,
           resume: () => paused = false, disconnect: () => source?.close() };
}
```

**SSE endpoints consumed by the UI:**

| Store | Endpoint | Consumer pages |
|-------|----------|---------------|
| `queryStream` | `/api/v1/dns/queries/stream` | Dashboard, Query Log |
| `dhcpLeaseStream` | `/api/v1/dhcp/leases/stream` | Dashboard, DHCP Leases |
| `systemEvents` | `/api/v1/system/events/stream` | Notification drawer, Cluster |
| `zoneEvents` | `/api/v1/dns/zones/events/stream` | Zones page (serial updates) |

**Reconnection strategy:** exponential back-off starting at 1s, cap at 30s, jitter ±500ms. While disconnected, a `● Reconnecting…` badge replaces the `● LIVE` indicator.

**Auth:** All SSE requests include `Authorization: Bearer <key>` via a server-side proxy route (`/api/stream/*`) in SvelteKit to avoid exposing the API key in browser network tabs.

---

## 9. Responsive Design

### Breakpoints

| Name | Width | Layout changes |
|------|-------|---------------|
| `mobile` | < 640px | Sidebar hidden; hamburger opens full-screen nav; single-column grids; tables scroll horizontally |
| `tablet` | 640–1024px | Sidebar icon-only (56px); two-column grids; condensed table columns |
| `desktop` | 1024–1440px | Full sidebar (240px); full layouts as designed |
| `wide` | > 1440px | Content area capped at 1440px; centred; sidebar unchanged |

### Mobile-specific adaptations

- **Navigation:** Full-screen slide-over nav triggered by hamburger (top-left); closes on item select.
- **Dashboard:** Stat cards stack 2×3; charts stack vertically; live feed shows 5 columns max (time, domain, result, type, latency).
- **Query Log:** Horizontal scroll on the table; sticky time + domain columns; other columns scroll.
- **Cluster view:** Node cards stack vertically.
- **Forms:** All slide-over drawers become full-screen sheets on mobile.
- **Add/edit actions:** Bottom action sheet instead of inline controls.

### Touch interactions

- Swipe right on sidebar to open, swipe left to close.
- Long-press on a table row opens the context menu (same as right-click).
- Pull-to-refresh on stat cards and live tables when not in SSE mode.

---

## 10. Accessibility

| Requirement | Implementation |
|-------------|---------------|
| WCAG 2.1 AA contrast | All colour pairs verified ≥ 4.5:1 (text) / 3:1 (UI). Dark and light themes both compliant. |
| Keyboard navigation | Full Tab order; all interactive elements reachable. Modal/drawer traps focus. `Esc` always closes overlays. |
| ARIA | Tables: `role="grid"`, `aria-sort`, `aria-rowcount`. Live feeds: `aria-live="polite"` with rate limiting (max 1 announcement/2s). Status badges: `role="status"`. |
| Screen readers | Sparkline charts include a `<title>` and `<desc>` summary. Gauges expose `aria-valuenow`, `aria-valuemin`, `aria-valuemax`. |
| Reduced motion | `@media (prefers-reduced-motion)` disables live-row slide-in animations and countUp; transitions set to 1ms. |
| Colour-blind safe | Result colouring uses both hue AND shape (badge text + icon) so red/green distinction is not the sole indicator. |

---

## 11. Keyboard Shortcuts

| Shortcut | Action |
|----------|--------|
| `Ctrl+K` / `⌘K` | Open Command Palette |
| `Ctrl+/` | Focus global search |
| `G D` | Go to Dashboard |
| `G Q` | Go to Query Log |
| `G Z` | Go to DNS Zones |
| `G B` | Go to Blocklists |
| `G H` | Go to DHCP Leases |
| `G C` | Go to Cluster |
| `Space` | Pause / Resume live feed (when feed has focus) |
| `Esc` | Close drawer / palette / modal |
| `N` | Add new record / scope / zone (context-dependent) |
| `?` | Show keyboard shortcut overlay |
| `Shift+↑/↓` | Select multiple table rows |
| `Delete` / `Backspace` | Delete selected rows (confirmation required) |

---

## 12. API Integration Map

Every UI page maps to specific API endpoints. This table is the source of truth for the API contract the UI depends on.

| Page / Component | Method | Endpoint | Notes |
|-----------------|--------|----------|-------|
| Dashboard – stat cards | GET | `/api/v1/stats/summary` | `?period=24h` |
| Dashboard – query volume | GET | `/api/v1/stats/queries/timeseries` | `?bucket=1h&periods=24` |
| Dashboard – top blocked | GET | `/api/v1/stats/domains/top` | `?filter=blocked&limit=10` |
| Dashboard – top clients | GET | `/api/v1/stats/clients/top` | `?limit=10` |
| Dashboard – live feed | SSE | `/api/v1/dns/queries/stream` | |
| Dashboard – upstream health | GET | `/api/v1/dns/upstreams` | |
| Dashboard – DHCP pool util | GET | `/api/v1/dhcp/scopes` | `?include=utilisation` |
| Query Log – history | GET | `/api/v1/dns/queries` | Paginated; filterable |
| Query Log – live | SSE | `/api/v1/dns/queries/stream` | `?q=&type=&result=&client=` |
| DNS Records – list | GET | `/api/v1/dns/records` | `?zone=&type=&source=&q=` |
| DNS Records – create | POST | `/api/v1/dns/records` | |
| DNS Records – update | PUT | `/api/v1/dns/records/:id` | |
| DNS Records – delete | DELETE | `/api/v1/dns/records/:id` | |
| DNS Records – bulk | POST | `/api/v1/dns/records/bulk` | Import |
| Zones – list | GET | `/api/v1/dns/zones` | |
| Zones – create | POST | `/api/v1/dns/zones` | |
| Zones – update | PUT | `/api/v1/dns/zones/:name` | |
| Zones – delete | DELETE | `/api/v1/dns/zones/:name` | |
| Zones – notify | POST | `/api/v1/dns/zones/:name/notify` | |
| Zones – DS record | GET | `/api/v1/dns/dnssec/zones/:name/ds` | |
| Blocklists – list | GET | `/api/v1/lists` | |
| Blocklists – sync | POST | `/api/v1/lists/:id/sync` | |
| Allow/Block rules | GET/POST/DELETE | `/api/v1/lists/rules` | |
| Forward Zones | GET/POST/PUT/DELETE | `/api/v1/dns/forwardzones` | |
| DHCP Scopes | GET/POST/PUT/DELETE | `/api/v1/dhcp/scopes` | |
| DHCP Leases – list | GET | `/api/v1/dhcp/leases` | |
| DHCP Leases – live | SSE | `/api/v1/dhcp/leases/stream` | |
| DHCP Leases – release | POST | `/api/v1/dhcp/leases/:id/release` | |
| DHCP Options | GET/POST/PUT/DELETE | `/api/v1/dhcp/options` | |
| DHCP Reservations | GET/POST/PUT/DELETE | `/api/v1/dhcp/reservations` | |
| Clients – list | GET | `/api/v1/clients` | |
| Clients – history | GET | `/api/v1/clients/:id/queries` | |
| Groups | GET/POST/PUT/DELETE | `/api/v1/groups` | |
| Analytics | GET | `/api/v1/stats/*` | Various |
| Cluster – nodes | GET | `/api/v1/cluster/nodes` | |
| Cluster – NATS | GET | `/api/v1/cluster/nats` | |
| System events – live | SSE | `/api/v1/system/events/stream` | |
| Certificates – list | GET | `/api/v1/dns/dnssec/keys` | |
| Certificates – renew | POST | `/api/v1/tls/certificates/:id/renew` | |
| DNSSEC keys | GET/POST/DELETE | `/api/v1/dns/dnssec/keys` | |
| Settings | GET/PATCH | `/api/v1/settings` | |
| API Keys | GET/POST/DELETE | `/api/v1/auth/keys` | |
| Cache flush | POST | `/api/v1/system/cache/flush` | Danger zone |

---

## 13. Deployment

The UI runs as a dedicated container: `dnsdave-ui`.

```yaml
# docker-compose.yml addition
  dnsdave-ui:
    image: ghcr.io/dnsdave/dnsdave-ui:latest
    container_name: dnsdave-ui
    restart: unless-stopped
    depends_on: [dnsdave-api]
    ports:
      - "3000:3000"
    environment:
      DNSDAVE_API_URL:  "http://dnsdave-api:8080"   # internal; proxied server-side
      DNSDAVE_UI_SECRET: "${DNSDAVE_UI_SECRET}"      # session signing key
      ORIGIN:            "https://dns.home.arpa"      # SvelteKit CSRF protection
    networks:
      - dnsdave
```

**API proxying:** The SvelteKit server proxies all `/api/*` and `/api/stream/*` calls to `dnsdave-api`. This means:
- The browser never sees the raw API URL – useful for split networks.
- The API key is stored server-side in a session cookie; never exposed to JS.
- SSE streams are tunnelled through the SvelteKit SSR server.

**Kubernetes:** The UI is a standard `Deployment` behind an `Ingress`. TLS terminates at the Ingress. The `dnsdave-api` service is reached via ClusterIP.

**Static export option:** `vite build --mode static` produces a fully static SPA. The browser calls the API directly; requires API key in `localStorage`. Suitable for air-gapped deployments where the API is directly reachable.

**First-run wizard:** If no API connection has been configured, the UI displays a setup page:
1. API URL field (auto-tries `http://localhost:8080` and `https://dns.home.arpa`)
2. API key field (with link to how to generate one)
3. Test connection button (shows result inline)
4. Save → redirects to Dashboard

**Build output:**
- SSR mode: `~8 MB` Docker image (node:22-alpine base)
- Static mode: `~2.5 MB` gzipped assets

---

## 14. UI Milestones

### UI-v0.1 – Core shell + Dashboard
- [ ] SvelteKit project scaffold, Tailwind, shadcn-svelte
- [ ] Application shell (sidebar, top bar, theme toggle)
- [ ] Dashboard page (stat cards, query volume chart, live feed mini, upstream health)
- [ ] First-run wizard (API URL + key setup)
- [ ] `LiveStore` SSE connection manager with reconnect + pause
- [ ] `StatusBadge`, `SparklineChart` components

### UI-v0.2 – DNS management
- [ ] Query Log page (full table, filters, detail drawer, export)
- [ ] DNS Records page (table, inline edit, bulk ops, add/import form)
- [ ] Zones page (zone list + detail, SOA editing, AXFR log)
- [ ] `RecordTable`, `DetailDrawer` components
- [ ] `CommandPalette` (Ctrl+K) with DNS record search
- [ ] Keyboard shortcuts: `G D`, `G Q`, `Space` pause

### UI-v0.3 – Blocklist + Forward Zones
- [ ] Blocklist page (card grid, sync progress, domain count sparkline)
- [ ] Allow/Block rules table
- [ ] Forward Zones page (table + test-query inline tool)
- [ ] Notification drawer (system events SSE)

### UI-v0.4 – DHCP
- [ ] DHCP Scopes page (`PoolGauge` component)
- [ ] DHCP Leases page (live table + event stream toggle)
- [ ] Reservations, Options (hierarchy tree), Client Classes
- [ ] `NodeCard` component (for scope cards)

### UI-v0.5 – Analytics + Clients + Cluster
- [ ] Analytics page (all charts, period selector, export)
- [ ] Clients page (per-client drill-down, group management)
- [ ] Cluster page (node cards, NATS stream health, Postgres HA status)
- [ ] System events SSE → notification drawer

### UI-v0.6 – Security + Settings + Polish
- [ ] Certificates page (TLS cert management, ACME renewal)
- [ ] DNSSEC Keys page (key lifecycle, DS export)
- [ ] Settings page (all tabs including Danger Zone)
- [ ] API Keys management
- [ ] Full responsive audit (mobile + tablet)
- [ ] Accessibility audit (WCAG 2.1 AA)
- [ ] Keyboard shortcut overlay (`?`)
- [ ] Light theme complete and tested

---

## Appendix A – Colour Accessibility Matrix

All foreground/background pairs used in the UI, verified at WCAG 2.1 AA:

| Use | Foreground | Background | Ratio | Result |
|-----|-----------|-----------|-------|--------|
| Body text | `#e8ecf5` | `#0c0e16` | 14.8:1 | ✓ AAA |
| Secondary text | `#8e96b0` | `#0c0e16` | 6.1:1 | ✓ AA |
| Muted text | `#5a637a` | `#0c0e16` | 3.8:1 | ✓ AA (large text) |
| Allowed badge | `#10b981` | `#131621` | 5.2:1 | ✓ AA |
| Blocked badge | `#ef4444` | `#131621` | 4.8:1 | ✓ AA |
| DNS accent | `#3b82f6` | `#131621` | 4.6:1 | ✓ AA |
| Focus ring | `#3b82f6` | `#0c0e16` | 5.1:1 | ✓ AA |
| Mono text | `#cbd5e1` | `#0f172a` | 9.7:1 | ✓ AAA |

---

## Appendix B – Page Load Targets

| Page | Target (cold, SSR) | Target (hot, client nav) |
|------|--------------------|--------------------------|
| Dashboard | < 800ms TTFB | < 100ms |
| Query Log | < 600ms TTFB | < 80ms |
| DNS Records (5000 rows) | < 700ms | < 100ms |
| DHCP Leases (1000 rows) | < 600ms | < 80ms |
| Analytics (all charts) | < 1.2s | < 200ms |

Charts and live feeds load skeleton placeholders within 100ms; data fills progressively.
