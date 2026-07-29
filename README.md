# StatLite

A tiny self-hosted metrics dashboard for small servers.

![StatLite dashboard](docs/images/dashboard.png)

StatLite supports Spring Boot Actuator and the lightweight StatLite Metrics JSON format for small applications that need basic health, traffic, latency, CPU, and runtime memory monitoring without a full observability stack. It uses local SQLite, raw samples only, simple charts, a localhost dashboard by default, and remains systemd-friendly.

**StatLite is not a Prometheus/Grafana replacement.** It is a small production-support tool for one-person / small-team self-hosted apps that need a focused dashboard for supported application integrations or StatLite self-monitoring.

Learn how to set up [lightweight Spring Boot monitoring without Prometheus and Grafana](https://pvrlabs.xyz/articles/lightweight-spring-boot-monitoring.html).

Supported target types are `spring` (Spring Boot Actuator), `statlite-metrics`
(the fixed `statlite-metrics/v1` JSON profile), and `statlite-health` (StatLite
self-monitoring). The legacy `statlite` value remains an alias for
`statlite-health`.

See [StatLite Metrics v1](docs/statlite-metrics-v1.md) for the application
producer contract.

See [Product and architecture](docs/product.md) for StatLite’s scope, design
principles, and normalized collection model.

## Quick Start

From a clone:

```bash
go build -o statlite ./cmd/statlite
./statlite
```

Open http://127.0.0.1:9090 — StatLite loads root `statlite.yaml` and monitors itself via `/healthz`.

The first self-monitor poll may fail briefly before the HTTP server is ready. That is expected; the dashboard should become healthy on the next poll rather than treating that initial failure as a permanent problem.

Edit `statlite.yaml` for the default single-target setup, or start from `examples/multi-target.yaml` if you want multiple targets in one instance.

See `examples/` for config templates and a demo app:

| Path | Description |
|------|-------------|
| `examples/multi-target.yaml` | Generic multi-target starter (illustrative only) |
| `examples/actuator.yaml` | Spring Boot Actuator (single target) |
| `examples/statlite.yaml` | Another StatLite instance (self-monitoring) |
| `examples/spring-actuator-demo/` | Standalone Spring Boot demo app for StatLite monitoring |
| `examples/python-fastapi-demo/` | Runnable FastAPI app exposing StatLite Metrics v1 |

### Installed binary

If StatLite is already on your `PATH` (release installer or Homebrew):

```bash
cp examples/actuator.yaml ./statlite.yaml   # or create your own config
# edit credentials and URLs
statlite --config ./statlite.yaml
```

Default config path is `statlite.yaml` in the current working directory when `--config` is omitted.

## Config

`statlite.yaml` is loaded by default. For production, point it at your app:

```bash
./statlite --config examples/actuator.yaml
```

Prefer environment variables for Actuator credentials, for example `username: "${STATLITE_ACTUATOR_USERNAME}"` and `password: "${STATLITE_ACTUATOR_PASSWORD}"`. StatLite expands `$VAR` and `${VAR}` across the entire config once at startup; restart it after changing a value. Plaintext credentials still work, but restrict that config file with `chmod 600` on a server. Credentials are never rendered in the dashboard or API responses. See [docs/configuration.md](docs/configuration.md) for escaping literal `${` syntax and other expansion details.

Details (targets, Basic Auth, storage, polling, self-monitoring, retention): [docs/configuration.md](docs/configuration.md).

## Deployment

The binary is self-contained. See [docs/install.md](docs/install.md) for user-level installation and [docs/systemd.md](docs/systemd.md) for systemd server provisioning. Installers and package managers install **only the binary** — they do not create config, initialize storage, install units, or start services.

### Dashboard access

StatLite does not include built-in dashboard/API authentication yet.

By default, examples bind the server to `127.0.0.1:9090` so the dashboard is reachable only from the local machine. For remote access, use an SSH tunnel, VPN, firewall-restricted private network, or an authenticated reverse proxy.

You may bind to `0.0.0.0:9090` if you intentionally want StatLite to listen on all interfaces, but do this only when access is protected externally. Without external protection, anyone who can reach the port can view dashboard/API data such as target names and operational metrics.

## Version and health

```bash
statlite --version
```

`GET /healthz` returns JSON including the same version string.

Process health semantics:

* Top-level `status` and the HTTP status code describe **StatLite itself**, not whether monitored targets are healthy.
* Target poll failures appear under `statlite.polling` and do **not** mark the process unhealthy.
* If the local SQLite store fails its health check, `/healthz` reports `status: "error"` and HTTP 503.

`type: "statlite-health"` targets are for StatLite self-monitoring only; `type: "statlite"` remains a compatibility alias. For application integrations, use `spring` or the fixed `statlite-metrics` profile. StatLite Metrics is not a general metrics protocol.

## API stability

`/api/*` is early/internal and **not yet a stable public API**. Fields and routes may change without a compatibility guarantee.

Missing optional metrics may appear as `null` (or be omitted from charts) and should degrade cleanly rather than failing the whole dashboard.

## Known Limitations (MVP)

StatLite is early and intentionally limited:

* **Raw samples only** — no derived rollups or downsampling. Query-time delta computation is used for counters.
* **No alerts** — dashboard-only. No alert manager, no notifications.
* **No dashboard auth** — see [Dashboard access](#dashboard-access) for safe deployment options.
* **Focused integrations** — arbitrary metric endpoints, Prometheus/OpenMetrics, labels, and custom dashboards are not supported in the MVP.
* **Credential handling** — use environment variables where possible; plaintext YAML credentials remain supported and should be restricted with `chmod 600`.
* **Dashboard CDN assets** — Chart.js and fonts may load from external CDNs. The backend is a single binary; full dashboard rendering still depends on those external frontend assets for now. Vendoring them into the binary is a post-MVP item.

## License

MIT
