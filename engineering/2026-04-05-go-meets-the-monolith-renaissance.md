---
title: "Go Meets the Monolith Renaissance"
excerpt: "The backend-first SPA pattern that Laravel developers love — now fully at home in Go. We built oullin/inertia-go for a real production engagement, open-sourced it because the community deserved it, and ported the official Inertia Kitchen Sink demo to prove it works. One binary. No API layer. No separate frontend deployment. Just Go, Inertia, and a component name."
slug: "2026-04-05-go-meets-the-monolith-renaissance"
published_at: 2026-04-05
author: "gocanto"
categories: "engineering"
tags: ["engineering", "llm", "inertia", "laravel", "typescript"]
---

<img width="740" height="617" alt="network-server-endpoint-management-system_999616-677" src="https://github.com/user-attachments/assets/7cc89c6a-491e-4fdd-9666-2980ca9a2683" />


# Go Meets the Monolith Renaissance: Introducing oullin/inertia-go

*The backend-first SPA pattern that [Laravel](https://laravel.com/) developers love — now fully at home in Go.*


There's a quiet architectural renaissance underway, and most engineers are too deep in their microservice dashboards to notice.

For years, the industry told us that building modern web apps meant one thing: a REST (or GraphQL) API on the backend, a JavaScript SPA on the frontend, two separate repositories, two separate deployment pipelines, two separate teams, and a JSON contract between them that nobody fully owned. We accepted the complexity because we wanted the UX. The deal was: great user experience costs architectural chaos.

Then [Inertia.js](https://inertiajs.com/) came along and said: *what if it didn't have to?*


## The Problem Inertia Solves

Inertia.js is not a framework. It's a protocol — a thin contract between your server-side routing and your client-side components. The insight is almost embarrassingly simple: your server already knows what page to show. Why not let it *say so directly*, instead of serialising everything into JSON for the client to interpret?

With Inertia, you keep your server-side routing. You keep your controllers. You keep your auth middleware, your sessions, your redirects. But instead of rendering HTML templates, you render **page components** — [Vue](https://vuejs.org/), [React](https://react.dev/), or [Svelte](https://svelte.dev/) — passing props down from the server as if you were calling a function.

No REST API. No client-side routing config. No separate frontend build server for development. Just a request, a component name, and a bag of props.

Think of it this way: traditional Server-Side Rendering (SSR) is a printing press — it stamps out complete HTML pages for every request. Single-Page Applications (SPAs) are a fax machine — they send raw data and hope the other end knows what to do with it. Inertia is an email with a rich attachment: structured, fast, and fully readable on both ends.

For the Laravel community, Inertia has been transformative. Entire SaaS products are running on it. The productivity gains are measurable and real.


## And Then There's Go

Go is having a moment in backend web development — and for good reason. Its concurrency model is genuinely excellent. Its binary compilation makes deployment trivially simple. Its memory footprint is a fraction of a comparable JVM process. For high-throughput services, regulated systems, or anything where you need a single compiled artefact that just *runs*, Go is one of the best choices available today.

But Go has largely been excluded from the Inertia ecosystem.

The existing Go adapters are a patchwork. Some are tied to specific frameworks — Echo, Gorilla — which fragments the ecosystem. Others implement only a subset of the Inertia protocol, omitting critical features such as asset versioning, SSR support, or proper partial reloads. A few have gone unmaintained. None of them feels like a first-class Go library: idiomatic, composable, tested, and ready to trust in production.

This is the gap we set out to close.


## Introducing `oullin/inertia-go`

`oullin/inertia-go` is our Go implementation of the Inertia.js server-side adapter. It is built with the same philosophy we apply to everything we ship at Oullin: **no ceremony, no magic, no framework lock-in, and nothing you didn't ask for.**

```
go get github.com/oullin/inertia-go
```

### Framework-agnostic by design

The adapter works with Go's standard `net/http` interface. That means it integrates cleanly with Chi, Gorilla Mux, the Go 1.22+ stdlib router, or any other `http.Handler`-compatible router without adaptation layers or vendor-specific shims.

```go
r := chi.NewRouter()
r.Use(inertia.Middleware(manager))

r.Get("/dashboard", func(w http.ResponseWriter, r *http.Request) {
    err := manager.Render(w, r, "Dashboard/Index", inertia.Props{
        "user":  currentUser(r),
        "stats": fetchDashboardStats(r.Context()),
    })
    if err != nil {
        http.Error(w, err.Error(), http.StatusInternalServerError)
    }
})
```

No special context types. No embedded structs. No interface compliance dance. If it speaks `http.Handler`, it works.


### Full protocol compliance

The Inertia protocol is deceptively involved. A correct implementation must handle:

- **Initial page loads** — render the full HTML shell with the page object embedded
- **Subsequent navigations** — respond with a JSON page object only, no HTML wrapper
- **Asset versioning** — detect when frontend assets have changed mid-session and force a full reload
- **Partial reloads** — return only the requested subset of props when the client specifies `X-Inertia-Partial-Data`
- **Shared props** — inject global data (auth state, flash messages) into every response without repeating it in every handler
- **External redirects** — distinguish between Inertia-internal navigation and redirects to non-Inertia URLs

Most community adapters implement the first two and stop. We implement all of them. The Inertia protocol spec is the test suite; our implementation passes it.


### Idiomatic Go: context, not magic

Shared props — the data you want available on every page, like the authenticated user or flash messages — are propagated through Go's `context.Context`, the way Go intends:

```go
func AuthMiddleware(manager *inertia.Manager) func(http.Handler) http.Handler {
    return func(next http.Handler) http.Handler {
        return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
            user := getUserFromSession(r)
            ctx := manager.WithProp(r.Context(), "auth", map[string]any{
                "user": user,
            })
            next.ServeHTTP(w, r.WithContext(ctx))
        })
    }
}
```

No global state. No thread-local storage. No service locator patterns sneaking in from the [PHP](https://www.php.net/) world. The data flows down the request chain the way Go developers expect.


### Embed-ready for single-binary deployment

One of Go's killer features is `embed.FS` — the ability to bundle static assets directly into your compiled binary. We fully support it:

```go
//go:embed templates
var templateFS embed.FS

manager := inertia.NewWithFS("https://app.example.com", "templates/app.gohtml", version, templateFS)
```

Deploy a single binary. No separate asset pipeline to manage at runtime. No `views/` directory to remember to include in your Docker image. The whole application — backend logic, templates, and frontend build artefacts — ships as one file.


## Proof of Work

Words are easy. Shipping is hard.

[PR #4](https://github.com/oullin/inertia-go/pull/4) represents a meaningful step in proving this adapter works under real conditions — not toy examples, but the kind of integration that exercises the tricky parts of the protocol. It demonstrates the full request lifecycle, including the cases that trip up simpler implementations: the version mismatch redirect, the partial reload header negotiation, and the 303 response on POST-redirect-GET flows.

This is the adapter speaking fluent Inertia. Not a subset. Not "mostly works." The conversation is complete.


## Who Should Use This

If you are building a Go web application and you want the developer experience of a modern SPA — reactive components, client-side routing, seamless page transitions — without the operational overhead of a separate frontend application, `oullin/inertia-go` is the right tool.

Specifically, this library is for you if:

**You came from Laravel** and miss how Inertia felt. The server was in charge, the frontend was smart, and the API layer was never a third wheel you had to maintain. You can have that in Go now.

**You're building internal tools or SaaS products** where operational simplicity matters. One binary. One deployment. One team. The monolith isn't a compromise — it's a deliberate, correct choice for a wide range of products.

**You work in regulated environments** — banking, healthcare, fintech — where your deployment surface needs to be minimal and auditable. A Go binary with embedded templates is about as simple a deployment artefact as exists.

**You want to own your frontend** with Vue, React, or Svelte without surrendering control of your routing and data fetching to a JavaScript framework that will have three major breaking versions by the time you read this.


## The Catch

Inertia is a protocol, not a solution to every architectural problem. It works best when:

- Your frontend and backend are **co-located** — same team, same repository, same deployment
- You are not building a **public API** that third parties consume
- You can accept **JavaScript as a rendering layer**, not a state management system

If you need a fully decoupled API for mobile clients, or you have a frontend team that deploys independently on a different cadence, Inertia may not be the right fit. That's fine — it's not trying to be everything.

But for the vast majority of web products? The monolith with a smart frontend is the right starting point. Inertia knows this. And now, Go does too.


## Standing on the Shoulders of Giants

Before anything else, a direct acknowledgement: none of this exists without the work of [Jonathan Reinink](https://reinink.ca/) and the Inertia.js team, and without [Laravel](https://laravel.com/) — the framework that made this pattern mainstream.

This adapter wasn't born in a vacuum. We built it at [Oullin](https://oullin.io/) as part of a real client engagement — a production Go application that needed exactly this pattern. The adapter solved a genuine problem for us, and once it did, open-sourcing it was the obvious next step. The Go community has given us a tremendous amount of quality infrastructure to build on. This is our way of giving some of that back.

Laravel's official Inertia adapter (`inertiajs/inertia-laravel`) is the reference implementation. It is the gold standard for how a server-side adapter should behave, and it is the spec we hold ourselves to. If you haven't seen what this looks like at full feature parity, the [**Inertia.js Kitchen Sink**](https://demo-v3.inertiajs.com/login) is the official demo app — every feature of the protocol, exercised in one place. Log in with the pre-filled credentials and explore what the full Inertia experience looks like.

That app is what we're porting to Go. Feature by feature, behaviour by behaviour.


## Getting Started

```bash
go get github.com/oullin/inertia-go
```

```go
package main

import (
    "net/http"

    "github.com/go-chi/chi/v5"
    "github.com/oullin/inertia-go"
)

func main() {
    manager := inertia.New(
        "http://localhost:3000",
        "./resources/templates/app.gohtml",
        "", // asset version — connect to your Vite manifest
    )

    r := chi.NewRouter()
    r.Use(inertia.Middleware(manager))

    r.Get("/", func(w http.ResponseWriter, r *http.Request) {
        manager.Render(w, r, "Home/Index", inertia.Props{
            "message": "Hello from Go.",
        })
    })

    http.ListenAndServe(":3000", r)
}
```

Pair it with [Vite](https://vite.dev/) on the frontend, point your `app.gohtml` at the Vite dev server during development, and you have a full-stack Go + React (or Vue, or Svelte) application running with a single `go run`.


### Run the demo locally

Want to see the Kitchen Sink running on a Go backend right now? Clone the repo and run:

```bash
make demo
```

That's it. The demo spins up a Go server wired to the full Inertia Kitchen Sink frontend — the same app you can explore at [demo-v3.inertiajs.com](https://demo-v3.inertiajs.com/login). Same features, same flows, different backend. It's the fastest way to see the protocol in action and validate that the Go adapter is speaking fluent Inertia.


## Final Thought

The industry spent a decade building walls between backends and frontends. Inertia spent a few years quietly tearing them down. We spent time making sure Go developers don't have to keep watching from the other side of the fence.

Systems that hold are systems that are simple enough to understand end-to-end. `oullin/inertia-go` is our contribution to that principle.

Start the repo. File an issue. Send a PR. We're building this in the open.


[**github.com/oullin/inertia-go**](https://github.com/oullin/inertia-go)


*Oullin builds software infrastructure and consults on system architecture for teams that need things to work. [oullin.io](https://oullin.io)*


