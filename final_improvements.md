# Final Improvements: Pre-Implementation Review and Checklist

A pre-build sanity check of the Parse for Artisans architecture and docs. The backend
(`../modal`) is built and deployed; the SDK and SaaS are not yet written. This doc lists what
to settle before implementation planning, plus a running checklist to work through across
sessions.

## How to use this doc

Each item has an ID (X = contradiction, G = spec gap, D = DX, O = known-open, N = next
artifact). Check items off as they are resolved. When an item is decided, record the decision
inline under the item so the reasoning survives.

## Verdict

The architecture is coherent and the data flow closes end to end for the two main paths (prod
BYO + webhook, local BYO + poll). Nothing here is architecture-breaking. The work before
building is to freeze the SDK to SaaS wire contract and resolve the managed-bucket submit path,
because everything else hangs off those two. Status: ready to plan, not yet ready to build.

---

## A. Contradictions to fix (doc-level, cheap)

- [x] **X1. Default output path contradicts itself.** **Decided: append `.md` to the full source
  filename** (`contracts/foo.pdf` becomes `parsed/contracts/foo.pdf.md`), which is collision-safe
  when two files differ only by extension (replace-the-extension would map `report.pdf` and
  `report.docx` to the same `parsed/report.md`). Fixed in `docs/parsing-documents.md` and
  `sdk-saas-contract.md`.
- [x] **X2. Stale "greenfield" line.** Fixed: `architecture.md` now describes `modal` as built,
  deployed and smoke-tested, pointing at `../modal/CLAUDE.md`.
- [x] **X3. Broken cross-reference.** Fixed: `../modal/CLAUDE.md` now points to
  `../architecture/docs/README.md`.

---

## B. Specification gaps to close before/early in implementation

These sit on the load-bearing interfaces. None are hard, but the SDK and SaaS cannot be built
without them.

- [x] **G1. The SDK to SaaS submit payload is incomplete.** **Resolved** by the full submit
  schema in `sdk-saas-contract.md` (explicit `extension`, `options`, `delivery`, BYO vs managed
  shape). The documented body is
  `{id, file_url, upload_url}` (`architecture.md:94`), but it omits two things that must travel:
  (a) the options (`force_ocr`, `pages`, `frontmatter`) that the SaaS forwards to Modal, and
  (b) a `filename`/`extension` field. The SaaS routes by extension, and parsing it out of a
  presigned URL path is fragile (encoding, query strings). Freeze a complete submit schema with
  an explicit extension and an options object. This is the single most important interface in
  the system. Tracked as deliverable N1.
- [x] **G2. The managed-bucket submit path is undefined**, and it is the default first-run
  experience. **Decided (June 2026): M1** (SDK uploads bytes to the SaaS at submit; SaaS presigns
  on its own managed bucket) and the source path resolves against Laravel's **default disk**
  (`config('filesystems.default')`) when `parse.disk` is unset. Specified in `sdk-saas-contract.md`.
- [x] **G3. No reaper on the webhook (prod) delivery path.** **Decided:** the SaaS reaps a request
  pending past its type's expected time budget plus a 15-minute margin, marking it failed with
  `error.type = timeout` and firing the customer webhook. Specified in `sdk-saas-contract.md`.
- [x] **G4. Testing story for SDK consumers.** **Decided: no bespoke fake in v1** (avoid
  overengineering). Consumers test their own code with Laravel built-ins: `Http::fake()` stubs the
  submit call, `Storage::fake()` backs `->markdown()` on the BYO path, and the public
  `ParseCompleted` / `ParseFailed` events can be dispatched directly in a test to drive a listener.
  Revisit a dedicated `Parse::fake()` only if customers ask. Optional later: a short "Testing"
  recipe in `docs/handling-results.md` showing the built-in approach.
- [x] **G5. Per-format page-equivalence rule for billing. Finalized** in
  `../pricing-and-cost/credit-definition.md`: formats with real pages (PDF pages, PPTX slides)
  bill those directly; non-paginated formats bill by extracted-text volume (min 1, on the
  emitted markdown with table formatting stripped) at two thresholds: prose (docx, email) one
  page per 3,000 chars, grids (xlsx, csv) one page per 10,000 chars. This replaces the
  inconsistent raw counts (docx fallback-1, email always-1, xlsx sheet count). The SaaS metering
  code follows that rule; user-facing version at `/docs/credit-definition`. Premium credit cost
  stays deferred (see O1).
- [x] **G6. `->pages()` on docx/email fails async.** **Decided:** validation is server-side, and
  the SaaS rejects bad option/extension combos **synchronously at submit** (`unsupported_option`)
  rather than letting them fail async. The SDK does not pre-validate, so the allowlist and option
  rules evolve server-side without an SDK release. This also upgrades the docx-`pages` case from an
  async `ParseFailed` to a synchronous `ParseException`. Specified in `sdk-saas-contract.md`.
- [x] **G7. Account-scope the client-supplied `id` on the SaaS.** **Resolved** in
  `sdk-saas-contract.md`: the job is keyed on `(account, id)` and a lookup for an `id` outside the
  calling account returns 404. The SDK mints the uuid and the SaaS keys its job on it; the job
  lookup and `GET /v1/parse/{id}` are scoped to the calling account.
- [ ] **G8. Correlation token from Modal.** Modal's webhook returns no job id, only the echoed
  `webhook_secret`. Do not use the secret as the lookup key. Embed the SaaS job id in the
  per-request `webhook_url` path and keep `webhook_secret` for auth only. Record this in the
  contract so it is not rediscovered mid-build. (SaaS-to-backend concern; lives in the backend/
  architecture docs, not the SDK-to-SaaS contract.)
- [x] **G9. Result metadata + `parse_requests` columns.** Done: the `parse_requests` table in
  `architecture.md` now includes `credits_used`, `started_at`, and `duration_ms` alongside
  `page_count` / `completed_at`, matching the result payload in `sdk-saas-contract.md`.

---

## C. DX improvements (make it easy for a Laravel dev)

- [x] **D1. No inline result path in code.** **Decided: add `->wait()` on the `ParseRequest`
  handle** as the synchronous escape hatch for CLI / tinker / one-off scripts (never a web request,
  it blocks). Signature `wait(int $timeout = 120, int $interval = 2): ParseRequest`. It polls
  `GET /v1/parse/{id}` itself, so it needs no queue worker or poll job, updates the local row, and
  on a terminal status returns the handle for chaining (`->wait()->markdown()`). Failure surfaces
  as a thrown `ParseFailedException` and a timeout as `ParseTimeoutException` (sync throws, async
  delivers the event, the same split as submission errors). `php artisan parse:file` uses it
  internally. Optional follow-up: show `->wait()` in the tinker example in `docs/introduction.md`
  so the quick-start returns a visible result.
- [ ] **D2. `->status()` silently stuck without a local worker.** Documented, but it will be the
  most common support question. Inherent to poll delivery. Consider having `parse:ping` or the
  poll job surface a clear hint when no queue worker appears to be running. Low priority.
- [x] **D3. Freeze the option set and remove the "not final" disclaimer.** Done in
  `docs/parsing-documents.md`: dropped "treat this list as the direction, not a contract," added
  `->ocrLanguage()`, and replaced the disclaimer with a per-format applicability note. Left
  `docs/handling-results.md:126` ("size/quota numbers are examples") in place: those numbers are
  genuinely not final (worker sizing pending the 1 GB run, O2; quota pending pricing).
- [x] **D4. `ocr_language` has no SDK surface.** **Decided:** expose it as `->ocrLanguage('spa')`
  in v1. Specified in `sdk-saas-contract.md`.
- [x] **D5. Verify the Laravel AI SDK example.** Verified against the official docs
  (`laravel.com/docs/ai-sdk`). The old example was wrong: there is no `Illuminate\Support\Facades\AI`
  facade and no `AI::text()->text()` chain. Fixed `docs/handling-results.md` to the official API:
  `use function Laravel\Ai\agent;` then `agent()->prompt("...")->text`.

---

## D. Known-open decisions (track, do not block planning)

Already acknowledged as open in `CLAUDE.md`/`architecture.md`. Listed so they are not lost.

- [~] **O1. Page-equivalents for non-paginated formats and premium credit cost** (OCR / large /
  high-accuracy above 1:1). Page-equivalence **finalized** (see G5 and
  `../pricing-and-cost/credit-definition.md`): prose 3,000 chars/page, grids 10,000 chars/page.
  Premium credit cost (above 1:1) is the only half still deferred.
- [ ] **O2. Worker resource numbers and the 1 GB validation run.** Provisional sizing is in
  `../modal/CLAUDE.md`; the pre-launch 1 GB PDF end-to-end run is the one remaining backend task
  and sets the final memory/timeout/disk numbers.
- [ ] **O3. Which backend gaps to close** (Office fidelity via LibreOffice for complex modern
  docs, standalone image OCR, structured/coordinate output). Demand-driven roadmap.
- [ ] **O4. PyMuPDF / `pymupdf4llm` AGPL licensing.** Fine during the free phase; budget an
  Artifex commercial license before charging. Real blocker for monetization, not for building.

---

## The managed-bucket question (active discussion)

Context: the review flagged "the SDK has no bucket credentials, so it cannot presign." That is
true only for the managed dev bucket, not for BYO. Clarification:

- **BYO (`parse.disk` set):** the disk is a normal Laravel filesystem disk whose credentials
  already live in the customer's `config/filesystems.php`. The SDK presigns GET (`file_url`) and
  PUT (`upload_url`) against that disk. No new credential config is introduced; we reuse the disk
  the customer already has. This is the case "set in config."
- **Managed (`parse.disk` unset):** there is no bucket and no credentials by design (the whole
  point is "no bucket to start"). The managed bucket is hosted by the SaaS, and the customer's
  app does not, and should not, hold write credentials to a shared bucket. So the SDK genuinely
  cannot presign in this mode.

**Invariant to preserve:** Modal always receives presigned `{file_url, upload_url}` and is
bucket-agnostic. The only thing that differs between modes is *who presigns* (the SDK for BYO,
the SaaS for managed) and *how `->markdown()` reads* (disk directly for BYO, SaaS API for
managed). Keep that invariant and the Modal contract never changes.

Two sub-decisions to make:

1. **How bytes reach the managed bucket (the submit mechanism).**
   - **Option M1 (recommended for v1): SDK uploads bytes to the SaaS at submit.** SDK reads the
     local source bytes and sends them to the SaaS (multipart, or a small pre-step). The SaaS
     stores them in the managed bucket, presigns GET + PUT on that bucket, and routes to Modal.
     Simplest; one extra hop. Managed is dev-only, small files, quota-limited, so the SaaS
     proxying bytes is a non-issue at this scale.
   - **Option M2 (defer): SaaS issues a presigned PUT to the SDK.** SDK asks the SaaS for an
     upload slot, gets back a presigned PUT plus the `file_url`/`upload_url` the SaaS will use,
     PUTs bytes directly to the managed bucket, then submits. Cleaner at scale (bytes skip the
     SaaS app), more round trips. Managed is not meant to scale, so this is not needed for v1.
   - **Option M3 (over-engineered for v1): ship a Flysystem adapter for the managed bucket.**
     Makes `parse.disk` always "set" (the managed disk talks to our API), unifying the SDK code
     path. Elegant but it is a real component to build and maintain; presigning still does not
     apply. Park it.
   - [x] **Decision: M1** (June 2026). SDK uploads local bytes to the SaaS at submit; the SaaS
     stores them in the managed bucket and presigns GET + PUT for Modal. M2 is the documented
     scale path if managed ever needs it; M3 parked.

2. **Which disk the source path resolves against in managed mode.** When `parse.disk` is unset,
   `Parse::file('contracts/foo.pdf')` must still read bytes from somewhere local. Options:
   fall back to Laravel's default disk (`config('filesystems.default')`), or treat the path
   against a documented default. Decide and document, since this is the literal first-run path.
   - [x] **Decision: default disk** (June 2026). When `parse.disk` is unset, the source path
     resolves against Laravel's default disk (`config('filesystems.default')`). Documented in
     `sdk-saas-contract.md`.

---

## N. Next artifact

- [x] **N1. Write the SDK to SaaS wire contract doc** (the API, no code yet). Drafted as
  `sdk-saas-contract.md`. Several choices in it are marked "Confirm" and need ratification. One document that
  freezes: the submit request (complete schema, incl. extension + options + the managed vs BYO
  shape), the `202` response, the status response (`GET /v1/parse/{id}`), the customer-webhook
  payload and signing, the error shapes (synchronous `ParseException` vs async `ParseFailed`),
  and how `->markdown()` reads in each storage mode. Resolves G1, G2, G7, G8 in one place. The
  backend half is already frozen in `../modal/CLAUDE.md`; this is the missing mirror. Create
  after the managed-bucket decision above.
