---
title: "Taming Complex State in Go: A Deep Dive into `oullin/workflow`"
excerpt: "Every non-trivial backend application manages state. An order is `placed`, then `paid`, then `shipped`, then `delivered`. A user account is `pending`, then `active`, then maybe `suspended`."
slug: "2026-03-13-taming-complex-state-in-go"
published_at: 2026-03-13
author: "gocanto"
categories: "go"
tags: ["engineering", "go", "workflow"]
---



# Taming Complex State in Go: A Deep Dive into `oullin/workflow`

## Where This Came From

This library didn't start as a library. It started as a solution.

[Oullin](https://oullin.io) was working with a client on a large-scale platform with complex domain objects that needed to move through elaborate, branching lifecycles — 
subscription billing flows, multi-step approval chains, parallel processing pipelines. The kind of real-world complexity that breaks naive `status` columns and scattered `if` blocks.

Rather than reaching for an off-the-shelf tool that didn't quite fit Go's idioms, the team built something purpose-made. Over time, as the engine matured and proved itself in 
production, it became clear that the core was too valuable to stay buried inside one client's codebase. So it was extracted, hardened, and open-sourced as `oullin/workflow` — a 
standalone, industrial-grade state management library for Go that any team can use.

---

## What Is It and Why Do You Need It?

Every non-trivial backend application manages state. An order is `placed`, then `paid`, then `shipped`, then `delivered`. A user account is `pending`, then `active`, 
then maybe `suspended`. On the surface, this looks simple — a database column and a few service methods.

It stops being simple and fast.

What happens when two things need to happen in parallel before the next step is allowed? What happens when certain transitions should only fire if a business rule is satisfied — 
say, a payment has cleared, or an identity check has passed? What happens when a regulatory requirement means you need a complete audit trail of every state change, including who 
triggered it and when? What happens when your workflow topology changes and you need to communicate that change to non-engineers?

These are the questions that turn a tidy `status` column into a sprawling mess of conditionals, side effects, and missed edge cases. `oullin/workflow` is the answer to all of them — 
a structured engine that brings mathematical rigour and production-grade tooling to state management in Go, without adding complexity to your domain models.


## Taming Complex State in Go

State management is one of those problems every backend developer thinks they've solved — until a product requirement shows up that breaks all their assumptions. A simple `status` column 
works fine until you need parallel approvals, conditional branching, or audit trails baked right into the transition logic. At that point, ad-hoc `if/switch` chains quietly become a 
maintenance nightmare.

`oullin/workflow` is a Go library that tackles this problem head-on. It's a high-performance, industrial-grade state management engine built on [Petri Net](https://en.wikipedia.org/wiki/Petri_net) 
theory — the same mathematical model that underpins workflow systems in the industry, enterprise BPM tools, and distributed systems research.


## The Core Idea: Two Modes, One Engine

The library ships two constructors for two distinct mental models.

**`workflow.NewStateMachine()`** enforces that your domain object lives in exactly one state at a time. This is the classic FSM (Finite-State Machine) you already know: `draft → pending → active`. Simple, linear, familiar.

**`workflow.New()`** gives you a full Petri Net engine where an object can occupy *multiple places simultaneously*. Think of a document that must go through legal review *and* finance approval in parallel 
before it can be published. Both reviews occur concurrently; the transition to `published` fires only when both are complete.

Choosing the right mode isn't just philosophical — it shapes your entire domain model.


## Defining a Workflow

The fluent builder API reads cleanly and keeps workflow definition close to your domain code:

```go
definition, _ := workflow.NewDefinitionBuilder().
    AddPlace("draft").
    AddPlace("payment_pending").
    AddPlace("active").
    SetInitialPlaces("draft").
    AddTransition("collect_payment", []string{"draft"}, []string{"payment_pending"}).
    AddTransition("activate", []string{"payment_pending"}, []string{"active"}).
    Build()
```

If you prefer keeping your workflow topology out of Go code entirely, the library supports declarative YAML:

```yaml
workflows:
  subscription_paid:
    type: state_machine
    initial_marking: draft
    places:
      - name: draft
      - name: payment_pending
      - name: active
    transitions:
      collect_payment:
        from: draft
        to: payment_pending
      activate:
        from: payment_pending
        to: active
        guard: payment_captured && identity_verified
```

That `guard` field in YAML is a nice touch — it lets non-Go folks (product managers, architects) read and reason about your state machine without needing to dig into code.


## Decoupled State via `MarkingStore`

One of the library's better design decisions is how it manages state on your domain objects. Rather than requiring your structs to embed a base type or implement a heavy interface, you wire up simple getter/setter functions:

```go
markingStore := &store.SingleState[*Subscription]{
    Getter: func(s *Subscription) string { return s.State },
    Setter: func(s *Subscription, state string) { s.State = state },
}
```

Your `Subscription` struct stays clean. No library imports bleeding into your domain layer. This pattern will feel familiar if you've worked with other Workflow components — the design is clearly inspired by that pattern and brought to idiomatic Go.

Applying a transition is then as straightforward as:

```go
sub := &Subscription{State: "draft"}
engine, _ := workflow.New("subscription_flow", definition, markingStore, nil)

_, err := engine.Apply(sub, "collect_payment", nil)
fmt.Println(sub.State) // payment_pending
```

---

## The Event Lifecycle: Seven Hooks, Zero Surprises

Where `oullin/workflow` really separates itself from a naive FSM (Finite-State Machine) implementation is the granular event system. Every transition fires through a strictly ordered sequence of seven events:

| Order | Event         | Purpose                                        |
|-------|---------------|------------------------------------------------|
| 1     | **Guard**     | Block the transition — validation, authorization |
| 2     | **Leave**     | Teardown logic for the old state               |
| 3     | **Transition**| The state change itself                        |
| 4     | **Enter**     | Setup logic for the new state                  |
| 5     | **Entered**   | Post-commit hooks — state is persisted         |
| 6     | **Completed** | The whole process finished                     |
| 7     | **Announce**  | What workflows to trigger next                 |

Guards are particularly powerful. They let you inject business logic that blocks a transition without coupling it to the engine itself:

```go
dispatcher.OnGuard(
    workflow.EventNameGuardNamed("blog", "publish"),
    func(event *events.GuardEvent[*Article]) {
        if event.Subject().Title == "" {
            event.SetBlocked(true, "Title is required to publish")
        }
    },
)
```

This is the kind of extensibility that matters in production. Your authorisation rules, feature flags, and external API checks — they all plug in at the `Guard` phase without touching transition definitions.


## Built-In Audit Trails

Compliance and debugging both demand that you know *who* changed *what* and *when*. The library ships an `audit.Trail` that attaches to your dispatcher and records every transition automatically:

```go
trail := &audit.Trail[*Subscription]{}
trail.Attach("subscription_flow", dispatcher)
```

No custom logging code to write. No risk of forgetting to record a particular transition. It's just there.


## Visualisation via Graphviz DOT

Complex workflows become impossible to reason about from code alone. `oullin/workflow` includes a `GraphvizDumper` that exports your workflow definition to DOT format, which you can pipe directly into the Graphviz CLI:

```go
dumper := &workflow.GraphvizDumper{}
dotOutput := dumper.Dump(definition)
// echo "$dotOutput" | dot -Tpng -o workflow.png
```

This is genuinely useful for documentation reviews, onboarding new engineers, and catching logical dead-ends before they hit production. The library also runs automated reachability validation at startup to detect potential deadlocks — a significant safety net that most homegrown FSMs skip entirely.


## Production Characteristics Worth Noting

- **Thread-safe registry** — safe to use across goroutines without external locking
- **O(1) state lookups** — no linear scans through place lists as your workflow grows
- **`slog`-compatible structured logging** — plays nicely with modern Go observability stacks
- **100% test coverage on critical paths** — verifiable via `make test-coverage`

The Go Report Card badge on the repo is A+ rated, and the CI pipeline runs on every commit.


## When Should You Reach for This?

`oullin/workflow` is a good fit when:

- Your domain objects move through meaningful states, and you've outgrown a `status` column plus scattered `if` checks
- You need parallel state occupation (parallel approvals, multi-step concurrent pipelines)
- You want audit trails and transition hooks without building them yourself
- Your workflow topology might change — YAML config means product changes don't always require code deploys

It's probably overkill for dead-simple two-state toggles or cases where a single `enum` column and a couple of service methods get the job done.

---

## Getting Started

```bash
go get github.com/oullin/workflow
```

The `examples/` directory in the repo walks through a complete working implementation with guards, audit trails, and the registry — a solid jumping-off point for integrating it into a real service.

For teams building order pipelines, subscription billing flows, document approval chains, or anything with meaningful lifecycle state, this library is worth a serious look. It brings Petri Net rigor to Go without the ceremony, and the API stays out of your domain model's way.

---

*Source: [github.com/oullin/workflow](https://github.com/oullin/workflow) · MIT License*
