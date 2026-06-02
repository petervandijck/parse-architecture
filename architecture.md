# Architecture

Systems and flows for **Parse for Artisans** (parseforartisans.com). This is the
technical companion to `docs.md` (user-facing). See `CLAUDE.md` for the high-level
overview and open decisions.

## Systems

| # | System | What it is | Owns |
|---|--------|-----------|------|
| 1 | **Client SDK** | Composer package the customer installs (`composer require ...`) | The `Parse::file()` / `->markdown()` surface. Talks only to the SaaS API over HTTPS with the customer's API key. |
| 2 | **SaaS app** | The Laravel application we run at parseforartisans.com | Auth, API keys, billing/metering, file storage, presigned URLs, file-type detection & routing, calling Modal, receiving Modal's webhook, serving results back to the SDK, customer webhooks. |
| 3 | **Parse backend** | Existing Modal serverless workers (`../parse-function`) | The actual parse. One endpoint per file type. Fully async. Never customer-facing. |
| 4 | **Queue** | Laravel queue (Redis/SQS) inside the SaaS app | Async/bulk jobs: dispatching to Modal, handling the Modal webhook, firing the customer webhook. |
| 5 | **Object storage** | Async: the **customer's** bucket. Sync: ephemeral SaaS-internal scratch. | Async: the SDK presigns `file_url` (GET) + `upload_url` (PUT) on the customer's own disk; the SaaS never touches bytes. Sync: the customer sends bytes (or a URL); the SaaS uses short-lived internal scratch and returns the markdown inline — no customer bucket. |

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
- **SaaS ↔ Customer (async result):** the SaaS POSTs the result to a **customer webhook**
  URL, signed so the customer can verify it. **Async delivery is webhook-based — no
  polling.**

## File-type routing

The developer always just calls `->markdown()` regardless of format. **File-type detection
and Modal endpoint routing live in the SaaS app** (by extension + MIME sniff), so the SDK
stays dumb and the customer never thinks about PDF-vs-docx-vs-xlsx. *(Final placement to be
confirmed — leaning SaaS.)*

## Flow A — Sync / blocking (`->markdown()`)

For the simple, interactive case. **Zero buckets for the customer** — they send the file
bytes (a multipart upload) *or* a URL, and get the markdown string back in the same HTTP
response. The SDK blocks; **the SaaS holds the request open** until Modal finishes. The only
storage involved is **ephemeral, SaaS-internal** — the customer never configures or sees it.

```
1. customer:  $md = $request->file('doc')->markdown();      // or Parse::url($href)->markdown()
2. SDK     →  POST /v1/parse (multipart bytes OR { url }, API key)  ──▶ SaaS
3. SaaS:      auth + meter, detect type, stash bytes in scratch storage,
              presign internal file_url + upload_url (ephemeral)
4. SaaS    →  POST trigger-<type> {file_url, upload_url,
              webhook_url, webhook_secret}                   ──▶ Modal   (202)
5. Modal:     download, parse, PUT markdown to upload_url    ──▶ SaaS scratch
6. Modal   →  POST webhook_url {status, url, ...}            ──▶ SaaS
7. SaaS:      verify webhook_secret, read markdown from scratch,
              record usage, discard scratch files
8. SaaS    →  200 { markdown: "..." }                        ──▶ SDK
9. SDK     →  returns the string to the customer
```

Notes:
- The customer sends **bytes or a URL** — no presigning, no bucket config. This is the
  difference from Flow B, where the customer's own bucket holds both ends.
- Scratch storage is SaaS-internal and short-lived (deleted after the response). Capped at
  the sync size limit (e.g. 20 MB) precisely because we hold it in our request path.
- A request-scoped timeout caps the wait; on timeout the SDK gets a clear "still processing,
  use async" error. Step 6→8 correlation (cache key on `webhook_secret`, or short internal
  poll of the job record) is a SaaS implementation detail.

## Flow B — Async / queued, bring-your-own-bucket (`->queue()`)

**This is the recommended async flow** for large files (up to ~200 MB to start) and bulk
imports (tens of thousands of files at once; the Modal backend is built for these bursts).

The key property: **the SaaS never touches the file bytes.** The customer's file already
lives in their own bucket. The SDK presigns a GET (the source) and a PUT (where the output
should land), generates a webhook secret, and POSTs a *tiny* JSON payload. The parse runs,
and the markdown lands **back in the customer's own bucket**. Fast to submit, private (data
never transits us), cheap (no storage on our side).

The SDK owns all the plumbing. The developer only configures a **disk** — the SDK generates
the presigned URLs, the webhook secret, and the webhook route automatically. **No polling.**

```
1. customer:  Parse::file('contracts/foo.pdf')->queue();   // path on their parse disk
2. SDK:       presign GET file_url + PUT upload_url on the customer's bucket,
              mint webhook_secret, create local job record
3. SDK     →  POST /v1/parse {file_url, upload_url,
              webhook_url, webhook_secret}  (tiny, API key)  ──▶ SaaS   (202 { id })
4. SaaS:      auth + meter, persist job, detect type
5. SaaS queue:POST trigger-<type> {file_url, upload_url,
              webhook_url:<SaaS>, webhook_secret:<SaaS>}     ──▶ Modal  (202)
6. Modal:     GET file_url (customer bucket) → parse
              → PUT markdown to upload_url (customer bucket) ──▶ customer bucket
7. Modal   →  POST <SaaS> webhook {status, url, page_count}  ──▶ SaaS
8. SaaS queue:verify, finalize metering, mark job done
9. SaaS    →  POST customer webhook_url {id, status,
              markdown_url, page_count, signature}           ──▶ customer app
10. SDK webhook route: verify signature, fire ParseCompleted event
11. customer: handle the event (markdown is already in their bucket)
```

Notes:
- **The SaaS proxies the webhook** (Modal → SaaS → customer) rather than letting Modal call
  the customer directly. This keeps Modal hidden, lets the SaaS record completion for
  billing, and gives the customer one stable, signed callback contract.
- **The output is already in the customer's bucket** by the time the webhook fires — the
  callback just carries the URL + metadata, not the markdown body.
- The SDK ships a **published webhook route** and a `ParseCompleted` event, so the developer
  writes a listener, not a controller. No `webhook:` argument needed on `->queue()`.

## Queues (inside the SaaS)

- **dispatch-to-modal** — POST the right `trigger-<type>` endpoint with the customer's
  presigned URLs (the SaaS does not presign or touch bytes in the async flow).
- **handle-modal-webhook** — verify `webhook_secret`, update job, meter usage. (Markdown is
  already in the customer's bucket; the SaaS only reads metadata.)
- **notify-customer** — sign and POST to the customer's webhook; retry with backoff.
- Built to absorb Modal's burst profile (`max_containers=100` per worker upstream).

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

### Chunked parsing (the path to 1 GB+)

For documents too big for even a `large` worker in one pass, split and parallelize (this is
the existing TODO in `parse-function/CLAUDE.md`):
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

**Status:** design only. Public docs cap async at **200 MB to start**; 1 GB+ depends on
shipping chunked parsing.

## What lives where (repos)

- `parse-architecture` (this repo) — architecture + user docs only. No app code.
- `parse-function` (`../parse-function`) — Modal workers. Already exists.
- SaaS app — separate repo, **not created yet**.
- Client SDK — separate Composer package, **not created yet**.

## Still open (see CLAUDE.md)

- Storage model: SaaS-managed S3 vs. bring-your-own bucket.
- Sync-wait mechanism details (hold-open vs. internal short poll) and timeout policy.
- Pricing/metering unit (page / file / MB).
- Final home of file-type detection (leaning SaaS).
- Worker tiers + where size-based routing lives (SaaS-side vs. Modal-side); chunked parsing
  for 1 GB+ — see "Scaling the parse backend" above. Async cap is **200 MB to start**.
