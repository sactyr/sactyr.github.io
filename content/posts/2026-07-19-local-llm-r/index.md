---
title: "Running Local LLMs from R: Hardware-Aware Model Selection with Ollama"
author: ''
date: "2026-07-19"
slug: []
categories:
  - Deep Dives
tags: 
  - R
  - LLM
  - Ollama
  - self-hosting
description: "Setting up a local LLM pipeline in R with Ollama, comparing two hardware-fit CLI tools (llmfit vs llm-checker), and two real bugs that ate more time than the setup itself."
draft: true
---

I wanted a local LLM pipeline I could call from R - no cloud API keys, no
per-token billing, just a prompt in and a chunk of text out. This post walks
through setting that up with Ollama and the `ollamar` package, and - more
usefully - the handful of things that quietly broke along the way.

Repo: [github.com/sactyr/local-llm-r](https://github.com/sactyr/local-llm-r)

## Hardware

Nothing exotic: a GTX 1080 (8GB VRAM), a Ryzen 7 5700G, and 32GB of system
RAM. Firmly mid-range by 2026 standards, which turned out to matter more
than expected.

## The chicken-and-egg problem: which models even fit?

Ollama will happily let you `pull` a model that's too big for your card and
find out the hard way. Rather than guess, I wanted the pipeline itself to
detect hardware and pick sensible models - partly for convenience, partly
because it makes the demo portable: clone the repo on different hardware,
get different (correct) recommendations.

### First attempt: llmfit

[`llmfit`](https://github.com/AlexsJones/llmfit) is a Rust CLI that scores
models against your detected hardware across quality/speed/fit/context. It's
fast and has zero dependencies (a single Scoop/Homebrew install), but its
model database spans multiple formats - Hugging Face safetensors, MLX,
GGUF - not just Ollama. In practice, most of its top recommendations for my
hardware turned out to be **not runnable in Ollama at all**: one was an
MLX-only checkpoint (Apple Silicon exclusive), another was a bitsandbytes
4-bit format Ollama can't load. Of the top 10 recommendations, only 3
resolved to something pullable - and even then, only by falling back to
Ollama's `hf.co/{repo}:{quant}` direct-pull syntax for community GGUF
conversions, since llmfit's `ollama_name` field was `null` for most results.

### Second attempt: llm-checker

[`llm-checker`](https://github.com/signerless/llm-checker) (npm-installed,
Node.js) takes a narrower, more useful approach for this specific job: it
restricts results to the Ollama catalog directly via an `--ollama-only`
flag, and prints a ready `ollama pull <tag>` command for each recommendation.
No format-mismatch resolution needed.

It's not perfect either - worth building a small sanity filter rather than
trusting recommendations blindly. Two issues I hit:

- One "recommendation" pointed at `qwen3-embedding` under a `qwen3` display
  name. Embedding models produce vectors, not text - pulling this into a
  text-generation pipeline would silently fail. Filtered out anything
  matching `embed|rerank` in the model name.
- My hardware tier was reported as "MEDIUM LOW, max model size 6GB" by
  `hw-detect`, yet one of the top-3 recommendations was a 36B-parameter,
  23.9GB model. It still ran (see benchmark below) via heavy CPU/RAM
  offload, just much slower - but it directly contradicted the tool's own
  hardware ceiling.

Lesson: treat automated hardware-fit tools as a starting point, not
ground truth. A thin validation layer between "tool recommends X" and
"pull X" earns its keep.

## Pulling the models

With the filter in place, three models came through cleanly:

| model | disk size | pull time |
|---|---|---|
| `gemma3` | 3.3GB | 40.8s |
| `qwen3.5` | 6.6GB | 110.4s |
| `qwen3.6` | 23.9GB | 382s |

(`ollamar::pull()` doesn't show progress the way the CLI `ollama pull` does
— it just blocks silently until done. Fine for a script, mildly nerve-wracking
to sit and watch.)

## Two bugs that cost more time than the setup itself

**`ollamar::test_connection()` returns an `httr2_response` object by
default, not a boolean.** A naive `if (!isTRUE(test_connection()))` check
fails even on a successful connection, because `isTRUE()` only matches a
literal logical `TRUE`. Fix: pass `logical = TRUE` explicitly.

**Reasoning models silently return empty text via the `/api/generate`
endpoint.** `qwen3.5` and `qwen3.6` are Qwen3-family models with chain-of-thought
"thinking" mode on by default. Ollama's `/api/generate` endpoint has a
known bug ([ollama/ollama#14793](https://github.com/ollama/ollama/issues/14793))
where `think: false` is silently ignored - the model burns its entire token
budget on hidden reasoning tokens and the visible `response` field comes
back empty, no error thrown. `/api/chat` with `think: false` at the
top level (not nested under `options`) works correctly. Non-reasoning
models like `gemma3` are unaffected either way, so routing everything
through `/api/chat` with `think: false` set is the simplest fix that
works uniformly.

## Results

Same prompt ("write a 3-sentence summary of what a GTX 1080 is"), all three
models, ~500-character target:

| model | elapsed | chars out | chars/sec |
|---|---|---|---|
| `gemma3` | 7.1s | 443 | 62.4 |
| `qwen3.5` | 12.2s | 363 | 29.8 |
| `qwen3.6` | 31.4s | 454 | 14.5 |

`qwen3.6` - the hardware-mismatched 36B model - is roughly 4x slower than
`gemma3` per character, which tracks with it running partly off system RAM
instead of VRAM. Still usable, just clearly the wrong default choice for
this card.

## Wrapping up

What started as "install Ollama and pull a model" turned into a more
useful exercise than expected: automated hardware-fit tools are a genuine
time-saver, but they're not infallible, and it's worth building a thin
validation layer rather than piping their output straight into `pull()`.
The two real bugs - a non-boolean connection check and a silent-failure
reasoning-model endpoint - are the kind of thing that cost far more
debugging time than the actual setup, and neither shows up in the tools'
documentation.

If you're setting up something similar, the full repo is a reasonable
starting point:
[github.com/sactyr/local-llm-r](https://github.com/sactyr/local-llm-r).
Swap in your own hardware, run `llm-checker check --ollama-only`, and see
what it recommends for you.