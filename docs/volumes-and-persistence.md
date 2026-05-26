# Volumes and Persistence

Containers are designed to be **disposable**. When a container is removed, everything written inside it is gone. That’s great for stateless tiers like the app and proxy, but it’s a disaster for the database. Volumes are how Docker keeps state alive across the container lifecycle.

## Named Volumes vs. Bind Mounts

**Bind mount** — maps a specific path on the **host** into the container:

```yaml
volumes:
  - ./proxy/nginx.conf:/etc/nginx/conf.d/default.conf:ro
```

Used for **configuration files** the operator owns and edits directly. The path is human-meaningful (`./proxy/nginx.conf`) and lives in the repo.

**Named volume** — Docker manages the storage location for you:

```yaml
volumes:
  db_data:
  proxy_logs:

services:
  db:
    volumes:
      - db_data:/var/lib/mysql
  proxy:
    volumes:
      - proxy_logs:/var/log/nginx
```

Used for **data the container owns** — database files, log files, anything that’s an internal implementation detail. Docker stores the bytes under `/var/lib/docker/volumes/<name>/` on the host and you reference it by name.

| | Bind mount | Named volume |
|---|---|---|
| Path on host | Chosen by you | Managed by Docker |
| Good for | Config files, source code in dev | Database files, log files, cache |
| Portability | Tied to host filesystem | Portable across hosts |
| Backups | Just copy the file | `docker run --rm -v db_data:/data ...` |

## Why `db_data` Matters

MySQL writes its data files to `/var/lib/mysql` inside the container. Without a volume, those files live in the container’s writable layer and disappear the moment you run `docker compose down` or rebuild the image.

Binding `db_data` to `/var/lib/mysql` means:

- `docker compose down` → books and orders **still there** on the next `up`.
- `docker compose pull && docker compose up -d` (image upgrade) → data carries forward to the new container.
- Host reboot → data carries forward.

The only thing that wipes `db_data` is explicitly removing it: `docker compose down -v` or `docker volume rm epicbook-docker-capstone_db_data`.

## How to Verify Persistence

```bash
# 1. Start the stack and insert something
docker compose up -d
docker exec -it epicbook-capstone-db-1 mysql -u root -p«password» -e   "USE bookstore; INSERT INTO books (title) VALUES ('Persistence Test');"

# 2. Tear down (containers go away, volume stays)
docker compose down

# 3. Start again from scratch
docker compose up -d

# 4. The row should still be there
docker exec -it epicbook-capstone-db-1 mysql -u root -p«password» -e   "USE bookstore; SELECT * FROM books WHERE title='Persistence Test';"
```

Inspect the volume directly:

```bash
docker volume ls
docker volume inspect epicbook-docker-capstone_db_data
```
