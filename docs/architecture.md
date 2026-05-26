# Architecture

EpicBook is a classic 3-tier web application. Each tier runs in its own container, on a shared private Docker network, with a single public entry point at the proxy.

```
Browser  →  Nginx (proxy:80)  →  Node.js (app:8080)  →  MySQL (db:3306)
             │                    │                      │
             └─ published port    └─ internal only       └─ internal only,
                to host :80          (no host port)         backed by db_data volume
```

## The Three Tiers

**1. Presentation tier — `proxy` (Nginx).**
Receives all inbound HTTP traffic on port 80. Performs reverse-proxy routing to the application, exposes a lightweight `/health` endpoint, and emits structured JSON access logs. The proxy is the only container that publishes a port to the host.

**2. Application tier — `app` (Node.js / Express).**
Implements the bookstore REST API and server-rendered views. Listens on `8080` inside the network, never directly on the host. It talks to MySQL by service name and serves requests proxied from Nginx.

**3. Data tier — `db` (MySQL 8).**
Persists books, users, and orders. Listens on `3306` inside the network only. The data directory `/var/lib/mysql` is bound to a named volume (`db_data`) so data survives container restarts and rebuilds.

## Why Each Tier Is Separate

- **Independent lifecycle.** The app can be rebuilt and redeployed without touching MySQL or Nginx. Each image can be versioned and rolled back on its own.
- **Resource isolation.** Memory pressure or a crash in one tier doesn’t take the others down. Compose’s `restart: unless-stopped` brings just the failed container back.
- **Security boundary.** The database has no host port and is unreachable from outside the Docker network. The app has no host port either; only the proxy is exposed.
- **Scaling story.** A separate proxy lets us terminate TLS, rate-limit, add caching, and (later) load-balance across multiple `app` replicas without changing the application code.

## Docker Networking by Service Name

Compose creates a user-defined bridge network (`epicbook-net`) and attaches all three services to it. On a user-defined bridge, Docker runs an embedded DNS resolver that maps each service name to its container IP.

That means the app connects to MySQL using:

```
DB_HOST=db
```

… not an IP address. Container IPs are not stable — they change on every restart — so hard-coding them would break the stack. Service-name DNS is stable, human-readable, and works the same locally and in EC2.

The same idea drives `proxy_pass http://app:8080` in `nginx.conf`: Nginx finds the app container by its service name on the shared network.

## Why the Proxy Sits in Front

Putting Nginx in front of Node.js is a deliberate production pattern:

- **Single ingress.** Only one process binds to port 80 on the host. Everything else stays internal.
- **Static-file efficiency.** Nginx serves bytes faster than Node and frees the application loop for actual logic.
- **Operational seam.** TLS termination, request logging, request size limits, and gzip belong at the edge — not sprinkled across application code.
- **Health visibility.** `/health` answers without ever touching the app, so an external load balancer or uptime monitor can probe Nginx cheaply.
- **Future-proofing.** When we add a second `app` replica, Nginx upstream config is the only place that needs to change.
