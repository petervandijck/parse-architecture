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
3. **Parse backend** — Modal serverless workers. The **new backend is `../modal`** and its
   initial version is **finished: built, deployed, and smoke-tested (June 12, 2026)** — all
   six V1 routes (pdf, docx, pptx, spreadsheet, email, legacy-office) are live as Modal app
   `parseforartisans-backend` (`petervandijck` workspace), with one small file per supported
   format (pdf/docx/pptx/xlsx/csv/doc/ppt/xls/eml/msg) verified end to end against the
   deployed endpoints. Detailed quality evals come later (`../evaluation`); the pre-launch
   **1 GB validation run** that finalizes worker sizing is still pending.
   **`../modal/CLAUDE.md` is the final word on the backend** — if this repo and that file
   drift, that file wins and this repo gets updated to follow. One endpoint per file type;
   fully async (returns `202`, PUTs markdown to a presigned URL, POSTs a webhook on
   completion). The **old `~/Herd/parse-function`** is historical reference only (shared
   with another project — don't modify). **Do not** expose Modal directly to customers —
   the SaaS app is always the gateway.

## Design goals

- **Dead-simple happy path.** `Parse::file($path)->parse()` submits; a `ParseCompleted` event
  hands you the result. No bucket or webhook setup needed to start (managed dev bucket).
- **Laravel-native.** Built for **Laravel 12 and 13**; composes cleanly with the Laravel AI SDK
  (light touch for now).
- **One mode: always async.** Parsing takes seconds to minutes, so it never blocks the
  request. You submit, then act on a `ParseCompleted` event; `->status()` lets a UI show
  progress in the meantime. There is no blocking string-return call — one honest contract.

## Client API surface

The full developer-facing API is in **`docs/`** (start at `docs/README.md`); don't duplicate it here. In one line:
`Parse::file($path)->parse()` (also `Parse::disk(...)->file(...)`, `Parse::files([...])`,
`Parse::url(...)`) returns a `ParseRequest` handle; a `ParseCompleted`/`ParseFailed` event
delivers the result; `->status()` shows progress. `->for($model)` ties the request to a
customer model (polymorphic, SDK-local) so the event hands that model back as
`$request->parsable`, the primary correlation path; `->withMeta()` carries non-model context.
Options (`->ocr()`, `->pages()`, `->frontmatter()`, `->to()`, `->withMeta()`, `->for()`) chain
before `->parse()`, a fast inline call (no queue to submit).

## Open decisions

- **Page-equivalents for non-paginated formats.** 1 credit = 1 page is locked (see Decided),
  but what counts as a "page" for an xlsx sheet, an email, or an image is deferred to
  Phase 1+ — as is premium credit cost (OCR / large / high-accuracy above 1:1). Business
  detail lives in the business repo's `pricing.md`; the architectural consequence is that
  the SaaS metering code needs a per-format page-equivalence rule.

### Decided

- **Pricing/metering unit: per page.** 1 credit = 1 page, $3 per 1,000 credits, 10,000 free
  credits/month, one tier (locked June 2026 — see the business repo's `pricing.md`). The
  SaaS meters the `page_count` Modal reports in its completion webhook — billing finalizes
  on completion, not at submit.
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
- **Legacy Office via LibreOffice — built & verified in the new backend.** `../modal` ships
  a LibreOffice-equipped worker for the legacy binary formats (`.doc`, `.ppt`, `.xls`),
  used as a **conversion shim**: convert legacy → modern (`.doc`→`.docx` etc.), then run
  the same parsers as the modern path, so legacy and modern files produce identical
  markdown. It's a separate worker image (LibreOffice is heavy) — modern paths stay light;
  SaaS extension routing sends legacy extensions to it unchanged. A LibreOffice
  "high-fidelity Office" tier for *complex modern* docs stays a roadmap option, not v1.
- **Large files, v1: one generously-sized worker per type — no tiers, no chunking.**
  Internal serial page-batching bounds peak memory regardless of document size, so a single
  worker (PDF: ~8–16 GB, ~2 h timeout, big disk) covers the 1 GB docs claim single-pass —
  slow on the extreme tail, fine under the async contract — and **must be tested end to end
  before launch** (the test also sets the resource numbers). Tiers (HEAD → size threshold →
  spawn, Modal-internal) and chunked parsing (split/fan-out/stitch) are both deferred cost/
  speed optimizations — build when usage shows the need; tiers first, chunking second.
- **File-type routing lives in the SaaS, by extension only.** Known extension →
  `trigger-<type>`; unknown → sync reject at submit. No byte sniffing anywhere — neither the
  SDK nor the SaaS ever has the bytes (BYO bucket), so routing runs on the filename. Modal
  verifies by construction: a parser failing on mismatched bytes returns a typed
  `ParseFailed` ("content doesn't match extension"). `Parse::url()` requires a known
  extension in the URL for v1 (a `->type('pdf')` override covers the rest). The SaaS↔Modal
  wiring is invisible to customers, so this stays cheap to revisit later.

## Backend reference (`../modal`, live)

- Async flow: POST `{file_url, upload_url, webhook_url, webhook_secret, options}` → `202` →
  worker PUTs markdown to `upload_url`, POSTs result to `webhook_url` (always — failures
  arrive as typed errors: `content_mismatch`/`corrupt`/`too_large`/`timeout`/`parse_error`).
- Auth to Modal: `Authorization: Bearer <PARSE_API_SECRET>` (server-to-server only).
- Live routes: `trigger-{pdf,docx,pptx,spreadsheet,email,legacy-office}` covering PDF
  (auto-OCR), docx, pptx, xlsx/csv, eml/msg, and doc/ppt/xls. No media/image routes in v1
  (the old backend's media paths belong to the other product).
- Built for bursts: `max_containers=100` per worker.
- Full contract details (options per route, `page_count` semantics, webhook payload):
  `../modal/CLAUDE.md`.

## Files in this repo

- `CLAUDE.md` — this file; architecture overview and decisions.
- `architecture.md` — systems, boundaries/auth, and the parse flow (storage + delivery) in detail.
- `docs/` — user-facing developer documentation (Laravel-style), split into pages; nav index in `docs/README.md`.
