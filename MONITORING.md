# DNSDave – Monitoring Stack

An optional, fully pre-configured observability stack that can be brought up alongside any DNSDave deployment. Everything is opt-in: DNSDave's core containers run and serve DNS/DHCP without the monitoring stack present. Adding the stack unlocks pre-built dashboards, alerting, and long-term log retention.

---

## Contents

1. [Architecture Overview](#1-architecture-overview)
2. [Quick Start (Docker Compose)](#2-quick-start-docker-compose)
3. [Stack Components](#3-stack-components)
4. [Prometheus Configuration](#4-prometheus-configuration)
5. [Alert Rules](#5-alert-rules)
6. [Grafana Dashboards](#6-grafana-dashboards)
7. [Loki – Log Aggregation](#7-loki--log-aggregation)
8. [Kubernetes Deployment](#8-kubernetes-deployment)
9. [Metrics Reference](#9-metrics-reference)
10. [Log Query Reference (LogQL)](#10-log-query-reference-logql)
11. [Customisation](#11-customisation)

---

## 1. Architecture Overview

```mermaid
graph TD
    subgraph DNSDave containers
        DNS["dnsdave-dns\n:9090/metrics"]
        DHCP["dnsdave-dhcp\n:9093/metrics"]
        API["dnsdave-api\n:8080/metrics"]
        SYNC["dnsdave-sync\n:9091/metrics"]
        LOG["dnsdave-log\n:9092/metrics"]
        STATS["dnsdave-stats\n:9094/metrics"]
        EXP["dnsdave-export\n:9096/metrics"]
        NB["dnsdave-netbox\n:9095/metrics"]
        NATS["NATS JetStream\n:8222 monitoring"]
    end

    subgraph Host
        NE["node-exporter\n:9100/metrics"]
        CA["cAdvisor\n:8081/metrics"]
        PGE["postgres-exporter\n:9187/metrics"]
    end

    subgraph Monitoring Stack
        PROM["Prometheus\n:9090"]
        AM["Alertmanager\n:9093"]
        LOKI["Loki\n:3100"]
        PT["Promtail\n(log collector)"]
        GRAF["Grafana\n:3000"]
        OC["OTel Collector\n:4317 (optional)"]
    end

    DNS & DHCP & API & SYNC & LOG & STATS & EXP & NB & NATS --> PROM
    NE & CA & PGE --> PROM
    PROM --> AM
    PROM --> GRAF
    LOKI --> GRAF
    PT --> LOKI
    EXP -->|"Loki push API"| LOKI
    EXP -->|"OTLP"| OC
    OC --> PROM & LOKI
    AM -->|"email / Slack / PagerDuty"| NOTIFY["Notification channels"]
```

**Data flows:**

| Signal type | Source | Collector | Storage | Dashboard |
|---|---|---|---|---|
| Metrics | `/metrics` on each container | Prometheus scrape | Prometheus TSDB | Grafana |
| Container logs | stdout / stderr | Promtail → Loki | Loki | Grafana Explore |
| DNS query events | `dnsdave-export` → Loki push | - | Loki | Query Log Explorer dashboard |
| DHCP events | `dnsdave-export` → Loki push | - | Loki | DHCP dashboard |
| Traces (optional) | `dnsdave-export` → OTLP | OTel Collector | Tempo | Grafana Explore |
| Host metrics | node-exporter | Prometheus scrape | Prometheus TSDB | System Resources dashboard |
| Container metrics | cAdvisor | Prometheus scrape | Prometheus TSDB | System Resources dashboard |
| Postgres metrics | postgres-exporter | Prometheus scrape | Prometheus TSDB | System Resources dashboard |

---

## 2. Quick Start (Docker Compose)

The monitoring stack is a separate Compose file that joins the same Docker network as the core DNSDave stack.

```bash
# Start the core DNSDave stack first (or alongside)
docker compose -f docker-compose.yml up -d

# Attach the monitoring stack
docker compose \
  -f docker-compose.yml \
  -f docker-compose.monitoring.yml \
  up -d

# Open Grafana
open http://localhost:3000
# Default credentials: admin / changeme  (change immediately)
```

`docker-compose.monitoring.yml`:

```yaml
# docker-compose.monitoring.yml
# Attach to an existing dnsdave network – adjust network name if needed.

networks:
  dnsdave:
    external: true

volumes:
  prometheus_data: {}
  grafana_data: {}
  loki_data: {}
  alertmanager_data: {}

services:

  # ── Prometheus ─────────────────────────────────────────────────────────
  prometheus:
    image: prom/prometheus:v2.52.0
    container_name: dnsdave-prometheus
    restart: unless-stopped
    command:
      - --config.file=/etc/prometheus/prometheus.yml
      - --storage.tsdb.path=/prometheus
      - --storage.tsdb.retention.time=30d
      - --web.enable-lifecycle
      - --web.enable-admin-api
    volumes:
      - ./deploy/monitoring/prometheus/prometheus.yml:/etc/prometheus/prometheus.yml:ro
      - ./deploy/monitoring/prometheus/rules/:/etc/prometheus/rules/:ro
      - prometheus_data:/prometheus
    ports:
      - "9090:9090"
    networks:
      - dnsdave

  # ── Alertmanager ───────────────────────────────────────────────────────
  alertmanager:
    image: prom/alertmanager:v0.27.0
    container_name: dnsdave-alertmanager
    restart: unless-stopped
    command:
      - --config.file=/etc/alertmanager/alertmanager.yml
      - --storage.path=/alertmanager
    volumes:
      - ./deploy/monitoring/alertmanager/alertmanager.yml:/etc/alertmanager/alertmanager.yml:ro
      - alertmanager_data:/alertmanager
    ports:
      - "9093:9093"
    networks:
      - dnsdave

  # ── Loki ───────────────────────────────────────────────────────────────
  loki:
    image: grafana/loki:3.0.0
    container_name: dnsdave-loki
    restart: unless-stopped
    command: -config.file=/etc/loki/loki.yml
    volumes:
      - ./deploy/monitoring/loki/loki.yml:/etc/loki/loki.yml:ro
      - loki_data:/loki
    ports:
      - "3100:3100"
    networks:
      - dnsdave

  # ── Promtail – Docker log collector ────────────────────────────────────
  promtail:
    image: grafana/promtail:3.0.0
    container_name: dnsdave-promtail
    restart: unless-stopped
    command: -config.file=/etc/promtail/promtail.yml
    volumes:
      - ./deploy/monitoring/promtail/promtail.yml:/etc/promtail/promtail.yml:ro
      - /var/run/docker.sock:/var/run/docker.sock:ro
      - /var/lib/docker/containers:/var/lib/docker/containers:ro
    networks:
      - dnsdave
    depends_on:
      - loki

  # ── Grafana ────────────────────────────────────────────────────────────
  grafana:
    image: grafana/grafana-oss:11.1.0
    container_name: dnsdave-grafana
    restart: unless-stopped
    environment:
      GF_SECURITY_ADMIN_USER: admin
      GF_SECURITY_ADMIN_PASSWORD: "${GRAFANA_ADMIN_PASSWORD:-changeme}"
      GF_SERVER_ROOT_URL: "${GRAFANA_ROOT_URL:-http://localhost:3000}"
      GF_FEATURE_TOGGLES_ENABLE: "ngalert"
      GF_ALERTING_ENABLED: "true"
    volumes:
      - grafana_data:/var/lib/grafana
      - ./deploy/monitoring/grafana/provisioning:/etc/grafana/provisioning:ro
      - ./deploy/monitoring/grafana/dashboards:/var/lib/grafana/dashboards:ro
    ports:
      - "3000:3000"
    networks:
      - dnsdave
    depends_on:
      - prometheus
      - loki

  # ── node-exporter – host metrics ───────────────────────────────────────
  node-exporter:
    image: prom/node-exporter:v1.8.1
    container_name: dnsdave-node-exporter
    restart: unless-stopped
    command:
      - --path.rootfs=/host
      - --collector.filesystem.mount-points-exclude=^/(sys|proc|dev|host|etc)($$|/)
    volumes:
      - /:/host:ro,rslave
    pid: host
    network_mode: host   # required for accurate host metrics
    ports:
      - "9100:9100"

  # ── cAdvisor – container metrics ───────────────────────────────────────
  cadvisor:
    image: gcr.io/cadvisor/cadvisor:v0.49.1
    container_name: dnsdave-cadvisor
    restart: unless-stopped
    privileged: true
    devices:
      - /dev/kmsg
    volumes:
      - /:/rootfs:ro
      - /var/run:/var/run:ro
      - /sys:/sys:ro
      - /var/lib/docker/:/var/lib/docker:ro
      - /dev/disk/:/dev/disk:ro
    ports:
      - "8081:8080"
    networks:
      - dnsdave

  # ── postgres-exporter ──────────────────────────────────────────────────
  postgres-exporter:
    image: prometheuscommunity/postgres-exporter:v0.15.0
    container_name: dnsdave-postgres-exporter
    restart: unless-stopped
    environment:
      DATA_SOURCE_NAME: "${POSTGRES_EXPORTER_DSN:-postgresql://dnsdave:changeme@postgres:5432/dnsdave?sslmode=disable}"
    ports:
      - "9187:9187"
    networks:
      - dnsdave

  # ── OpenTelemetry Collector (optional – uncomment to enable) ───────────
  # otel-collector:
  #   image: otel/opentelemetry-collector-contrib:0.102.0
  #   container_name: dnsdave-otel-collector
  #   command: ["--config=/etc/otel/otel-collector.yml"]
  #   volumes:
  #     - ./deploy/monitoring/otel/otel-collector.yml:/etc/otel/otel-collector.yml:ro
  #   ports:
  #     - "4317:4317"   # OTLP gRPC
  #     - "4318:4318"   # OTLP HTTP
  #   networks:
  #     - dnsdave
```

---

## 3. Stack Components

| Component | Version | Purpose | Default port |
|---|---|---|---|
| **Prometheus** | 2.52 | Metrics scraping, storage, alerting rules | 9090 |
| **Alertmanager** | 0.27 | Alert routing (email, Slack, PagerDuty, webhook) | 9093 |
| **Loki** | 3.0 | Log aggregation and indexed log storage | 3100 |
| **Promtail** | 3.0 | Docker log collection → Loki | - |
| **Grafana** | 11.1 | Dashboards, alerting UI, log exploration | 3000 |
| **node-exporter** | 1.8 | Host OS metrics (CPU, memory, disk, network) | 9100 |
| **cAdvisor** | 0.49 | Per-container CPU, memory, network, disk I/O | 8081 |
| **postgres-exporter** | 0.15 | Postgres query latency, connection pool, table stats | 9187 |
| **OTel Collector** | 0.102 | Optional OTLP receiver for traces; fan-out to Tempo | 4317/4318 |

All images are pinned to specific versions. Update by editing `docker-compose.monitoring.yml` and re-pulling.

---

## 4. Prometheus Configuration

### `deploy/monitoring/prometheus/prometheus.yml`

```yaml
global:
  scrape_interval:     15s
  evaluation_interval: 15s
  scrape_timeout:      10s
  external_labels:
    cluster: 'dnsdave-local'

alerting:
  alertmanagers:
    - static_configs:
        - targets: ['alertmanager:9093']

rule_files:
  - /etc/prometheus/rules/*.yml

scrape_configs:

  # ── DNSDave containers ─────────────────────────────────────────────────

  - job_name: 'dnsdave-dns'
    static_configs:
      - targets: ['dnsdave-dns:9090']
    relabel_configs:
      - source_labels: [__address__]
        target_label: instance
        replacement: 'dns-node-1'

  - job_name: 'dnsdave-dhcp'
    static_configs:
      - targets: ['dnsdave-dhcp:9093']

  - job_name: 'dnsdave-api'
    static_configs:
      - targets: ['dnsdave-api:8080']
    metrics_path: /metrics

  - job_name: 'dnsdave-sync'
    static_configs:
      - targets: ['dnsdave-sync:9091']

  - job_name: 'dnsdave-log'
    static_configs:
      - targets: ['dnsdave-log:9092']

  - job_name: 'dnsdave-stats'
    static_configs:
      - targets: ['dnsdave-stats:9094']

  - job_name: 'dnsdave-export'
    static_configs:
      - targets: ['dnsdave-export:9096']

  - job_name: 'dnsdave-netbox'
    static_configs:
      - targets: ['dnsdave-netbox:9095']

  # ── NATS via nats-surveyor ─────────────────────────────────────────────
  - job_name: 'nats'
    static_configs:
      - targets: ['nats:8222']
    metrics_path: /metrics

  # ── Infrastructure ─────────────────────────────────────────────────────
  - job_name: 'node-exporter'
    static_configs:
      - targets: ['host.docker.internal:9100']

  - job_name: 'cadvisor'
    static_configs:
      - targets: ['cadvisor:8080']

  - job_name: 'postgres'
    static_configs:
      - targets: ['postgres-exporter:9187']

  # ── Prometheus self-monitoring ─────────────────────────────────────────
  - job_name: 'prometheus'
    static_configs:
      - targets: ['localhost:9090']
```

### Multi-node configuration

For clusters with multiple DNS nodes, replace the static `dnsdave-dns` target with a file-based or DNS-based service discovery block:

```yaml
  - job_name: 'dnsdave-dns'
    file_sd_configs:
      - files:
          - /etc/prometheus/targets/dns-nodes.yml
        refresh_interval: 30s
```

`dns-nodes.yml` is updated by `dnsdave-api` via the Prometheus file SD API whenever a cluster node joins or leaves.

---

## 5. Alert Rules

All rules live in `deploy/monitoring/prometheus/rules/`. They are split into topic files for maintainability.

### `rules/dns.yml`

```yaml
groups:
  - name: dnsdave.dns
    interval: 30s
    rules:

      - alert: DnsHighQueryLatencyP99
        expr: |
          histogram_quantile(0.99,
            rate(dnsdave_dns_query_duration_us_bucket[5m])
          ) > 10000
        for: 2m
        labels:
          severity: warning
          team: network
        annotations:
          summary: "DNS p99 latency above 10 ms"
          description: "p99 query latency is {{ $value | humanizeDuration }} on {{ $labels.instance }}."
          runbook: "https://docs.dnsdave.io/runbooks/dns-latency"

      - alert: DnsHighQueryLatencyP99Critical
        expr: |
          histogram_quantile(0.99,
            rate(dnsdave_dns_query_duration_us_bucket[5m])
          ) > 50000
        for: 1m
        labels:
          severity: critical
        annotations:
          summary: "DNS p99 latency above 50 ms – service degraded"

      - alert: DnsHighErrorRate
        expr: |
          rate(dnsdave_dns_queries_total{response_type="error"}[5m])
          / rate(dnsdave_dns_queries_total[5m]) > 0.01
        for: 3m
        labels:
          severity: warning
        annotations:
          summary: "DNS error rate above 1%"
          description: "{{ $value | humanizePercentage }} of queries are returning SERVFAIL."

      - alert: DnsAllUpstreamsDown
        expr: |
          sum(dnsdave_dns_upstream_healthy) == 0
        for: 30s
        labels:
          severity: critical
        annotations:
          summary: "All DNS upstream resolvers are unhealthy"

      - alert: DnsDnssecValidationFailures
        expr: |
          rate(dnsdave_dns_dnssec_validation_failures_total[5m]) > 0.1
        for: 2m
        labels:
          severity: warning
        annotations:
          summary: "DNSSEC validation failures detected"

      - alert: DnsContainerDown
        expr: up{job="dnsdave-dns"} == 0
        for: 1m
        labels:
          severity: critical
        annotations:
          summary: "dnsdave-dns container is down on {{ $labels.instance }}"
```

### `rules/dhcp.yml`

```yaml
groups:
  - name: dnsdave.dhcp
    rules:

      - alert: DhcpPoolUtilisationHigh
        expr: dnsdave_dhcp_pool_utilisation > 0.80
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "DHCP pool {{ $labels.scope_id }} above 80% utilisation"
          description: "{{ $value | humanizePercentage }} of addresses in use. Consider expanding the pool."

      - alert: DhcpPoolUtilisationCritical
        expr: dnsdave_dhcp_pool_utilisation > 0.95
        for: 1m
        labels:
          severity: critical
        annotations:
          summary: "DHCP pool {{ $labels.scope_id }} above 95% – imminent exhaustion"

      - alert: DhcpPoolExhausted
        expr: dnsdave_dhcp_pool_utilisation >= 1.0
        for: 0m
        labels:
          severity: critical
        annotations:
          summary: "DHCP pool {{ $labels.scope_id }} is EXHAUSTED"

      - alert: DhcpContainerDown
        expr: up{job="dnsdave-dhcp"} == 0
        for: 1m
        labels:
          severity: critical
        annotations:
          summary: "dnsdave-dhcp is down – no new leases will be issued"

      - alert: DhcpHighNakRate
        expr: |
          rate(dnsdave_dhcp_nak_total[5m]) > 5
        for: 3m
        labels:
          severity: warning
        annotations:
          summary: "High DHCP NAK rate on scope {{ $labels.scope_id }}"
          description: "{{ $value | humanize }} NAKs/s – possibly rogue DISCOVER or stale leases."
```

### `rules/blocklist.yml`

```yaml
groups:
  - name: dnsdave.blocklist
    rules:

      - alert: BlocklistSyncFailed
        expr: increase(dnsdave_sync_errors_total[1h]) > 3
        for: 0m
        labels:
          severity: warning
        annotations:
          summary: "Blocklist sync for list {{ $labels.list_id }} has failed 3+ times in 1 hour"

      - alert: BlocklistDomainCountDropped
        expr: |
          (dnsdave_sync_domains_total - dnsdave_sync_domains_total offset 1h)
          / dnsdave_sync_domains_total offset 1h < -0.20
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Blocklist {{ $labels.list_id }} domain count dropped >20% – list may be empty or corrupted"

      - alert: SyncContainerDown
        expr: up{job="dnsdave-sync"} == 0
        for: 2m
        labels:
          severity: warning
        annotations:
          summary: "dnsdave-sync is down – blocklists will not be updated"
```

### `rules/infrastructure.yml`

```yaml
groups:
  - name: dnsdave.infrastructure
    rules:

      - alert: ApiContainerDown
        expr: up{job="dnsdave-api"} == 0
        for: 1m
        labels:
          severity: critical
        annotations:
          summary: "dnsdave-api is down – UI and all API clients will be unavailable"

      - alert: NatsConnectionLost
        expr: nats_varz_connections == 0
        for: 30s
        labels:
          severity: critical
        annotations:
          summary: "NATS has no connected clients – entire event bus is isolated"

      - alert: NatsStreamLagHigh
        expr: nats_consumer_num_pending > 10000
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "NATS consumer {{ $labels.consumer_name }} has {{ $value }} pending messages"

      - alert: PostgresDown
        expr: pg_up == 0
        for: 30s
        labels:
          severity: critical
        annotations:
          summary: "Postgres is unreachable – API and log containers will fail"

      - alert: PostgresConnectionsHigh
        expr: |
          pg_stat_activity_count{state="active"}
          / pg_settings_max_connections > 0.80
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Postgres connection pool above 80%"

      - alert: PostgresReplicationLag
        expr: pg_replication_lag > 30
        for: 2m
        labels:
          severity: warning
        annotations:
          summary: "Postgres replication lag is {{ $value | humanizeDuration }}"

      - alert: CertificateExpirySoon
        expr: dnsdave_tls_cert_expiry_seconds < 2592000   # 30 days
        for: 0m
        labels:
          severity: warning
        annotations:
          summary: "TLS certificate for {{ $labels.domain }} expires in < 30 days"

      - alert: CertificateExpiryCritical
        expr: dnsdave_tls_cert_expiry_seconds < 604800   # 7 days
        for: 0m
        labels:
          severity: critical
        annotations:
          summary: "TLS certificate for {{ $labels.domain }} expires in < 7 days"

      - alert: NodeHighCpuUsage
        expr: |
          100 - (avg by(instance)(rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100) > 90
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Host {{ $labels.instance }} CPU above 90%"

      - alert: NodeHighMemoryUsage
        expr: |
          (1 - (node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes)) > 0.90
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Host {{ $labels.instance }} memory above 90%"

      - alert: NodeDiskAlmostFull
        expr: |
          (node_filesystem_avail_bytes{fstype!~"tmpfs|fuse.lxcfs"}
           / node_filesystem_size_bytes) < 0.10
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Disk {{ $labels.mountpoint }} on {{ $labels.instance }} below 10% free"
```

### `rules/security.yml`

```yaml
groups:
  - name: dnsdave.security
    rules:

      - alert: AuthFailureSpike
        expr: |
          rate(dnsdave_api_auth_failures_total[5m]) > 5
        for: 1m
        labels:
          severity: warning
        annotations:
          summary: "High authentication failure rate – possible brute-force attempt"

      - alert: UserAccountLocked
        expr: increase(dnsdave_iam_lockouts_total[5m]) > 0
        for: 0m
        labels:
          severity: info
        annotations:
          summary: "User account locked out due to repeated authentication failures"

      - alert: NetboxSyncFailing
        expr: |
          rate(dnsdave_netbox_pushes_total{status="error"}[15m]) > 0.1
        for: 10m
        labels:
          severity: warning
        annotations:
          summary: "NetBox sync is experiencing persistent errors"

      - alert: NetboxHighConflicts
        expr: |
          rate(dnsdave_netbox_conflicts_total[1h]) > 10
        for: 0m
        labels:
          severity: info
        annotations:
          summary: "High NetBox sync conflict rate – review conflict strategy"
```

### `deploy/monitoring/alertmanager/alertmanager.yml`

```yaml
global:
  resolve_timeout: 5m
  smtp_smarthost: 'smtp.example.com:587'
  smtp_from: 'dnsdave-alerts@example.com'
  smtp_auth_username: 'alerts@example.com'
  smtp_auth_password: '${SMTP_PASSWORD}'

route:
  receiver: 'default'
  group_by: ['alertname', 'cluster']
  group_wait: 30s
  group_interval: 5m
  repeat_interval: 4h

  routes:
    - match:
        severity: critical
      receiver: 'pagerduty'
      continue: true

    - match:
        severity: critical
      receiver: 'slack-critical'

    - match:
        severity: warning
      receiver: 'slack-warnings'

receivers:
  - name: 'default'
    email_configs:
      - to: 'network-team@example.com'
        send_resolved: true

  - name: 'slack-critical'
    slack_configs:
      - api_url: '${SLACK_WEBHOOK_CRITICAL}'
        channel: '#dnsdave-critical'
        title: '🚨 {{ .GroupLabels.alertname }}'
        text: '{{ range .Alerts }}{{ .Annotations.summary }}{{ end }}'
        send_resolved: true

  - name: 'slack-warnings'
    slack_configs:
      - api_url: '${SLACK_WEBHOOK_WARNINGS}'
        channel: '#dnsdave-alerts'
        title: '⚠️ {{ .GroupLabels.alertname }}'
        text: '{{ range .Alerts }}{{ .Annotations.summary }}{{ end }}'

  - name: 'pagerduty'
    pagerduty_configs:
      - routing_key: '${PAGERDUTY_ROUTING_KEY}'
        description: '{{ .GroupLabels.alertname }}: {{ range .Alerts }}{{ .Annotations.summary }}{{ end }}'

inhibit_rules:
  - source_match:
      severity: 'critical'
    target_match:
      severity: 'warning'
    equal: ['alertname', 'cluster', 'instance']
```

---

## 6. Grafana Dashboards

Ten pre-built dashboards are provisioned automatically. They live in `deploy/monitoring/grafana/dashboards/` as JSON files, loaded by Grafana's provisioning subsystem on startup. No manual import is needed.

### Dashboard provisioning configuration

`deploy/monitoring/grafana/provisioning/dashboards/dnsdave.yml`:

```yaml
apiVersion: 1

providers:
  - name: 'DNSDave'
    folder: 'DNSDave'
    type: file
    disableDeletion: false
    updateIntervalSeconds: 30
    options:
      path: /var/lib/grafana/dashboards
```

`deploy/monitoring/grafana/provisioning/datasources/datasources.yml`:

```yaml
apiVersion: 1

datasources:
  - name: Prometheus
    type: prometheus
    access: proxy
    url: http://prometheus:9090
    isDefault: true
    jsonData:
      httpMethod: POST
      exemplarTraceIdDestinations:
        - name: trace_id
          datasourceUid: tempo

  - name: Loki
    type: loki
    access: proxy
    url: http://loki:3100
    jsonData:
      derivedFields:
        - name: trace_id
          matcherRegex: '"trace_id":"(\w+)"'
          url: '$${__value.raw}'
          datasourceUid: tempo

  # Uncomment if Tempo is running:
  # - name: Tempo
  #   type: tempo
  #   access: proxy
  #   url: http://tempo:3200
  #   uid: tempo
```

---

### Dashboard 1 – DNSDave Overview

**File:** `dns-overview.json`

The entry-point dashboard for at-a-glance status. Intended for a wall display or NOC screen.

**Panels:**

| Panel | Type | Query |
|---|---|---|
| Queries per second (QPS) | Stat + sparkline | `rate(dnsdave_dns_queries_total[1m])` |
| Block rate | Stat + trend | `rate(dnsdave_dns_blocked_total[1m]) / rate(dnsdave_dns_queries_total[1m])` |
| Cache hit rate | Stat + gauge | `rate(dnsdave_dns_cache_hits_total[5m]) / rate(dnsdave_dns_queries_total[5m])` |
| p50 / p95 / p99 latency | Stat multi | `histogram_quantile(0.99, rate(dnsdave_dns_query_duration_us_bucket[5m]))` |
| Active DHCP leases | Stat | `sum(dnsdave_dhcp_leases_active)` |
| DHCP pool utilisation (worst scope) | Gauge | `max(dnsdave_dhcp_pool_utilisation)` |
| Container health matrix | State timeline | `up{job=~"dnsdave-.*"}` |
| Upstream resolver health | State timeline | `dnsdave_dns_upstream_healthy` |
| Active alerts | Alert list panel | Current firing alerts from Alertmanager |
| Queries over time (stacked by result type) | Time series | `rate(dnsdave_dns_queries_total[2m])` by `response_type` |

---

### Dashboard 2 – DNS Performance

**File:** `dns-performance.json`

Detailed resolution performance for capacity planning and tuning.

**Panels:**

| Panel | Type | Notes |
|---|---|---|
| Query latency heatmap | Heatmap | `dnsdave_dns_query_duration_us_bucket` |
| Latency percentiles over time | Time series | p50, p95, p99, p999 |
| Queries by response type over time | Time series | allowed / blocked / cached / upstream / NXDOMAIN / error |
| Top 20 queried domains (last 1h) | Table | Loki LogQL aggregation |
| Top 20 blocked domains (last 1h) | Table | Loki LogQL aggregation |
| Upstream resolver latency per resolver | Time series | `dnsdave_dns_upstream_latency_us` |
| Upstream error rate per resolver | Time series | `rate(dnsdave_dns_upstream_errors_total[5m])` |
| Cache hit rate over time | Time series | 24h view |
| Bloom filter false-positive rate | Time series | `rate(dnsdave_dns_bloom_false_positives_total[5m])` |
| DNSSEC validation failures | Time series | `rate(dnsdave_dns_dnssec_validation_failures_total[5m])` |
| Forward zone hits per zone | Bar chart | `rate(dnsdave_dns_forward_zone_hits_total[5m])` by `zone` |
| DNS query rate by client group | Time series | `rate(dnsdave_dns_queries_total[2m])` by `client_group` |

---

### Dashboard 3 – DHCP Dashboard

**File:** `dhcp.json`

DHCP scope health, lease lifecycle, and pool utilisation.

**Panels:**

| Panel | Type | Notes |
|---|---|---|
| Pool utilisation gauges (one per scope) | Gauge array | `dnsdave_dhcp_pool_utilisation` by `scope_id` |
| Active leases per scope | Bar chart | `dnsdave_dhcp_leases_active` |
| Lease events over time (acquire/release/expire) | Time series | `rate(dnsdave_dhcp_leases_total[5m])` by `event_type` |
| DHCP offer latency | Time series | `histogram_quantile(0.95, rate(dnsdave_dhcp_offer_latency_us_bucket[5m]))` |
| NAK rate per scope | Time series | `rate(dnsdave_dhcp_nak_total[5m])` |
| Dynamic DNS registrations | Counter | `rate(dnsdave_dhcp_dynamic_dns_registrations_total[5m])` |
| Lease expiry timeline (next 24h) | Bar chart | From Loki log events |
| Pool utilisation over time (all scopes) | Time series | 7-day trend |
| DHCPv6 leases | Stat | `dnsdave_dhcpv6_leases_active` |

---

### Dashboard 4 – Blocklist Analytics

**File:** `blocklists.json`

Blocklist health, domain counts, and sync schedules.

**Panels:**

| Panel | Type | Notes |
|---|---|---|
| Total blocked domains | Stat | `sum(dnsdave_sync_domains_total)` |
| Domains per list | Bar chart | `dnsdave_sync_domains_total` by `list_id` |
| Domain count trend (all lists) | Time series | 30-day view; spike/drop detection |
| Last successful sync per list | Table | Derived from `dnsdave_sync_last_success_timestamp` |
| Sync duration per list | Bar chart | `dnsdave_sync_duration_s` |
| Sync error rate | Time series | `rate(dnsdave_sync_errors_total[1h])` |
| Blocked queries over time | Time series | `rate(dnsdave_dns_blocked_total[5m])` |
| Block rate per client group | Bar chart | By `client_group` label |

---

### Dashboard 5 – Client Activity

**File:** `client-activity.json`

Per-client and per-group query behaviour.

**Panels:**

| Panel | Type | Notes |
|---|---|---|
| Top 20 clients by query volume | Table | Loki LogQL aggregation on `client_ip` |
| Top 20 clients by block rate | Table | Loki LogQL aggregation |
| Query volume per client group | Time series | `dnsdave_dns_queries_total` by `client_group` |
| Block rate per client group | Time series | |
| Query volume heatmap (client × hour) | Heatmap | Loki |
| Client first-seen / last-seen timeline | Timeline | From DHCP lease log |

---

### Dashboard 6 – System Resources

**File:** `system-resources.json`

Host OS and container-level resource consumption.

**Panels:**

| Panel | Type | Notes |
|---|---|---|
| CPU usage per container | Time series | `rate(container_cpu_usage_seconds_total[5m])` by `name` |
| Memory RSS per container | Time series | `container_memory_rss` by `name` |
| Network I/O (DNS container) | Time series | `rate(container_network_receive_bytes_total[5m])` |
| Disk read/write (Postgres container) | Time series | `rate(container_blkio_device_usage_total[5m])` |
| Host CPU (all cores) | Time series | node-exporter |
| Host memory breakdown | Stacked area | used / cached / free |
| Host disk usage per mount | Gauge | `node_filesystem_avail_bytes` |
| Postgres active connections | Time series | `pg_stat_activity_count` |
| Postgres query latency | Time series | `pg_stat_statements_seconds` (requires `pg_stat_statements`) |
| NATS message rate | Time series | `nats_varz_in_msgs_total`, `nats_varz_out_msgs_total` |
| NATS JetStream storage used | Gauge | `nats_jsz_storage_used` |

---

### Dashboard 7 – Cluster Health

**File:** `cluster-health.json`

Multi-node status for HA deployments.

**Panels:**

| Panel | Type | Notes |
|---|---|---|
| Node status matrix | State timeline | `up{job="dnsdave-dns"}` by `instance` |
| QPS per node | Time series | `rate(dnsdave_dns_queries_total[1m])` by `instance` |
| Latency spread across nodes | Time series | p99 per node – should be near-identical |
| NATS consumer lag per node | Time series | `nats_consumer_num_pending` by `instance` |
| Config propagation lag | Histogram | `dnsdave_config_propagation_duration_ms` |
| Cluster leader indicator | Stat | `dnsdave_dhcp_is_leader` (1 = leader, 0 = standby) |

---

### Dashboard 8 – NetBox Integration

**File:** `netbox-sync.json`

Push/pull sync health and object coverage.

**Panels:**

| Panel | Type | Notes |
|---|---|---|
| Push rate (success / conflict / error) | Time series | `rate(dnsdave_netbox_pushes_total[5m])` by `status` |
| Pull rate | Time series | `rate(dnsdave_netbox_pulls_total[5m])` |
| NATS consumer lag (NetBox) | Gauge | `dnsdave_netbox_nats_lag` |
| Total synced objects | Stat | `dnsdave_netbox_objects_synced_total` by `object_type` |
| Conflict count by type | Bar chart | `dnsdave_netbox_conflicts_total` by `object_type` |
| Push latency to NetBox API / Diode | Histogram | `dnsdave_netbox_push_duration_ms` |
| Device auto-create rate | Time series | `rate(dnsdave_netbox_devices_created_total[1h])` |
| Last successful sync timestamp | Stat | `dnsdave_netbox_last_sync_timestamp` |

---

### Dashboard 9 – Security and Audit

**File:** `security-audit.json`

Authentication events, permission errors, and API key usage.

**Panels:**

| Panel | Type | Notes |
|---|---|---|
| Authentication failures over time | Time series | `rate(dnsdave_api_auth_failures_total[5m])` |
| Account lockouts | Stat + timeline | `dnsdave_iam_lockouts_total` |
| Permission denied (403) rate | Time series | `rate(dnsdave_api_requests_total{status="403"}[5m])` |
| API key usage by key name | Table | Loki LogQL on audit log |
| Top users by write operations | Table | Loki LogQL on audit log |
| Audit events (live Loki tail) | Logs panel | `{container="dnsdave-api"} |= "audit"` |
| TLS certificate expiry calendar | Table | `dnsdave_tls_cert_expiry_seconds` by `domain` |

---

### Dashboard 10 – Query Log Explorer

**File:** `query-log-explorer.json`

Loki-backed interactive query explorer. Complements the in-UI log viewer for longer retention periods and ad-hoc analysis.

**Panels:**

| Panel | Type | Notes |
|---|---|---|
| Live query stream | Logs panel | `{container="dnsdave-dns"} \| json \| line_format "{{.domain}} {{.qtype}} {{.result}} {{.latency_us}}µs"` |
| Query volume over time | Time series | Derived from log volume |
| Filters (label filters) | Dashboard variables | `client_ip`, `qtype`, `response_type`, `domain` (regex) |
| Slow queries (>100ms) | Logs panel | `{container="dnsdave-dns"} \| json \| latency_us > 100000` |
| Blocked queries | Logs panel | `{container="dnsdave-dns"} \| json \| response_type="blocked"` |
| NXDOMAIN queries | Logs panel | `{container="dnsdave-dns"} \| json \| response_type="nxdomain"` |
| Queries from a specific client | Logs panel | Driven by `client_ip` variable |

---

## 7. Loki – Log Aggregation

### `deploy/monitoring/loki/loki.yml`

```yaml
auth_enabled: false

server:
  http_listen_port: 3100
  grpc_listen_port: 9096

common:
  path_prefix: /loki
  storage:
    filesystem:
      chunks_directory: /loki/chunks
      rules_directory: /loki/rules
  replication_factor: 1
  ring:
    instance_addr: 127.0.0.1
    kvstore:
      store: inmemory

schema_config:
  configs:
    - from: 2024-01-01
      store: tsdb
      object_store: filesystem
      schema: v13
      index:
        prefix: index_
        period: 24h

limits_config:
  retention_period: 90d
  ingestion_rate_mb: 16
  ingestion_burst_size_mb: 32
  max_query_series: 5000

compactor:
  working_directory: /loki/compactor
  compaction_interval: 10m
  retention_enabled: true
  retention_delete_delay: 2h
```

### `deploy/monitoring/promtail/promtail.yml`

```yaml
server:
  http_listen_port: 9080

positions:
  filename: /tmp/positions.yaml

clients:
  - url: http://loki:3100/loki/api/v1/push

scrape_configs:
  - job_name: docker
    docker_sd_configs:
      - host: unix:///var/run/docker.sock
        refresh_interval: 5s
        filters:
          - name: label
            values: ["com.docker.compose.project"]
    relabel_configs:
      - source_labels: ['__meta_docker_container_name']
        regex: '/(.*)'
        target_label: container
      - source_labels: ['__meta_docker_container_label_com_docker_compose_service']
        target_label: service
      - source_labels: ['__meta_docker_container_label_com_docker_compose_project']
        target_label: compose_project
    pipeline_stages:
      - json:
          expressions:
            level: level
            msg: msg
            domain: domain
            qtype: qtype
            result: result
            latency_us: latency_us
            client_ip: client_ip
            trace_id: trace_id
      - labels:
          level:
          result:
      - timestamp:
          source: ts
          format: RFC3339Nano
```

Promtail auto-discovers all Docker containers by label and parses the DNSDave structured JSON log format, promoting key fields to Loki labels for fast indexed filtering.

---

## 8. Kubernetes Deployment

For Kubernetes, the recommended approach is the [kube-prometheus-stack](https://github.com/prometheus-community/helm-charts/tree/main/charts/kube-prometheus-stack) Helm chart (which bundles Prometheus Operator, Grafana, Alertmanager, node-exporter, and kube-state-metrics) combined with the [Grafana Loki stack](https://github.com/grafana/loki/tree/main/production/helm/loki).

### Quick install

```bash
# Add Helm repos
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo add grafana https://grafana.github.io/helm-charts
helm repo update

# Install kube-prometheus-stack
helm install monitoring prometheus-community/kube-prometheus-stack \
  --namespace monitoring --create-namespace \
  --values deploy/k8s/monitoring/kube-prometheus-stack-values.yml

# Install Loki stack
helm install loki grafana/loki-stack \
  --namespace monitoring \
  --values deploy/k8s/monitoring/loki-values.yml
```

### `ServiceMonitor` for DNSDave containers

The Prometheus Operator uses `ServiceMonitor` CRDs to define scrape targets without editing `prometheus.yml`. Each DNSDave container needs a corresponding `ServiceMonitor`:

```yaml
# deploy/k8s/monitoring/servicemonitors.yml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: dnsdave-dns
  namespace: dnsdave
  labels:
    release: monitoring
spec:
  selector:
    matchLabels:
      app: dnsdave-dns
  endpoints:
    - port: metrics
      interval: 15s
      path: /metrics
---
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: dnsdave-dhcp
  namespace: dnsdave
  labels:
    release: monitoring
spec:
  selector:
    matchLabels:
      app: dnsdave-dhcp
  endpoints:
    - port: metrics
      interval: 15s
---
# Repeat for dnsdave-api, dnsdave-sync, dnsdave-log,
# dnsdave-stats, dnsdave-export, dnsdave-netbox
```

### `PrometheusRule` for alert rules

```yaml
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: dnsdave-alerts
  namespace: dnsdave
  labels:
    release: monitoring
spec:
  groups:
    - name: dnsdave.dns
      rules:
        - alert: DnsHighQueryLatencyP99
          expr: |
            histogram_quantile(0.99,
              rate(dnsdave_dns_query_duration_us_bucket[5m])
            ) > 10000
          for: 2m
          labels:
            severity: warning
          annotations:
            summary: "DNS p99 latency above 10 ms on {{ $labels.instance }}"
        # ... (remaining rules as above)
```

### Grafana dashboard `ConfigMap`

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: dnsdave-grafana-dashboards
  namespace: monitoring
  labels:
    grafana_dashboard: "1"
data:
  dns-overview.json: |
    # (paste dashboard JSON here, or use Helm values to reference a file)
```

With the `grafana_dashboard: "1"` label, Grafana's sidecar container automatically imports the dashboard from the ConfigMap without a restart.

### `kube-prometheus-stack-values.yml` (key extracts)

```yaml
grafana:
  adminPassword: "changeme"
  additionalDataSources:
    - name: Loki
      type: loki
      url: http://loki:3100
      access: proxy
  sidecar:
    dashboards:
      enabled: true
      label: grafana_dashboard

prometheus:
  prometheusSpec:
    retention: 30d
    storageSpec:
      volumeClaimTemplate:
        spec:
          accessModes: ["ReadWriteOnce"]
          resources:
            requests:
              storage: 50Gi
    additionalScrapeConfigs:
      - job_name: 'nats'
        static_configs:
          - targets: ['nats.dnsdave.svc.cluster.local:8222']
        metrics_path: /metrics

alertmanager:
  config:
    route:
      receiver: 'slack'
    receivers:
      - name: 'slack'
        slack_configs:
          - api_url: 'https://hooks.slack.com/services/...'
            channel: '#dnsdave-alerts'
```

---

## 9. Metrics Reference

Complete list of all Prometheus metrics exposed by DNSDave containers, consolidated from §12 of `DESIGN.md`.

### `dnsdave-dns` (`:9090/metrics`)

| Metric | Type | Labels | Description |
|---|---|---|---|
| `dnsdave_dns_queries_total` | Counter | `response_type`, `qtype` | Total queries processed |
| `dnsdave_dns_query_duration_us` | Histogram | `response_type` | Query processing time in microseconds |
| `dnsdave_dns_blocked_total` | Counter | - | Queries blocked by blocklist |
| `dnsdave_dns_cache_hits_total` | Counter | - | Answer cache hits |
| `dnsdave_dns_cache_misses_total` | Counter | - | Answer cache misses |
| `dnsdave_dns_bloom_false_positives_total` | Counter | - | Bloom filter false-positive checks |
| `dnsdave_dns_nats_publish_errors_total` | Counter | - | Failed NATS event publishes |
| `dnsdave_dns_forward_zone_hits_total` | Counter | `zone` | Queries forwarded per conditional zone |
| `dnsdave_dns_dnssec_validation_failures_total` | Counter | - | DNSSEC validation failures |
| `dnsdave_dns_upstream_healthy` | Gauge | `upstream_id` | 1 if upstream is reachable |
| `dnsdave_dns_upstream_latency_us` | Histogram | `upstream_id` | Upstream query round-trip time |
| `dnsdave_dns_upstream_errors_total` | Counter | `upstream_id`, `error` | Upstream resolution errors |
| `dnsdave_tls_cert_expiry_seconds` | Gauge | `domain` | Seconds until TLS certificate expires |
| `dnsdave_dns_affinity_resolution_total` | Counter | `result` (`matched`/`fallback`) | Network-affinity resolution outcomes |

### `dnsdave-dhcp` (`:9093/metrics`)

| Metric | Type | Labels | Description |
|---|---|---|---|
| `dnsdave_dhcp_leases_active` | Gauge | `scope_id` | Current active leases per scope |
| `dnsdave_dhcp_leases_total` | Counter | `scope_id`, `event_type` | Lease lifecycle events (acquire/release/expire/renew) |
| `dnsdave_dhcp_pool_utilisation` | Gauge | `scope_id` | Fraction of pool addresses in use (0.0–1.0) |
| `dnsdave_dhcp_offer_latency_us` | Histogram | `scope_id` | DISCOVER→OFFER processing time |
| `dnsdave_dhcp_nak_total` | Counter | `scope_id`, `reason` | DHCPNAK responses sent |
| `dnsdave_dhcp_dynamic_dns_registrations_total` | Counter | `scope_id` | Dynamic DNS updates sent on lease |
| `dnsdave_dhcp_relay_requests_total` | Counter | `scope_id` | Requests received via relay agent |
| `dnsdave_dhcpv6_leases_active` | Gauge | `scope_id` | Active DHCPv6 IA_NA leases |
| `dnsdave_dhcp_is_leader` | Gauge | - | 1 if this node is the active DHCP leader |

### `dnsdave-api` (`:8080/metrics`)

| Metric | Type | Labels | Description |
|---|---|---|---|
| `dnsdave_api_requests_total` | Counter | `method`, `path`, `status` | HTTP requests |
| `dnsdave_api_request_duration_ms` | Histogram | `method`, `path` | API handler latency |
| `dnsdave_api_auth_failures_total` | Counter | `reason` | Authentication failures |
| `dnsdave_iam_lockouts_total` | Counter | - | User account lockouts triggered |
| `dnsdave_api_active_sessions` | Gauge | - | Active JWT sessions |
| `dnsdave_api_ws_connections` | Gauge | - | Active SSE connections |

### `dnsdave-sync` (`:9091/metrics`)

| Metric | Type | Labels | Description |
|---|---|---|---|
| `dnsdave_sync_domains_total` | Gauge | `list_id` | Domain count per blocklist |
| `dnsdave_sync_duration_s` | Histogram | `list_id` | Full sync round-trip time |
| `dnsdave_sync_errors_total` | Counter | `list_id` | Sync fetch or parse errors |
| `dnsdave_sync_last_success_timestamp` | Gauge | `list_id` | Unix timestamp of last successful sync |

### `dnsdave-export` (`:9096/metrics`)

| Metric | Type | Labels | Description |
|---|---|---|---|
| `dnsdave_export_events_total` | Counter | `backend`, `status` | Events forwarded per backend |
| `dnsdave_export_event_latency_ms` | Histogram | `backend` | Time to deliver one event batch |
| `dnsdave_export_queue_depth` | Gauge | `backend` | In-memory queue depth per backend |
| `dnsdave_export_backend_errors_total` | Counter | `backend`, `error` | Delivery failures |

### `dnsdave-netbox` (`:9095/metrics`)

| Metric | Type | Labels | Description |
|---|---|---|---|
| `dnsdave_netbox_pushes_total` | Counter | `mode`, `object_type`, `status` | Push operations |
| `dnsdave_netbox_pulls_total` | Counter | `object_type`, `status` | Pull operations |
| `dnsdave_netbox_push_duration_ms` | Histogram | - | NetBox API / Diode round-trip latency |
| `dnsdave_netbox_nats_lag` | Gauge | - | Unacked NATS messages |
| `dnsdave_netbox_conflicts_total` | Counter | `strategy`, `object_type` | Sync conflicts |
| `dnsdave_netbox_devices_created_total` | Counter | - | Auto-created device records |
| `dnsdave_netbox_objects_synced_total` | Gauge | `object_type` | Total objects under sync management |
| `dnsdave_netbox_last_sync_timestamp` | Gauge | `direction` | Unix timestamp of last push/pull |

---

## 10. Log Query Reference (LogQL)

Useful LogQL queries for Grafana Explore and the Query Log Explorer dashboard.

### DNS queries

```logql
# All queries in the last 15 minutes
{container="dnsdave-dns"} | json

# Only blocked queries
{container="dnsdave-dns"} | json | response_type = "blocked"

# Slow queries (>100 ms)
{container="dnsdave-dns"} | json | latency_us > 100000

# Queries for a specific domain (substring match)
{container="dnsdave-dns"} | json | domain =~ ".*malware.*"

# Queries from a specific client
{container="dnsdave-dns"} | json | client_ip = "192.168.1.42"

# NXDOMAIN responses
{container="dnsdave-dns"} | json | response_type = "nxdomain"

# DNSSEC failures
{container="dnsdave-dns"} | json | level = "warn" | msg =~ ".*dnssec.*"

# Query volume over time (metric from logs)
sum by (response_type) (
  rate({container="dnsdave-dns"} | json | __error__="" [1m])
)
```

### DHCP events

```logql
# All DHCP lease events
{container="dnsdave-dhcp"} | json | msg =~ "lease.*"

# Leases for a specific MAC address
{container="dnsdave-dhcp"} | json | mac_address = "aa:bb:cc:dd:ee:ff"

# NAK events
{container="dnsdave-dhcp"} | json | msg =~ ".*nak.*"

# Leases in scope "LAN"
{container="dnsdave-dhcp"} | json | scope_id = "scope-lan"
```

### API and authentication

```logql
# All 4xx/5xx API responses
{container="dnsdave-api"} | json | status >= 400

# Authentication failures
{container="dnsdave-api"} | json | msg =~ ".*auth.*fail.*"

# Requests from a specific user
{container="dnsdave-api"} | json | username = "alice"

# Audit trail for DNS record changes
{container="dnsdave-api"} | json
  | resource_type = "dns_records"
  | action =~ "create|edit|delete"
```

### Error aggregation

```logql
# All errors across all DNSDave containers
{service=~"dnsdave-.*"} | json | level = "error"

# Errors by container in the last hour
sum by (container) (
  count_over_time(
    {service=~"dnsdave-.*"} | json | level = "error" [1h]
  )
)
```

### NetBox sync

```logql
# All NetBox sync events
{container="dnsdave-netbox"} | json

# Conflicts only
{container="dnsdave-netbox"} | json | action = "conflict"

# Failed syncs
{container="dnsdave-netbox"} | json | sync_status = "error"
```

---

## 11. Customisation

### Adjusting data retention

| Component | Default | Configuration |
|---|---|---|
| Prometheus | 30 days | `--storage.tsdb.retention.time` in compose command |
| Loki | 90 days | `retention_period` in `loki.yml` `limits_config` |
| Alertmanager | 120h silences | `--data.retention` flag |

For long-term metrics retention, replace Prometheus with **VictoriaMetrics** (drop-in compatible; significantly lower disk and CPU cost). Update the scrape target in `datasources.yml` to point to VictoriaMetrics instead.

### Using an existing Grafana instance

If you already have Grafana, skip the `grafana` service in `docker-compose.monitoring.yml` and manually import the dashboard JSON files from `deploy/monitoring/grafana/dashboards/`. Add Prometheus and Loki as new data sources pointing at the new containers.

### Using an existing Prometheus

Point the existing Prometheus at the DNSDave `/metrics` endpoints by adding the scrape jobs from §4 to your existing `prometheus.yml`. Copy the alert rules from `deploy/monitoring/prometheus/rules/` into your rule directory.

### Reducing resource consumption (Raspberry Pi)

For low-resource deployments, a lightweight profile is available:

```bash
docker compose \
  -f docker-compose.yml \
  -f docker-compose.monitoring.yml \
  -f docker-compose.monitoring.lite.yml \
  up -d
```

`docker-compose.monitoring.lite.yml` overrides:
- Prometheus retention reduced to 7 days.
- Loki retention reduced to 14 days; single-binary mode.
- cAdvisor removed (high overhead on ARM).
- node-exporter kept (very low overhead).
- Grafana image swapped for `grafana/grafana-oss:latest` with `GF_RENDERING_SERVER_URL` unset (no image rendering).
- Prometheus scrape interval increased to 60s.

### Adding custom alert notification channels

Edit `deploy/monitoring/alertmanager/alertmanager.yml` to add receivers for your preferred channels. Alertmanager supports email, Slack, PagerDuty, OpsGenie, VictorOps, Discord (via webhook), and generic HTTP webhooks natively.

### Enabling distributed tracing (Tempo)

1. Uncomment the `otel-collector` service in `docker-compose.monitoring.yml`.
2. Set `mode = "otlp"` in `export.toml` for `dnsdave-export`.
3. Install Tempo:
   ```bash
   docker compose -f docker-compose.monitoring.yml \
     -f docker-compose.monitoring.tempo.yml up -d
   ```
4. Add Tempo as a Grafana datasource pointing at `http://tempo:3200`.
5. The `trace_id` field in all DNSDave log lines and NATS events becomes a clickable link in Grafana Explore, jumping directly to the trace for a given query or DHCP event.

---

## File Structure

```
deploy/
└── monitoring/
    ├── prometheus/
    │   ├── prometheus.yml           # Scrape config and alertmanager reference
    │   └── rules/
    │       ├── dns.yml              # DNS alert rules
    │       ├── dhcp.yml             # DHCP alert rules
    │       ├── blocklist.yml        # Blocklist alert rules
    │       ├── infrastructure.yml   # Host, NATS, Postgres, cert alerts
    │       └── security.yml         # Auth, IAM, NetBox security alerts
    ├── alertmanager/
    │   └── alertmanager.yml         # Routing, receivers, inhibition rules
    ├── loki/
    │   └── loki.yml                 # Loki storage and retention config
    ├── promtail/
    │   └── promtail.yml             # Docker log discovery and pipeline
    ├── grafana/
    │   ├── provisioning/
    │   │   ├── dashboards/
    │   │   │   └── dnsdave.yml      # Dashboard provider config
    │   │   └── datasources/
    │   │       └── datasources.yml  # Prometheus + Loki data sources
    │   └── dashboards/
    │       ├── dns-overview.json
    │       ├── dns-performance.json
    │       ├── dhcp.json
    │       ├── blocklists.json
    │       ├── client-activity.json
    │       ├── system-resources.json
    │       ├── cluster-health.json
    │       ├── netbox-sync.json
    │       ├── security-audit.json
    │       └── query-log-explorer.json
    └── otel/
        └── otel-collector.yml       # OTel Collector pipeline (optional)
```
