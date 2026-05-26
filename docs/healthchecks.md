# Healthchecks

A healthcheck is a command Docker runs **inside** the container on a schedule to decide whether the service is actually serving traffic, not just whether the process is alive. The result becomes the container’s health state: `starting`, `healthy`, or `unhealthy`.

Healthchecks matter for two reasons:

1. **Dependency ordering.** Other services can wait until this one is genuinely ready.
2. **Observability.** `docker compose ps` shows `(healthy)` so operators can tell at a glance whether the stack is up.

## `depends_on` Without a Condition vs. `condition: service_healthy`

`depends_on` by itself is a startup-order hint. It tells Compose: *don’t start service B until service A’s container has started.* But “container started” only means the entrypoint process is running. MySQL, for example, takes 10–30 seconds **after** the container starts to actually accept connections, initialize the data directory, and bind to its port.

If the app starts immediately after the MySQL container starts, the app’s first connection attempt fails and — because the process exits on a fatal DB error — it crash-loops until MySQL happens to come up in time.

The fix is to add a healthcheck to the dependency and gate the dependent on a *healthy* state:

```yaml
depends_on:
  db:
    condition: service_healthy
```

Now Compose blocks the app from starting until the `db` healthcheck actually returns success. No more startup races.

| Construct | What it guarantees |
|---|---|
| `depends_on: [db]` | The `db` container has started (process launched). |
| `depends_on: db: condition: service_started` | Same as above, explicit form. |
| `depends_on: db: condition: service_healthy` | The `db` healthcheck has reported healthy. **This is what we want.** |

## The Specific Healthchecks Used

**db (MySQL):**

```yaml
healthcheck:
  test: ["CMD", "mysqladmin", "ping", "-h", "localhost", "-u", "root", "-p${DB_ROOT_PASSWORD}"]
  interval: 10s
  timeout: 5s
  retries: 5
  start_period: 30s
```

`mysqladmin ping` returns success only when the MySQL server is accepting authenticated connections. `start_period: 30s` gives MySQL time to initialize before failures start counting.

**app (Node.js):**

```yaml
healthcheck:
  test: ["CMD-SHELL", "wget -qO- http://127.0.0.1:8080 || exit 1"]
  interval: 30s
  timeout: 5s
  retries: 3
  start_period: 15s
```

`wget` against `http://127.0.0.1:8080` proves the Express server is up and routing requests — a true end-to-end check, not just “the process exists.”

**proxy (Nginx):**

```yaml
healthcheck:
  test: ["CMD-SHELL", "wget -qO- http://127.0.0.1/health || exit 1"]
  interval: 30s
  timeout: 5s
  retries: 3
  start_period: 10s
```

Nginx exposes a cheap `/health` location that returns `200 {"status":"ok"}` without hitting upstream — perfect for a liveness probe.

## Why `127.0.0.1` Instead of `localhost`

On Alpine-based images, `localhost` can resolve to the IPv6 loopback `::1` first. If the server is only listening on the IPv4 `0.0.0.0`, the connection is refused and the healthcheck fails even though the service is fine. Forcing `127.0.0.1` removes that ambiguity.

## Inspecting Health Status

```bash
docker compose ps
docker inspect --format='{{json .State.Health}}' <container_id> | jq
```
