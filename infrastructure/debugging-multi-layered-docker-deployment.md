---
title: "Debugging a Multi-Layered Docker & Caddy Deployment"
excerpt: "Troubleshooting process for deploying a Go API with PostgreSQL on a VPS with Caddy and Docker."
slug: "2025-11-15-debugging-multi-layered-docker-deployment"
published_at: 2025-11-15
author: "gocanto"
categories: "infrastructure"
tags: ["infrastructure", "Docker", "Caddy", "PostgreSQL", "Go", "tls", "proxy"]
---
![reverse-proxy](https://github.com/user-attachments/assets/da7d8035-c151-42c3-9db8-1495e14a95d5)

## Debugging a Multi-Layered Docker + Caddy + mTLS Deployment
> **TL;DR**

There is a built “relay” path on the web tier that forwards signed requests to an internal **mTLS** gateway on the API tier. 
Along the way we hit:

- **421 Misdirected** Request loops caused by strict **SNI/Host** checks on the **mTLS** listener,
- a few **Caddy file** gotchas (handle_path ordering, named matcher scope), 
- a **CA** sync issue between web and **API** containers (client cert rejected),
- and a stray **404** from path rewriting.

The final config delivers 200s end-to-end with mTLS, and we added guards (**405** on GET) and **CI** validation.

## Glossary of Terms

| Term                                       | Definition                                                                                               |
|--------------------------------------------|----------------------------------------------------------------------------------------------------------|
| **Makefile**                               | A file used by `make` to define automation rules for building and deployment.                            |
| **Docker Compose**                         | A tool for defining and running multi-container Docker applications.                                     |
| **SNI (Server name indication)**           | It is an addition to the TLS encryption protocol that enables a client device to specify the domain name |
| **Security Profile**                       | A set of rules in AppArmor that defines what actions a process can perform.                              |
| **TLS (Transport Layer Security)**         | Protocol for encrypting data between servers and clients.                                                |
| **mTLS** (Mutual Transport Layer Security) | mTLS is a security protocol that provides two-way authentication and encryption for network connections. |
| **CA** (A Certificate Authority)           | It a trusted entity that issues digital certificates to verify the identity of websites.                 |
| **Docker Secret**                          | A secure way to store sensitive data (like usernames or passwords) in Docker.                            |
| **Entrypoint Script**                      | The startup script that runs when a container is launched.                                               |
| **`security_opt`**                         | A Docker Compose option that modifies container security settings.                                       |
| **VPS (Virtual Private Server)**           | A virtualised server instance hosted by a provider.                                                      |
| **`docker exec`**                          | A Docker command used to run a command inside a running container.                                       |


## Background & Architecture

- **Two Caddy layers, two Compose projects, one shared network.**
  - **API stack** (`oullin_proxy_prod`) terminates public TLS for `oullin.io` and also exposes a **private** mTLS listener on `:8443` for exactly one path: `/api/generate-signature`. Downstream is the Go API at `api:8080`.
  - Web stack (`web_caddy_prod`) serves the SPA on HTTP `:80` and implements a relay entrypoint `/relay/*` that internally calls the API’s mTLS gateway on `oullin_proxy_prod:8443`.
  - Both stacks join the external `caddy_net` network and can reach each other by container name (confirmed with `docker network inspect`).
- **Why mTLS for a single path?** — We wanted a hardened back-channel for signing operations. The public listener explicitly blocks `/api/generate-signature*`; only the internal `:8443` mTLS listener will serve it.

## What We Saw in Development vs Production

-   **Development**
    -   Web Caddy is listening on `:80` only (no auto-HTTPS).
    -   The relay used `handle_path` to strip `/relay` and forward to the API mTLS endpoint.
    -   Local tests from containers worked once certs were aligned and the path rewrite was correct.
-   **Production**
    -   Cloudflare → API Caddy (`oullin_proxy_prod`) for `oullin.io`. The default site in API Caddy reverse-proxies SPA traffic to `web:80`.
    -   The **relay** path still originates at the **web** layer (because SPA code calls `/relay/...`), which means:  
        Cloudflare → API Caddy → **web Caddy** `/relay/*` → **API Caddy :8443 (mTLS)** → `api:8080`.


## Initial Symptoms

-   **Container-side curl to mTLS failed with client-auth errors**

    ```shell
      TLS alert unknown ca
    ```

    > The API side required a client cert signed by its CA; the web side was using an outdated CA bundle (or wrong CA). We synced a fresh `ca.pem`/client pair into `web_caddy_prod` and errors moved on.

-   **End-to-end calls returned `421 Misdirected Request`**  
    Reproducible via both the browser and `curl -H 'Host: oullin.io' ... /relay/generate-signature`.  
    The debug logs showed the upstream (API’s `:8443`) returning `421`, not web Caddy itself.

    Why? On Caddy, enabling TLS client authentication implies `strict_sni_host`. The server expects the **SNI** (TLS `ServerName`) to match the **Host** it sees for that site; if they don’t match, the request is rejected with `421`. [Caddy Web Server](https://caddyserver.com/docs/caddyfile/options?utm_source=chatgpt.com)  
    Also, `421` broadly means “this server isn’t configured to answer for that scheme/authority”—consistent with an SNI/Host mismatch. [developer.mozilla.org](https://developer.mozilla.org/en-US/docs/Web/HTTP/Reference/Status/421?utm_source=chatgpt.com)

-   **A stray `404` after we “fixed” 421**  
    We were rewriting `/relay/*` to `/api{path}` but the API’s `:8443` site needed to **strip** `/api` before proxying to `api:8080`. Missing that produced 404s at the backend route.