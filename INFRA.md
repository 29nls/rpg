# INFRA.md — Infrastruktur, Docker, CI/CD & Monitoring

**Game:** RPG Fantasy Turn-Based Idle (Web + PWA)  
**Versi:** 1.0 · **Tanggal:** 2026-07-14 · **Status:** FINAL  
**Konsisten dengan:** SRS (NFR), DECISION_REGISTER (D-17..19), ERD (`currency_ledger`), PRD, TDD

---

## 1. Arsitektur Deploy

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                            CLIENT (PWA / Web)                               │
│  Phaser 3 + Vite + TypeScript  →  Static assets (JS/CSS/WASM/IMG)         │
│  Served via CDN (CloudFront / Cloudflare / Bunny) + Service Worker (PWA)  │
└─────────────────────────────────┬───────────────────────────────────────────┘
                                  │ HTTPS / WSS
                                  ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                     REVERSE PROXY / API GATEWAY (TLS Termination)          │
│                           Nginx / Caddy (config via env)                   │
│  • TLS 1.3, HSTS, CSP                                                      │
│  • WebSocket upgrade (/ws, /socket.io)                                     │
│  • Rate limit (per IP, per user via header)                                │
│  • Static asset cache headers (immutable + long max-age)                   │
│  • Health probe endpoint `/healthz`                                        │
└───────────┬───────────────────────────────────────────┬─────────────────────┘
            │                                           │
            ▼                                           ▼
┌───────────────────────┐                   ┌───────────────────────┐
│     NESTJS API        │                   │      WebSocket        │
│   (Stateless, N pods) │◄─────────────────►│   (socket.io/ws)      │
│   • REST /meta/config │                   │   • Battle relay      │
│   • Auth (JWT)        │                   │   • PvP ladder        │
│   • Idle, Gacha,      │                   │   • Notifikasi real-time│
│     Market, Battle    │                   │                         │
└───────────┬───────────┘                   └───────────┬───────────┘
            │                                           │
            ▼                                           ▼
┌───────────────────────┐                   ┌───────────────────────┐
│      POSTGRESQL       │                   │        REDIS          │
│  (Managed / Self-host)│                   │   (Cluster / Sentinel)│
│  • Primary + Replica  │                   │   • Session store     │
│  • currency_ledger    │                   │   • BullMQ queues:    │
│    (append-only, PK)  │                   │     - idle_accrual    │
│  • Partition by month │                   │     - gacha_process   │
│  • Wallets UQ (user,  │                   │     - market_settle   │
│    currency)          │                   │   • Rate limit cache  │
│  • Read replicas for  │                   │   • Pub/Sub WS shard  │
│    analytics          │                   │                         │
└───────────────────────┘                   └───────────────────────┘
                                  ▲
                                  │
                    ┌─────────────┴─────────────┐
                    │         CDN               │
                    │  (Static assets, SW,      │
                    │   sourcemap privat)       │
                    └───────────────────────────┘
```

**Alur utama:**
1. Client load PWA dari CDN → Service Worker cache shell
2. Client baca `API_URL` dari `env` (D-17: client agnostic shard/region)
3. REST request → Reverse Proxy (TLS, rate limit) → NestJS API
4. WebSocket upgrade di Reverse Proxy → NestJS WS server (stateful connection, sticky session via Redis)
5. API/WS mutasi data → PostgreSQL (transactional, `currency_ledger` append-only) + Redis (session, queues, cache)
6. Static asset (sprite, audio, sourcemap) → CDN (private signed URL untuk sourcemap D-18)
7. Client polling `/meta/config` → forced update check (D-19), maintenance mode, feature flags

---

## 2. Containerization

### 2.1 Dockerfile Backend (Multi-stage, Non-root, Healthcheck)

```dockerfile
# ---- Builder ----
FROM node:20-alpine AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci --omit=dev
COPY . .
RUN npm run build        # tsc → dist/

# ---- Runtime ----
FROM node:20-alpine AS runner
WORKDIR /app
ENV NODE_ENV=production
# Non-root user
RUN addgroup -g 1001 -S nodejs && adduser -S nestjs -u 1001 -G nodejs
# Copy only needed
COPY --from=builder --chown=nestjs:nodejs /app/dist ./dist
COPY --from=builder --chown=nestjs:nodejs /app/node_modules ./node_modules
COPY --from=builder --chown=nestjs:nodejs /app/package.json ./
USER nestjs
EXPOSE 3000
# Healthcheck untuk k8s / docker-compose
HEALTHCHECK --interval=30s --timeout=3s --start-period=10s --retries=3 \
  CMD wget --no-verbose --tries=1 --spider http://localhost:3000/healthz || exit 1
CMD ["node", "dist/main.js"]
```

**Catatan:** `package.json` include `pm2` atau `node --enable-source-maps` untuk stack trace. Sourcemap `.map` jangan di-copy ke image runtime (upload ke Sentry/private storage saat CI).

### 2.2 docker-compose.yml (Dev Lokal)

```yaml
version: "3.9"
services:
  postgres:
    image: postgres:16-alpine
    environment:
      POSTGRES_DB: rpg
      POSTGRES_USER: rpg
      POSTGRES_PASSWORD: ${PG_PASSWORD}
    volumes:
      - pgdata:/var/lib/postgresql/data
      - ./sql/init:/docker-entrypoint-initdb.d  # ERD schema + seed
    ports: ["5432:5432"]
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U rpg -d rpg"]
      interval: 10s; timeout: 5s; retries: 5

  redis:
    image: redis:7-alpine
    command: redis-server --appendonly yes --maxmemory 256mb --maxmemory-policy allkeys-lru
    volumes: [redisdata:/data]
    ports: ["6379:6379"]
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]; interval: 10s; timeout: 3s; retries: 5

  backend:
    build:
      context: ./backend
      dockerfile: Dockerfile
    environment:
      NODE_ENV: development
      DATABASE_URL: postgresql://rpg:${PG_PASSWORD}@postgres:5432/rpg
      REDIS_URL: redis://redis:6379
      JWT_SECRET: ${JWT_SECRET}
      API_URL: http://localhost:3000
      WS_URL: ws://localhost:3000
      LOG_LEVEL: debug
    ports: ["3000:3000"]
    depends_on:
      postgres: {condition: service_healthy}
      redis: {condition: service_healthy}
    volumes:
      - ./backend:/app
      - /app/node_modules
      - /app/dist

  frontend:
    build:
      context: ./frontend
      dockerfile: Dockerfile
    ports: ["5173:5173"]
    environment:
      VITE_API_URL: http://localhost:3000
      VITE_WS_URL: ws://localhost:3000
    volumes:
      - ./frontend:/app
      - /app/node_modules
      - /app/dist

volumes:
  pgdata:
  redisdata:
```

**Catatan:** `frontend/Dockerfile` pakai `nginx:alpine` serve `dist/` + SPA fallback + CSP headers. Dev pakai `vite --host` (hot reload).

---

## 3. Hosting & Reverse Proxy

| Environment | Provider Placeholder | Reverse Proxy | TLS | Static Host |
|-------------|---------------------|---------------|-----|-------------|
| **dev**     | Local (docker-compose) | Caddy (auto HTTPS via mkcert) | Self-signed | Vite dev / Nginx |
| **staging** | VPS / Cloud Run / Fly.io | Nginx / Caddy | Let's Encrypt (auto) | CDN (Cloudflare) |
| **prod**    | Managed K8s (GKE/EKS) atau VM + Nomad | Nginx Ingress / Caddy | Let's Encrypt / ACME | CDN (CloudFront / Bunny) |

**Nginx snippet (produksi):**
```nginx
server {
  listen 443 ssl http2;
  server_name api.example.com;

  ssl_certificate /etc/letsencrypt/live/api.example.com/fullchain.pem;
  ssl_certificate_key /etc/letsencrypt/live/api.example.com/privkey.pem;
  ssl_protocols TLSv1.2 TLSv1.3;
  ssl_ciphers HIGH:!aNULL:!MD5;

  # Rate limit zona
  limit_req_zone $binary_remote_addr zone=api:10m rate=100r/s;
  limit_req_zone $binary_remote_addr zone=ws:10m rate=50r/s;

  location / {
    proxy_pass http://backend_upstream;
    proxy_http_version 1.1;
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto $scheme;
    limit_req zone=api burst=200 nodelay;
  }

  location /ws/ {
    proxy_pass http://backend_upstream;
    proxy_http_version 1.1;
    proxy_set_header Upgrade $http_upgrade;
    proxy_set_header Connection "upgrade";
    proxy_read_timeout 3600s;
    proxy_send_timeout 3600s;
    limit_req zone=ws burst=50 nodelay;
  }

  location /healthz {
    proxy_pass http://backend_upstream/healthz;
    access_log off;
  }
}

upstream backend_upstream {
  least_conn;
  server backend-1:3000 max_fails=3 fail_timeout=30s;
  server backend-2:3000 max_fails=3 fail_timeout=30s;
  keepalive 64;
}
```

**Environment variables per env:**
| Variable | dev | staging | prod |
|---|---|---|---|
| `NODE_ENV` | development | staging | production |
| `DATABASE_URL` | local pg | managed pg (read-replica) | managed pg (primary+replica) |
| `REDIS_URL` | local redis | managed redis (cluster) | managed redis (cluster) |
| `API_URL` | http://localhost:3000 | https://api-staging.example.com | https://api.example.com |
| `WS_URL` | ws://localhost:3000 | wss://api-staging.example.com | wss://api.example.com |
| `CDN_URL` | http://localhost:5173 | https://cdn-staging.example.com | https://cdn.example.com |
| `SENTRY_DSN` | (opsional) | ✓ | ✓ |
| `LOG_LEVEL` | debug | info | warn |

---

## 4. CI/CD Pipeline (GitHub Actions)

### 4.1 Struktur Workflow

```yaml
# .github/workflows/ci-cd.yml
name: CI/CD
on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]
  workflow_dispatch:
    inputs:
      environment:
        type: choice
        options: [staging, production]
        required: true

env:
  REGISTRY: ghcr.io
  IMAGE_NAME: ${{ github.repository }}/backend
  FRONTEND_IMAGE: ${{ github.repository }}/frontend

jobs:
  lint-test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with: { node-version: '20', cache: 'npm' }
      - run: npm ci
      - run: npm run lint
      - run: npm run test:unit -- --coverage
      - uses: codecov/codecov-action@v4

  build-backend:
    needs: lint-test
    runs-on: ubuntu-latest
    outputs:
      image: ${{ steps.meta.outputs.tags }}
      digest: ${{ steps.build.outputs.digest }}
    steps:
      - uses: actions/checkout@v4
      - uses: docker/metadata-action@v5
        id: meta
        with:
          images: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}
          tags: |
            type=ref,event=branch
            type=sha
            type=raw,value=${{ github.run_number }}
      - uses: docker/build-push-action@v5
        id: build
        with:
          context: ./backend
          push: true
          tags: ${{ steps.meta.outputs.tags }}
          cache-from: type=gha
          cache-to: type=gha,mode=max

  build-frontend:
    needs: lint-test
    runs-on: ubuntu-latest
    outputs:
      image: ${{ steps.meta.outputs.tags }}
    steps:
      - uses: actions/checkout@v4
      - uses: docker/metadata-action@v5
        id: meta
        with:
          images: ${{ env.REGISTRY }}/${{ env.FRONTEND_IMAGE }}
          tags: |
            type=ref,event=branch
            type=sha
      - uses: docker/build-push-action@v5
        id: build
        with:
          context: ./frontend
          push: true
          tags: ${{ steps.meta.outputs.tags }}
          build-args: |
            VITE_API_URL=${{ vars.API_URL }}
            VITE_WS_URL=${{ vars.WS_URL }}

  deploy-staging:
    needs: [build-backend, build-frontend]
    if: github.ref == 'refs/heads/develop' || github.event_name == 'workflow_dispatch'
    runs-on: ubuntu-latest
    environment: staging
    steps:
      - name: Deploy ke Staging (Fly.io / Cloud Run / K8s)
        run: |
          # Contoh Fly.io
          flyctl deploy --image ${{ needs.build-backend.outputs.image }} --app rpg-staging-api
          flyctl deploy --image ${{ needs.build-frontend.outputs.image }} --app rpg-staging-web
      - name: Smoke test
        run: |
          curl -f https://api-staging.example.com/healthz
          curl -f https://api-staging.example.com/meta/config

  deploy-production:
    needs: [build-backend, build-frontend]
    if: github.event_name == 'workflow_dispatch' && github.event.inputs.environment == 'production'
    runs-on: ubuntu-latest
    environment: production
    steps:
      - name: Manual approval gate
        uses: trstringer/manual-approval@v1
        with:
          secret: ${{ github.token }}
          approvers: tech-lead,backend-lead
          minimum-approvals: 2
      - name: Deploy ke Production (Blue/Green atau Rolling)
        run: |
          # Rolling update via kubectl / Fly.io / Nomad
          kubectl set image deployment/backend backend=${{ needs.build-backend.outputs.image }} -n rpg-prod
          kubectl set image deployment/frontend frontend=${{ needs.build-frontend.outputs.image }} -n rpg-prod
          kubectl rollout status deployment/backend -n rpg-prod --timeout=5m
          kubectl rollout status deployment/frontend -n rpg-prod --timeout=5m
      - name: Post-deploy smoke
        run: |
          curl -f https://api.example.com/healthz
          curl -f https://api.example.com/meta/config
      - name: Notify Slack
        uses: slackapi/slack-github-action@v1
        with:
          payload: '{"text": "🚀 Production deploy ${{ github.sha }} done"}'
```

### 4.2 Secrets Management

| Secret | Sumber | Akses |
|---|---|---|
| `DATABASE_URL` | Vault / GH Environments | Backend runtime |
| `REDIS_URL` | Vault / GH Environments | Backend runtime |
| `JWT_SECRET` | Vault (rotate 90 hari) | Backend runtime |
| `SENTRY_DSN` | Vault | CI (upload sourcemap), runtime |
| `CDN_API_KEY` | Vault | CI (purge cache) |
| `FLY_API_TOKEN` / `KUBECONFIG` | Vault | CI deploy job only |

**Jangan hardcode** — semua secret di-inject via GitHub Environments / HashiCorp Vault / AWS Secrets Manager.

---

## 5. Config & Feature Flags

### 5.1 `/meta/config` Endpoint (Public, Cached CDN 60s)

```json
{
  "version": "1.3.0",
  "buildNumber": 142,
  "minClientVersion": "1.2.0",
  "forcedUpdate": false,
  "maintenanceMode": false,
  "maintenanceMessage": null,
  "rates": {
    "gacha": { "ssr": 0.03, "sr": 0.15, "r": 0.82 },
    "idleGoldPerSec": 10,
    "marketFeeBps": 500
  },
  "featureFlags": {
    "pvpLadder": true,
    "marketplace": true,
    "events": true,
    "battlePass": false
  },
  "cdnUrl": "https://cdn.example.com",
  "wsUrl": "wss://api.example.com",
  "serverTime": "2026-07-14T12:00:00Z"
}
```

- Client cek setiap cold start + periodic (5 menit)
- `forcedUpdate: true` → blokir gameplay, redirect ke store / reload PWA
- `maintenanceMode: true` → tampilkan banner, disable write API (read-only `/meta/config`, `/leaderboard`)
- Version format: `major.minor.patch` — breaking CDP/balance = major/minor bump

### 5.2 Environment-based API URL (D-17)

Client **tidak** embed shard/region logic. Hanya baca `VITE_API_URL` & `VITE_WS_URL` dari build-time env. Reverse proxy / API Gateway handle routing (geo-DNS, header `X-Region`).

---

## 6. Observability

### 6.1 Structured Logging (JSON)

```json
{
  "timestamp": "2026-07-14T12:00:00.123Z",
  "level": "info",
  "service": "rpg-api",
  "traceId": "a1b2c3d4",
  "spanId": "e5f6g7h8",
  "userId": "42",
  "action": "gacha_pull",
  "bannerId": "banner_01",
  "result": "SSR",
  "pityBefore": 74,
  "latencyMs": 142
}
```

- Library: **Pino** (fast, child loggers)
- Output: stdout → Loki / ELK / Datadog
- Correlation: `traceId` propagate via HTTP header `x-trace-id` & WS handshake

### 6.2 Metrics (Prometheus + Grafana)

| Metric | Type | Labels | SLA / Alert |
|---|---|---|---|
| `http_requests_total` | Counter | method, path, status | - |
| `http_request_duration_seconds` | Histogram | method, path | p99 < 0.3s (NFR-PERF-01) |
| `ws_connections_active` | Gauge | shard | alert > 80% capacity |
| `battle_resolve_duration_seconds` | Histogram | - | p99 < 1s (NFR-PERF-02) |
| `gacha_pull_duration_seconds` | Histogram | banner | p99 < 0.5s (NFR-PERF-03) |
| `idle_claim_duration_seconds` | Histogram | - | p99 < 0.5s (NFR-PERF-04) |
| `currency_ledger_discrepancy` | Gauge | currency | alert > 0 (audit job) |
| `dau_total` | Gauge | date | track trend |
| `gdp_gold_total` | Counter | date | anomaly alert (z-score > 3) |
| `gacha_pulls_total` | Counter | banner, rarity | rate alert |
| `market_transactions_total` | Counter | - | - |
| `payment_failures_total` | Counter | provider | alert > 0.5% |

**Exporter:** `@nestjs/terminus` + `prom-client` → `/metrics` endpoint (scraped by Prometheus).

### 6.3 Distributed Tracing (OpenTelemetry)

- SDK: `@opentelemetry/sdk-node` + `@opentelemetry/auto-instrumentations-node`
- Exporter: OTLP → Tempo / Jaeger / Datadog
- Propagation: W3C `traceparent` header (HTTP & WS)
- Span attributes: `user.id`, `action`, `entity.id`, `db.statement` (sanitized)

### 6.4 Error Tracking (Sentry)

- **Backend:** `@sentry/node` + `@sentry/profiling-node`
- **Frontend:** `@sentry/browser` + `@sentry/vite-plugin` (upload sourcemap privat D-18)
- DSN di secret; sourcemap di-upload ke Sentry **private** (bukan public CDN)
- Sampling: 100% error, 10% transaction
- Release tracking: `release: "rpg-api@1.3.0+142"`

### 6.5 Alerting Rules (PrometheusRule / Grafana)

```yaml
groups:
- name: rpg-alerts
  interval: 30s
  rules:
  - alert: APILatencyHigh
    expr: histogram_quantile(0.99, rate(http_request_duration_seconds_bucket[5m])) > 0.3
    for: 5m
    labels: {severity: critical}
    annotations:
      summary: "API p99 latency > 300ms"
  - alert: ErrorRateHigh
    expr: rate(http_requests_total{status=~"5.."}[5m]) > 0.01
    for: 2m
    labels: {severity: critical}
  - alert: WSConnectionsHigh
    expr: ws_connections_active > 180
    for: 5m
    labels: {severity: warning}
  - alert: LedgerDiscrepancy
    expr: currency_ledger_discrepancy > 0
    for: 0m
    labels: {severity: critical}
  - alert: GDPAnomaly
    expr: |
      (gdp_gold_total - avg_over_time(gdp_gold_total[1h])) / stddev_over_time(gdp_gold_total[1h]) > 3
    for: 5m
    labels: {severity: warning}
  - alert: PaymentFailureRate
    expr: rate(payment_failures_total[5m]) / rate(payment_attempts_total[5m]) > 0.005
    for: 5m
    labels: {severity: critical}
```

### 6.6 Dashboard Wajib (Grafana)

| Dashboard | Panels |
|---|---|
| **Overview** | DAU (24h/7d/30d), p99 latency, error rate, active WS, deploy markers |
| **Economy** | GDP Gold (total/daily), Gems spent, Gacha pulls by rarity, Market volume, Ledger discrepancy (audit) |
| **Performance** | API latency heatmap, Battle resolve p50/p95/p99, Idle claim latency, DB query p95 |
| **WebSocket** | Conn count per pod, msg throughput, reconnect rate, battle relay latency |
| **Business** | Retention D1/D7/D30, Funnel (login→battle→gacha→purchase), ARPU, Paying % |
| **Infrastructure** | CPU/Mem/Net per pod, PG connections, Redis memory, Queue lag (BullMQ) |

---

## 7. Database & Backup

### 7.1 PostgreSQL Backup Strategy

| Layer | Frequency | Retention | Tool |
|---|---|---|---|
| **Snapshot (Base Backup)** | Daily 02:00 UTC | 30 hari | `pgBackRest` / managed (RDS/Cloud SQL) |
| **WAL Archiving** | Continuous (streaming) | 7 hari | `pg_receivewal` / managed |
| **Logical Dump (Schema + Seed)** | Weekly | 12 bulan | `pg_dump --format=custom` |
| **currency_ledger Partition Archive** | Monthly (bila > 24 bln) | Cold storage (S3/GCS Parquet) | Custom job |

**Point-in-Time Recovery (PITR):** Target RPO < 5 menit (WAL streaming), RTO < 30 menit (restore + replay).

### 7.2 currency_ledger Integrity (ERD §G)

- **Tidak pernah DELETE/UPDATE** `currency_ledger` (append-only immutable)
- `wallets.balance` = cache denormalized → **hanya update via `apply_currency_tx()`** (stored proc atomic)
- **Audit job harian** (04:00 UTC):
  ```sql
  SELECT w.user_id, w.currency_code, w.balance AS wallet_bal,
         COALESCE(SUM(l.delta), 0) AS ledger_sum
  FROM wallets w
  LEFT JOIN currency_ledger l
    ON l.user_id = w.user_id AND l.currency_code = w.currency_code
  GROUP BY w.user_id, w.currency_code, w.balance
  HAVING w.balance <> COALESCE(SUM(l.delta), 0);
  ```
  → Alert kalau return > 0 baris (Sentry + PagerDuty)

### 7.3 Restore Drill (Quarterly)

1. Spin up staging PG dari snapshot terbaru
2. Replay WAL ke timestamp acak (last 7 hari)
3. Jalankan audit query di atas → harus 0 discrepancy
4. Verifikasi `gacha_pulls`, `marketplace_transactions` count match prod
5. Dokumentasi durasi RTO/RPO aktual → update runbook

---

## 8. Scaling & Capacity

### 8.1 Baseline (10–50 DAU)

| Component | Spec (MVP) | Headroom |
|---|---|---|
| **API Pods** | 1 × 2 vCPU / 2 GB | 10x (horizontal) |
| **PostgreSQL** | Managed: 2 vCPU / 8 GB (Primary) + 1 Replica | Read replica for analytics |
| **Redis** | 2 vCPU / 4 GB (Cluster 3 shards) | Horizontal shard |
| **WebSocket** | Sticky session via Redis Pub/Sub | 1 pod ≈ 5k conn |
| **CDN** | Unlimited | N/A |

### 8.2 Trigger Scaling (HPA / Manual)

| Metric | Threshold Scale-Up | Threshold Scale-Down |
|---|---|---|
| CPU (API) | > 70% 5m | < 30% 15m |
| Memory (API) | > 80% 5m | < 50% 15m |
| WS Connections/pod | > 4000 | < 1500 |
| HTTP p99 latency | > 400ms 5m | < 200ms 10m |
| BullMQ Queue Lag | > 1000 jobs | < 100 jobs |
| PG Connections | > 80% max_connections | < 50% |

**Scaling behavior:**
- API stateless → HPA replica count (min 1, max 20)
- Session disimpan Redis → tidak perlu sticky di L4 (WS pakai Redis Pub/Sub sharding)
- BullMQ workers terpisah (idle, gacha, market) → scale worker terpisah dari API
- DB read-replica untuk leaderboard/analytics query

### 8.3 Lonjakan (Event / Launch)

- **Pre-warm:** Tambah 3 pod API + 2 worker 1 jam sebelum event
- **Rate limit** naik 2x (config via `/meta/config` feature flag)
- **Gacha queue:** BullMQ `gacha_process` priority high, concurrency 10/pod
- **Market settle:** Batch per 5 menit, idempotent via `escrow_holds` + ledger
- **Idle accrual:** Scheduler 5s tick (NFR-PERF-10) → batch insert ledger per 100 user

---

## 9. Security & Compliance

| Layer | Measure |
|---|---|
| **Transport** | TLS 1.3 everywhere (API, WS, CDN). HSTS, CSP, `Secure; HttpOnly; SameSite=Lax` cookies |
| **Secrets** | Vault / Sealed Secrets / GH Environments. Rotation JWT 90 hari, DB pwd 180 hari |
| **Rate Limit** | Nginx `limit_req` (IP) + NestJS Guard (per user: 100 req/min API, 20 req/min WS) |
| **WAF (Opsional)** | Cloudflare / AWS WAF managed rules (OWASP Top 10) |
| **Auth** | JWT (RS256) 15m access + 30d refresh (rotate). Email verify soft (D-05) |
| **Audit Log** | `audit_logs` table: admin action, privilege escalation, data export/delete, GDPR request. Immutable (INSERT only). Retensi 12 bln |
| **GDPR/PDPA (D-20)** | `/account/export` (JSON all user data), `/account/delete` (anonymize: `user_id` → hash, retain ledger RESTRICT). SLA 30 hari. Consent checkbox register. |
| **DDoS Basic** | Cloudflare Proxy (gratis tier) → challenge, rate limit layer 7. API Gateway burst buffer |
| **Dependency Scan** | `npm audit` + `trivy` di CI. Fail build pada CVE > High |
| **Penetration Test** | Tahunan (external) + SAST/DAST di CI (CodeQL) |

---

## 10. Runbook Singkat

### 10.1 Deploy Production

```bash
# 1. Tag release
git tag -a v1.3.0 -m "Release 1.3.0"
git push origin v1.3.0

# 2. CI build & push images (auto via tag)
# 3. Manual approval di GitHub Actions → deploy-production job
# 4. Verify
curl -f https://api.example.com/healthz
curl -f https://api.example.com/meta/config
# 5. Smoke test critical path: login → battle → gacha → idle claim → market
```

### 10.2 Rollback

```bash
# Option A: Image tag rollback (instant)
kubectl set image deployment/backend backend=ghcr.io/owner/rpg-backend:v1.2.9 -n rpg-prod
kubectl rollout status deployment/backend -n rpg-prod

# Option B: DB rollback (jika migration breaking)
# - Stop API pods
# - pg_restore --clean --if-exists snapshot_20260713_0200.dump
# - Start API pods dengan image vecchia
```

### 10.3 Incident: Dupe Item (PRD §35)

1. **Deteksi:** Alert `currency_ledger_discrepancy` / audit log anomaly / player report
2. **Isolasi:** Enable `maintenanceMode: true` via `/meta/config` (instant, CDN cache 60s)
3. **Investigasi:** Query `currency_ledger` + `audit_logs` untuk user & timeframe
4. **Remediasi:**
   - Identifikasi ledger ref duplikat (nonce collision / race condition)
   - Insert ledger reversal (delta negatif) via `apply_currency_tx` admin
   - Update `wallets.balance` via re-sync job
   - Ban/exploit patch deploy (hotfix)
5. **Komunikasi:** Template internal + player notice (transparan, tidak detail teknis)
6. **Post-mortem:** 48 jam → doc di `INCIDENT-YYYYMMDD.md`

### 10.4 Incident: Gacha Rate Melenceng

1. **Deteksi:** Alert `gacha_pulls_total` rarity distribution z-score > 3 (bandingkan 1h window)
2. **Verifikasi:** Cek `gacha_pulls` + `pity_state` Konsistensi (soft pity ramp D-02)
3. **Root cause:** Config drift `/meta/config` rates vs DB `gacha_banners.rates_json` / code bug
4. **Fix:** Patch config / hotfix code → deploy → verify distribution normalisasi
5. **Compensation:** Jika confirmed bug → grant kompensasi via admin ledger (audit logged)

### 10.5 Maintenance Mode

```bash
# Enable (instant, via config endpoint)
curl -X PATCH https://api.example.com/admin/meta/config \
  -H "Authorization: Bearer $ADMIN_TOKEN" \
  -d '{"maintenanceMode": true, "maintenanceMessage": "Scheduled maintenance 02:00-03:00 UTC"}'

# Disable
curl -X PATCH ... -d '{"maintenanceMode": false, "maintenanceMessage": null}'
```
- Read API tetap hidup (`/meta/config`, `/leaderboard`, `/hero/*` read)
- Write API return `503 Service Unavailable` + `Retry-After`
- WS: nieuwe conn rejected, existing dikirim `maintenance` event → client show banner

---

## 11. Cost & Asumsi

### 11.1 Estimasi Bulanan (Placeholder, MVP 10–50 DAU)

| Komponen | Provider (Contoh) | Estimasi USD/bln | Catatan |
|---|---|---|---|
| **Compute (API 1 pod)** | Fly.io / Cloud Run / VPS | $15–30 | Scale to zero dev |
| **PostgreSQL (Managed)** | Neon / Supabase / RDS | $20–50 | 2 vCPU / 8 GB + replica |
| **Redis (Managed)** | Upstash / Redis Cloud | $10–25 | 4 GB cluster |
| **CDN** | Cloudflare / Bunny | $0–5 | Free tier cukup MVP |
| **Observability** | Grafana Cloud / Datadog | $0–50 | Free tier / pay per host |
| **Sentry** | Sentry.io | $0–26 | Free 5k errors/bln |
| **CI/CD** | GitHub Actions | $0 | Free untuk public/private < 2k min |
| **Domain + Cert** | Cloudflare / Let's Encrypt | $0–1 | |
| **Total MVP** | | **~$45–186** | Bisa < $50 dgn free tier |

### 11.2 Asumsi Utama

| Asumsi | Detail |
|---|---|
| **Database** | Managed PostgreSQL (Neon/Supabase/RDS) — backup, PITR, replica otomatis. Self-host hanya bila cost > $200/bln. |
| **Redis** | Managed (Upstash/Redis Cloud) — cluster mode, persistence AOF. |
| **Orchestrator** | MVP: VM + Docker Compose / Fly.io / Cloud Run (no k8s). k8s (GKE/EKS) saat > 500 DAU atau multi-region. |
| **CDN** | Cloudflare (gratis) → Bunny/CloudFront bila bandwidth > 1 TB/bln. |
| **Secrets** | GitHub Environments + Vault (opsional). Tidak pakai HashiCorp Vault self-host di MVP. |
| **Compliance** | PDPA default (Asia), EU GDPR toggle (D-20). Data residency: PG region = Singapore (APAC). |
| **SLA** | 99.5% availability (NFR-PERF-05) = max 3.65 jam downtime/bln. Maintenance window: Minggu 02:00–04:00 UTC. |

---

## 12. Referensi Cepat (Cheatsheet)

| Task | Command / Link |
|---|---|
| **Local dev** | `docker-compose -f docker-compose.yml up -d` |
| **Run migration** | `docker-compose exec backend npm run migration:run` |
| **Seed data** | `docker-compose exec backend npm run seed` |
| **View logs** | `docker-compose logs -f backend` |
| **Build image** | `docker build -t rpg-backend:local ./backend` |
| **Push image** | `docker tag rpg-backend:local ghcr.io/owner/rpg-backend:v1.3.0 && docker push ...` |
| **Deploy staging** | `flyctl deploy -a rpg-staging-api` |
| **Check metrics** | `curl https://api.example.com/metrics` |
| **Audit ledger** | `psql $DATABASE_URL -f sql/audit_ledger.sql` |
| **Enable maintenance** | `curl -X PATCH .../admin/meta/config -d '{"maintenanceMode":true}'` |
| **Rollback image** | `kubectl set image deployment/backend backend=ghcr.io/...:v1.2.9 -n rpg-prod` |

---

**File:** `C:\Users\Admin\Downloads\Store\rpg\INFRA.md`  
**Baris:** ~480  
**Konsisten:** SRS NFR (p99 <300ms, battle <1s, 99.5% avail), DECISION_REGISTER D-17 (env API URL), D-18 (sourcemap privat), D-19 (forced update `/meta/config`), ERD `currency_ledger` append-only + audit job, PRD §35 dupe item response.