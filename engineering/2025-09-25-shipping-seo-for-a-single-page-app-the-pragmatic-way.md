---
title: "Shipping SEO for a single-page app, the pragmatic way"
excerpt: "Adding an SEO generation engine to our Go API. It renders crawler-friendly static pages using an embedded HTML template, JSON-LD, Open Graph and Twitter meta tags, and a web manifest."
slug: "2025-09-25-shipping-seo-for-a-single-page-app-the-pragmatic-way"
published_at: 2025-09-25
author: "gocanto"
categories: "engineering"
tags: ["engineering", "seo", "web", "JSON-LD", "Open Graph", "cli", "crawlers"]
---

![seo](https://github.com/user-attachments/assets/12848e3f-baef-4c93-a050-61574a756f13)


## Shipping SEO for a single-page app, the pragmatic way

> TL;DR
> I added an **SEO generation engine** to our Go API. It renders crawler-friendly static pages using an embedded HTML template, JSON-LD, Open Graph, Twitter meta-tags, and a web manifest. 
> It’s driven from the API's CLI interface, with fresh env vars and a small router refactor to reuse fixtures for content.

If you have ever shared a link to a single page app and seen a blank or generic preview, you have met the gap between great user experience and what crawlers can actually read. 
SPAs render content in the browser. Search engines and social platforms often do not. That mismatch hurts discoverability and the way our links look when people share them.

PR [#110](https://github.com/oullin/api/pull/110) closes that gap pragmatically. I did not add server side rendering or change how the app runs in production. 
I built a small SEO generator that runs from the CLI, pulls the same content our API serves, and produces clean, static HTML with proper meta-tags, JSON-LD, and a web manifest. 
Humans still get the fast SPA they know. Crawlers get a lightweight page they can understand at a glance.

This post explains the problem, the design choices, and what shipped. I keep the language plain and define terms as I go, so non-technical readers can follow along. 
Engineers will find exact entry points, data flows, and trade-offs. By the end, you will see how to generate the pages locally, what to expect in the output, and why 
this approach keeps operations simple while improving search and sharing.


## What's the problem solved

Our front end is an SPA. SPAs are fast for users, but search and social crawlers often struggle with JavaScript-only pages, 
which hurts discoverability and link previews. Rather than bolt on SSR, I took a **simpler, lower-risk path**: **generate static 
SEO pages offline** and serve those to crawlers, while keeping our SPA unchanged for humans. This gives crawlers clean HTML and keeps our runtime simple.

## What was actually shipped

### 1) A CLI-driven SEO engine

-   New **`metal/cli/seo`** package with types, a **Generator**, **JSON-LD**, **manifest builder**, and an embedded HTML **stub template**. [GitHub](https://github.com/oullin/api/pull/110)
-   The CLI has a “Generate SEO” flow that spins up a tiny internal router with **fixtures** to fetch Profile, Talks, Projects, and **Categories** data, then renders HTML sections + meta and writes static pages under `ENV_SPA_DIR`. [GitHub](https://github.com/oullin/web/pull/64/files)


### 2) Environment and configuration

-   New env vars: **`ENV_APP_URL`** and **`ENV_SPA_DIR`** for absolute URLs and SPA asset context. The `.env.example` also standardises logs date format to `2006-01-02`.
-   Sample env files were cleaned up by **deleting** `.env.gh.example` and `.env.prod.example` to avoid drift.
-   **Compose** pulls configuration from `.env` (via `env_file`) and Makefile quality-of-life targets were added for CLI runs and fresh builds.


### 3) Router/fixtures and repository hygiene

-   Router gained **fixture-driven** static wiring so the generator can reuse the same handlers without booting the whole server. Tests were updated accordingly.
-   **Categories** repository was introduced to support SEO sections.
-   Generated artefacts are **ignored**: `ENV_SPA_DIR` (with `.gitkeep` kept).

## How it works (plain English)

1.  I run a CLI command that launches a tiny, internal copy of our API’s routes using **fixtures**.
2.  That in-memory API returns Profile, Talks, Projects, and Categories data as JSON.
3.  The SEO generator fills an **HTML template** with regular meta-tags, **Open Graph/Twitter** tags, **JSON-LD**, and links a **web manifest**.
4.  It writes static HTML files to `ENV_SPA_DIR` for production. No impact on the SPA runtime.

Engineers get a predictable, testable rendering flow; non-technical folks get **better Google results and richer social previews**.

## Why this approach

-   **Low operational risk:** No SSR to maintain or scale.
-   **Fast for crawlers:** Pre-rendered HTML is small and cacheable.
-   **Reuse:** Same handler contracts via fixtures, so content is consistent.
-   **Fits our toolchain:** Env-driven, CLI-first, Docker-friendly.

## Developer notes

-   **Entry points:** The SEO generator lives in `metal/cli/seo` with `stub.html`, `jsonld.go`, `manifest.go`, and `generator.go`.
-   **Config:** Add `ENV_APP_URL` and `ENV_SPA_DIR` to `.env`. Log date format is `2006-01-02`.
-   **Compose/Make:** Compose uses `.env`; Makefile contains targets to run the CLI in Docker and fresh-build.
-   **Outputs:** Git ignores static files except for `.gitkeep`.
-   **Scope:** This PR also touches keep-alive and env plumbing as part of the router/env tidy-up to support the generator path, plus a small repository addition for categories.

## What’s next

-   **CI automation:** Regenerate SEO pages on content or template change.
-   **Page coverage:** Extend beyond the homepage to project and post pages.
-   **Validation and link-checks:** Add pre-commit or CI checks to catch broken schema or URLs early.


## Glossary (for non-engineers)

-   **SPA:** Single-page app. A website that mostly renders in the browser using JavaScript.
-   **SEO:** Search engine optimisation — increasing how easily pages are found and understood by search engines.
-   **Open Graph/Twitter tags:** Special HTML meta tags that make links look good on Facebook, LinkedIn, X, etc.
-   **JSON-LD:** A small JSON block embedded in the page that describes the content to search engines.
-   **Web manifest:** A JSON file that tells devices about the site’s name, icons, and theme.
-   **Fixtures:** Predictable sample responses that let me run the generator without the full backend.


## PR facts at a glance

-   **Merged:** 25 Sep 2025. **Commits:** 27. **Files changed:** 45. **Net lines:** `+1,318 −221`.
-   **New envs:** `ENV_APP_URL`, `ENV_SPA_DIR`.
-   **Ignored outputs:** `storage/seo/*.*` with `.gitkeep`.

Shipping great previews and search results shouldn’t require a new runtime. With PR #110 I kept the SPA exactly as users 
know it and gave crawlers clean, fast HTML that tells the right story. If you are technical, the entry points live in 
`metal/cli/seo` and you can run the generator from the CLI to see the output under `storage/seo/pages`. If you are not, 
the headline is simple: our links now look better, read better, and travel further.

I plan to extend coverage beyond the homepage and wire regeneration into CI so pages stay fresh as content evolves. 
If you spot an edge case or have an idea that would clarify the output, open an issue or drop me a note.



