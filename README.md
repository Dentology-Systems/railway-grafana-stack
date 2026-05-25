# Grafana Stack on Railway

[![Deploy on Railway](https://railway.com/button.svg)](https://railway.com/template/8TLSQD?referralCode=IFlm92)

## What is this template

This template deploys a complete Grafana observability stack on Railway with just one click! The stack includes four integrated services:

- **Grafana**: The leading open-source analytics and monitoring solution
- **Loki**: A horizontally-scalable, highly-available log aggregation system
- **Prometheus**: A powerful metrics collection and alerting system
- **Tempo**: A high-scale distributed tracing backend

This template is perfect for teams who need a comprehensive observability solution for their railway project without the hassle of manual configuration and infrastructure management.

### Key Features

- **Pre-configured Integration**: _All services come pre-connected_, so Grafana is ready to query your data immediately.
- **Persistent Storage**: All four services use Railway volumes to ensure your data, dashboards, and configurations persist between updates and deploys.
- **Version Control**: Pin specific Docker image versions for each service using environment variables.
- **Customizable**: Fork the repository to customize configuration files for any service. You can take full control and edit anything you'd need to as you scale.
- **One-Click Deploy**: Get a complete Grafana-based observability stack running in minutes.

---

## Dentology fork — deployment notes

> This fork (`Dentology-Systems/railway-grafana-stack`) runs on Railway inside the **Dentology** project. Beyond the upstream template it adds a **combined telemetry proxy**, **per-tRPC-endpoint metrics** via Tempo's metrics-generator, and several Railway-specific fixes. This section documents how it's wired and the gotchas we hit, so they don't bite again.

### How telemetry flows

`dentology-app` runs on **Vercel** (not Railway), so it can't reach Railway's IPv6 private network directly. It ships telemetry to a public, authenticated **combined proxy**, which forwards to Tempo/Loki over the private network:

```
dentology-app (Vercel)
  │  traces:  POST ${OTEL_HOST}/v1/traces           (app/lib/otel.ts; OTEL_HOST + OTEL_API_KEY)
  │  logs:    POST ${LOKI_HOST}/loki/api/v1/push    (pino-loki; LOKI_HOST + LOKI_API_KEY)
  ▼   (public, X-API-Key auth)
dentology-telemetry-proxy.up.railway.app  ← Caddy (telemetry-proxy/), validates X-API-Key
  │  /v1/*   ─▶ tempo.railway.internal:4318   (OTLP HTTP)
  │  /loki/* ─▶ loki.railway.internal:3100    (Loki push)
  ▼
Tempo ─(metrics-generator, remote_write)─▶ Prometheus ─▶ Grafana
Loki  ───────────────────────────────────────────────▶ Grafana
```

App-side env vars (set in **Vercel**); telemetry only ships when `NODE_ENV==='production'` and the vars are set:

| Signal | Code | Env vars | Endpoint |
|--------|------|----------|----------|
| Traces | `apps/dentology/app/lib/otel.ts` (OTLP HTTP) | `OTEL_HOST`, `OTEL_API_KEY` | `${OTEL_HOST}/v1/traces` |
| Logs | `apps/dentology/app/lib/log.ts`, `libs/shared/logger/` (pino-loki) | `LOKI_HOST`, `LOKI_API_KEY` | `${LOKI_HOST}/loki/api/v1/push` |

Both `OTEL_HOST` and `LOKI_HOST` point at the **same** combined proxy domain.

### The combined telemetry proxy (`telemetry-proxy/`)

A single pinned **Caddy** service (`caddy:2.8.4-alpine`) that path-routes and authenticates:

- `/v1/*` → Tempo (`TEMPO_INTERNAL_URL`, default `http://tempo.railway.internal:4318`)
- `/loki/*` → Loki (`LOKI_INTERNAL_URL`, default `http://loki.railway.internal:3100`)
- every request needs `X-API-Key: $API_KEY`; `/health` is open for Railway's healthcheck

It replaced two hand-rolled **Bun-function** proxies (`Tempo proxy`, `Loki proxy`) that cached DNS / reused dead keep-alive connections and so **got stuck (15s timeouts → 502) after any Tempo/Loki redeploy**, needing manual restarts. Caddy's Go transport re-resolves DNS and retries dead connections, so it self-heals. (`tempo-proxy/` is a single-purpose Caddy variant kept until that legacy service is decommissioned.)

### Per-tRPC-endpoint metrics (Tempo metrics-generator)

There is **no "aggregation plugin"** — RED metrics per endpoint come from **Tempo's metrics-generator**, configured in `tempo/tempo.yml`:

- processors are enabled per-tenant via the **`overrides`** block — defining the `metrics_generator` block alone does nothing
- it **remote-writes** to Prometheus (`metrics_generator.storage.remote_write`)
- Prometheus must accept it (`--web.enable-remote-write-receiver`)

Generated series are labelled by `span_name` (= the tRPC procedure, e.g. `trpc.query.sidebar.getUnscreenedAppointments`):

```promql
# request rate per endpoint
sum by (span_name) (rate(traces_spanmetrics_calls_total{span_name=~"trpc.*"}[5m]))
# p95 latency per endpoint
histogram_quantile(0.95, sum by (span_name, le) (rate(traces_spanmetrics_latency_bucket{span_name=~"trpc.*"}[5m])))
```

(Prometheus datasource UID `grafana_prometheus`, Loki UID `grafana_lokiq`.)

### Railway gotchas & lessons learned

- **Pin every image — never `latest`.** Tempo was on `latest`, which floated to `v3.0.0-rc.1` and broke startup (3.0 removed the top-level `compactor` and `metrics_generator.traces_storage` fields). Pinned to `2.8.1`; Loki/Grafana/Caddy are pinned too.
- **A dashboard `VERSION` variable overrides the Dockerfile `ARG VERSION`.** Each service's image tag is ultimately driven by its `VERSION` service variable (passed as a build arg), so a Dockerfile pin is silently ignored unless they match. Keep them in sync.
- **Railway's private network is IPv6-only — bind `[::]`, not `0.0.0.0`.** Services on `0.0.0.0` are unreachable via `*.railway.internal` (symptom: callers hang and time out). Fixed Tempo's OTLP receivers (`[::]:4317` / `[::]:4318` in `tempo.yml`) and Prometheus (`--web.listen-address=[::]:9090`).
- **Prometheus CMD flags** (`prometheus/dockerfile`): `--web.enable-remote-write-receiver` (accept Tempo's generated metrics), `--web.listen-address=[::]:9090` (IPv6), `--storage.tsdb.no-lockfile` (avoid the TSDB volume-lock deadlock when a redeploy's new container starts before the old releases the lock).
- **Deploying via the CLI:** `railway up <dir> --path-as-root --service "<name>"` uploads just that folder as the build context — no GitHub link or Root Directory needed. Connecting a **GitHub repo** or a **custom domain** via the CLI fails with a misleading `Unauthorized` (those need dashboard authorization). A **capitalized `Dockerfile`** auto-detects; a lowercase `dockerfile` needs `RAILWAY_DOCKERFILE_PATH=/<dir>/dockerfile`. Mutating CLI commands need a non-expired `railway login`. Note: a service deployed with `railway up` does **not** auto-deploy on git push unless its Source is connected to GitHub in the dashboard.
- **Railway UI service "groups" are cosmetic** — they don't affect private networking; all services in the same project+environment share the network.
- **Local dev:** `docker-compose.yml` runs the whole stack (including `telemetry_proxy`) locally; Railway does **not** use it — each service builds independently from its folder.

---

## Quick Start Guide

1. Click the "Deploy on Railway" button at the top of this page
2. Enter your desired Grafana admin username in the `GF_SECURITY_ADMIN_USER` variable
3. Leave all other variables at their defaults (or customize as needed)
4. Wait for your stack to deploy (this typically takes 3-5 minutes)
5. Navigate to the Grafana URL provided by Railway
6. Log in with your admin username and the auto-generated password found in the `GF_SECURITY_ADMIN_PASSWORD` environment variable
7. Hook up your applications to the datasources.
8. Create dashboards, alerts, and explore your data in Grafana!

## Optional Variables

| Variable | Description | Default |
|----------|-------------|---------|
| `GF_SECURITY_ADMIN_USER` | Username for the Grafana admin account | Required input |
| `GF_SECURITY_ADMIN_PASSWORD` | Password for the Grafana admin account | Auto-generated secure string |
| `GF_DEFAULT_INSTANCE_NAME` | Name of your Grafana instance | `Grafana on Railway` |
| `GF_INSTALL_PLUGINS` | Comma-separated list of Grafana plugins to install | `grafana-simple-json-datasource,grafana-piechart-panel,grafana-worldmap-panel,grafana-clock-panel` |

### Internal Service URLs

The Grafana service exposes these environment variables that you can reference in your other Railway applications to easily send data to your observability stack:

| Variable | Description | Usage |
|----------|-------------|-------|
| `LOKI_INTERNAL_URL` | Internal URL for the Loki service | Use in your applications to send logs to and query Loki |
| `PROMETHEUS_INTERNAL_URL` | Internal URL for the Prometheus service | Use in your applications to send metrics to and query Prometheus |
| `TEMPO_INTERNAL_URL` | Internal URL for the Tempo service | Use in your applications to query Tempo |

These variables make it easy to configure your other Railway services to send telemetry data to your observability stack.

Tempo also exposes a few variables to make it easier to push tracing information to the service using either HTTP or GRPC

| Variable | Description | Usage |
|----------|-------------|-------|
| `INTERNAL_HTTP_INGEST` | Internal HTTP ingest server URL for Tempo | Use in your applications to send traces to tempo via HTTP |
| `INTERNAL_GRPC_INGEST` | Internal GRPC ingest server URL for Tempo | Use in your applications to send traces to tempo via GRPC |

### Version Control

Each service has its own `VERSION` environment variable that can be set independently in each service's settings in the Railway dashboard:

- **Grafana Service**: Set `VERSION` to control the Grafana Docker image tag
- **Loki Service**: Set `VERSION` to control the Loki Docker image tag
- **Prometheus Service**: Set `VERSION` to control the Prometheus Docker image tag
- **Tempo Service**: Set `VERSION` to control the Tempo Docker image tag

By default, all services use the `latest` tag, but you can pin specific versions for stability:

Examples:
- Grafana: `VERSION=11.5.2`
- Loki: `VERSION=3.4.2`
- Prometheus: `VERSION=v3.2.1`
- Tempo: `VERSION=v2.7.1`

This allows you to update each component independently as needed.

## Project Structure & Services

This template deploys four interconnected services:

### Grafana
- The central visualization and dashboarding platform
- Pre-configured with connections to all other services
- Persistent volume for storing dashboards, users, and configurations
- Comes with useful plugins pre-installed
- Exposes internal URLs for other Railway services to connect to Loki, Prometheus, and Tempo

### Prometheus
- Time-series database for metrics collection
- Configured with sensible defaults for monitoring
- Persistent volume for metrics data

### Loki
- Log aggregation system designed to be cost-effective
- Horizontally scalable architecture
- Persistent volume for log storage

### Tempo
- Distributed tracing system for tracking requests across services
- High-performance trace storage
- Persistent volume for trace data

All services are deployed using official Docker images and configured to work together seamlessly.

## Connecting Your Applications

### Using [Locomotive](https://railway.com/template/jP9r-f) for Loki

You can easily ingest *all* of your railway logs into Loki from *any* service using [Locomotive](https://railway.com/template/jP9r-f). Just spin up their template, drop in your Railway API key, the ID of the services you want to monitor, and a link to your new Loki instance and logs will start flowing! no code changes needed anywhere!

### Using OpenTelemetry libraries for Tempo 

Tempo is a bit different than both Prometheus and Loki in that exposes separate GRPC and HTTP servers on ports `:4317` and `:4318` respectively specifically for ingesting your tracing data or "spans".

When configuring your application to send traces to Tempo, please use one of the preconfigured variables in the Tempo service: `INTERNAL_HTTP_INGEST` or `INTERNAL_GRPC_INGEST`.

Another thing to note is that the ingest API endpoint for the HTTP server is `/v1/traces`. For a working example of this in a node.js express API, see `/examples/api/tracer.js` in our GitHub repository.

### Using otherwise standard observability tooling

To send data from your other Railway applications to this observability stack:

1. In your application's Railway service, add environment variables that reference the internal URLs:
   ```
   LOKI_URL=${{Grafana.LOKI_INTERNAL_URL}}
   PROMETHEUS_URL=${{Grafana.PROMETHEUS_INTERNAL_URL}}
   TEMPO_URL=${{Grafana.TEMPO_INTERNAL_URL}}
   ```
2. Configure your application's logging, metrics, or tracing libraries to use these URLs
3. Your application data will automatically appear in your Grafana dashboards

## Customizing Your Stack

To customize the configuration of Loki, Prometheus, or Tempo:

1. Fork the [GitHub repository](https://github.com/yourusername/grafana-railway-template)
2. Modify the configuration files in their respective directories
3. In Railway, disconnect the service you want to customize
4. Reconnect the service to your forked repository
5. Deploy the updated service

The pre-configured Grafana connections will continue to work with your customized services.

## Additional Resources

- [Locomotive: a loki transport for railway services](https://railway.com/template/jP9r-f)
- [Grafana Documentation](https://grafana.com/docs/grafana/latest/)
- [Loki Documentation](https://grafana.com/docs/loki/latest/)
- [Prometheus Documentation](https://prometheus.io/docs/introduction/overview/)
- [Tempo Documentation](https://grafana.com/docs/tempo/latest/)
- [Grafana Community Forums](https://community.grafana.com/)
- [Grafana Plugins Directory](https://grafana.com/grafana/plugins/)

---

Developed and maintained by [Mykal](https://mykal.codes). For issues or suggestions, please open an issue on the [GitHub repository](https://github.com/MykalMachon/grafana-stack-railway).
