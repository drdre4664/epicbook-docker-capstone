# EpicBook — Docker Capstone

A containerized 3-tier Node.js bookstore application demonstrating production-grade Docker practices: multi-stage builds, Docker Compose orchestration, Nginx reverse proxy, healthchecks with dependency conditions, named volumes for data persistence, structured JSON logging, and AWS EC2 deployment via Terraform.

---

## Architecture

```
                +----------------+
                |    Browser     |
                +--------+-------+
                         |
                         | HTTP :80
                         v
                +--------+-------+
                |     Nginx      |   <-- proxy (reverse proxy, JSON access logs)
                |   (alpine)     |
                +--------+-------+
                         |
                         | proxy_pass :8080
                         v
                +--------+-------+
                |    Node.js     |   <-- app (Express, multi-stage build)
                |  (node:18)     |
                +--------+-------+
                         |
                         | TCP :3306
                         v
                +--------+-------+
                |     MySQL      |   <-- db (named volume: db_data)
                |   (mysql:8)    |
                +----------------+
```

All three services run on a private bridge network (`epicbook-net`) and reference each other by service name rather than IP.

---

## Tech Stack

| Layer            | Technology         | Purpose                                           |
|------------------|--------------------|---------------------------------------------------|
| Containers       | Docker             | Image build and runtime isolation                 |
| Orchestration    | Docker Compose     | Multi-service lifecycle, networks, volumes        |
| Reverse Proxy    | Nginx (alpine)     | TLS-ready edge, JSON access logs, health endpoint |
| Application      | Node.js 18 / Express | REST API for the bookstore                      |
| Database         | MySQL 8            | Relational store for books and orders             |
| Infrastructure   | Terraform          | AWS EC2 provisioning for cloud deployment         |

---

## Project Structure

```
epicbook-docker-capstone/
├── app/
│   ├── Dockerfile              # Multi-stage Node.js build
│   ├── package.json
│   └── server.js
├── proxy/
│   └── nginx.conf              # Reverse proxy + JSON access log format
├── docs/
│   ├── architecture.md
│   ├── healthchecks.md
│   ├── multi-stage-builds.md
│   ├── volumes-and-persistence.md
│   └── logging.md
├── docker-compose.yml          # Three services: db, app, proxy
├── .env.example                # Template for required environment variables
├── .gitignore
├── RUNBOOK.md                  # Day-2 operations guide
└── README.md
```

---

## What I Built

- **Multi-stage Dockerfile** for the Node.js app — a `builder` stage installs dependencies, then a slim runtime stage copies only `node_modules` and source code. Smaller final image, fewer attack-surface packages in production.
- **Docker Compose orchestration** for three services (`db`, `app`, `proxy`) on a private bridge network, with named volumes for persistent state and per-service log rotation.
- **Healthchecks with `condition: service_healthy`** so `app` only starts after MySQL is actually accepting connections (not just after the container starts), and `proxy` only starts after `app` reports healthy.
- **Nginx reverse proxy** terminating port 80, proxying upstream to `app:8080`, exposing a lightweight `/health` endpoint, and emitting structured JSON access logs.
- **Named volumes** (`db_data`, `proxy_logs`) so MySQL data and Nginx logs survive container restarts and rebuilds.
- **AWS EC2 deployment** with Terraform provisioning the instance, then the same Compose stack runs unchanged on the cloud host.

---

## How to Run Locally

```bash
# 1. Clone
git clone https://github.com/drdre4664/epicbook-docker-capstone.git
cd epicbook-docker-capstone

# 2. Create your .env from the template and fill in values
cp .env.example .env
# edit .env to set DB_ROOT_PASSWORD, DB_NAME, DB_USER, DB_PASSWORD

# 3. Build and start the stack
docker compose up --build
```

Once all services report `(healthy)` in `docker compose ps`, the app is reachable at <http://localhost>.

---

## How to Deploy to AWS

```bash
# 1. Provision the EC2 instance with Terraform
cd terraform/
terraform init
terraform apply

# 2. Copy project files to the instance
scp -i ~/.ssh/<your-key>.pem -r ../docker-compose.yml ../app ../proxy ../.env \
    ubuntu@<EC2-PUBLIC-IP>:~/epicbook-capstone/

# 3. SSH in and start the stack
ssh -i ~/.ssh/<your-key>.pem ubuntu@<EC2-PUBLIC-IP>
cd ~/epicbook-capstone
docker compose up -d
```

The Terraform module opens port 80 in the security group so the proxy is reachable from the internet.

---

## Lessons Learned

| Problem | Root Cause | Fix |
|---|---|---|
| App crash-looped on startup | `depends_on` only waits for container start, not MySQL readiness | Added a healthcheck and `condition: service_healthy` on the dependency |
| Proxy healthcheck failing | Alpine's `localhost` resolves to IPv6, but Nginx was listening on IPv4 | Changed healthcheck URL to use `127.0.0.1` instead of `localhost` |
| Docker Compose not found on EC2 | Ubuntu's default apt repo doesn't ship `docker-compose-plugin` | Installed Docker from the official repo via `get.docker.com` |
| Two Docker versions conflicting on EC2 | Both `docker.io` (Ubuntu) and Docker CE (official) were installed | Removed `docker.io` and kept only the official Docker CE packages |

---

## Documentation

In-depth notes on each design decision live under `docs/`:

- [`docs/architecture.md`](docs/architecture.md) — three-tier separation and Docker networking
- [`docs/healthchecks.md`](docs/healthchecks.md) — `depends_on` vs `service_healthy`
- [`docs/multi-stage-builds.md`](docs/multi-stage-builds.md) — image size and security benefits
- [`docs/volumes-and-persistence.md`](docs/volumes-and-persistence.md) — named volumes vs bind mounts
- [`docs/logging.md`](docs/logging.md) — JSON log driver and Nginx structured logs

See [`RUNBOOK.md`](RUNBOOK.md) for day-2 operations (start, stop, restart, recovery, DB access).
