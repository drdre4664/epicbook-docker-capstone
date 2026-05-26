# EpicBook Runbook

Day-2 operations guide for running, restarting, inspecting, and recovering the EpicBook stack.

## Stack Overview

- **proxy** — Nginx on port 80, routes traffic to `app`
- **app** — Node.js / Express on port 8080
- **db** — MySQL on port 3306 (internal only, not published)

## Start the Stack

```bash
docker compose up -d
```

## Stop the Stack

```bash
docker compose down
```

## Restart a Single Service

```bash
docker compose restart app
docker compose restart db
docker compose restart proxy
```

## Check Status

```bash
docker compose ps
```

All services should show `(healthy)`.

## Check Logs

```bash
docker compose logs
docker compose logs app
docker compose logs proxy
docker compose logs db
docker compose logs -f proxy
```

## Database Access

```bash
docker exec -it epicbook-capstone-db-1 mysql -u epicbook -pepicbook123 bookstore
```

## Recovery Procedures

### App container crashed

Docker restarts it automatically via `restart: unless-stopped`. Check logs if it keeps crashing:

```bash
docker compose logs app
```

### Database won't start

Check logs:

```bash
docker compose logs db
```

Data is stored in the `db_data` named volume and survives restarts.

### Proxy returning 502

The app is down. Check and recover:

```bash
docker compose ps
docker compose logs app
docker compose restart app
```

## Update the App Image

```bash
docker compose down
docker compose build app
docker compose up -d
```

## Cloud Deployment (EC2)

**SSH:**

```bash
ssh -i ~/.ssh/John-key.pem ubuntu@<EC2-PUBLIC-IP>
```

**Project location:** `~/epicbook-capstone`
