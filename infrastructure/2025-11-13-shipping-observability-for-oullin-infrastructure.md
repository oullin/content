# Shipping Observability for Oullin API: What We Built and How You Use It

Over the last cycle, I added a focused observability stack to the Oullin API. It gives us a clear picture of the app in production without exposing sensitive endpoints. This post explains what we built, how it’s wired, and the exact steps to run, access, and maintain it.

## What we shipped

-   **Prometheus** to scrape metrics from the API, Postgres, and the reverse proxy.
-   **Grafana** with pre-provisioned data sources and dashboards stored in the repo.
-   **An API `/metrics` endpoint** (Prometheus client) enabled behind the internal Docker network.
-   **A Postgres exporter** to collect DB health and performance signals.
-   **Compose profiles** (`local`, `prod`) so you can switch between laptop and VPS without changing files.
-   **A small runbook** with Make targets, file paths, and examples for day-to-day work.

## Folder layout (at a glance)

```bash
infra/metrics/
  grafana/
    provisioning/         # datasources + dashboards as code
  prometheus/
    prometheus.yml        # scrape configs
  docker-compose.yml      # grafana + prometheus (+ exporters)
```


Keeping dashboards and data sources in the repo means a fresh machine comes up ready, with no manual clicks required.

## How it’s wired

-   **API** exposes `/metrics` only inside the Docker network.
-   **Prometheus** scrapes targets by **service name** over the shared network (no localhost hacks).
-   **Grafana** reads from Prometheus and serves the dashboards.
-   **Caddy** continues to handle public traffic; it does **not** route the monitoring UIs or `/metrics`.
    

## Run it locally vs in production

**Local (fast feedback):**

```bash
# from repo root
docker compose -f infra/metrics/docker-compose.yml --profile local up -d
```

**Production (locked down):**

```bash
# create shared network once
docker network create oullin_net || true

# monitoring lives in its own services
docker compose -p oullin_infra -f docker-compose.yml --profile prod up -d

# app lives in a different project
docker compose -p oullin_app -f docker-compose.yml --profile prod up -d
```

Splitting projects prevents monitoring bounces during app deploys.

## Access without exposing anything

Keep Prometheus and Grafana private. To view them on a VPS:

```bash
# from your laptop
ssh -N \
  -L 3000:<grafana_container_ip>:3000 \
  -L 9090:<prom_container_ip>:9090 \
  user@your-vps
```

Then open:

-   Grafana → `http://localhost:3000`
    
-   Prometheus → `http://localhost:9090`
    

You can use the Grafana admin password you configured. Do not publish ports in `prod`.

## Make targets you’ll actually use

-   **Bring monitoring up** (respecting profiles):  
    `make metrics-up`
    
-   **Bring monitoring down (local only)**:  
    `make metrics-down`
    
-   **Tail logs when debugging**:  
    `make metrics-logs`
    
-   **Export dashboards from Grafana to the repo** (so changes are versioned):  
    `make grafana-export`
    

These wrap Docker Compose commands to keep the workflow repeatable.

## Health, persistence, and safety nets

-   **Health checks**:
    
    -   Prometheus: `/-/ready`
        
    -   Grafana: `/api/health`
        
-   **Restart policy**: `unless-stopped` for both.
    
-   **Volumes**: persistent data for Prometheus and Grafana, so re-creates don’t lose history.
    
-   **Resource limits** (recommended in `prod`): cap CPU and memory to protect the VPS.
    

## Prometheus scrape basics

-   Targets are defined under `infra/metrics/prometheus/prometheus.yml`.
    
-   Jobs point to container DNS (for example, `api:8080/metrics`).
    
-   You can add more exporters by declaring a new job and joining the same network.
    

## Dashboards and datasources as code

-   Grafana **provisioning** files live under `infra/metrics/grafana/provisioning/`.
    
-   Datasources are set to the Prometheus service URL on the Docker network.
    
-   Dashboards are JSON files in the repo. You can edit in Grafana, then run `make grafana-export` to save them back to git. No manual rework after rebuilds.
    

## Safe deployment routine (copy/paste)

```bash
# 0) Ensure monitoring is up separately
docker compose -p oullin_infra -f docker-compose.yml --profile prod up -d

# 1) Deploy/update the app without stopping the world
docker compose -p oullin_app -f docker-compose.yml pull
docker compose -p oullin_app -f docker-compose.yml up -d

# 2) Quick smoke checks on the VPS
curl -sf http://prometheus:9090/-/ready
curl -sf http://grafana:3000/api/health
docker exec -it $(docker ps -qf name=prometheus) sh -lc 'wget -qO- http://api:8080/metrics | head'
```

## Systemd on the VPS (so it survives reboots)

Create `/etc/systemd/system/oullin-metrics.service`:

```bash
[Unit]
Description=Oullin Monitoring Stack
After=docker.service
Requires=docker.service

[Service]
Type=oneshot
WorkingDirectory=/srv/oullin
ExecStart=/usr/bin/docker compose -p oullin_infra -f docker-compose.yml --profile prod up -d
ExecStop=/usr/bin/docker compose -p oullin_infra -f docker-compose.yml --profile prod stop
RemainAfterExit=yes

[Install]
WantedBy=multi-user.target
```

Then:

```bash
sudo systemctl daemon-reload
sudo systemctl enable oullin-metrics
sudo systemctl start oullin-metrics
```

## What this gives us

-   **Throughput and latency** per handler, plus error rates.
    
-   **Runtime signals** (goroutines, GC, memory, CPU).
    
-   **Database health** via exporter metrics.
    
-   **Reverse proxy visibility** for upstream timings.
    
-   **Ready-to-use dashboards** you can extend without losing changes.
    

## Guardrails we follow

-   Never expose `/metrics` or the UIs publicly.
    
-   Keep monitoring and the app in separate Compose **projects**.
    
-   Back up the Grafana and Prometheus volumes.
    
-   Version dashboards and datasources in git.
    
-   No `docker compose down` during deploys; use `up -d` and update in place.
    

## What’s next

-   Add SLO panels for latency and error rate.
    
-   Expand Postgres coverage (slow queries, index bloat).
    
-   Consider a lightweight logs view if needed, only if it stays small on the VPS.
    
-   Nightly backup of monitoring volumes.
    

* * *

This setup is small, safe, and repeatable. It gives us the right signals to debug and the confidence to ship more often—without opening anything we shouldn’t.


