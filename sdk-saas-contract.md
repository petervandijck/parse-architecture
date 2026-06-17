# SDK to SaaS wire contract (v1)

The HTTP contract between the **Client SDK** (the Composer package in the customer's app) and
the **SaaS** (`parseforartisans.com`). This is the design spec for the API, not code. It covers
the customer-facing half only. How the SaaS runs the actual parse (backend orchestration) is out
of scope here.

Status: **draft**, decisions applied June 2026 (see "Decisions" at the end). Choices follow
`architecture.md`, `CLAUDE.md`, and `final_improvements.md`.

## Principles and invariants

1. **One async contract.** `->parse()` submits and returns a handle. Results arrive by event
   (`ParseCompleted` / `ParseFailed`). There is no blocking string return in code.
2. **The SDK and SaaS exchange only ids and URLs, never the markdown body.** The result lands in
   storage out of band; the event and status responses carry metadata only.
3. **Presigning is the only thing the storage mode changes.** Under BYO the SDK presigns against
   the customer disk; under managed the SaaS presigns against its own bucket. Everything else in
   this contract is identical across modes.
4. **The SDK owns correlation.** The SDK mints the request `id` (uuid) and keys everything on it.
   `->for()` / `->withMeta()` never leave the SDK. The SaaS only echoes the `id`.
5. **Validation is server-side.** The SDK does not pre-check extensions or option/format
   compatibility; it submits and the SaaS validates. This keeps the allowlist and option rules
   evolving server-side without an SDK release.
6. **Bytes vs metadata.** Under BYO the customer's file *bytes* never transit the SaaS. The SaaS
   does see metadata (the presigned URLs, which contain the object key, and the file extension).
   Under managed, bytes do transit the SaaS by design (dev-only bucket).
7. **Two error channels.** Submission problems are synchronous (`->parse()` throws
   `ParseException`). Parse-time problems are asynchronous (`ParseFailed` event).

## Identifiers and correlation

| Token | Minted by | Scope | Travels |
|:--|:--|:--|:--|
| `id` (uuid) | SDK | account | SDK to SaaS, echoed in status + webhook back to SDK |
| API key `pfa_...` | SaaS (dashboard) | account | every SDK to SaaS call |
| Signing secret `whsec_...` | SaaS (dashboard) | account | not sent; both sides hold it for HMAC |

The SaaS keys its job on `(account, id)`. The `id` is **account-scoped**: a lookup for an `id`
that does not belong to the calling API key's account returns 404, never another account's data
(resolves G7).

## Auth and secrets

| Hop | Credential | Header |
|:--|:--|:--|
| SDK to SaaS | API key `pfa_...` | `Authorization: Bearer pfa_...` |
| SaaS to SDK (webhook) | account `whsec_...` | `X-Parse-Signature: t=<ts>,v1=<hmac>` |

## Storage modes

Selected by `parse.disk` (or a per-call `->disk('s3')`).

| | BYO (disk set) | Managed (disk unset) |
|:--|:--|:--|
| Who presigns `file_url` / `upload_url` | the SDK, against the customer disk | the SaaS, against its managed bucket |
| Source bytes | already in the customer bucket | the SDK reads the **default disk** (`config('filesystems.default')`) and uploads them to the SaaS at submit (decision **M1**) |
| Submit content type | `application/json` | `multipart/form-data` (metadata + file) |
| `->markdown()` reads | the customer disk directly (`Storage::disk($disk)->get($output_path)`) | the SaaS API (`GET /v1/parse/{id}/markdown`) |
| `->to(path)` | sets the output object key | not meaningful; the result is addressed by `id` (recorded on the row, does not change retrieval) |

## Endpoints

| Method | Path | Purpose |
|:--|:--|:--|
| POST | `/v1/parse` | submit one file (BYO = JSON, managed = multipart) |
| POST | `/v1/parse/batch` | submit many files in one call |
| GET | `/v1/parse/{id}` | status (drives `->status()` and the poll job) |
| GET | `/v1/parse/{id}/markdown` | fetch the result (managed mode only) |
| GET | `/v1/ping` | `parse:ping` health/key check |

All paths are versioned under `/v1`. Base URL `https://parseforartisans.com`.

---

### POST `/v1/parse` (submit)

**BYO mode (JSON):**

```json
{
  "id": "9f1c0b6e-...-uuid",
  "extension": "pdf",
  "filename": "contracts/foo.pdf",
  "source": {
    "mode": "byo",
    "file_url": "https://bucket.example/contracts/foo.pdf?X-Amz-...",
    "upload_url": "https://bucket.example/parsed/contracts/foo.pdf.md?X-Amz-..."
  },
  "delivery": {
    "mode": "webhook",
    "callback_url": "https://customer-app.example/parse/webhook"
  },
  "options": {
    "force_ocr": false,
    "pages": "1-20",
    "frontmatter": false,
    "ocr_language": null
  }
}
```

**Managed mode (multipart/form-data):** the same JSON minus `source` (the SaaS provides the
URLs) sent as a `payload` part, plus a `file` part carrying the bytes the SDK read from the
default disk. The SaaS stores the bytes, presigns GET + PUT on its managed bucket, and proceeds
identically from there.

Field notes:

- `extension` is the routing key. The SaaS routes on this field (it does not parse the URL or
  sniff bytes). Unknown extension is a synchronous reject (see errors). `filename` is optional and
  for diagnostics/logging only.
- The SaaS validates option/extension compatibility at submit and rejects an incompatible combo
  synchronously (`unsupported_option`), for example `pages` on a docx. The SDK does not pre-check.
- `delivery.mode` is `webhook` or `poll`. For `poll` (local dev), `callback_url` is omitted and
  the SaaS makes no outbound call; the SDK polls `GET /v1/parse/{id}`. For `webhook`, the SaaS
  POSTs the result to `callback_url` on completion. Sending `callback_url` per request keeps the
  zero-config promise (no dashboard webhook step). The SaaS SSRF-hardens it: require an `https`
  URL, reject when the host resolves to a private, loopback, link-local, or cloud-metadata
  (`169.254.169.254`) address, and do not follow redirects.
- `options` is the canonical object below. `to`, `for`, and `withMeta` are SDK-local and are
  **not** sent (`to` is applied by the SDK when presigning `upload_url`).

**202 response:**

```json
{ "id": "9f1c0b6e-...-uuid", "status": "pending" }
```

### POST `/v1/parse/batch`

`{ "items": [ <submit object>, ... ] }` (JSON for BYO; multipart with multiple `file` parts and
a `payload` manifest for managed). Returns `{ "items": [ { "id", "status" }, ... ] }`. The SaaS
fans out one parse per item; each item fires its own webhook/event. Batch is BYO-first; managed
batch works but is dev-sized.

### Submit errors (synchronous, become `ParseException`)

Any non-2xx from submit is thrown by `->parse()` as `ParseForArtisans\Exceptions\ParseException`.

```json
{ "error": { "type": "unsupported_type", "message": "Extension 'rtf' is not supported." } }
```

| HTTP | `type` | When |
|:--|:--|:--|
| 401 | `invalid_api_key` | missing/invalid key (checked before the body) |
| 422 | `unsupported_type` | extension not in the allowlist |
| 422 | `unsupported_option` | an option is not valid for the file type (e.g. `pages` on docx) |
| 402 | `quota_exceeded` | account is out of credits at submit |
| 400 | `invalid_request` | malformed body / missing required field |
| 413 | `file_too_large` | managed upload over the dev cap |

Note: quota at submit is a coarse gate (page count is unknown until completion), so the true
usage is finalized later. Quota can be slightly overshot by in-flight jobs; acceptable for v1.

### GET `/v1/parse/{id}` (status)

Account-scoped. Drives `->status()` (the local DB row is kept current from this) and the poll job.

```json
{
  "id": "9f1c0b6e-...-uuid",
  "status": "pending",
  "page_count": null,
  "credits_used": null,
  "started_at": null,
  "completed_at": null,
  "duration_ms": null,
  "error": null
}
```

On a terminal state `status` is `completed` or `failed`. On `completed` the result fields are
filled (same set as the webhook body below). On `failed`, `error` is set and the result fields
stay null. 404 if the `id` is unknown for this account. There is no `markdown_url`: the SDK
resolves the result itself (BYO from the disk by `output_path`; managed via the markdown endpoint
by `id`).

### GET `/v1/parse/{id}/markdown` (managed read)

Returns the stored markdown (`Content-Type: text/markdown`) for managed-mode requests. Used by
`->markdown()` when no disk is configured. Account-scoped; 404 after retention (~1 day) or for a
BYO request (BYO results never land in the managed bucket). For BYO, `->markdown()` never calls
this; it reads the customer disk.

### GET `/v1/ping`

```json
{ "ok": true, "plan": "starter" }
```

Backs `php artisan parse:ping`. 401 on a bad key.

---

## The customer webhook (SaaS to SDK)

Only under `delivery.mode = webhook` (production). The SaaS POSTs to the `callback_url` the SDK
supplied at submit.

**Headers:** `X-Parse-Signature: t=<unix_ts>,v1=<hex hmac-sha256>` where the signed string is
`"<t>.<raw_body>"` and the key is the account `whsec_...` (Stripe-style).

**Body (success):**

```json
{
  "id": "9f1c0b6e-...-uuid",
  "status": "completed",
  "page_count": 12,
  "credits_used": 12,
  "started_at": "2026-06-17T12:34:48Z",
  "completed_at": "2026-06-17T12:34:56Z",
  "duration_ms": 8430,
  "error": null
}
```

**Body (failure):** `status: "failed"`, the result fields null, and
`error: { "type": "...", "message": "..." }` using the typed errors below.

Result fields:

- `page_count` is the per-type count actually parsed (the billable quantity).
- `credits_used` is the credits charged for the request. In v1, 1 credit = 1 page, so this
  equals `page_count`; it is a separate field so premium pricing (OCR / large / high-accuracy)
  can diverge later without changing the payload shape.
- `started_at` / `completed_at` are ISO 8601 UTC timestamps for when the parse began and
  finished. `duration_ms` is the parse wall-clock (`completed_at - started_at`), provided for
  convenience. The SDK already holds the submit time on the row (`created_at`), so the customer
  can also see queue wait (`started_at - created_at`).

These fields are persisted on the `parse_requests` row and exposed on the `ParseRequest` (e.g.
`$request->page_count`, `$request->credits_used`, `$request->started_at`,
`$request->completed_at`, `$request->duration_ms`).

**SDK behavior on receipt:** verify the signature (reject if `t` is older than a 5-minute
tolerance window, to block replay); look up the `parse_requests` row by `id`; if already
terminal, ack and stop (idempotent); otherwise update the row and fire the event. Like the status
endpoint, no markdown body: the SDK resolves markdown by mode. The webhook and the poll job never
run together (delivery is one mode per environment).

### Async failure `error.type` taxonomy

The values the SaaS sends in `error.type` on a `failed` result (surfaced to the customer as
`$request->error`):

| `type` | Meaning |
|:--|:--|
| `content_mismatch` | extension said one thing, the bytes are another |
| `corrupt` | the file could not be opened |
| `too_large` | the file exceeded the size cap for its type |
| `timeout` | the parse ran past its time budget |
| `parse_error` | a parse failure not covered above |

(Option/format mistakes are caught synchronously at submit as `unsupported_option`, so they do
not appear here in normal use.)

## The options object

Canonical. SDK methods map to it; the SaaS validates applicability at submit.

| Wire field | SDK method | Applies to | If sent off-route |
|:--|:--|:--|:--|
| `force_ocr` (bool) | `->ocr(true)` | pdf | ignored |
| `pages` (string, e.g. `"1-20"`) | `->pages('1-20')` | pdf (pages), pptx (slides), spreadsheet (sheets) | rejected at submit (`unsupported_option`) on docx/email |
| `frontmatter` (bool) | `->frontmatter(true)` | all | n/a |
| `ocr_language` (string or null) | `->ocrLanguage('spa')` | pdf | ignored; null = default set |

## page_count and billing semantics

`page_count` is the per-type count the SaaS meters into `credits_used` (1 credit = 1 page; see
`CLAUDE.md`): PDF pages, pptx slides, spreadsheet sheets, 1 for email, and Word's estimate for
docx (fallback 1). When `pages` narrows the range, `page_count` is the count actually parsed, and
frontmatter (if requested) carries the document total. The per-format page-equivalence rule for
non-paginated formats is open as **G5 / O1**. Metering finalizes on completion, not at submit.

## Result resolution: `->markdown()`

- **BYO:** `Storage::disk($parse->disk)->get($parse->output_path)`. No API call.
- **Managed:** `GET /v1/parse/{id}/markdown` with the API key.

The SDK picks the path from the row (`disk` null => managed). The output is already in place by
the time the event fires; the event carries metadata, never the markdown body.

## Reliability: terminal-state guarantee

- **Poll (local):** the poll job has a capped attempt count / TTL; if a result never lands it
  fires `ParseFailed` so the event always eventually arrives.
- **Webhook (prod):** the SaaS reaps a request that has been pending past its type's expected
  time budget plus a 15-minute margin, marking it failed with `error.type = timeout` and firing
  the customer webhook, so a stalled parse cannot leave a request `pending` forever (resolves G3).

---

## Decisions (resolved June 2026)

1. Base URL `https://parseforartisans.com`, all paths under `/v1`.
2. Delivery and `callback_url` travel per request in the submit payload (no dashboard webhook
   step). SSRF rule: https only, reject private/loopback/link-local/metadata hosts, no redirects.
3. No `markdown_url` in status or webhook payloads; the SDK resolves markdown by mode.
4. Signature `X-Parse-Signature: t=<ts>,v1=<hmac>` over `"<t>.<body>"`, 5-minute replay window.
5. `ocr_language` is exposed in the SDK as `->ocrLanguage()`.
6. Result payload carries `page_count`, `credits_used`, `started_at`, `completed_at`,
   `duration_ms`, persisted on the `parse_requests` row.
7. Validation is server-side. The SaaS rejects bad option/extension combos synchronously at
   submit (`unsupported_option`); the SDK does not pre-validate (resolves G6).
8. Webhook reaper: per-type time budget plus 15-minute margin, `error.type = timeout` (resolves G3).

## What this resolves

Closes **G1** (complete submit schema), **G2** (managed submit via M1 + default-disk source),
**G3** (webhook reaper), **G6** (server-side option validation at submit), and **G7**
(account-scoped `id`). Remaining: **G5 / O1** (per-format page-equivalence), and the DX items in
`final_improvements.md`. Note: the `parse_requests` table in `architecture.md` needs the new
result columns (`credits_used`, `started_at`, `duration_ms`) added when that doc is updated.
