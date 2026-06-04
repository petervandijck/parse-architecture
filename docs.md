# Parse for Artisans

Turn PDFs, Word docs, spreadsheets, and more into clean Markdown — built for Laravel.

```php
use ParseForArtisans\Facades\Parse;

// Submit a file. Returns immediately with a handle — parsing runs in the background.
// Works on any format; we detect the type. (Try playbook.docx, revenue.xlsx, ...)
$document = Parse::file('contract.pdf')->parse();

$document->id;        // a parse reference (uuid) the SDK tracks for you
$document->status();  // 'pending'
```

```php
// The result arrives via a Laravel event when it's ready — write a listener.
use ParseForArtisans\Events\ParseCompleted;

class StoreParsedDocument
{
    public function handle(ParseCompleted $event): void
    {
        $markdown = $event->request->markdown();
        // ...store it, index it, send it to an LLM
    }
}
```

That's the whole shape: **`->parse()` to submit, a `ParseCompleted` event when the Markdown is
ready.** Parsing a real document takes anywhere from a few seconds to a few minutes, so it
never blocks your app — you hand us the file and get on with the request.

Point it at one file or tens of thousands; the same two steps handle both.

```php
// A whole batch — one handle per file, each fires its own event as it finishes.
$paths = Storage::disk('s3')->files('contracts');   // ['contracts/1.pdf', 'contracts/2.docx', ...]
$batch = Parse::disk('s3')->files($paths)->parse();
```

> **Need to show a user the result while they wait?** Don't block the request — submit with
> `->parse()` and poll [`->status()`](#checking-status) from your frontend until it flips to
> `completed`, then display the Markdown.

---

## What it supports

You call `->parse()` on anything below — we detect the type and route it for you. There's
nothing to configure per format.

| Format | Extensions | What you get |
|:--|:--|:--|
| **PDF** | `.pdf` | Clean Markdown. **Scanned PDFs are OCR'd automatically** — no flag needed. |
| **Word** | `.docx`, `.doc` | Markdown with headings, lists, and tables preserved. |
| **PowerPoint** | `.pptx`, `.ppt` | One `## Slide N` section per slide, including slide tables. |
| **Spreadsheet** | `.xlsx`, `.csv` | Each sheet (or the CSV) rendered as a Markdown table, one `## Sheet` per tab. |
| **Email** | `.eml`, `.msg` | Headers (from/to/subject/date) as YAML frontmatter, body text, and an attachment list. |

Highlights:

- **Automatic OCR.** Scanned or image-only PDFs are detected and run through OCR — you don't
  pick a mode.
- **Multi-column layouts.** Two- and multi-column pages (think scientific papers) are read in
  the correct reading order, not jumbled left-to-right across columns.
- **Tables.** Tables are detected and emitted as proper Markdown tables.
- **Document structure.** Headings, lists, bold/italic, and code blocks come through as real
  Markdown — not a flat wall of text.
- **Hyperlinks preserved.** Links in the source stay as Markdown links in the output.
- **Email fields & attachments.** Email comes back with from/to/subject/date as frontmatter,
  the body, and a list of attachments (name + size).
- **Type auto-detection.** You never pass a format; we sniff it from the file.
- **Optional frontmatter.** Add `->frontmatter()` to prepend YAML metadata (author, dates,
  page/slide/sheet counts).
- **Page ranges.** Parse just the pages you want with `->pages('1-20')`.
- **Large files.** Documents up to **1 GB** each.
- **Massive bulk.** Queue **tens of thousands of files at once** — the backend scales out to
  absorb the burst and writes each result straight back to your bucket.
- **One consistent format.** Every input type — PDF, Word, Excel, email — comes out as the
  same clean Markdown.
- **Private by design.** With your own bucket, your file bytes never pass through us: your
  bucket in, your bucket out.

> Legacy `.doc` / `.ppt` (the pre-2007 binary formats) are supported alongside the modern
> `.docx` / `.pptx`. More formats are on the way.

---

## How it works

One flow, start to finish:

1. **`->parse()`** hands us the job and returns a `ParseRequest` handle right away. It's a
   small, fast API call — **you can call it inline in a controller; you don't need Laravel's
   queue.**
2. We parse the document and write the Markdown **straight to your bucket** (or our managed
   dev bucket — see [Local development](#local-development)).
3. When it's done, you get a **`ParseCompleted` Laravel event**. The Markdown is already
   waiting; the event just tells you it's ready and hands you the `ParseRequest`.

You never juggle ids, secrets, presigned URLs, or webhook routes — the SDK owns all of it.

---

## Installation

### 1. Sign up & get your keys

Create an account at [parseforartisans.com](https://parseforartisans.com), add billing, and
grab two values from the dashboard: your **API key** and your **webhook signing secret**.

```
pfa_...      # API key
whsec_...    # webhook signing secret (verifies result callbacks)
```

Keep them secret.

### 2. Install the package

```bash
composer require parseforartisans/laravel
```

### 3. Add your keys

```env
PARSE_API_KEY=pfa_...
PARSE_WEBHOOK_SECRET=whsec_...
```

### 4. Publish config & migrate

```bash
php artisan vendor:publish --tag=parse-config
php artisan migrate
```

The migration adds a small `parse_requests` table the SDK uses to track submissions and match
results back to them — so you never handle ids or secrets yourself.

`config/parse.php`:

```php
'disk'     => env('PARSE_DISK'),               // your bucket. Leave unset to use our managed dev bucket.
'output'   => 'parsed',                        // prefix where Markdown is written (default: parsed/)
'delivery' => env('PARSE_DELIVERY', 'auto'),   // auto | webhook | poll  (see below)
```

**Delivery** controls how the result reaches your app:

| Mode | What happens |
|:--|:--|
| `auto` *(default)* | `poll` only when `APP_ENV=local`; `webhook` everywhere else (staging, production, …) — keyed off `APP_ENV`. |
| `webhook` | We POST a signed callback to the SDK's route; it fires the event. No polling. |
| `poll` | The SDK checks for the result on your queue and fires the same event. For local dev, where we can't reach an inbound webhook. See [Local development](#local-development). |

`auto` means everything but your laptop uses webhooks; local just works — you rarely set this by hand.

---

## Test your setup

Confirm your key works and the service is reachable:

```bash
php artisan parse:ping
```

```
✔ Connected to parseforartisans.com
✔ API key valid (plan: starter)
✔ Ready to parse
```

Parse a file from the command line to see the Markdown (the command submits, waits, and prints
the result for you). No PDF handy? Point it at a public URL:

```bash
php artisan parse:file https://parseforartisans.com/samples/invoice.pdf
```

Or a path on your disk:

```bash
php artisan parse:file contracts/contract.pdf
```

Add `--save=out.md` to write the result to a file instead of printing it.

---

## Submitting documents

Use the `Parse` facade. It takes a path on a Laravel filesystem disk (resolved like
`Storage::get()`, so you never hand-build an OS path):

```php
use ParseForArtisans\Facades\Parse;

$document = Parse::file('contracts/foo.pdf')->parse();           // default disk
$document = Parse::disk('s3')->file('contracts/foo.pdf')->parse();
```

Add per-request options and your own context:

```php
$document = Parse::file('contracts/foo.pdf')
    ->to('contracts/foo.md')                    // optional: defaults to parsed/contracts/foo.pdf.md
    ->withMeta(['invoice_id' => $invoice->id])  // optional: your context, handed back in the event
    ->parse();

$document->id;        // uuid
$document->status();  // 'pending'
```

By default the output mirrors the source path under `parsed/` — `contracts/foo.pdf` →
`parsed/contracts/foo.md`. Override per-file with `->to(...)`.

### Parse a URL

Already have the document at a public URL? Skip the upload — we fetch it. Try it right now in
`php artisan tinker` with our sample file:

```php
$document = Parse::url('https://parseforartisans.com/samples/invoice.pdf')->parse();
```

### From a file upload

There's no inline Markdown to return, and parsing happens in your bucket, so the idiomatic
Laravel move is **store the upload, then parse the stored path**:

```php
use Illuminate\Http\Request;

public function store(Request $request)
{
    $request->validate([
        'document' => ['required', 'file', 'mimes:pdf,docx,xlsx', 'max:51200'],
    ]);

    $path = $request->file('document')->store('uploads', 's3');  // your disk
    $doc  = Parse::disk('s3')->file($path)->parse();

    // Hand the id to your frontend so it can poll ->status(), or just wait for the event.
    return response()->json(['parse_id' => $doc->id]);
}
```

You don't tell us the file type — we detect it (PDF, Word, PowerPoint, Excel, email, and more)
and route it automatically.

### Parse a whole batch

Hand `Parse::files()` an array (or collection) of paths. They're submitted as one batch and
you get back a collection of `ParseRequest` models — one per file:

```php
$paths = Storage::disk('s3')->files('contracts');

$batch = Parse::disk('s3')->files($paths)
    ->frontmatter(true)        // options apply to every file in the batch
    ->parse();

$batch->count();               // how many were submitted
$batch->pluck('id');           // the parse references the SDK is tracking
```

Each file fires its own `ParseCompleted` event as it finishes — so you handle results the same
way whether you submitted one file or ten thousand.

> Submitting **thousands** at once? Wrap the `->parse()` call in your own queued job so the
> submissions run in the background with retries. The SDK doesn't force a queue; it's your call.

---

## Handling the result

The package registers a signed webhook route, verifies the callback, matches it back to your
`ParseRequest`, and fires a Laravel event — write a listener, not a controller:

```php
use ParseForArtisans\Events\ParseCompleted;

class StoreParsedDocument
{
    public function handle(ParseCompleted $event): void
    {
        $request = $event->request;          // the ParseRequest, now 'completed'
        $request->meta['invoice_id'];        // your context, reunited
        $request->page_count;

        $markdown = $request->markdown();    // reads from your bucket (or fetches the managed copy)

        // ...store, index, summarize
    }
}
```

`ParseFailed` fires on errors, carrying `$request->error`. Register both like any Laravel event
listener.

> `$request->markdown()` reads the result from your bucket. You can also go straight to the
> file yourself: `Storage::disk($request->disk)->get($request->output_path)`.

---

## Checking status

The Markdown is always delivered by the [event](#handling-the-result). But you'll often want to
show the *end user* where their document is — "still parsing", "done", or "something went
wrong". That's what `->status()` is for.

It reads the local `parse_requests` row, which the SDK keeps current as results come in (the
webhook updates it in production; the background poll does locally). It's a plain DB read — no
API call — so it's cheap to hit on every page load:

```php
$document = Parse::find($id);   // look up the row by id

$label = match ($document->status()) {
    'pending'   => 'Still parsing…',
    'completed' => 'Done',
    'failed'    => 'Something went wrong: ' . $document->error,
};
```

Drive a progress badge or a Livewire/polling spinner with it. `->status()` only tells you *where
the job is* — it never fetches the Markdown. The result still arrives through the
`ParseCompleted` event.

---

## Local development

Two things differ on a laptop, and the SDK handles both so your code stays identical to
production:

**1. No bucket? Use ours.** Leave `parse.disk` unset and we write results to a **managed dev
bucket** — zero setup, nothing to configure. It's quota-limited (sized for development, not
production) and results are kept for **~1 day**, so fetch what you need promptly. For
production, set `PARSE_DISK` to your own bucket; your bytes then never transit us.

**2. No inbound webhook? We poll instead.** We can't reach a webhook on `localhost`, so with
`delivery=auto` your local environment uses **`poll`**: `->parse()` quietly dispatches a small
background job that checks our API, then updates the `parse_requests` row and fires the same
`ParseCompleted` event when the result lands. It rides the queue worker you're already running —
if you use Laravel's `composer run dev`, that includes `queue:listen`, so **events just work
locally with nothing extra**.

```bash
composer run dev        # serve + queue:listen + vite — events fire locally
```

> That background poll is also what advances [`->status()`](#checking-status) locally (in
> production the webhook keeps the row current). So **run a worker locally to see status move or
> events fire** — and it needs a real queue driver (`database` or `redis`, the Laravel default),
> not `sync`.

Prefer real webhooks locally for true production parity? Run a tunnel (ngrok, Expose, Herd
Share), point `APP_URL` at it, and set `PARSE_DELIVERY=webhook`.

---

## Options

Chain options before `->parse()`:

```php
$document = Parse::file($path)
    ->ocr(true)              // force OCR (auto-detected by default for scanned PDFs)
    ->pages('1-20')          // only these pages
    ->frontmatter(true)      // prepend YAML frontmatter (author, dates, page count)
    ->parse();
```

> The exact option set is still being finalized — treat this list as the direction, not a
> contract.

---

## Use it with the Laravel AI SDK

Parsed Markdown drops straight into a prompt:

```php
use Illuminate\Support\Facades\AI;

public function handle(ParseCompleted $event): void
{
    $markdown = $event->request->markdown();

    $summary = AI::text("Summarize this contract:\n\n{$markdown}")->text();
}
```

---

## Limits & errors

| Aspect | Your bucket (BYO) | Managed dev bucket |
|:--|:--|:--|
| **Max file size** | up to 1 GB | smaller, dev-sized cap |
| **Total storage** | your own | quota-limited |
| **Result retention** | yours to keep | ~1 day |
| **Best for** | production, bulk | local development |

Results are delivered by **event** (`ParseCompleted` / `ParseFailed`) — parse-time problems
arrive as a `ParseFailed` event carrying `$request->error`, not as a thrown exception. Only
**submission** problems (bad key, unsupported type, quota exceeded) throw
`ParseForArtisans\Exceptions\ParseException` from `->parse()` itself.

> Size and quota numbers above are examples and not yet final.
