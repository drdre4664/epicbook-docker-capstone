# Logging

Logs are how you know what the stack actually did. Two pieces have to be right:

1. The **driver** that captures stdout/stderr from each container.
2. The **format** the application writes in.

EpicBook configures both: every service uses Docker’s `json-file` driver with rotation, and Nginx is configured to emit structured JSON access logs.

## The `json-file` Logging Driver

Docker can ship container output through several drivers (`json-file`, `local`, `syslog`, `journald`, `fluentd`, etc.). `json-file` is the default and writes one JSON object per line to a file under `/var/lib/docker/containers/<id>/<id>-json.log` on the host.

Each line looks like:

```json
{"log":"GET /books 200\n","stream":"stdout","time":"2025-05-26T10:14:22.512Z"}
```

This is what `docker compose logs` reads when you ask for output.

## Why We Rotate Logs: `max-size` and `max-file`

On a busy server, an unrotated log file grows forever and eventually fills the disk — a classic outage cause. Compose lets us cap that per-service:

```yaml
logging:
  driver: json-file
  options:
    max-size: "10m"
    max-file: "3"
```

- **`max-size: "10m"`** — each log file is capped at 10 MB. When it hits the cap, Docker rotates it.
- **`max-file: "3"`** — keep at most 3 rotated files. The oldest is deleted when a new rotation happens.

Upper bound per service: `10 MB × 3 = 30 MB`. With three services, the worst-case footprint is ~90 MB — predictable and safe.

## Structured JSON Access Logs in Nginx

Nginx’s default `combined` log format is a single space-separated line. It’s human-readable but a pain to grep, ship to ELK, or query in CloudWatch Logs Insights. The proxy is configured with a JSON format instead:

```nginx
log_format json_combined escape=json
    '{'
    '"time":"$time_iso8601",'
    '"method":"$request_method",'
    '"uri":"$request_uri",'
    '"status":$status,'
    '"bytes_sent":$body_bytes_sent,'
    '"request_time":$request_time,'
    '"remote_addr":"$remote_addr",'
    '"user_agent":"$http_user_agent"'
    '}';

access_log /var/log/nginx/access.log json_combined;
```

The `escape=json` directive tells Nginx to properly escape quotes and backslashes inside values — critical so a user-agent string with quotes in it doesn’t break parsing.

A log line now looks like:

```json
{"time":"2025-05-26T10:14:22+00:00","method":"GET","uri":"/books/42","status":200,"bytes_sent":1843,"request_time":0.012,"remote_addr":"203.0.113.7","user_agent":"Mozilla/5.0 ..."}
```

The file is written into the `proxy_logs` named volume so logs survive container rebuilds.

## Reading Structured Logs

**Tail and pretty-print:**

```bash
docker exec epicbook-capstone-proxy-1 tail -f /var/log/nginx/access.log | jq
```

**Filter by status code:**

```bash
docker exec epicbook-capstone-proxy-1 cat /var/log/nginx/access.log \
  | jq -c 'select(.status >= 500)'
```

**Slowest requests:**

```bash
docker exec epicbook-capstone-proxy-1 cat /var/log/nginx/access.log \
  | jq -s 'sort_by(-.request_time) | .[0:10]'
```

**Container-level (Docker driver output):**

```bash
docker compose logs -f proxy
docker compose logs --tail=100 app
```

Because every line is valid JSON, the same logs can be shipped unchanged to Loki, CloudWatch, Datadog, or any other structured log backend.
