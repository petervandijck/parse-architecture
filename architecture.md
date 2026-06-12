# Architecture

Systems and flows for **Parse for Artisans** (parseforartisans.com). This is the
technical companion to `docs.md` (user-facing). See `CLAUDE.md` for the high-level
overview and open decisions.

> **Parse backend:** `../modal/CLAUDE.md` (the V1 build spec) is the **final word** — build
> work happens there. The backend sections below follow it; if they drift, update this file
> to match, not the other way around.

## Systems

| # | System | What it is | Owns |
|---|--------|-----------|------|
| 1 | **Client SDK** | Composer package the customer installs (`composer require ...`) | The `Parse::file()->parse()` / `->status()` / `->markdown()` surface + `ParseCompleted`/`ParseFailed` events. Talks only to the SaaS API over HTTPS with the customer's API key. Owns a `parse_requests` table (correlation + status), a signed webhook route (webhook delivery), and a self-releasing poll job (poll delivery). |
| 2 | **SaaS app** | The Laravel application we run at parseforartisans.com | Auth, API keys, billing/metering, presigned URLs, file-type detection & routing, calling Modal, receiving Modal's webhook, the status endpoint, customer webhooks. Hosts the managed dev bucket. |
| 3 | **Parse backend** | Modal serverless workers — new backend in `../modal` (greenfield, to build); old `~/Herd/parse-function` is the working contract reference | The actual parse. One endpoint per file type. Fully async. Never customer-facing. |
| 4 | **Queue** | Laravel queue (Redis/SQS) inside the SaaS app | Dispatching to Modal, handling the Modal webhook, firing the customer webhook. (A separate poll job runs on the *customer's* queue, owned by the SDK — see Delivery.) |
| 5 | **Object storage** | Default: the **customer's** bucket (BYO). Dev: a SaaS-hosted **managed bucket**. | BYO: the SDK presigns `file_url` (GET) + `upload_url` (PUT) on the customer's own disk; the SaaS never touches bytes. Managed: SaaS-hosted, quota-limited, ~1-day retention — for local dev with no bucket; the customer uploads bytes to us, the result lands with us, and `->markdown()` fetches it via the API. |

```
   composer require            HTTPS + API key            HTTP (Bearer secret)
customer app ──────▶ Client SDK ──────────▶ SaaS app ──────────────────▶ Modal worker
                         ▲                  │  ▲   │                         │
                         │                  │  │   └── presigned PUT/GET ──▶ S3 ◀──┘
                         └── markdown ───────┘  └────── webhook (result) ────────┘
```

## Boundaries & auth

- **Customer ↔ SDK:** plain PHP calls.
- **SDK ↔ SaaS:** HTTPS, customer **API key** (`Authorization: Bearer pfa_...`). This is the
  only credential a customer ever sees.
- **SaaS ↔ Modal:** server-to-server, shared **`PARSE_API_SECRET`** Bearer token. Never
  exposed to customers.
- **Modal ↔ SaaS (webhook):** Modal POSTs back with a per-request **`webhook_secret`** the
  SaaS generated; the SaaS verifies it before trusting the callback.
- **SaaS ↔ Customer (result):** under `webhook` delivery the SaaS POSTs the result to the
  SDK's webhook route, **HMAC-signed with a single per-account signing secret**
  (`PARSE_WEBHOOK_SECRET`, Stripe-style). The SDK verifies the signature against that one
  config value — no per-request secret is stored on the customer side. Under `poll` delivery
  (local dev) the SDK instead reads the result from the authenticated `GET /v1/parse/{id}`
  status endpoint. See "Delivery" below.

## Client SDK input API

The SDK speaks Laravel's **filesystem disk** language (`docs.md` covers the call forms). The
architectural point: the same configured disk is reused to presign the source GET + output PUT
(see "The parse flow") and is the one knob that selects BYO vs. the managed dev bucket — one
disk concept end to end.

## File-type routing — decided

**The SaaS routes by extension.** Allowlist and routing are the same step: known extension →
the matching `trigger-<type>` Modal endpoint; unknown extension → synchronous reject at
submit (`ParseException`). The SDK stays dumb and the customer just calls `->parse()`
regardless of format.

Why extension-only, no MIME/byte sniffing: **nobody upstream of Modal has the bytes.** Under
BYO the file sits in the customer's bucket — the SDK only presigns URLs and the SaaS only
ever sees `{id, file_url, upload_url}`. Sniffing would mean ranged GETs against presigned
URLs (latency + cost at bulk scale, and it dilutes "your bytes never transit us"). For our
customer base — Laravel apps parsing files their own code stored, typically validated with
`mimes:` at upload — the extension is present ~always and lying ~never.

The rare mismatch is handled by **verification by construction**: the Modal worker is the
only component that reads the file, and a parser failing on wrong bytes reports a typed
`ParseFailed` ("extension says pdf, content is docx") through the normal failure path.

Edge cases: `Parse::url()` requires a known extension in the URL for v1; a `->type('pdf')`
override covers extension-less URLs. (A HEAD-for-Content-Type fallback can come later if
people hit this.)

This decision is **internal and reversible** — customers never see the SaaS↔Modal wiring, so
collapsing to a single Modal endpoint with a dispatcher later would be a zero-impact
refactor. We pick extension routing because it ships: it keeps the proven per-type endpoint
contract with no new components on the greenfield backend's critical path.

## The parse flow (`->parse()`)

One mode: always async. There is a single flow; two axes vary underneath it — **where the bytes
live** (storage) and **how the result reaches the app** (delivery) — but the customer's code is
identical in every case.

The SDK owns all the plumbing: it mints the id, presigns URLs, persists a `ParseRequest` row,
and either runs a webhook route or a poll job to deliver the event. `->parse()` is a fast
inline HTTP call (tiny payload) — it does **not** require Laravel's queue to submit; customers
reach for their own queue only to fan out bulk submissions.

```
1. customer:  $parse = Parse::file('contracts/foo.pdf')->parse();   // path on their parse disk
2. SDK:       mint id (uuid), presign GET file_url + PUT upload_url on the
              customer's bucket, INSERT parse_requests row (status=pending)
3. SDK     →  POST /v1/parse {id, file_url, upload_url}  (tiny, API key) ──▶ SaaS  (202)
4. SaaS:      auth + quota check, persist job, route by extension
5. SaaS queue:POST trigger-<type> {file_url, upload_url,
              webhook_url:<SaaS>, webhook_secret:<SaaS>}     ──▶ Modal  (202)
6. Modal:     GET file_url (customer bucket) → parse
              → PUT markdown to upload_url (customer bucket) ──▶ customer bucket
7. Modal   →  POST <SaaS> webhook {status, url, page_count}  ──▶ SaaS
8. SaaS queue:verify webhook_secret, finalize metering, mark job done
9.            ── delivery to the customer (webhook or poll — see below) ──
10. SDK:      verify, look up parse_requests by id (idempotent),
              mark completed, fire ParseCompleted event
11. customer: handle the event (markdown is already in their bucket)
```

Notes:
- **The SaaS proxies the webhook** (Modal → SaaS → customer) rather than letting Modal call
  the customer directly. This keeps Modal hidden, lets the SaaS record completion for
  billing, and gives the customer one stable, signed callback contract.
- **Metering is per page** (1 credit = 1 page; $3 per 1,000 credits, 10,000 free/month —
  locked, see the business repo's `pricing.md`). The `page_count` Modal reports in the step-7
  webhook is the billable quantity, so metering finalizes at step 8, on completion — step 4
  only checks auth/quota. Page-equivalents for non-paginated formats (xlsx/email/image) and
  premium credit costs are still open.
- **The output is already in the bucket** by the time the event fires — the result carries the
  URL + metadata, not the markdown body. `->markdown()` reads it (from the customer bucket
  directly, or via the API for the managed bucket).
- The SDK ships the `ParseCompleted` / `ParseFailed` events and publishes editable listener
  stubs (via `php artisan parse:install`, auto-discovered on Laravel 12/13), so the developer
  customizes a listener rather than writing a controller. No `webhook:` argument on `->parse()`.
- **Batch submit:** `Parse::files([...])->parse()` presigns each file, INSERTs one
  `parse_requests` row per file, and sends them as a single batch payload to the SaaS, which
  fans out one Modal trigger per file. Returns a collection of `ParseRequest`. Each file gets
  its own event — the result path is identical to single-file.

### Storage — BYO bucket (default) vs. managed dev bucket

| | BYO bucket (default / prod) | Managed dev bucket |
|:--|:--|:--|
| **Source bytes** | already in the customer's bucket; SDK presigns a GET | customer uploads to the SaaS |
| **Output** | PUT back to the customer's bucket; **SaaS never touches bytes** | held by the SaaS |
| **`->markdown()` reads** | the customer disk directly | the SaaS API |

Selected by the configured disk — `parse.disk` set → BYO, unset → managed (the zero-config
fallback, so a fresh install parses with no bucket setup). Limits/retention live in `docs.md`.

### Delivery — webhook (default) vs. poll (local)

Both paths fire the **same** `ParseCompleted`/`ParseFailed` event; only the transport differs.
`parse.delivery` selects it: `auto` (default) resolves to **`poll` when `APP_ENV=local`,
`webhook` everywhere else** (staging, production, …).

- **`webhook`** — at step 9 the SaaS POSTs `{id, status, markdown_url, page_count}` to the
  SDK's published route, **HMAC-signed with the single per-account `PARSE_WEBHOOK_SECRET`**
  (Stripe-style). The SDK verifies the signature and fires the event. Reactive; no queue. The
  per-request `webhook_secret` is internal to the Modal → SaaS hop only — it never reaches the
  customer's table.
- **`poll`** — for local dev, where the SaaS can't reach an inbound webhook. `->parse()`
  dispatches a self-releasing job onto the customer's queue; it polls `GET /v1/parse/{id}`,
  `release()`s itself while `pending`, and fires the same event on a terminal status. Needs a
  real queue driver (`database`/`redis`, not `sync`); rides the `queue:listen` worker that
  Laravel's `composer run dev` already runs. A capped attempt count / TTL fires `ParseFailed`
  if the job never completes, so the event always eventually arrives.

**`->status()` reads the local `parse_requests` row** (a DB read, not an API call); the row is
kept current by whichever delivery is active. Consequence: locally it only advances while a
worker runs the poll job. It reports progress only — the result is delivered by the event.

### Client SDK state — the `parse_requests` table

The SDK ships a migration + Eloquent model so async submissions can be correlated and
deduped without the developer handling ids or secrets:

| column | purpose |
|:--|:--|
| `id` (uuid) | SDK-generated correlation handle; sent to the SaaS, echoed in the webhook |
| `disk`, `source_path`, `output_path` | where the file is and where the markdown lands |
| `status` | `pending` → `completed` / `failed` |
| `page_count`, `error` | filled on completion |
| `meta` (json) | customer-supplied context (e.g. `invoice_id`), returned in the event |
| `created_at`, `completed_at` | timing / pruning |

No secret column — webhook verification uses the single `PARSE_WEBHOOK_SECRET`. Delivery is
idempotent (looks up by `id`, ignores already-terminal rows) so duplicate deliveries — and the
webhook/poll paths never running together — are safe. `->parse()` returns this model; the
`ParseCompleted` event carries it; `->status()` reads its `status` column.

## Queues

**Inside the SaaS:**
- **dispatch-to-modal** — POST the right `trigger-<type>` endpoint with the presigned URLs
  (for BYO, the SaaS does not presign or touch bytes).
- **handle-modal-webhook** — verify `webhook_secret`, update job, meter usage. (Markdown is
  already in the bucket; the SaaS only reads metadata.)
- **notify-customer** (webhook delivery) — sign and POST to the customer's webhook; retry with
  backoff.
- Built to absorb Modal's burst profile (`max_containers=100` per worker upstream).

**On the customer side (SDK):** the **poll-parse-status** job exists only under `poll`
delivery (local dev) — it polls `GET /v1/parse/{id}` and fires the event. It runs on the
customer's own queue and is never used when delivery is `webhook`.

## Scaling the parse backend — worker tiers & large documents

The current `parse-function` has one worker per file type with fixed memory/timeout. To
handle a wide size range — from a 1-page invoice to a multi-hundred-MB scan — without
over-provisioning every job:

### Worker sizing — v1 decided: one size per type, no tiers

**v1 has no worker tiers and no size routing.** Each `trigger-<type>` endpoint is one
generously-sized function (the PDF worker: ~8–16 GB memory, ~2 h timeout cap, big ephemeral
disk — exact numbers set by the pre-launch validation test). `trigger-pdf` just *is* the
worker; no entry function, no HEAD probe, nothing routes on size.

Why this isn't reckless: **internal serial page-batching bounds peak memory regardless of
document size** (see below), so one size covers the advertised range. Tiers only buy cost
efficiency — paying big-memory-seconds for 1-page invoices — and at Modal's memory pricing
that waste is ~$0.0005 per invoice against $0.003/page revenue: real at scale, irrelevant
at zero customers. A wrong size number is fixed by editing one config value and redeploying.

**When tiers come back:** when usage shows it — unit economics at real volume, or a job
class needing a different resource profile. The design is settled and waiting: Modal-internal,
a thin per-type entry function doing HEAD → `Content-Length` → threshold → `.spawn()` the
right-sized worker, tier reported in the completion webhook for billing. Tiers are the cheap
first step; chunking (below) the expensive second.

### Large documents — v1 is single-pass; chunking is deferred

**Committed target: PDFs up to 1 GB** (advertised in the docs, explicitly scoped to PDF —
other formats carry lower caps; a 1 GB xlsx would melt any engine). The v1 path is
**one single-pass worker, not distributed chunking**:

- generous memory/timeout/disk on the per-type worker; drop `MAX_PAGES_FOR_OCR = 200`;
- bound memory **inside** the worker by processing pages in serial batches —
  `pymupdf4llm.to_markdown(pages=[...])` and `pdf2image`'s `first_page`/`last_page` already
  support this. Internal batching: no cross-container coordination, no stitch step, no
  partial-failure handling.

The cost is wall-clock on the extreme tail: a 2,000-page scanned PDF OCR'd serially takes
hours. That's acceptable under the always-async contract (no latency promise), and free
early access is the right time to learn whether anyone actually feels it.

**Validate before launch:** run a genuinely large file — including a big scanned PDF —
through the PDF worker end to end. The 1 GB number in the docs must be tested, not
aspirational; the test also sets the worker's memory/timeout/disk numbers.

**Chunked parsing (split → fan out → stitch) is deferred.** It's a *speed* optimization for
huge documents, not a capability requirement, and its design is a sketch, not final: split
into page ranges (e.g. 50 pages), fan out across containers (already scales to
`max_containers=100`), stitch the markdown back together (`## Page N` markers) and write
one result to the bucket. A 2,000-page PDF becomes 40 parallel 50-page jobs instead of one
serial grind. Build it when single-pass jobs prove too slow for real customers.

**Status:** docs advertise **up to 1 GB for PDFs** (lower caps elsewhere); v1 delivers it
with one generously-sized single-pass worker per type (raised caps/timeouts/disk + internal page batching), **to
build in the new backend (`../modal`)**. Tiers and chunking deferred.

## Backend quality, gaps & risks (TBD)

The parse backend (`~/Herd/parse-function/CLAUDE.md`) is **not
final** — it's a per-format Python stack we can adjust. Benchmarking against
[parsel](https://github.com/shipfastlabs/parsel) (a local PHP library built on
[liteparse](https://github.com/run-llama/liteparse) — the same engine `parse-function`
already evaluated and rejected) surfaced where our underlying packages are weaker. None of
these are decided; we'll pick which gaps to close later.

### Our current engines

| Type | Engine | Output |
|:--|:--|:--|
| PDF | `pymupdf4llm` (MuPDF) | Markdown — headings, tables, multi-column reading order, links |
| Scanned PDF | Tesseract (via `pdf2image`) | Markdown (OCR), capped at 200 pages |
| `.docx` | `mammoth` | Markdown |
| `.pptx` | `python-pptx` | Markdown (text + tables) |
| `.xlsx` | `openpyxl` | Markdown tables |
| `.eml`/`.msg` | stdlib `email` / `extract-msg` | Markdown + frontmatter |
| images | `pyvips` (optimize only) | **No markdown** |

### Where we're likely weaker than the parsel/liteparse stack

- **Office fidelity.** parsel uses **LibreOffice**, which does true layout-fidelity
  conversion. Our `mammoth` / `python-pptx` / `openpyxl` are lightweight but lower-fidelity
  on complex documents. With LibreOffice now in the v1 stack for legacy formats (below), a
  "high-fidelity Office" tier for *complex modern* docs costs much less to add later — the
  image and worker already exist. *Still a roadmap option, not v1.*
- **Legacy `.doc`/`.ppt`/`.xls` + `.csv` — decided, to build in `../modal` v1.** The user
  docs advertise them. Legacy binary formats go through a **LibreOffice-equipped worker used
  as a conversion shim**: convert legacy → modern (`soffice` headless, `.doc`→`.docx`,
  `.ppt`→`.pptx`, `.xls`→`.xlsx`), then run the *same* parsers as the modern path — so
  legacy and modern files yield identical markdown. Separate worker image (LibreOffice is
  heavy; one job per container, so its concurrency-unsafety doesn't bite); modern paths
  stay light. SaaS extension routing sends legacy extensions to this worker's endpoint —
  fits the per-type routing unchanged. `.csv` is a trivial add to the spreadsheet path.
- **Standalone image OCR (clear hole).** parsel OCRs images (ImageMagick → PDF → Tesseract).
  Our image worker only **optimizes** (resize to JPEG) — it produces no text/markdown.
  *Option: add an image→markdown OCR path; cheap, obvious.*
- **Structured / coordinate output.** liteparse (PDFium) returns bounding boxes, font info,
  and confidence per token — enabling citations, highlighting, redaction. We emit markdown
  only. *Demand-driven roadmap item — see "Roadmap: structured output" below.*
- **Pluggable OCR.** parsel can route to EasyOCR/PaddleOCR for higher accuracy; we're fixed
  on Tesseract at the default tier.

### Where we're likely stronger

- **Markdown-first.** liteparse outputs text/JSON only — you'd build markdown yourself.
- **Complex-layout markdown.** liteparse's own README concedes weakness on dense tables,
  multi-column, and scanned PDFs; `pymupdf4llm` handles multi-column reading order + tables.
- **Scale & ops.** Cloud autoscale + async bulk vs. a synchronous local library whose heavy
  LibreOffice path is slow and concurrency-unsafe. Zero-install DX (`composer` + key) vs. a
  binary install chain on the customer's server.

### Licensing risk (our side)

- **`pymupdf4llm` = PyMuPDF / MuPDF is AGPL.** For a commercial paid SaaS this likely
  requires an Artifex commercial license — **resolve before charging.** (liteparse's PDFium
  is BSD, so cleaner.) Track this as a real blocker, not a nice-to-have.

### Reality check

Neither stack matches true vision-model cloud parsers (LlamaParse/Reducto) — both top out at
Tesseract-class OCR. A future vision-model worker tier is the path to close that, and is
something only the cloud architecture (not a local library) can do.

## Roadmap: structured output — bounding boxes + page images (build on demand)

**Not building yet — add when customers ask.** Captured here so we know the cost up front.
The engine work is nearly free (the PDF worker already imports `fitz` / PyMuPDF); the real
cost is the output plumbing and a new API/SDK contract.

- **Bounding boxes — easy (~1–1.5 days incl. OCR path).** PyMuPDF already exposes what PDFium
  does, and richer: `page.get_text("words")` gives per-word `(x0,y0,x1,y1)`; `get_text("dict")`
  adds font/size/color. For scanned PDFs, `pytesseract.image_to_data()` gives per-word boxes
  **+ confidence** (matches parsel's confidence field). Serialize to JSON → one extra presigned
  URL. Document the PDF-points coordinate system and emit page width/height for normalization.
- **Page images — easy to render, plumbing is the work (~2–3 days).** `page.get_pixmap(dpi=…)`
  renders PNGs with no new deps (no poppler needed). The hard part is **N variable outputs** vs.
  today's single `upload_url`. Options:
  - **A. Zip bundle** — render all pages, PUT one `screenshots.zip` to a single presigned URL.
    Keeps the current one-URL contract; simplest. *(Recommended for v1.)*
  - **B. Scoped credentials** — SDK mints short-lived STS creds for a `screenshots/` prefix; the
    worker writes `page-N.png` individually. Cleanest for per-page URLs; more setup. *(Later.)*
  - **C. Two-phase** — probe page count, presign N URLs, then render. Chattier round-trips.
- **Scope: PDF + images only.** docx/xlsx/pptx have no page raster without rendering via
  LibreOffice first (ties to the high-fidelity Office tier above).
- **Output-shape decision (the real design call, not difficulty).** These aren't a markdown
  string. They want a distinct **structured mode** — e.g. `Parse::file(...)->layout()` returning
  page objects `{ markdown, image_url, words: [{text, bbox, confidence}] }` — separate from
  `->markdown()`. Decide the surface when we commit to building.

## What lives where (repos)

All repos live under one parent, `~/Herd/parseforartisans/`:

- `architecture` (this repo) — architecture + user docs only. No app code.
- `app` — the SaaS Laravel app (Laravel 13, Livewire 4, Flux, Pest). Early scaffold.
- `modal` — the **new** Modal parse backend. Greenfield, to be written from scratch.
- `evaluation` — eval harness vs LlamaParse (103-file corpus).
- `sdk` — the client Composer package. **Not created yet.**

The **old** backend, `~/Herd/parse-function` (outside the parent), is the working contract
reference — shared with another project, reference only, don't modify.

## Still open (see CLAUDE.md)

- Page-equivalents for non-paginated formats (xlsx / email / image) and premium credit
  costs — the per-page unit itself is locked (1 credit = 1 page, $3/1k, 10k free/month).
- Worker resource numbers (memory/timeout/disk per type — set by the pre-launch validation
  test), and the deferred tier and chunking designs; v1 shape is decided (one size per
  type, single-pass, no size routing); see "Scaling the parse backend" above. The **1 GB**
  docs claim must be validated end to end before launch.
- Which backend gaps to close (Office fidelity via LibreOffice, image OCR, structured/coord
  output) and the **PyMuPDF AGPL licensing** question — see "Backend quality, gaps & risks".
  `parse-function` is not final and can be adjusted.
