---
title: "Stop Installing Python Just to Convert a File to Markdown"
excerpt: "If you work with LLMs, you know the drill. You have a <.docx>, a <.pdf>, or a spreadsheet, and you need it as Markdown — clean, structured text that models can actually work with."
slug: "2026-02-26-stop-installing-python-just-to-convert-files-to-markdown"
published_at: 2026-02-26
author: "gocanto"
categories: "infrastructure"
tags: ["infrastructure", "Docker", "LLMs", "make", "html", "csv"]
---

![file](https://github.com/user-attachments/assets/f953c79e-8618-42ee-8d14-c542c9f74dc5)


Microsoft's [markitdown](https://github.com/microsoft/markitdown) does this well. It converts office documents, PDFs, images, HTML, CSV, and more into Markdown while preserving headings, tables, and lists. 
The catch? It's a Python tool. That means you need Python installed, a virtual environment set up, and a handful of dependencies managed. 
Fine if you're already in that world. Not great if you just want to convert a file and move on.

[to-markdown](https://github.com/gocanto/to-markdown) wraps the whole thing in Docker so you never touch Python at all. You run a single `make` command and get your Markdown file back. That's it.

## The idea behind it

The project is built around a simple opinion: file conversion should be boring. You put a file in, you get a file out, and the tool stays out of your way.

There are three design choices that make this work.

**Deterministic paths.** Input files go in `storage/input`. Output files are in `storage/output`. There's no guessing where things end up. If you convert `report.docx`, you'll find `report.md` exactly where you expect it. 
This makes the tool easy to script and easy to reason about.

**Quiet output.** On success, the command prints one line — the path to the generated file. No Docker build logs, no pip install noise, no progress bars. 
If something goes wrong, it prints a clear error message along with the last 30 lines of the relevant log. You get a signal when you need it and silence when you don't.

**Zero local dependencies beyond Docker and Make.** No Python. No `pip install`. No virtual environments. If you have Docker running and `make` available (which you almost certainly do), you're ready to go.

## How it works under the hood

The project is structured around a `Makefile` and Docker Compose. Here's what happens when you run `make convert file=report.docx`:

1. The Makefile validates that you passed a filename and that the file actually exists in the input directory.
2. It checks whether the Docker image has already been built. If not, it builds silently and shows output only if the build fails.
3. It runs the `markitdown` container via `docker compose run`, mounting your `storage` directory as a volume at `/work`.
4. The container converts the file and writes the result to the output directory.
5. The Makefile confirms the output file exists and prints its path.

All logs are written to temporary files that are automatically cleaned up. The whole thing is wrapped in `set -e` with a `trap` for cleanup, so partial failures don't leave artefacts behind.

Here's the relevant part of the Makefile that handles the conversion:

```makefile
input_path="$(FROM_DIR)/$(file)"; \
output_file="$$(basename "$(file)")"; \
output_file="$${output_file%.*}.$(OUTPUT_EXT)"; \
output_path="$(TO_DIR)/$${output_file}"; \
mkdir -p "$(TO_DIR)"; \
```

The filename manipulation strips the original extension and replaces it with whatever `OUTPUT_EXT` is set to (defaults to `md`). Simple string operations — no magic.

## Getting started

Clone the repo, copy the example env file, and you're set:

```sh
git clone https://github.com/gocanto/to-markdown.git
cd to-markdown
cp .env.example .env
```

Drop a file into `storage/input` and convert it:

```sh
cp ~/Documents/report.docx storage/input/
make convert file=report.docx
```

Output:

```
storage/output/report.md
```

If you ever need a clean rebuild (say, after bumping the `markitdown` version), run:

```sh
make fresh
```

That triggers a `docker compose build --no-cache` so everything gets rebuilt from scratch.

## Configuration

The `.env` file gives you four knobs to turn:

- `FROM_DIR` — where to read input files from (default: `storage/input`)
- `TO_DIR` — where to write output files to (default: `storage/output`)
- `OUTPUT_EXT` — the file extension for converted files (default: `md`)
- `MARKITDOWN_VERSION` — the pinned version of the `markitdown` package installed in the image

Pinning the version matters. It means your conversions are reproducible. The same file, the same version, the same output — regardless of when or where you run it.

## Why this matters for non-Python developers

If your stack is Go, Node, PHP, or TypeScript, you probably don't have a Python environment set up on your machine — and you shouldn't have to just for file conversion. 
Docker gives you that isolation for free. The Python runtime, the dependencies, and the tool itself all live inside the container. Your host stays clean.

This also makes it easy to plug into existing workflows. Need to convert uploaded documents as part of a queue job? Shell out to `make convert` from a queued command. 
Processing files in a Node.js pipeline? Spawn the command as a child process. Building a Go service that ingests documents? Call it from your handler. 
The interface is always the same: a file goes in, a Markdown file comes out.

## What we actually support

Since this is a wrapper, it supports everything the upstream `markitdown` library does. That includes PDF, Word (`.docx`), Excel (`.xlsx`), PowerPoint (`.pptx`), HTML, CSV, JSON, XML, images (with optional LLM-powered descriptions), 
audio files, and even ZIP archives where it iterates over the contents. The Markdown output preserves document structure — headings, lists, tables, and links all come through.

## Final thoughts

[to-markdown](https://github.com/gocanto/to-markdown) doesn't try to do anything clever. It takes a good tool, puts it in a box, and gives you a one-liner to use it. The value is in what it removes: setup friction, 
runtime dependencies, and noisy output. If you convert files to Markdown more than once, it's worth the 30 seconds to set up.

Check out the project on [GitHub](https://github.com/gocanto/to-markdown), and if it's useful to you, give it a star.

