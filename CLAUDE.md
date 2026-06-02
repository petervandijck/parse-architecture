# parse-architecture

Architecture and user-facing docs for **Parse for Artisans** (parseforartisans.com) — a
document-to-markdown parsing service for Laravel / PHP developers. Think LlamaParse or
Reducto, but Laravel-native: install a package, pay, get an API key, and call
`->markdown()`.

This repo is **design + docs only**. No application code lives here yet — it's where we
settle the architecture and write the developer-facing documentation before building.

## The three pieces

```
┌─────────────────────┐     ┌──────────────────────────┐     ┌─────────────────────┐
│  Client SDK         │     │  SaaS app                │     │  parse-function     │
│  (Composer package) │ ──▶ │  parseforartisans.com    │ ──▶ │  (Modal workers)    │
│  Parse::file()      │ API │  auth, billing, storage, │ HTTP│  PDF/docx/xlsx/...   │
│  ->markdown()       │ ◀── │  orchestration, webhooks │ ◀── │  → markdown         │
└─────────────────────┘     └──────────────────────────┘   webhook└─────────────────┘
```

1. **Client SDK** — a Laravel/Composer package developers install. Exposes the simple
   `->markdown()` surface. Talks only to our SaaS API with their API key. Never touches
   Modal or S3 directly.
2. **SaaS app** (`parseforartisans.com`) — the Laravel application we sell. Handles signup,
   billing, API keys, file storage, presigned URLs, calling Modal, receiving Modal's
   webhook, and serving results back to the SDK. **Not in this repo.**
3. **parse-function** (`../parse-function`) — existing Modal serverless backend. One
   endpoint per file type; fully async (returns `202`, PUTs markdown to a presigned URL,
   POSTs a webhook on completion). See its `CLAUDE.md` / `openapi.yaml`. **Do not** expose
   Modal directly to customers — the SaaS app is always the gateway.

## Design goals

- **Dead-simple happy path.** `$request->file('doc')->markdown()` returns a string.
- **Laravel-native.** First-class with Laravel 13 native file uploads and `UploadedFile`.
  Composes cleanly with the Laravel AI SDK (light touch for now).
- **Two equal DX modes:**
  - **Sync / blocking** — `->markdown()` blocks, SDK polls our API, returns the string.
    For the simple, interactive case.
  - **Async / queued** — dispatch a job, get the result via a Laravel event/webhook.
    For bulk imports (100s–1000s of files; the Modal backend is built for these bursts).

## Client API surface (target)

```php
// Facade — core entry point. Accepts a path or an UploadedFile.
$markdown = Parse::file($path)->markdown();

// UploadedFile macro — Laravel 13 native uploads.
$markdown = $request->file('document')->markdown();

// Async / queued (bulk).
Parse::file($path)->queue();   // dispatches; result via event/webhook
```

Both modes share one fluent builder; `->markdown()` resolves it synchronously,
`->queue()` dispatches it. (Exact builder API is still being shaped in docs.md.)

## Open decisions

- **Storage model.** SaaS-managed S3 (dev never touches a bucket) vs. bring-your-own
  bucket (SDK generates presigned URLs from the dev's own disk). Leaning: default
  SaaS-managed, allow BYO for power users. **To be discussed.**
- **Pricing unit.** Per page? Per file? Per MB? (Affects what the SaaS meters.)
- **Sync-wait mechanism.** How the SaaS holds the request open until Modal's webhook lands
  (hold-open vs. internal short poll) and the timeout policy.

### Decided

- **Async delivery is webhook-based, not polling.** The SaaS POSTs a signed result to the
  customer's webhook URL.
- **File-type detection/routing lives in the SaaS** (by extension + MIME), so the SDK stays
  dumb and the dev just calls `->markdown()` regardless of format. *(Leaning final.)*

## Backend reference (parse-function)

- Async flow: POST `{file_url, webhook_url, upload_url, webhook_secret}` → `202` → worker
  PUTs markdown to `upload_url`, POSTs result to `webhook_url`.
- Auth to Modal: `Authorization: Bearer <PARSE_API_SECRET>` (server-to-server only).
- Supported today: PDF (auto-OCR), docx, pptx, xlsx, eml, msg, plus media/image
  optimization. PDFs can also return a truncated copy for LLM ingestion.
- Built for bursts: `max_containers=100` per worker.

## Files in this repo

- `CLAUDE.md` — this file; architecture overview and decisions.
- `architecture.md` — systems, boundaries/auth, and the sync + async flows in detail.
- `docs.md` — user-facing developer documentation (Laravel-style). Starts empty.
