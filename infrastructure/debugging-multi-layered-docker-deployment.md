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

The final config delivers 200s end-to-end with mTLS, and we added guards (**405** on GET) and **CI/Makefile** validation.

## Glossary of Terms

| Term                                       | Definition                                                                                               |
|--------------------------------------------|----------------------------------------------------------------------------------------------------------|
| **Makefile**                               | A file used by `make` to define automation rules for building and deployment.                            |
| **Docker Compose**                         | A tool for defining and running multi-container Docker applications.                                     |
| **SNI (Server Name Indication)**           | An extension to the TLS protocol that allows a client to specify the hostname it's connecting to         |
| **Security Profile**                       | A set of rules in AppArmor that defines what actions a process can perform.                              |
| **TLS (Transport Layer Security)**         | Protocol for encrypting data between servers and clients.                                                |
| **mTLS** (Mutual Transport Layer Security) | mTLS is a security protocol that provides two-way authentication and encryption for network connections. |
| **CA** (A Certificate Authority)           | A trusted entity that issues digital certificates to verify the identity of websites.                    |
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

## The Step-by-Step Debugging & Hotfixes

### A) Prove basic network and container naming

-   Confirm both stacks share `caddy_net` and can resolve each other:
```docker
docker network inspect caddy_net | jq '.[]?.Containers | to_entries[] | {name: .value.Name, ipv4: .value.IPv4Address}'
```
-   Result showed `oullin_proxy_prod`, `web_caddy_prod`, `oullin_api` on `172.19.0.0/16`.


### B) Fix the mTLS trust chain (web → API)

-   Rotate/regenerate CA and client leaf on the API side.
-   **Sync** `ca.pem`, `client.pem`, `client.key` into `web_caddy_prod` (mounted `caddy/mtls:ro`).
-   Test from a web container:
```shell
curl -vk \
  --cert /etc/caddy/mtls/client.pem \
  --key  /etc/caddy/mtls/client.key \
  --cacert /etc/caddy/mtls/ca.pem \
  https://oullin_proxy_prod:8443/api/generate-signature
```

-   (Client cert flags are standard `curl` TLS client-auth options.)

### C) Eliminate `421` (SNI/Host consistency)

-   In **web** Caddy’s relay `reverse_proxy`, set:

```shell
"reverse_proxy https://oullin_proxy_prod:8443 {
  header_up Host oullin_proxy_prod
  transport http {
    tls
    tls_server_name oullin_proxy_prod
    tls_client_auth /etc/caddy/mtls/client.pem /etc/caddy/mtls/client.key
    tls_trust_pool file /etc/caddy/mtls/ca.pem
  }
}"
```

-   (Client cert flags are standard `curl` TLS client-auth options.)


### C) Eliminate `421` (SNI/Host consistency)

-   In **web** Caddy’s relay `reverse_proxy`, set:

```shell
reverse_proxy https://oullin_proxy_prod:8443 {
  header_up Host oullin_proxy_prod
  transport http {
    tls
    tls_server_name oullin_proxy_prod
    tls_client_auth /etc/caddy/mtls/client.pem /etc/caddy/mtls/client.key
    tls_trust_pool file /etc/caddy/mtls/ca.pem
  }
}
```
-   This ensures the TLS SNI and HTTP Host seen by API Caddy match its site. `421` disappeared. (Strict SNI behaviour with client auth is documented.) [Caddy Docs](https://caddyserver.com/docs/caddyfile/options?utm_source=chatgpt.com)


### D) Fix the path to rewrite chain and method guards

**Web Caddy (`:80`)** — strip `/relay`, rewrite to `/api{path}` and **deny GET**:

```shell
":80 {
  @relay_get {
    path /relay/*
    method GET
  }

  handle_path /relay/* {
    handle @relay_get {
      respond 405
    }

    rewrite * /api{path}

    reverse_proxy https://oullin_proxy_prod:8443 {
      header_up Host oullin_proxy_prod
      transport http {
        tls
        tls_server_name oullin_proxy_prod
        tls_client_auth /etc/caddy/mtls/client.pem /etc/caddy/mtls/client.key
        tls_trust_pool file /etc/caddy/mtls/ca.pem
      }
    }
  }

  # Static SPA
  handle {
    root * /usr/share/caddy
    try_files {path} /index.html
    file_server
  }

  log {
    output stdout
    format json
  }
}"
```
Why `handle_path`? Because it **strips** `/relay` before sub-handlers run. [Caddy Docs](https://caddyserver.com/docs/caddyfile/directives/handle_path?utm_source=chatgpt.com)

**API Caddy (`:8443`)** — require client certs, **strip `/api`** then proxy:
```shell
":8443 {
  tls /etc/caddy/mtls/server.pem /etc/caddy/mtls/server.key {
    client_auth {
      mode require_and_verify
      trust_pool file /etc/caddy/mtls/ca.pem
    }
  }

  encode gzip zstd

  @sig path /api/generate-signature*
  handle @sig {
    uri strip_prefix /api
    reverse_proxy api:8080
  }

  # deny anything else on the mTLS port
  handle {
    respond 403
  }
}"
```

-   `uri strip_prefix /api` is the fix for the earlier `404` (backend expects `/generate-signature*`).
-   `trust_pool` is the modern way to point Caddy at your client-CA trust source. [Caddy Docs](https://caddyserver.com/docs/caddyfile/directives/tls?utm_source=chatgpt.com)


### E) CI & validation

-   **Pin Caddy 2.10.0** everywhere (web image, API build arg, CI Docker image).
-   Add **format + validate** steps in CI for both Caddy files:

```docker
caddy fmt --overwrite /etc/caddy/Caddyfile
caddy validate --config /etc/caddy/Caddyfile
```

-   Add an integration test that POSTS to `/relay/generate-signature` in a compose-up ephemeral environment to catch regressions.


### F) Final guards & hygiene

-   Keep `caddy/mtls` mounts `:ro` on both stacks.
-   Remove redundant `header_up` for `X-Forwarded-*` (Caddy forwards by default). [Caddy Docs](https://caddyserver.com/docs/caddyfile/directives/reverse_proxy?utm_source=chatgpt.com)
-   Optionally short-circuit **CORS preflight** on `/relay/*` with a `204` response if you want to reduce hops.


## The Win: Verified 200s

After SNI/Host alignment + path fixes + CA sync, the browser showed:

```yaml
Status Code: 200 OK
content-type: application/json
etag: W/"0.0.1"
via: 1.1 Caddy  →  2.0 Caddy  →  1.1 Caddy
```

That means: Cloudflare → API Caddy (public) → web Caddy (relay) → API Caddy (:8443 mTLS) → Go API is working exactly as designed.


## Practical Lessons

-   **When you enable client auth, expect strict SNI behavior.** Always set `tls_server_name` (SNI) and make sure the upstream sees a matching `Host`. Otherwise you’ll chase `421`s. [Caddy Docs](https://caddyserver.com/docs/caddyfile/options?utm_source=chatgpt.com)
-   **Use `handle_path` whenever you intend to strip path prefixes.** It avoids double-rewrites and confusing `404`s. [Caddy Docs](https://caddyserver.com/docs/caddyfile/directives/handle_path?utm_source=chatgpt.com)
-   **Scope named matchers outside `handle_path`.** Use nested `handle` blocks to apply them (e.g., 405 on GET). [Caddy Docs+1](https://caddyserver.com/docs/caddyfile/directives/handle_path?utm_source=chatgpt.com)
-   **Pin Caddy versions across CI/Dev/Prod.** The parser gets stricter over time; pinning avoids surprise breakages.

## References & Further Reading

-   **Caddy file `handle_path`** — strips the matched prefix; cannot use named matchers directly in the header. [Caddy Docs](https://caddyserver.com/docs/caddyfile/directives/handle_path?utm_source=chatgpt.com)
-   **Caddy file `handle` (ordering, nesting)** — how handlers sort and interact. [Caddy Docs](https://caddyserver.com/docs/caddyfile/directives/handle?utm_source=chatgpt.com)
-   **`reverse_proxy` and HTTP transport TLS options** (SNI, client cert to upstream). [Caddy Docs](https://caddyserver.com/docs/caddyfile/directives/reverse_proxy?utm_source=chatgpt.com)
-   **Client-auth & `trust_pool` in `tls`** (modern CA trust configuration). [Caddy Docs](https://caddyserver.com/docs/caddyfile/directives/tls?utm_source=chatgpt.com)
-   **Why you saw `421`** — definition and strict SNI behaviour when client auth is enabled. [developer.mozilla.org+1](https://developer.mozilla.org/en-US/docs/Web/HTTP/Reference/Status/421?utm_source=chatgpt.com)
-   **When to prefer `handle_path` for path stripping** — community guidance/examples. [Caddy Community](https://caddy.community/t/how-to-get-the-reverse-proxy-directive-to-strip-the-path-of-the-request-before-forwarding-the-request-upstream/16525?utm_source=chatgpt.com)