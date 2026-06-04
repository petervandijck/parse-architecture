# parse-architecture

Architecture and user-facing docs for **Parse for Artisans** (parseforartisans.com) — a
document-to-markdown parsing service for Laravel / PHP developers. Think LlamaParse or
Reducto, but Laravel-native: install a package, pay, get an API key, and call
`->parse()`.

This repo is **design + docs only**. No application code lives here yet — it's where we
settle the architecture and write the developer-facing documentation before building.

## The three pieces

```
┌─────────────────────┐     ┌──────────────────────────┐     ┌─────────────────────┐
│  Client SDK         │     │  SaaS app                │     │  parse-function     │
│  (Composer package) │ ──▶ │  parseforartisans.com    │ ──▶ │  (Modal workers)    │
│  Parse::file()      │ API │  auth, billing, storage, │ HTTP│  PDF/docx/xlsx/...   │
│  ->parse()          │ ◀── │  orchestration, webhooks │ ◀── │  → markdown         │
└─────────────────────┘     └──────────────────────────┘   webhook└─────────────────┘
```

1. **Client SDK** — a Laravel/Composer package developers install. Exposes the simple
   `->parse()` surface (result via a `ParseCompleted` event). Talks only to our SaaS API with
   their API key. Never touches Modal or S3 directly.
2. **SaaS app** (`parseforartisans.com`) — the Laravel application we sell. Handles signup,
   billing, API keys, file storage, presigned URLs, calling Modal, receiving Modal's
   webhook, and serving results back to the SDK. **Not in this repo.**
3. **parse-function** (`../parse-function`) — existing Modal serverless backend. One
   endpoint per file type; fully async (returns `202`, PUTs markdown to a presigned URL,
   POSTs a webhook on completion). See its `CLAUDE.md` / `openapi.yaml`. **Do not** expose
   Modal directly to customers — the SaaS app is always the gateway.

## Design goals

- **Dead-simple happy path.** `Parse::file($path)->parse()` submits; a `ParseCompleted` event
  hands you the result. No bucket or webhook setup needed to start (managed dev bucket).
- **Laravel-native.** Built for **Laravel 12 and 13**; composes cleanly with the Laravel AI SDK
  (light touch for now).
- **One mode: always async.** Parsing takes seconds to minutes, so it never blocks the
  request. You submit, then act on a `ParseCompleted` event; `->status()` lets a UI show
  progress in the meantime. There is no blocking string-return call — one honest contract.

## Client API surface

The full developer-facing API is in **`docs.md`** — don't duplicate it here. In one line:
`Parse::file($path)->parse()` (also `Parse::disk(...)->file(...)`, `Parse::files([...])`,
`Parse::url(...)`) returns a `ParseRequest` handle; a `ParseCompleted`/`ParseFailed` event
delivers the result; `->status()` shows progress. Options (`->ocr()`, `->pages()`,
`->frontmatter()`, `->to()`, `->withMeta()`) chain before `->parse()`, a fast inline call (no
queue to submit).

## Open decisions

- **Pricing unit.** Per page? Per file? Per MB? (Affects what the SaaS meters.)

### Decided

- **One mode: always async**, event-delivered. No sync/blocking string call. `->parse()`
  returns a handle; a `ParseCompleted`/`ParseFailed` event carries the result. `->status()`
  reports progress for a UI — it **reads the local `parse_requests` row** (kept current by the
  webhook in prod, the poll job locally), never the API.
- **Storage: BYO bucket is the default** (presigned GET+PUT on the dev's disk; SaaS never
  touches bytes). A **SaaS-hosted managed dev bucket** (quota-limited, ~1-day retention) is the
  zero-config fallback when no disk is set — so local dev needs no bucket.
- **Delivery: `webhook` by default, `poll` only on local.** `parse.delivery=auto` resolves via
  `APP_ENV` — `poll` when `local`, `webhook` everywhere else (staging/prod). Both fire the same
  event; they never run together. Poll rides the customer's queue (`composer run dev`); a
  capped TTL guarantees the event eventually arrives. The same poll job advances `->status()`
  locally, so a worker must be running locally for status to move or events to fire.
- **File-type detection/routing lives in the SaaS** (by extension + MIME), so the SDK stays
  dumb and the dev just calls `->parse()` regardless of format. *(Leaning final.)*

## Backend reference (parse-function)

- Async flow: POST `{file_url, webhook_url, upload_url, webhook_secret}` → `202` → worker
  PUTs markdown to `upload_url`, POSTs result to `webhook_url`.
- Auth to Modal: `Authorization: Bearer <PARSE_API_SECRET>` (server-to-server only).
- Supported today: PDF (auto-OCR), docx, pptx, xlsx, eml, msg, plus media/image
  optimization. PDFs can also return a truncated copy for LLM ingestion.
- Built for bursts: `max_containers=100` per worker.

## Files in this repo

- `CLAUDE.md` — this file; architecture overview and decisions.
- `architecture.md` — systems, boundaries/auth, and the parse flow (storage + delivery) in detail.
- `docs.md` — user-facing developer documentation (Laravel-style).
