# Building Oullin: How I Built My Personal Site and Blog Engine with Go, Docker, Caddy, Vue, and a Modest VPS

> Today I’m launching my personal website and blog — [oullin.io](https://oullin.io). I built it from the ground up to be fast, simple, and entirely under my control. I also built it to scratch a deeper itch: I love programming, and I try to learn something new every day. Building this stack in public keeps me curious and keeps me honest about what works in the real world.
>
> The entire codebase is public and open source. If you want to explore or reuse parts of it, please take a look at the link I've included to the [oullin organisation](https://github.com/oullin) in GitHub. The READMEs cover local setup with Docker Compose and production notes with Caddy. I want you to know that issues and pull requests are welcome if you spot bugs, have ideas, or want to contribute.

I wanted a personal site that felt **fast**, **simple**, and **mine**. No heavy CMS. No mystery plugins. Just a small stack I could understand from start to finish and iterate on forever.

This post is the story of how I built [oullin.io](https://oullin.io) from zero: the decisions, the trade-offs, the stack, and the scars. I’ll keep it technical without losing plain English, and share a few copy-paste snippets you can adapt.


## Why build from scratch?

- **Ownership.** I wanted to control the data model, the content pipeline, and the performance footprint.
- **Learning.** Shipping a real thing forces decisions: how I structure Go packages, how I proxy with Caddy, how I containerise and deploy on a no-frills VPS.
- **Longevity.** A simple codebase ages better than a pile of plugins.


## The stack at a glance

- **Go (API + generators).** One binary for content APIs and a tiny admin; separate CLI tasks for SEO/meta generation.
- **Vue + Vite (web).** A lean SPA for the reader experience.
- **Caddy (edge).** TLS, HTTP/2/3, caching hints, clean reverse proxy.
- **PostgreSQL (persistence).** Boring, reliable, well-documented.
- **Docker Compose (ops).** One file, multiple profiles (local vs prod).
- **A modest VPS.** Enough CPU/RAM to be honest about performance.


## Architecture in 30 seconds

```
[Browser]  ↔  [Caddy]  ↔  [Go API]  ↔  [PostgreSQL]
    │             │             │
    └─ Static assets (Vite)     └─ Migrations, content store
```

- **Caddy** serves the Vue app and proxies `/api/*` to the Go service.
- The **Go API** returns posts, pages, tags, and search; it also exposes a private admin endpoint.
- **PostgreSQL** stores canonical content (markdown + metadata).
- A **Go CLI** task pre-generates SEO pages and JSON-LD when I publish.


## Local dev: one command, no foot-guns

I keep local and prod close, but not identical. Compose profiles let me toggle dev-only bits (hot reload, bind mounts) without changing the file.

```yaml
# docker-compose.yml (excerpt)
services:
  web:
    image: node:22-alpine
    working_dir: /app
    volumes: [ "./web:/app" ]
    command: ["sh","-c","npm ci && npm run dev"]
    ports: [ "5173:5173" ]
    profiles: [ "local" ]

  api:
    build: ./api
    environment:
      - DB_HOST=api-db
    depends_on: [ "api-db" ]
    ports: [ "8080:8080" ]
    profiles: [ "local", "prod" ]

  api-db:
    image: postgres:16-alpine
    environment:
      - POSTGRES_DB=oullin
      - POSTGRES_USER=oullin
      - POSTGRES_PASSWORD=devpass
    volumes:
      - oullin_db_data:/var/lib/postgresql/data
    profiles: [ "local", "prod" ]

volumes:
  oullin_db_data: {}
```

Local loop:

```bash
docker compose --profile local up -d --build
```

- Vue runs with Vite hot refresh.
- Go auto-rebuilds on file change (tiny `air` config).
- DB is a named volume; blowing it away is a one-liner when I change schemas.


## Production: predictable over clever

In production, I want fewer moving parts:

- **No bind mounts.** Only images.
- **Environment via Docker secrets** (DB creds, tokens).
- **Healthchecks** on API and DB.
- **Caddy** as the edge proxy and static file server.

A minimal prod profile:

```yaml
# docker-compose.prod.yml (overlay)
services:
  web:
    build:
      context: ./web
      dockerfile: Dockerfile
    command: ["node","server.mjs"] # for prerender or asset serve
    ports: [ "8081:8081" ]

  api:
    image: ghcr.io/oullin/api:latest
    secrets: [ "pg_username", "pg_password" ]
    environment:
      - DB_HOST=api-db
      - DB_USER_FILE=/run/secrets/pg_username
      - DB_PASS_FILE=/run/secrets/pg_password
    healthcheck:
      test: ["CMD","/app/healthcheck"]
      interval: 10s
      timeout: 2s
      retries: 5

  caddy:
    image: caddy:2.10
    volumes:
      - ./caddy/Caddyfile:/etc/caddy/Caddyfile:ro
      - caddy_data:/data
      - caddy_config:/config
    ports: [ "80:80", "443:443" ]
    depends_on: [ "web","api" ]

secrets:
  pg_username:
    file: /etc/oullin/secrets/pg_username
  pg_password:
    file: /etc/oullin/secrets/pg_password

volumes: { caddy_data: {}, caddy_config: {} }
```

Deploying is intentionally boring:

```bash
docker compose -f docker-compose.yml -f docker-compose.prod.yml   --profile prod up -d --build
```


## Caddy: the friendly edge

Caddy’s config is short and readable. It handles certificates, HTTP/3, and sane defaults without a tangle of directives.

```caddy
# caddy/Caddyfile (excerpt)
oullin.io, www.oullin.io {
  encode zstd gzip
  @api path /api/*

  handle @api {
    reverse_proxy api:8080
  }

  handle {
    root * /srv/web
    try_files {path} /index.html
    file_server
  }

  header {
    Strict-Transport-Security "max-age=31536000; includeSubDomains; preload"
    Referrer-Policy "strict-origin-when-cross-origin"
    Permissions-Policy "interest-cohort=()"
  }
}
```

- **`try_files` → `/index.html`** gives the SPA clean URLs.
- **Reverse proxy** routes `/api` to Go.
- **Headers** are set once at the edge, not inside every app.


## The Go API: small, fast, explicit

The API does three things well:

1. **Serve content** (posts, pages, tags).
2. **Index content** (for a simple search).
3. **Generate structured data** (Open Graph, JSON-LD, canonical links).

A thin handler keeps the shape obvious:

```go
func (h *PostsHandler) Show(w baseHttp.ResponseWriter, r *baseHttp.Request) *http.ApiError {
	slug := payload.GetSlugFrom(r)

	if slug == "" {
		return http.BadRequestError("Slugs are required to show posts content")
	}

	post := h.Posts.FindBy(slug)
	if post == nil {
		return http.NotFound(fmt.Sprintf("The given post '%s' was not found", slug))
	}

	items := payload.GetPostsResponse(*post)
	if err := json.NewEncoder(w).Encode(items); err != nil {
		slog.Error(err.Error())

		return http.InternalError("There was an issue processing the response. Please, try later.")
	}

	return nil
}
```

**Why Go?** It’s fast, static, and predictable under load. The standard library covers most needs. Concurrency is there when I want it, invisible when I don’t.

**Data model?** Markdown + front-matter for content; normalised tables for tags, authors, and redirects. I keep migrations as SQL files (plain text is future-proof).


## Vue front-end: lightweight and delightful

Vue + Vite is a pleasant default:

- Fast dev server, tiny build output.
- Component model that maps well to post layouts and lists.
- Enough flexibility for progressive enhancement.

I render lists quickly, then **hydrate** details (related posts, reading time, share buttons) after first paint. It keeps the page interactive without drowning in JS.

### About SEO and rendering

For this project, I avoided a heavy SSR layer. Instead, I generate **static SEO pages** (lightweight HTML for crawlers and social previews) with a [Go CLI task](https://oullin.io/post/2025-09-25-shipping-seo-for-a-single-page-app-the-pragmatic-way) and let the SPA own the interactive experience. It’s a nice middle path: great previews without runtime SSR complexity.


## A few war stories (and what they taught me)

- **Security profiles can trip Docker on VPSes.** On one provider, the default AppArmor profile blocked harmless operations during image builds and DB startup. The fix was to (temporarily) relax the profile during builds and, when necessary, mark a couple of containers as `apparmor:unconfined`. Lesson: *production is a place, not a theory*. Test on the real box early.

- **Secrets are sharp.** I once mounted the wrong DB username via Docker secrets. Everything “looked” fine until the API kept failing auth. Lesson: verify **source files** for secrets, not just the container mounts. A single wrong line can cost hours.

- **Make fewer layers.** I started with a fancy deployment script that wrapped `make` that wrapped `compose` that wrapped… everything. When errors hit, the stack traces weren’t helpful. I now deploy with **direct `docker compose` commands** and a small Makefile for local chores.


## Observability: trust but verify

- **Healthchecks** on API and DB (Compose) to detect boot loops early.
- **`docker logs -f`** tails everything during deploy; it’s amazing how far raw logs get you if your services start fast.
- **Curl checks with a custom User-Agent** (e.g. `Twitterbot/1.0`) to validate social meta and canonical tags.
- **Database logs** in slow-query mode for the first week after migrating content.


## Performance: simple wins

- Serve **immutable assets** (`cache-control: immutable, max-age=31536000`).
- Inline only **critical CSS** for above-the-fold; lazy-load the rest.
- Ship **SVGs** for icons, **AVIF/WebP** for images with PNG fallbacks.
- Keep payloads small: compress JSON, paginate lists, and prefer **select N + join** over N+1 queries.


## Publishing flow (the part I enjoy most)

1. Draft in markdown (with a few front-matter fields).
2. Run `make publish`:
   - Validate front-matter.
   - Embed canonical + JSON-LD.
   - Build images for `web` and `api`.
   - Run migrations if needed.
3. `docker compose ... up -d --build`
4. Sanity checks:
   - `curl -I https://oullin.io` (headers look right?)
   - `curl -A "Twitterbot/1.0" ...` (previews OK?)
   - Quick read on mobile.

Everything either **ships** or **fails loudly**. That’s the point.


## What I’d do differently next time

- Consider **hybrid SSG** for sections like the blog index if the content shape grows (e.g. monthly archives), while keeping the SPA for reading experience.
- Add a tiny **admin UI** for drafts and image uploads (right now it’s CLI-driven).
- Bake in **structured audit logs** on publish to make rollbacks even easier.


## Minimal, reusable snippets

**Caddy header hardening**

```bash
header {
  Strict-Transport-Security "max-age=31536000; includeSubDomains; preload"
  Referrer-Policy "strict-origin-when-cross-origin"
  X-Content-Type-Options "nosniff"
  X-Frame-Options "DENY"
  Content-Security-Policy "default-src 'self'; img-src 'self' https: data:; font-src 'self' https: data:; script-src 'self'; style-src 'self' 'unsafe-inline'"
}
```

**Go: safe JSON helpers**

```go
func JSON(w http.ResponseWriter, status int, v any) {
    w.Header().Set("Content-Type", "application/json; charset=utf-8")
    w.WriteHeader(status)
    _ = json.NewEncoder(w).Encode(v)
}
```

**Compose: healthcheck pattern**

```yaml
healthcheck:
  test: ["CMD-SHELL","wget -qO- http://localhost:8080/health || exit 1"]
  interval: 10s
  timeout: 3s
  retries: 3
  start_period: 5s
```


## Plain-English glossary

- **API** — A program that other programs talk to.
- **Caddy** — A friendly web server that handles HTTPS and proxies requests.
- **Container** — A lightweight runtime box for an app and its dependencies.
- **Docker Compose** — A file that says which containers to run and how they connect.
- **JSON-LD** — Structured data embedded on a page so search engines understand it.
- **Proxy** — A server that forwards requests to another server.
- **SPA (Single-Page App)** — A web app that loads once and updates the page with JavaScript.
- **SSG (Static-Site Generation)** — Building HTML pages ahead of time, at publish time.
- **TLS** — Encryption for web traffic (what makes the lock icon appear).
- **VPS** — A rented server with your own OS, CPU, and disk.


## Closing

Building [oullin.io](https://oullin.io) from scratch was not about reinventing the wheel. It was about owning the wheel: understanding every turn from browser to database, and making a stack I can grow with. The payoff is a site that loads fast, behaves predictably, and keeps me excited to ship the next post.

If you’re considering doing the same: pick boring tools, write small services, keep configs readable, and deploy with the fewest steps you can live with. The rest is just writing.

**Open source repos:**  
- API: [https://github.com/oullin/api](https://github.com/oullin/api)  
- Web: [https://github.com/oullin/web](https://github.com/oullin/web)
- Infra: [https://github.com/oullin/infra](https://github.com/oullin/infra)
- Articles: [https://github.com/oullin/content](https://github.com/oullin/content)
