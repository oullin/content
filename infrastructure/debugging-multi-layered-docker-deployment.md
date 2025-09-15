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

    Why? On Caddy, enabling TLS client authentication implies `strict_sni_host`. The server expects the **SNI** (TLS `ServerName`) to match the **Host** it sees for that site; if they don’t match, the request is rejected with `421`. [Caddy Docs](https://caddyserver.com/docs/caddyfile/options)  
    Also, `421` broadly means “this server isn’t configured to answer for that scheme/authority”—consistent with an SNI/Host mismatch. [Mozilla Docs](https://developer.mozilla.org/en-US/docs/Web/HTTP/Reference/Status/421)

-   **A stray `404` after we “fixed” 421**  

    We were rewriting `/relay/*` to `/api{path}` but the API’s `:8443` site needed to **strip** `/api` before proxying to `api:8080`. Missing that produced 404s at the backend route.

## Root Causes (with Caddy file gotchas)

-   **SNI/Host mismatch on the mTLS hop**  

    API Caddy’s `:8443` listener had client auth enabled. That turns on strict SNI checks; if the web relay connects with SNI 
    different from the Host the upstream Caddy expects, it emits `421`. Fix: on the _web_ side of the relay, set both the **TLS SNI** 
    and the **forwarded Host** to the upstream service name exposed by Caddy (`oullin_proxy_prod`).

    -   `tls_server_name oullin_proxy_prod` (SNI) in the transport
    -   `header_up Host oullin_proxy_prod` to make the HTTP Host consistent  
        (The reverse proxy module uses TLS to the upstream; SNI and client cert options live under `transport http { tls ... }`.) [Caddy Docs](https://caddyserver.com/docs/modules/http.reverse_proxy.transport.http)

-   **Using `handle_path` correctly**  

    `handle_path` is the right tool to **strip** the matched prefix before routing to sub-handlers (as opposed to `handle`, which preserves the path). That’s how we remove `/relay` on the web tier, and `/api` on the API mTLS tier. [Caddy Docs](https://caddyserver.com/docs/caddyfile/directives/handle_path)

-   **Named matchers + `handle_path` scope**  

    `handle_path` accepts a _path matcher_ right in its header and cannot be combined with named matchers there; named matchers must be defined at site scope and used in nested `handle` blocks. We used that to return `405` on accidental GETs to `/relay/*`. [Caddy Docs](https://caddyserver.com/docs/caddyfile/directives/handle_path)

-   **Client auth config names**  

    On the **upstream TLS** hop (web → API :8443), we used `tls_client_auth <cert> <key>` and `tls_server_name`. On the **server-side** TLS (API `:8443` site), we used `client_auth { mode require_and_verify; trust_pool file /etc/caddy/mtls/ca.pem }`. (`trust_pool` is the current CA trust mechanism for client auth.) [Caddy Docs](https://caddyserver.com/docs/modules/http.reverse_proxy.transport.http)