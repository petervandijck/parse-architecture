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
| 5 | **Object storage** | S3 (SaaS-managed by default) | Source file bytes + resulting markdown. Generates the presigned `file_url` (GET) and `upload_url` (PUT) Modal needs. *Storage model still open — see CLAUDE.md.* |

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

For the simple, interactive case. The SDK blocks; **the SaaS holds the HTTP request open**
until Modal's webhook lands, then returns the markdown in the response. (No customer-side
polling — the wait happens server-side on one request.)

```
1. customer:  $md = $request->file('doc')->markdown();
2. SDK     →  POST /v1/parse (multipart, API key)            ──▶ SaaS
3. SaaS:      auth + meter, store bytes in S3,
              detect type, presign file_url + upload_url
4. SaaS    →  POST trigger-<type> {file_url, upload_url,
              webhook_url, webhook_secret}                   ──▶ Modal   (202)
5. Modal:     download, parse, PUT markdown to upload_url    ──▶ S3
6. Modal   →  POST webhook_url {status, url, ...}            ──▶ SaaS
7. SaaS:      verify webhook_secret, fetch markdown from S3,
              record usage for billing
8. SaaS    →  200 { markdown: "..." }                        ──▶ SDK
9. SDK     →  returns the string to the customer
```

Notes: a request-scoped timeout caps how long the SaaS waits; on timeout the SDK gets a
clear "still processing, use async" error. Mechanism for step 6→8 correlation (e.g. cache
key on `webhook_secret`, or short internal poll of the job record) is an implementation
detail of the SaaS.

## Flow B — Async / queued (`->queue()`)

For bulk imports (100s–1000s of files; the Modal backend is built for these bursts). The
customer supplies (or pre-registers) a **webhook URL**; the SaaS notifies them on
completion. **No polling.**

```
1. customer:  Parse::file($path)->queue(webhook: 'https://app.test/parsed');
2. SDK     →  POST /v1/parse?async=1 (API key, callback url) ──▶ SaaS
3. SaaS:      auth + meter, store bytes, create job record,
              return { id } immediately                      ──▶ SDK (202)
4. SaaS queue: dispatch job → presign + POST trigger-<type> ──▶ Modal (202)
5. Modal:     parse, PUT markdown to upload_url              ──▶ S3
6. Modal   →  POST webhook_url {status, url, ...}            ──▶ SaaS
7. SaaS queue: verify, record usage, mark job done
8. SaaS    →  POST customer webhook {id, status, markdown_url
              or markdown, signature}                        ──▶ customer app
9. customer:  verify signature, handle the result
```

The customer's app receives the result in its own controller/listener — the SDK ships a
ready-made signed-webhook verifier and an optional Laravel event to make step 9 a one-liner.

## Queues (inside the SaaS)

- **dispatch-to-modal** — presign URLs, POST the right `trigger-<type>` endpoint.
- **handle-modal-webhook** — verify `webhook_secret`, pull markdown, update job, meter usage.
- **notify-customer** — sign and POST to the customer's webhook; ret(retry with backoff).
- Built to absorb Modal's burst profile (`max_containers=100` per worker upstream).

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
