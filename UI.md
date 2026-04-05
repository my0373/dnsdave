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
| NetBox integration | `Box` (NetBox logo approximation) |
| NetBox synced | `RefreshCw` |
| NetBox conflict | `AlertCircle` |
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

  ○ INTEGRATIONS
    NetBox          ✅ synced  (or ⚠ 2 conflicts / ❌ error)
```

- Section headers are non-clickable separators.
- Active item highlighted with `--dns` left border + faint background tint.
- `● live` badge pulses (2s animation) when SSE stream is connected.
- The `NetBox` sidebar item shows an inline status indicator: a green dot when healthy, an amber dot with conflict count when unresolved conflicts exist, or a red dot on error. Hovering shows a tooltip with last sync time.
- Collapse button at bottom toggles icon-only mode; state persisted in `localStorage`.
- **System status footer** (bottom of sidebar, always visible):
  - NATS: `● Connected` / `● Degraded` / `● Down`
  - Postgres: `● Connected` / `● Error`
  - Cert expiry: `● 87 days` / `⚠ 12 days` / `✗ Expired`
  - NetBox: `● Synced` / `⚠ Conflicts` / `● Not configured` (if disabled)

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

**Purpose:** Full searchable, filterable, pausable live stream of every DNS query processed. Memory footprint is explicitly budgeted and enforced so the page cannot cause Chrome tab OOM regardless of query volume or how long it is left open.

**Layout:**

```
┌─────────────────────────────────────────────────────────────────┐
│  [● LIVE]  [⏸ Pause]  [⟳ Clear]    Showing 847 / 10,000 rows  │
│  [Search domain or regex…] [Client ▼] [Type ▼] [Result ▼] [⋯]  │
│  Active filters: result=BLOCK  client=192.168.1.50  [× Clear all]│
├──────────┬────────────────────┬──────┬────────┬──────────┬──────┤
│ Time     │ Domain             │ Type │ Result │ Src      │  ms  │
├──────────┼────────────────────┼──────┼────────┼──────────┼──────┤
│ 14:23:01 │ ads.google.com     │  A   │ BLOCK  │ blocklist│   –  │← red left border
│ 14:23:01 │ server.home.arpa   │  A   │ PASS   │ local    │   0  │← violet
│ 14:23:00 │ api.github.com     │  A   │ PASS   │ cache    │   0  │← cyan
│ 14:23:00 │ example.com        │  A   │ PASS   │ upstream │  12  │← green
│ 14:22:59 │ svc.corp.example   │  A   │ PASS   │ forward  │   8  │← indigo
└──────────┴────────────────────┴──────┴────────┴──────────┴──────┘
│ Buffer: 10,000 rows · 18.4 MB   [Export CSV]  [Export JSON]     │
└─────────────────────────────────────────────────────────────────┘
```

**Full column set (configurable – drag to reorder, click eye icon to hide):**

| Column | Width | Notes |
|--------|-------|-------|
| Time | 90px | HH:mm:ss.SSS; hover shows full ISO timestamp in tooltip |
| Client IP | 120px | Clickable → adds to client filter |
| Client name | 140px | Resolved from `clients` table; greyed if unknown |
| Domain | auto | Monospace; full FQDN; filter-match highlighted |
| Type | 52px | A / AAAA / CNAME / MX / TXT / PTR … |
| Result | 80px | Coloured badge: PASS / BLOCK / NXDOMAIN / SERVFAIL |
| Source | 80px | local / cache / upstream / forward / blocklist |
| Upstream | 140px | Which upstream responded (if applicable) |
| Latency | 56px | µs or ms; `–` for cached/local |
| DNSSEC | 48px | AD / – / FAIL badge |

---

#### 6.2.1 Filter System

Filtering operates on **two tiers** that work together:

**Tier 1 – Server-side SSE parameters** (coarse-grained, reduces wire volume):

The SSE subscription URL carries query parameters that are applied on the server before events are sent. Changing any of these restarts the SSE connection.

| Parameter | Example | Effect |
|-----------|---------|--------|
| `q` | `q=ads.` | Server emits only events whose domain contains `ads.` |
| `result` | `result=BLOCK` | Server emits only blocked queries |
| `client` | `client=192.168.1.50` | Server emits only queries from this client |
| `qtype` | `qtype=A,AAAA` | Server emits only A and AAAA queries |

When all server-side parameters are clear, the full stream is received and all filtering is done in the browser.

**Tier 2 – Client-side filter (in Web Worker, zero main-thread cost):**

Applied to the in-memory ring buffer without restarting the stream. Changes are debounced at 150 ms; results are sent back to the main thread as a compact index array (not the full rows).

| Control | Type | Client-side only |
|---------|------|-----------------|
| Domain search | Text – plain substring or `/regex/` | Yes |
| Client multi-select | Dropdown, searches IP and hostname | Partially – triggers SSE restart only when the server param set changes |
| Type checkboxes | A / AAAA / CNAME / MX / TXT / PTR / other | Yes |
| Result checkboxes | PASS / BLOCK / NXDOMAIN / SERVFAIL / ERROR | Yes |
| Source checkboxes | local / cache / upstream / forward / blocklist | Yes |
| Min latency | Number input (µs) | Yes |
| Time range | Live / Last 5m / Last 1h / Custom | Live = client-side; historical = history API |
| DNSSEC status | AD / no-AD / FAIL | Yes |

**Filter URL state:** all active filter values are written to the URL query string (`?q=&result=&client=…`) on every change so that the browser's back button and copy-URL restore the same view.

**Filter presets:** up to 10 named filter combinations can be saved to `localStorage` (key: `dnsdave.log.presets`). A `[☆ Save preset]` button appears when any filter is active. Presets are listed in a dropdown next to the filter bar.

**Active filter chips:** each active filter renders as a dismissible chip below the filter bar (as shown in the layout). Clicking a chip removes that filter.

**Filter match highlighting:** when a domain text filter is active, the matched portion of the domain string is highlighted with a `<mark>` element styled in the theme's accent colour. Regex matches highlight each capture group.

---

#### 6.2.2 Memory Architecture

The log viewer is the highest-throughput component in the UI. At 10,000 QPS sustained (a realistic busy home lab or small office), the naive approach of pushing objects into a reactive Svelte store would cause continuous GC pressure, main-thread jank, and memory growth until Chrome kills the tab.

**Design goals:**
- Hard memory cap: ≤ 25 MB for the log buffer regardless of query volume or session length.
- Main thread never touches raw log data – only the filtered visible slice.
- Zero allocations per incoming event on the hot path (object pool reuse).
- Chrome `performance.memory.usedJSHeapSize` stays flat when the buffer is full.

**Implementation – Shared Ring Buffer in a Web Worker:**

```
┌─ Main Thread (SvelteKit page) ──────────────────────────────────┐
│                                                                   │
│   <LogViewer>                                                     │
│      ↕ postMessage (FilterSpec)          ↑ postMessage           │
│      ↓                                   (FilterResult)          │
│   svelte-virtual-list  ←── visible rows slice (index array) ─────┤
│      renders only ~30 visible rows                               │
└──────────────────────────────────────────────────────────────────┘
          ↑ SSE text events (raw JSON strings)
          │
┌─ log.worker.ts (Dedicated Worker) ──────────────────────────────┐
│                                                                   │
│  Ring Buffer (pre-allocated):                                     │
│    rows[0..9999] – object pool, slot reused oldest-first         │
│    head: number   – write pointer                                 │
│    size: number   – current fill level                           │
│                                                                   │
│  On SSE event:                                                    │
│    1. Parse JSON into next pool slot (reuse object, no alloc)    │
│    2. Advance head pointer                                        │
│    3. Re-apply current FilterSpec to affected window             │
│    4. If new row passes filter: append its index to visibleIdx[] │
│    5. postMessage({ type: 'append', indices: [i], stats })       │
│       (only if main thread is not already pending a render)      │
│                                                                   │
│  On FilterSpec change (from main thread):                        │
│    1. Full scan of ring buffer (max 10,000 iterations)           │
│    2. Compile regex once before scan                             │
│    3. Build visibleIdx[] (Uint16Array – compact, transferable)   │
│    4. postMessage({ type: 'filter', indices: Uint16Array },      │
│                   [indices.buffer])   ← Transferable, zero-copy  │
└──────────────────────────────────────────────────────────────────┘
```

**Ring buffer pre-allocation:**

The worker allocates all 10,000 row objects at startup and reuses them in a circular pattern. When the buffer is full, the oldest slot is overwritten in-place – no allocation, no GC.

```typescript
// log.worker.ts (illustrative)
const CAPACITY = 10_000;
const pool: LogRow[] = Array.from({ length: CAPACITY }, () => ({} as LogRow));
let head = 0;
let size = 0;

function ingest(raw: string): void {
  const slot = head % CAPACITY;
  const obj = pool[slot];
  // Assign fields in-place – reuse the object; no `new`
  const parsed = JSON.parse(raw);
  obj.ts = parsed.ts;
  obj.domain = parsed.domain;
  obj.qtype = parsed.qtype;
  obj.result = parsed.result;
  obj.client = parsed.client;
  obj.source = parsed.source;
  obj.latency_us = parsed.latency_us;
  obj.dnssec_ad = parsed.dnssec_ad;
  head = (head + 1) % CAPACITY;
  if (size < CAPACITY) size++;
}
```

**Memory budget:**

| Item | Budget |
|------|--------|
| Ring buffer (10,000 pre-allocated row objects) | ≤ 12 MB |
| Visible index array (`Uint16Array`, max 10,000 × 2 bytes) | 20 KB |
| DOM nodes (svelte-virtual-list renders ~30 rows) | ≤ 2 MB |
| Filter state, presets, misc | < 1 MB |
| **Total budget** | **≤ 25 MB** |

The status bar displays live `buffer: N rows · X.X MB` to make the budget visible to the user. The MB figure is an estimate: `size × avgRowBytes` where `avgRowBytes` is sampled from the last 100 ingest events.

**When the buffer is full:** oldest rows are silently overwritten. The status bar shows `● 10,000 rows (ring full)`. No alert, no pause – the viewer simply becomes a sliding window.

**Pausing:** when the user clicks `⏸ Pause`, the worker stops updating `visibleIdx` and stops posting messages. Incoming SSE events still fill the ring buffer in the worker (so nothing is lost), but the main thread view freezes. On resume, the worker posts a full `filter` result immediately.

**Tab visibility:** an `IntersectionObserver` on the log viewer container detects when it scrolls off-screen; a `visibilitychange` listener detects when the browser tab is hidden. In both cases, the worker reduces its postMessage rate to 1 update/second (instead of per-event) to avoid waking the main thread unnecessarily.

---

#### 6.2.3 Virtual Scroll

`svelte-virtual-list` renders only the visible viewport rows – typically 25–35 rows at 36px row height in a 1080p window. The list is fed the `visibleIdx[]` array (indices into the ring buffer) and reads row data on demand:

```svelte
<!-- LogViewer.svelte (illustrative) -->
<VirtualList
  items={$visibleIndices}
  itemHeight={36}
  let:item={idx}
>
  <LogRow row={worker.getRow(idx)} filter={$activeFilter} />
</VirtualList>
```

The `getRow(idx)` call reads a value from the shared pool that the worker has already written. Because the worker holds the pool and the main thread only reads indices, there is no serialisation overhead for rows that are not currently visible.

**Row height:** fixed at 36px (single-line mode, default). An optional "expanded" mode shows the domain on two lines (52px) for very long FQDNs. The height toggle is global for the current session and stored in `localStorage`.

**Scroll behaviour:**
- **Live mode (default):** the list auto-scrolls to top (newest) on each update. A `↓ N new rows` toast appears if the user has manually scrolled away from the top; clicking it snaps back.
- **Paused / filtered mode:** scroll position is preserved.
- **Keyboard:** `↑` / `↓` navigate rows; `Enter` opens the detail drawer for the focused row; `Esc` closes the drawer.

---

#### 6.2.4 Row Detail Drawer

Opens as a right-side slide-over (480px wide) when a row is clicked. The drawer reads the full row from the ring buffer by index – the pool object is still valid because the drawer is closed before the slot could be reused (a 10,000-row buffer at 1,000 QPS gives ~10 seconds before a slot is recycled; the drawer includes a `⚠ This entry has been recycled` warning if the slot was overwritten whilst the drawer was open).

**Drawer content:**

```
┌─ Query Detail ──────────────────────────────────────── [×] ───┐
│  ads.google.com                         14:23:01.842           │
│  ─────────────────────────────────────────────────────────     │
│  Client     192.168.1.50  (mylaptop)   Group: default          │
│  Result     ● BLOCKED                  Source: blocklist       │
│  Type        A                         Latency: –              │
│  DNSSEC      –                                                  │
│                                                                 │
│  EDNS0 Flags    RD=1  DO=0  CD=0  AD=0                        │
│                                                                 │
│  Response    NXDOMAIN (synthesised)                            │
│                                                                 │
│  Blocklist match                                                │
│    List: StevenBlack hosts (updated 3h ago)                    │
│    Rule: ads.google.com                                         │
│                                                                 │
│  ─────────────────────────────────────────────────────────     │
│  [✓ Allow this domain]   [⛔ Re-block]   [View client history] │
└───────────────────────────────────────────────────────────────┘
```

For upstream/forward results, the drawer additionally shows the full RRset with TTL and (if DNSSEC) the RRSIG record.

---

#### 6.2.5 Export

Export always pulls from the **history API** (paginated), never from the in-memory ring buffer, so it is not limited to the 10,000-row window.

| Format | Endpoint | Notes |
|--------|----------|-------|
| CSV | `GET /api/v1/dns/queries?format=csv&…` | Streamed; `Content-Disposition: attachment` |
| JSON (NDJSON) | `GET /api/v1/dns/queries?format=ndjson&…` | One JSON object per line |
| JSON (array) | `GET /api/v1/dns/queries?format=json&…` | Full array; limited to 100,000 rows |

All active filters are forwarded as query parameters in the export request so that "Export CSV" exports only what is currently visible.

**Data source:**
- Live view: SSE `/api/v1/dns/queries/stream?q=&result=&client=&qtype=`
- History view: `GET /api/v1/dns/queries?q=&result=&client=&qtype=&from=&to=&page=&per_page=200`

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

### 6.13 NetBox Integration

**Purpose:** First-class bidirectional integration with NetBox 4.5+. Configure push and pull behaviour, manage API credentials, map NetBox organisational objects to DNSDave objects, and monitor sync activity — all from a single page.

**Navigation:** `Settings → Integrations → NetBox` (also reachable via the global Command Palette as "NetBox Integration").

**Layout – tabbed panel page:**

```
┌─ NetBox Integration ──────────────────────── [● Connected] [Test] ─┐
│                                                                      │
│  [Connection] [Push] [Pull] [Mappings] [Activity]                   │
│                                                                      │
└──────────────────────────────────────────────────────────────────────┘
```

---

#### Tab 1 – Connection

```
┌─ Connection ─────────────────────────────────────────────────────────┐
│                                                                       │
│  NetBox URL    [https://netbox.example.com              ]  [Test]    │
│                                                                       │
│  API Token     [••••••••••••••••••••••••••••  8f3a]  [Show] [⟳]    │
│                ↳ Last rotated: 12 days ago                           │
│                                                                       │
│  Mode          ● REST API (direct)                                   │
│                ○ Diode (gRPC – recommended for high-volume push)     │
│                                                                       │
│  Diode URL     [grpc://diode-server:8080        ]  (shown if Diode) │
│  Diode Key     [••••••••••••  9c2d]  [Show] [⟳]                    │
│                                                                       │
│  ┌─ Connection Status ─────────────────────────────────────────────┐ │
│  │  ✅ Connected to NetBox 4.5.2  ·  Response: 3 ms               │ │
│  │  Authenticated as: dnsdave-svc  ·  Permissions: read + write   │ │
│  │  Diode: ✅ Connected  ·  Queue depth: 0                        │ │
│  └──────────────────────────────────────────────────────────────── │
│                                                                       │
│  Webhook Secret   [•••••••••••••••  ] [Show] [⟳ Regenerate]        │
│  Webhook URL      https://dnsdave:8080/api/v1/integrations/netbox/  │
│                   webhook                              [📋 Copy]    │
│                                                                       │
│  [Save Connection]                                                    │
└───────────────────────────────────────────────────────────────────────┘
```

**Behaviour:**
- `[Test]` fires `POST /api/v1/integrations/netbox/test` and displays the result inline: NetBox version, response latency, authenticated user, and permission check (must have IPAM read/write and DCIM read).
- Token and keys are stored AES-GCM encrypted in Postgres; the UI displays only the last 4 characters.
- `[⟳]` (rotate) opens a confirmation modal; the old token is invalidated only after the new one is saved and tested successfully.
- Webhook URL: display only (generated from the DNSDave API base URL). A `[📋 Copy]` button copies it to clipboard. Alongside it, a `[📜 Generate NetBox Script]` button downloads a Python script that, when run against the connected NetBox instance, creates all required event rules automatically.
- Mode toggle: switching from REST to Diode shows the Diode URL and key fields with a smooth CSS transition.

---

#### Tab 2 – Push (DNSDave → NetBox)

```
┌─ Push ────────────────────────────────────────────────────────────────┐
│                                                                        │
│  ☑ Enable push                                                        │
│                                                                        │
│  ┌─ What to push ──────────────────────────────────────────────────┐  │
│  │  ☑  Prefixes          DHCP scopes → NetBox Prefixes             │  │
│  │  ☑  IP Ranges         DHCP pool bounds → NetBox IP Ranges       │  │
│  │  ☑  IP Addresses      DHCP leases → NetBox IP Addresses         │  │
│  │  ☑  DNS names         DNS A/AAAA records → IP Address dns_name  │  │
│  │  ☐  Devices           Auto-create Device records for new leases  │  │
│  │       └── Max auto-create per hour: [10     ]                   │  │
│  │           Dry-run mode:  ☑ (log what would be created)          │  │
│  │           MAC prefix allowlist: [aa:bb:cc, 11:22:33   ]         │  │
│  └─────────────────────────────────────────────────────────────────┘  │
│                                                                        │
│  ┌─ Conflict strategy ────────────────────────────────────────────┐   │
│  │  When a NetBox object already exists:                           │   │
│  │  ● Overwrite   DNSDave data always wins                        │   │
│  │  ○ Skip        Leave NetBox unchanged                           │   │
│  │  ○ Tag         Add "dnsdave-conflict" tag and alert             │   │
│  │  ○ Merge       Fill only empty NetBox fields                    │   │
│  └──────────────────────────────────────────────────────────────── │   │
│                                                                        │
│  ┌─ Lifecycle ────────────────────────────────────────────────────┐   │
│  │  On lease expiry:    [Free (status → available)  ▼]            │   │
│  │  On lease release:   [Free (status → available)  ▼]            │   │
│  │  On scope deleted:   [Keep (Prefix status → container) ▼]      │   │
│  └────────────────────────────────────────────────────────────────┘   │
│                                                                        │
│  ┌─ Context ──────────────────────────────────────────────────────┐   │
│  │  Default VRF:     [              ▼]  (blank = Global Table)    │   │
│  │  Default Site:    [              ▼]                             │   │
│  │  Default Tenant:  [              ▼]                             │   │
│  │  Tags applied:    [dnsdave            ×] [+ Add tag]            │   │
│  └────────────────────────────────────────────────────────────────┘   │
│                                                                        │
│  ┌─ Push Status ──────────────────────────────────────────────────┐   │
│  │  Last push:  2 minutes ago                                      │   │
│  │  Objects:    Prefixes 12  IP Ranges 12  Addresses 847  DNS 412  │   │
│  │  Errors:     0    Conflicts: 0    Skipped: 3                    │   │
│  └────────────────────────────────────────────────────────────────┘   │
│                                                                        │
│  [Save Push Config]              [▶ Push Now]   [☐ Dry-run ▼]        │
└────────────────────────────────────────────────────────────────────────┘
```

**Behaviour:**
- `[▶ Push Now]` fires `POST /api/v1/integrations/netbox/sync/push` and switches to the Activity tab, which streams live progress via SSE.
- Dry-run mode can be set for a single run (`[☐ Dry-run ▼]` dropdown: "Once" or "Always") or toggled globally. When dry-run is active, a yellow `DRY RUN` badge appears in the tab header.
- Lifecycle dropdowns: "Free" (status → available), "Remove" (delete the IP Address from NetBox), "Keep" (do nothing). A warning tooltip on "Remove" explains this is irreversible.
- Device auto-creation: collapsing sub-section that expands when the "Devices" checkbox is ticked. Max per-hour cap and MAC allowlist guard-rail inputs appear inside it.

---

#### Tab 3 – Pull (NetBox → DNSDave)

```
┌─ Pull ─────────────────────────────────────────────────────────────────┐
│                                                                         │
│  ☑ Enable pull                                                         │
│                                                                         │
│  Pull mode:                                                             │
│  ● Webhook     NetBox sends events to DNSDave instantly                │
│  ○ Scheduled   Poll NetBox every [15   ] minutes                       │
│  ○ Manual      Only when triggered from this page                      │
│                                                                         │
│  ┌─ What to pull ──────────────────────────────────────────────────┐   │
│  │  ☑  Prefixes             NetBox Prefixes  → DHCP scopes         │   │
│  │  ☑  IP Ranges            NetBox IP Ranges → DHCP pool bounds    │   │
│  │  ☑  IP Addresses         NetBox IPs       → DNS records         │   │
│  │  ☑  Devices              Devices + IPs    → DNS A + PTR records │   │
│  │  ☑  Virtual Machines     VMs + IPs        → DNS A + PTR records │   │
│  │  ☑  Static Reservations  IPs with MAC CF  → DHCP reservations  │   │
│  │                                                                  │   │
│  │  Auto-create DNS zones for unknown domains:  ○ Yes  ● No        │   │
│  └──────────────────────────────────────────────────────────────────┘  │
│                                                                         │
│  ┌─ Filters ──────────────────────────────────────────────────────┐    │
│  │  Only pull objects tagged:    [dnsdave            ×] [+ tag]   │    │
│  │  Filter by site:              [All sites           ▼]          │    │
│  │  Filter by VRF:               [All VRFs            ▼]          │    │
│  │  Filter by tenant:            [All tenants         ▼]          │    │
│  └──────────────────────────────────────────────────────────────── │    │
│                                                                         │
│  ┌─ Conflict strategy ─────────────────────────────────────────────┐   │
│  │  When a DNSDave object already exists:                          │   │
│  │  ● Skip        Leave DNSDave unchanged (safest default)         │   │
│  │  ○ Overwrite   NetBox data wins                                 │   │
│  │  ○ Merge       Fill only empty DNSDave fields                   │   │
│  │  ○ Tag         Mark conflict; queue for manual review           │   │
│  │  ○ Ask         Open a conflict resolution dialog per object     │   │
│  └──────────────────────────────────────────────────────────────── │   │
│                                                                         │
│  ┌─ Device FQDN template ──────────────────────────────────────────┐   │
│  │  Template:  [{name}.{site}.{default_zone}          ]           │   │
│  │  Preview:   router-01.london.corp.example.com                   │   │
│  │  Variables: {name} {site} {tenant} {role} {default_zone}       │   │
│  └──────────────────────────────────────────────────────────────── │   │
│                                                                         │
│  ┌─ Pull Status ──────────────────────────────────────────────────┐    │
│  │  Last pull:     14 minutes ago  (next in 1 min)                 │    │
│  │  Last webhook:  32 seconds ago  (2 events received)             │    │
│  │  Pulled:        Prefixes 8   IPs 241   Devices 34   VMs 12      │    │
│  │  Pending:       0    Conflicts: 2    Errors: 0                   │    │
│  └──────────────────────────────────────────────────────────────── │    │
│                                                                         │
│  [Save Pull Config]  [▶ Pull Now]  [🔍 Preview Pull]  [☐ Dry-run ▼]  │
└─────────────────────────────────────────────────────────────────────────┘
```

**Behaviour:**
- `[🔍 Preview Pull]` fires `GET /api/v1/integrations/netbox/sync/preview` and opens a modal showing a diff table: "Would create N, update M, delete K, conflicts P" with expandable row detail for each action.
- Preview modal has a `[▶ Apply]` button to execute the pull immediately from within the modal.
- Webhook mode: when selected, a status indicator shows the last received event and how many events have been processed in the last hour.
- FQDN template live preview: the preview line updates as the user types, using the first device returned by the NetBox API as a sample object.
- Conflict count badge: a red badge on the tab header `Pull ②` appears when there are unresolved conflicts. Clicking it filters the Activity tab to conflicts only.

---

#### Tab 4 – Mappings

```
┌─ Mappings ──────────────────────────────────────────────────────────────┐
│  Map NetBox organisational objects to DNSDave objects.                   │
│                                                                          │
│  [VRF → Scope] [Site → Zone] [Tenant → Client Group] [Tag → Filter]     │
│                                                                          │
│  ── VRF → DHCP Scope ─────────────────────────────────────────────────  │
│  NetBox VRF                    DNSDave DHCP Scope                        │
│  [corporate           ▼]  →   [Office LAN (scope_office)  ▼]  [× Del]  │
│  [iot-network         ▼]  →   [IoT Segment (scope_iot)    ▼]  [× Del]  │
│  [+ Add VRF mapping]                                                     │
│                                                                          │
│  ── Site → DNS Zone Suffix ───────────────────────────────────────────  │
│  NetBox Site                   DNSDave Zone                              │
│  [london              ▼]  →   [london.corp.example  ▼]  [× Del]        │
│  [new-york            ▼]  →   [ny.corp.example      ▼]  [× Del]        │
│  [+ Add site mapping]                                                    │
│                                                                          │
│  ── Tenant → Client Group ────────────────────────────────────────────  │
│  NetBox Tenant                 DNSDave Client Group                      │
│  [finance             ▼]  →   [Finance (group_finance)  ▼]  [× Del]   │
│  [+ Add tenant mapping]                                                  │
│                                                                          │
│  [Save Mappings]                                                         │
└──────────────────────────────────────────────────────────────────────────┘
```

**Behaviour:**
- All dropdowns are populated by live API calls to both NetBox (`GET /api/ipam/vrfs/`, etc.) and DNSDave. They are searchable with a debounced text filter.
- Saving a mapping fires `POST /api/v1/integrations/netbox/mappings`.
- Mappings take effect on the next sync operation.
- A `[Test mapping]` button on each row runs a dry-run of that specific mapping against the current NetBox data and shows how many objects would be affected.

---

#### Tab 5 – Activity

```
┌─ Activity ──────────────────────────────────────────────────────────────┐
│  [● LIVE]  [⏸]  [⟳ Clear]    Showing 312 events                        │
│  [All ▼] [Direction: All ▼] [Status: All ▼] [Object type: All ▼]       │
├──────────┬──────────┬─────────────┬───────────┬────────────┬────────────┤
│ Time     │ Direction│ Object type │ NetBox ID │ Action     │ Status     │
├──────────┼──────────┼─────────────┼───────────┼────────────┼────────────┤
│ 10:23:01 │ ← pull   │ IP Address  │ nb:317    │ created    │ ✅ synced  │
│ 10:23:00 │ → push   │ IP Address  │ nb:318    │ updated    │ ✅ synced  │
│ 10:22:48 │ ← webhook│ Prefix      │ nb:42     │ updated    │ ✅ synced  │
│ 10:21:33 │ → push   │ IP Address  │ nb:205    │ skipped    │ ⏭ skip    │
│ 10:20:11 │ ← pull   │ Device      │ nb:88     │ conflict   │ ⚠ conflict │← amber row
└──────────┴──────────┴─────────────┴───────────┴────────────┴────────────┘
│  [Export CSV]  [Export JSON]                                             │
└──────────────────────────────────────────────────────────────────────────┘
```

**Clicking a conflict row opens a resolution drawer:**

```
┌─ Conflict: Device nb:88 ──────────────────────────────────────────────┐
│  Device: "router-01" (NetBox ID 88)                                    │
│  Direction: Pull (NetBox → DNSDave)                                    │
│                                                                         │
│  Conflict on: dns_record "router-01.london.corp.example"               │
│                                                                         │
│  NetBox value:  A  192.168.10.1  (from device primary_ip4)             │
│  DNSDave value: A  192.168.10.2  (manually set, last modified 3h ago)  │
│                                                                         │
│  Resolution:                                                            │
│  ○ Keep DNSDave value (192.168.10.2)  — ignore NetBox                  │
│  ○ Use NetBox value   (192.168.10.1)  — overwrite DNSDave              │
│  ○ Skip permanently   — never sync this object again                   │
│                                                                         │
│  [Apply resolution]                    [Cancel]                        │
└─────────────────────────────────────────────────────────────────────────┘
```

**Activity tab behaviour:**
- Live SSE stream from `GET /api/v1/integrations/netbox/log/stream`. Same ring-buffer + Web Worker pattern as the Query Log (see §6.2) — max 5,000 events in-memory, virtual scroll.
- Direction filter: "All" / "→ Push" / "← Pull" / "← Webhook".
- Status filter: "All" / "✅ Synced" / "⚠ Conflict" / "❌ Error" / "⏭ Skipped".
- Conflict rows are highlighted amber; a `[Resolve]` button opens the drawer above.
- Error rows show a `[Retry]` button that fires `POST /api/v1/integrations/netbox/objects/:id/resolve` with `{ action: "retry" }`.
- `[Export CSV/JSON]` exports the filtered log via the history API.

---

#### Status widget (sidebar + dashboard)

A compact status widget appears in the sidebar under the `Integrations` nav group:

```
NetBox   ✅ 3m ago   ⚠ 2 conflicts
```

Clicking it navigates to the Mappings page filtered to conflicts. The Dashboard page shows a "NetBox sync" card with last push/pull times, total managed objects, and a sparkline of sync events per hour.

---

#### Setup Wizard (first-run)

When NetBox integration is not yet configured, the page shows an empty state with a `[Connect NetBox]` wizard button. The wizard steps through:

1. Enter NetBox URL and API token → test connection.
2. Choose mode (REST API or Diode). If Diode is chosen, shows docker-compose snippet.
3. Configure push toggles (pre-selected sensible defaults).
4. Configure pull toggles + filters.
5. Click `[Generate NetBox Event Rules]` → downloads a Python script to run against NetBox that creates all required event rules. Alternatively, pastes the configuration into a text box if copy-paste is preferred.
6. Review and save.

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

### 7.6 `LogViewer`

The primary full-featured log component used on the Query Log page. All heavy lifting (ring buffer, filtering, indexing) happens off-main-thread in `log.worker.ts`.

```typescript
// Props
interface LogViewerProps {
  stream:        string;          // SSE endpoint base URL
  historyUrl:    string;          // paginated history API URL
  columns?:      ColumnDef[];     // override visible columns
  capacity?:     number;          // ring buffer size; default 10_000
  rowHeight?:    number;          // px; default 36
  initialFilter?: FilterSpec;     // applied on mount (from URL params)
  onRowClick?:   (row: LogRow) => void;
}

// FilterSpec – serialisable; written to URL query string
interface FilterSpec {
  q?:            string;    // plain substring or /regex/
  result?:       string[];  // e.g. ['BLOCK', 'NXDOMAIN']
  client?:       string[];
  qtype?:        string[];
  source?:       string[];
  minLatencyUs?: number;
  dnssec?:       'ad' | 'fail' | 'none';
}

// Worker message protocol
type WorkerIn =
  | { type: 'event';  raw: string }         // raw SSE JSON string
  | { type: 'filter'; spec: FilterSpec }    // new filter from UI
  | { type: 'pause' }
  | { type: 'resume' }
  | { type: 'clear' };

type WorkerOut =
  | { type: 'append'; indices: Uint16Array; stats: BufferStats }
  | { type: 'filter'; indices: Uint16Array; stats: BufferStats }
  | { type: 'stats';  stats: BufferStats };

interface BufferStats {
  total:       number;  // rows in ring buffer
  visible:     number;  // rows matching current filter
  estimatedMb: number;  // live memory estimate
  full:        boolean; // ring buffer at capacity
}
```

**Behaviour:**

- On mount: spawns `log.worker.ts`, opens SSE connection via SvelteKit server-side proxy, forwards each raw event string to the worker via `postMessage`.
- On `WorkerOut.append`: appends new visible indices to the displayed slice; auto-scrolls to top if in live mode.
- On `WorkerOut.filter`: replaces `visibleIndices` entirely with the new `Uint16Array` (zero-copy, transferred ownership).
- On unmount: terminates the worker and closes the SSE connection. The ring buffer is discarded with it – no lingering memory.
- Exposes `pause()`, `resume()`, `clear()`, and `updateFilter(spec)` via a Svelte `bind:this`.

**`log.worker.ts` responsibilities:**

- Maintains the pre-allocated ring buffer (object pool, no GC-triggering allocations on the hot path).
- Compiles `FilterSpec.q` into a `RegExp` once per filter change; caches it.
- On a full filter scan (10,000 iterations max): builds a `Uint16Array` of matching indices and transfers it to the main thread with zero-copy (`postMessage(msg, [msg.indices.buffer])`).
- On each new ingest: checks only the new row against the current filter; appends its index if it matches – no full rescan needed.
- Debounces filter re-scans at 150 ms when multiple `filter` messages arrive in quick succession (e.g., user typing).
- Emits a `stats` message every 2 seconds (not per-event) for the status bar.

**`MiniLogFeed` (lightweight variant):** used on the Dashboard for the 5-row live query preview. Uses the same worker protocol but with `capacity: 500` and no filter controls.

#### 7.6.1 `MiniLogFeed`

```typescript
interface MiniLogFeedProps {
  maxRows?:   number;   // default 5 (display); buffer 500
  onViewAll?: () => void;
}
```

Lightweight live preview. Shares the same `log.worker.ts` module but with a 500-row capacity. Clicking a row navigates to the full Query Log filtered to that client. "View all" navigates to an unfiltered Query Log.

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

### 8.1 Architecture Overview

The UI uses **two distinct real-time patterns**, chosen based on throughput:

| Stream | Volume | Pattern | Rationale |
|--------|--------|---------|-----------|
| DNS query log | Up to 10,000/s | Web Worker + ring buffer | Main-thread isolation; zero GC pressure |
| DHCP lease events | < 10/s | Lightweight `LiveStore` | Low volume; main-thread reactive store is fine |
| System events | < 1/s | Lightweight `LiveStore` | Low volume; drives notifications |
| Zone serial updates | < 1/s | Lightweight `LiveStore` | Low volume; drives zone list badge |

### 8.2 DNS Query Log – Web Worker Pattern

The query log stream bypasses Svelte's reactive store entirely. Raw SSE event strings are forwarded to `log.worker.ts` without parsing on the main thread.

```
SSE connection (SvelteKit proxy)
    │  raw JSON strings
    ▼
log.worker.ts (Dedicated Worker – separate V8 heap)
    │  pre-allocated ring buffer (10,000 slots)
    │  FilterSpec applied here
    │  Uint16Array of matching indices
    │  postMessage(..., [transferable])   ← zero-copy
    ▼
LogViewer.svelte (main thread)
    │  stores only the Uint16Array (20 KB max)
    │  reads row data from worker on demand via getRow(idx)
    ▼
svelte-virtual-list
    │  renders ~30 DOM rows at any time
    ▼
Chrome tab heap
```

**Why this pattern keeps Chrome memory low:**

1. The ring buffer lives in the worker's V8 heap, not the main thread's. Chrome's "Memory" DevTools tab shows the main tab heap stays small.
2. The main thread only ever holds a `Uint16Array` of at most 20 KB (10,000 × 2 bytes). It never holds parsed log row objects.
3. `svelte-virtual-list` recycles DOM nodes – only ~30 nodes exist regardless of buffer size.
4. Transferring the `Uint16Array` with `Transferable` semantics moves ownership to the main thread at zero copy cost; the worker immediately receives a new buffer to write into.
5. No array spread (`[item, ...rows]`) on the hot path – the ring buffer is mutated in-place.

**SSE proxy route** (`src/routes/api/stream/+server.ts`):

All SSE connections are proxied through SvelteKit's server-side routes. This has two benefits: the API key is never exposed in browser network tabs (it is added server-side), and the SvelteKit edge can add gzip compression to the event stream.

```typescript
// src/routes/api/stream/queries/+server.ts
export async function GET({ url, locals }) {
  const upstream = new URL('/api/v1/dns/queries/stream', DNSDAVE_API_URL);
  url.searchParams.forEach((v, k) => upstream.searchParams.set(k, v));

  const res = await fetch(upstream, {
    headers: { 'Authorization': `Bearer ${locals.apiKey}` }
  });

  return new Response(res.body, {
    headers: {
      'Content-Type':  'text/event-stream',
      'Cache-Control': 'no-cache',
      'X-Accel-Buffering': 'no',
    }
  });
}
```

**Worker lifecycle:**

```typescript
// src/lib/log-worker-bridge.ts
export class LogWorkerBridge {
  private worker: Worker;
  private source: EventSource | null = null;

  constructor(private endpoint: string) {
    this.worker = new Worker(
      new URL('./log.worker.ts', import.meta.url),
      { type: 'module' }
    );
  }

  connect(params: Record<string, string>) {
    const url = new URL(this.endpoint, location.origin);
    Object.entries(params).forEach(([k, v]) => url.searchParams.set(k, v));

    this.source = new EventSource(url.toString());
    this.source.onmessage = (e) => {
      // Forward raw string – no JSON.parse on main thread
      this.worker.postMessage({ type: 'event', raw: e.data });
    };
    this.source.onerror = () => this.reconnect();
  }

  applyFilter(spec: FilterSpec) {
    this.worker.postMessage({ type: 'filter', spec });
  }

  onMessage(handler: (msg: WorkerOut) => void) {
    this.worker.onmessage = (e) => handler(e.data);
  }

  destroy() {
    this.source?.close();
    this.worker.terminate();   // frees ring buffer heap immediately
  }

  private reconnect() {
    this.source?.close();
    const delay = Math.min(30_000, exponentialBackoff() + jitter());
    setTimeout(() => this.connect(this.lastParams), delay);
  }
}
```

### 8.3 Lightweight `LiveStore` (Low-volume Streams)

For DHCP, system events, and zone updates – all of which arrive at fewer than 10 events per second – a simple reactive store is appropriate. The key fix over a naïve implementation is using a **fixed-size circular array** rather than spread-and-slice:

```typescript
// src/lib/stores/live.ts
export function createLiveStore<T>(endpoint: string, capacity = 500) {
  const buffer: T[] = new Array(capacity);
  let head = 0;
  let size = 0;
  const { subscribe, set } = writable<T[]>([]);
  let source: EventSource | null = null;
  let paused = false;
  let snapshotDirty = false;

  function flush() {
    if (!snapshotDirty) return;
    snapshotDirty = false;
    // Build ordered snapshot without spread – reuse slice
    const snap = size < capacity
      ? buffer.slice(0, size).reverse()
      : [...buffer.slice(head), ...buffer.slice(0, head)].reverse();
    set(snap);
  }

  function connect() {
    source = new EventSource(endpoint);
    source.onmessage = (e) => {
      if (paused) return;
      buffer[head % capacity] = JSON.parse(e.data) as T;
      head = (head + 1) % capacity;
      if (size < capacity) size++;
      snapshotDirty = true;
      requestAnimationFrame(flush);   // batch to one DOM update per frame
    };
    source.onerror = () => {
      source?.close();
      setTimeout(connect, Math.min(30_000, exponentialBackoff()));
    };
  }

  return {
    subscribe,
    connect,
    pause:      () => { paused = true; },
    resume:     () => { paused = false; },
    disconnect: () => source?.close(),
  };
}
```

**Key properties:**
- `requestAnimationFrame(flush)` batches Svelte store updates to one per frame – no matter how fast events arrive, the DOM updates at most 60 times/second.
- The snapshot copy is built once per frame, not once per event.
- `paused = true` drops incoming events from the store but the SSE connection stays open (no reconnect needed on resume).

### 8.4 SSE Endpoints

| Store / component | Proxy route | Upstream endpoint | Consumer pages |
|---|---|---|---|
| `LogWorkerBridge` | `/api/stream/queries` | `/api/v1/dns/queries/stream` | Dashboard (mini), Query Log |
| `dhcpLeaseStore` | `/api/stream/dhcp/leases` | `/api/v1/dhcp/leases/stream` | Dashboard, DHCP Leases |
| `systemEventStore` | `/api/stream/system` | `/api/v1/system/events/stream` | Notification drawer, Cluster |
| `zoneEventStore` | `/api/stream/zones` | `/api/v1/dns/zones/events/stream` | Zones page |

### 8.5 Connection Status

A global `connectionStore` tracks the health of every SSE connection:

```typescript
type ConnectionState = 'live' | 'reconnecting' | 'paused' | 'disconnected';
```

The top bar `● LIVE` indicator reflects the **worst** state across all active connections. Individual page headers show per-stream state. Whilst reconnecting, the last known data remains visible – it is never cleared on disconnect.

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
| NetBox – config | GET/PUT | `/api/v1/integrations/netbox/config` | Token masked in GET response |
| NetBox – test connection | POST | `/api/v1/integrations/netbox/test` | Returns version, latency, permissions |
| NetBox – status | GET | `/api/v1/integrations/netbox/status` | Object counts, last sync times |
| NetBox – status live | SSE | `/api/v1/integrations/netbox/status/stream` | Real-time sync counters |
| NetBox – push now | POST | `/api/v1/integrations/netbox/sync/push` | Optional body: `{ types: [...], dry_run: bool }` |
| NetBox – pull now | POST | `/api/v1/integrations/netbox/sync/pull` | Optional body: `{ types: [...], dry_run: bool }` |
| NetBox – pull preview | GET | `/api/v1/integrations/netbox/sync/preview` | Dry-run diff; no writes |
| NetBox – webhook receiver | POST | `/api/v1/integrations/netbox/webhook` | HMAC-SHA256 validated; no auth key needed |
| NetBox – objects list | GET | `/api/v1/integrations/netbox/objects` | `?status=conflict&type=device&direction=pull` |
| NetBox – object detail | GET | `/api/v1/integrations/netbox/objects/:id` | Full source + DNSDave data + conflict detail |
| NetBox – resolve conflict | POST | `/api/v1/integrations/netbox/objects/:id/resolve` | `{ action: "keep_dnsdave" \| "use_netbox" \| "skip" }` |
| NetBox – mappings | GET/POST/PUT/DELETE | `/api/v1/integrations/netbox/mappings` | VRF/site/tenant/tag → scope/zone/group |
| NetBox – activity log | GET | `/api/v1/integrations/netbox/log` | Paginated; filterable by direction/status/type |
| NetBox – activity live | SSE | `/api/v1/integrations/netbox/log/stream` | Real-time sync event stream |

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

### UI-v0.7 – NetBox Integration Page
- [ ] Connection tab: URL, token, mode (REST/Diode), test button, connection status card
- [ ] Push tab: object-type toggles, conflict strategy, lifecycle dropdowns, context (VRF/site/tenant/tags), push status card, `[Push Now]` + dry-run
- [ ] Pull tab: mode (webhook/poll/manual), object-type toggles, filter controls (tag/site/VRF/tenant), conflict strategy, FQDN template with live preview, pull status card, `[Pull Now]` + preview modal
- [ ] Mappings tab: VRF→Scope, Site→Zone, Tenant→Client Group sub-tabs; searchable dropdowns populated from live NetBox + DNSDave API
- [ ] Activity tab: virtualised live log (Web Worker ring buffer, same pattern as Query Log); direction/status/type filters; conflict resolution drawer
- [ ] Setup wizard: 6-step first-run flow; `[Generate NetBox Event Rules]` script download
- [ ] Sidebar `Integrations` nav group with NetBox status widget (● synced / ⚠ N conflicts)
- [ ] Dashboard `NetBox sync` card (last push/pull, object counts, sparkline)
- [ ] Conflict review badge on Pull tab header
- [ ] NetBox webhook URL display with copy button
- [ ] Token rotation flow (confirm modal; test before invalidating old token)

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
