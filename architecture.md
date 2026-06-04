# Architecture

Systems and flows for **Parse for Artisans** (parseforartisans.com). This is the
technical companion to `docs.md` (user-facing). See `CLAUDE.md` for the high-level
overview and open decisions.

## Systems

| # | System | What it is | Owns |
|---|--------|-----------|------|
| 1 | **Client SDK** | Composer package the customer installs (`composer require ...`) | The `Parse::file()->parse()` / `->status()` / `->markdown()` surface + `ParseCompleted`/`ParseFailed` events. Talks only to the SaaS API over HTTPS with the customer's API key. Owns a `parse_requests` table (correlation + status), a signed webhook route (webhook delivery), and a self-releasing poll job (poll delivery). |
| 2 | **SaaS app** | The Laravel application we run at parseforartisans.com | Auth, API keys, billing/metering, presigned URLs, file-type detection & routing, calling Modal, receiving Modal's webhook, the status endpoint, customer webhooks. Hosts the managed dev bucket. |
| 3 | **Parse backend** | Existing Modal serverless workers (`../parse-function`) | The actual parse. One endpoint per file type. Fully async. Never customer-facing. |
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

## File-type routing

**File-type detection and Modal endpoint routing live in the SaaS app** (by extension + MIME
sniff), so the SDK stays dumb and the customer just calls `->parse()` regardless of format.
*(Final placement to be confirmed — leaning SaaS.)*

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
4. SaaS:      auth + meter, persist job, detect type
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
- **The output is already in the bucket** by the time the event fires — the result carries the
  URL + metadata, not the markdown body. `->markdown()` reads it (from the customer bucket
  directly, or via the API for the managed bucket).
- The SDK ships the `ParseCompleted` / `ParseFailed` events, so the developer writes a
  listener, not a controller. No `webhook:` argument on `->parse()`.
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

## Scaling the parse backend — worker tiers & large documents (TBD)

The current `parse-function` has one worker per file type with fixed memory/timeout. To
handle a wide size range — from a 1-page invoice to a multi-hundred-MB scan — without
over-provisioning every job, two complementary directions, **both TBD**:

### Worker tiers (different memory/timeout per size)

Deploy the same parse logic as **several Modal functions sized differently** — e.g.
`small` (low memory, short timeout), `medium`, `large` (high memory, long timeout, bigger
ephemeral disk). Route a job to the right tier by file size / page count, so small jobs stay
cheap and fast while big jobs get the resources they need. Routing can live in **either**
place:
- **SaaS-side routing** — the SaaS inspects size (from the upload, a HEAD on `file_url`, or
  a cheap page-count probe) and calls the matching `trigger-<type>-<tier>` endpoint.
- **Modal-side routing** — the SaaS hits one endpoint; a thin dispatcher inside Modal probes
  the file and `.spawn()`s the appropriately-sized worker.

Trade-off to settle: SaaS-side keeps Modal dumb and makes cost/routing visible to billing;
Modal-side keeps the SaaS contract to a single endpoint per type. Leaning SaaS-side for
visibility, but undecided.

### Chunked parsing (the path to 1 GB)

**Committed target: support up to 1 GB per file** (advertised in the docs). The backend will
be built to deliver this. For documents too big for even a `large` worker in one pass, split
and parallelize (this is the existing TODO in `parse-function/CLAUDE.md`):
1. **Split** into page ranges (e.g. 50 pages each) — `pymupdf4llm.to_markdown(pages=[...])`
   and `pdf2image`'s `first_page`/`last_page` already support this.
2. **Fan out** the chunks across many containers in parallel (already scales to
   `max_containers=100`).
3. **Stitch** the chunk markdown back together (with `## Page N` markers) and write the
   single result to the customer's bucket.

This also lifts today's hard caps that block big files: `MAX_PAGES_FOR_OCR = 200`, the
30-minute OCR `timeout`, single-pass memory, and ephemeral disk for the download. It makes
huge docs *faster*, not just possible (a 2,000-page PDF → 40 parallel 50-page jobs instead
of one serial grind).

**Status:** docs advertise **up to 1 GB**; the backend work (chunked parsing + raised
caps/timeouts/disk) is committed and **to build in `parse-function`**.

## Backend quality, gaps & risks (TBD)

The parse backend ([`../parse-function/CLAUDE.md`](../parse-function/CLAUDE.md)) is **not
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

- **Office fidelity (biggest gap).** parsel uses **LibreOffice**, which does true
  layout-fidelity conversion and handles legacy `.doc`/`.ppt`. Our `mammoth` / `python-pptx`
  / `openpyxl` are lightweight but lower-fidelity on complex documents. *Option: a
  LibreOffice-backed "high-fidelity Office" worker tier (fits the worker-tiers plan above).*
- **Legacy `.doc`/`.ppt` + `.csv` (committed in the docs, not built yet).** The user docs now
  advertise `.doc`, `.ppt`, and `.csv`. These need new backend paths: `.doc`/`.ppt` likely via
  a **LibreOffice** worker (our pure-Python libs are `.docx`/`.pptx`-only); `.csv` is a trivial
  add to the spreadsheet path. **To build in `parse-function`.**
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

- `parse-architecture` (this repo) — architecture + user docs only. No app code.
- `parse-function` (`../parse-function`) — Modal workers. Already exists.
- SaaS app — separate repo, **not created yet**.
- Client SDK — separate Composer package, **not created yet**.

## Still open (see CLAUDE.md)

- Pricing/metering unit (page / file / MB).
- Final home of file-type detection (leaning SaaS).
- Worker tiers + where size-based routing lives (SaaS-side vs. Modal-side); chunked parsing
  for the **1 GB** target — see "Scaling the parse backend" above. Docs advertise up to 1 GB.
- Which backend gaps to close (Office fidelity via LibreOffice, image OCR, structured/coord
  output) and the **PyMuPDF AGPL licensing** question — see "Backend quality, gaps & risks".
  `parse-function` is not final and can be adjusted.
