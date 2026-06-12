# Parse for Artisans

Turn PDFs, Word docs, spreadsheets, and more into clean Markdown — built for Laravel.

```php
use ParseForArtisans\Facades\Parse;

$document = Parse::file('contract.pdf')->parse();
$document->status();  // 'pending', 'completed'
```

The result arrives via a Laravel event when it's ready. The SDK publishes a listener that you can customize
```php
// app/Listeners/HandleParsedDocument.php:
use ParseForArtisans\Events\ParseCompleted;

class HandleParsedDocument
{
    // This event fires when parsing is finished
    public function handle(ParseCompleted $event): void
    {
        $markdown = $event->request->markdown();
        // ...store it, index it, send it to an LLM
    }
}
```

---

## How it works

1. **`->parse()`** hands us the job and returns a `ParseRequest` handle right away.
2. We parse the document and write the Markdown straight to your bucket.
3. We ping a webhook and you get a **`ParseCompleted`** Laravel event. The Markdown is already
   ready.

Our SDK handles webhooks, secrets, presigned URLs.

> For local development, if you're running a queue (**`composer dev`** works) it will just work.
> No need to set up webhook endpoints etc.

---

## Installation

### 1. Sign up & get your keys

Create an account at [parseforartisans.com](https://parseforartisans.com), and
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

### 4. Run the installer

```bash
php artisan parse:install
```

One command does it all: publishes `config/parse.php`, drops the event listeners into
`app/Listeners/` (`HandleParsedDocument`, `HandleFailedParse`), and runs the migration that adds
a small `parse_requests` table the SDK uses to track submissions and match results back to them
— so you never handle ids or secrets yourself. Customize the listeners later — see
[Handling the result](#handling-the-result).

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

## Handling the result

You never write webhook-handling code. The SDK ships the route, verifies the signed callback,
and matches it back to your `ParseRequest` — then fires a Laravel event. All you write is what
to *do* with a finished document.

And you don't even start from scratch: [`parse:install`](#4-run-the-installer) dropped a
listener into `app/Listeners/HandleParsedDocument.php`. Open it and fill in the body:

```php
<?php

namespace App\Listeners;

use ParseForArtisans\Events\ParseCompleted;

class HandleParsedDocument
{
    /**
     * Create the event listener.
     */
    public function __construct() {}

    /**
     * Handle the event.
     */
    public function handle(ParseCompleted $event): void
    {
        $document = $event->request;          // the ParseRequest, now 'completed'
        $markdown = $document->markdown();    // the parsed Markdown

        // Your logic — store it, index it, send it to an LLM, notify the user…
        // $document->meta['invoice_id'] and $document->page_count are here too.
    }
}
```

On **Laravel 12 and 13 this listener is auto-discovered** — it's live the moment the file
exists, with nothing to register. A companion `HandleFailedParse` stub is published for errors:

```php
<?php

namespace App\Listeners;

use ParseForArtisans\Events\ParseFailed;

class HandleFailedParse
{
    public function handle(ParseFailed $event): void
    {
        report("Parse failed: {$event->request->error}");
        // Your logic — flag the document, alert someone, queue a retry…
    }
}
```

> **Prefer to wire your own?** Skip the publish and write any listener for `ParseCompleted` /
> `ParseFailed` — they're auto-discovered the same way. The published stubs are just a head
> start; once published they're yours, so a later SDK update won't overwrite them.

> `$document->markdown()` reads the result from your bucket (or fetches the managed copy). You
> can also go straight to the file: `Storage::disk($document->disk)->get($document->output_path)`.

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

## Batch processing

Hand `Parse::files()` an array (or collection) of paths instead of a single file. They're
submitted as one batch and you get back a collection of `ParseRequest` models — one per file:

```php
$paths = Storage::disk('s3')->files('contracts');

$batch = Parse::disk('s3')->files($paths)
    ->frontmatter(true)        // options apply to every file in the batch
    ->parse();

$batch->count();               // how many were submitted
$batch->pluck('id');           // the parse references the SDK is tracking
```

Each file fires its own `ParseCompleted` event as it finishes — so your published listener
handles results the same way whether you submitted one file or ten thousand.

> Submitting **thousands** at once? Wrap the `->parse()` call in your own queued job so the
> submissions run in the background with retries. The SDK doesn't force a queue; it's your call.

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

## What it supports

You call `->parse()` on anything below — we detect the type and route it for you. There's
nothing to configure per format.

| Format | Extensions | What you get |
|:--|:--|:--|
| **PDF** | `.pdf` | Clean Markdown. **Scanned PDFs are OCR'd automatically** — no flag needed. |
| **Word** | `.docx`, `.doc` | Markdown with headings, lists, and tables preserved. |
| **PowerPoint** | `.pptx`, `.ppt` | One `## Slide N` section per slide, including slide tables. |
| **Spreadsheet** | `.xlsx`, `.xls`, `.csv` | Each sheet (or the CSV) rendered as a Markdown table, one `## Sheet` per tab. |
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
- **Type auto-detection.** You never pass a format; we detect it and route automatically.
- **Optional frontmatter.** Add `->frontmatter()` to prepend YAML metadata (author, dates,
  page/slide/sheet counts).
- **Page ranges.** Parse just the pages you want with `->pages('1-20')`.
- **Large files.** PDFs up to **1 GB** each; other formats top out lower (see
  [Limits](#limits--errors)).
- **Massive bulk.** Queue **tens of thousands of files at once** — the backend scales out to
  absorb the burst and writes each result straight back to your bucket.
- **One consistent format.** Every input type — PDF, Word, Excel, email — comes out as the
  same clean Markdown.
- **Private by design.** With your own bucket, your file bytes never pass through us: your
  bucket in, your bucket out.

> Legacy `.doc` / `.ppt` / `.xls` (the pre-2007 binary formats) are supported alongside the
> modern `.docx` / `.pptx` / `.xlsx`. More formats are on the way.

---

## Limits & errors

| Aspect | Your bucket (BYO) | Managed dev bucket |
|:--|:--|:--|
| **Max file size** | PDFs up to 1 GB; other formats lower | smaller, dev-sized cap |
| **Total storage** | your own | quota-limited |
| **Result retention** | yours to keep | ~1 day |
| **Best for** | production, bulk | local development |

Results are delivered by **event** (`ParseCompleted` / `ParseFailed`) — parse-time problems
arrive as a `ParseFailed` event carrying `$request->error`, not as a thrown exception. Only
**submission** problems (bad key, unsupported type, quota exceeded) throw
`ParseForArtisans\Exceptions\ParseException` from `->parse()` itself.

> Size and quota numbers above are examples and not yet final.
