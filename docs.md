# Parse for Artisans

Turn PDFs, Word docs, spreadsheets, and more into clean Markdown — with one method call.
Built for Laravel.

```php
// From a file on disk
$markdown = Parse::file('contract.pdf')->markdown();

// From a URL
$markdown = Parse::url('https://parseforartisans.com/samples/invoice.pdf')->markdown();

// With options (pick a disk, force OCR, limit pages, add frontmatter)
$markdown = Parse::disk('s3')->file('contracts/contract.pdf')
    ->ocr(true)
    ->pages('1-20')
    ->frontmatter(true)
    ->markdown();
```

That's the whole thing. Point it at a file or a URL, get Markdown back.

---

## What it supports

You call `->markdown()` on anything below — we detect the type and route it for you. There's
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
- **Large files.** Documents up to **200 MB** via the [async flow](#large-files--bulk-the-async-flow)
  (the sync call is capped at 20 MB).
- **Massive bulk.** Queue **tens of thousands of files at once** — the backend scales out to
  absorb the burst and writes each result straight back to your bucket.
- **One consistent format.** Every input type — PDF, Word, Excel, email — comes out as the
  same clean Markdown.
- **Private by design.** In the async flow your file bytes never pass through us: your bucket
  in, your bucket out.

> Legacy `.doc` / `.ppt` (the pre-2007 binary formats) are supported alongside the modern
> `.docx` / `.pptx`. More formats are on the way.

---

## Installation

### 1. Sign up & get your keys

Create an account at [parseforartisans.com](https://parseforartisans.com), add billing, and
grab two values from the dashboard: your **API key** and your **webhook signing secret**.

```
pfa_...      # API key
whsec_...    # webhook signing secret (used to verify async callbacks)
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

The API key is all you need for the **sync** flow. The webhook secret is used by the **async**
flow to verify incoming callbacks.

### 4. Publish config & migrate

The package auto-registers — no config file required for sync. To use the **async** flow,
publish the config and run the migration (it adds a small `parse_requests` table the SDK uses
to track submissions and match them to webhooks):

```bash
php artisan vendor:publish --tag=parse-config
php artisan migrate
```

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

Parse a file from the command line to see the Markdown. No PDF handy? Point it at a URL —
this one is a public sample, so you can run it as-is:

```bash
php artisan parse:file https://parseforartisans.com/samples/invoice.pdf
```

Or a local file:

```bash
php artisan parse:file storage/app/contract.pdf
```

Add `--save=out.md` to write the result to a file instead of printing it.

---

## Usage

### Parse a file

Use the `Parse` facade. It takes a path on your filesystem disk (or an `UploadedFile`):

```php
use ParseForArtisans\Facades\Parse;

$markdown = Parse::file('contract.pdf')->markdown();           // default disk
$markdown = Parse::disk('s3')->file('contracts/foo.pdf')->markdown();
```

Paths resolve against your configured disk (just like `Storage::get()`), so you never
hand-build an OS path. Absolute filesystem paths and `UploadedFile` instances work too.

### Parse a URL

Already have the document at a public URL? Skip the upload entirely — we fetch it. You can
run this right now in `php artisan tinker` with our sample file — no PDF needed:

```php
use ParseForArtisans\Facades\Parse;

$markdown = Parse::url('https://parseforartisans.com/samples/invoice.pdf')->markdown();
```

Both of these are the **sync flow**: you send a file or URL, you get the Markdown back in the
same request. Good for files up to **20 MB** (and a single document a user is waiting on). No
bucket needed on your end — bring-your-own-bucket is only for the
[async flow](#large-files--bulk-the-async-flow).

### From a file upload

Working with a Laravel 13 file upload? Call `->markdown()` straight on the uploaded file:

```php
use Illuminate\Http\Request;

public function store(Request $request)
{
    $request->validate([
        'document' => ['required', 'file', 'mimes:pdf,docx,xlsx', 'max:20480'], // 20 MB
    ]);

    $markdown = $request->file('document')->markdown();

    return response($markdown)->header('Content-Type', 'text/markdown');
}
```

You don't tell us the file type — we detect it (PDF, Word, PowerPoint, Excel, email, and
more) and route it automatically.

### Options

Chain options before `->markdown()`:

```php
$markdown = Parse::file($path)
    ->ocr(true)              // force OCR (auto-detected by default for scanned PDFs)
    ->pages('1-20')          // only these pages
    ->frontmatter(true)      // prepend YAML frontmatter (author, dates, page count)
    ->markdown();
```

> The exact option set is still being finalized — treat this list as the direction, not a
> contract.

---

## Large files & bulk: the async flow

The sync flow blocks until the parse finishes, so it's capped (e.g. **20 MB**). Most of the time,
you'll want to use the **async flow**. It doesn't block your app, and you can send large amounts of files.

- You ping the service (no need to queue this, it's a quick api call).
- The SDK handles signing URLs, webhook, secrets etc.
- A few minutes later you receive a Laravel event (through the webhook): your markdown is available.

### 1. Point the SDK at your bucket

Make sure you've done the async setup from [Installation](#4-publish-config--migrate)
(`PARSE_WEBHOOK_SECRET` + `php artisan migrate`), then set the disk your documents live on in
`config/parse.php`:

```php
'disk'   => 's3',        // any Laravel filesystem disk
'output' => 'parsed',    // prefix where parsed Markdown is written (default: parsed/)
```

By default the output mirrors the source path under `parsed/` — `contracts/foo.pdf` →
`parsed/contracts/foo.md`. Override per-file with `->to(...)`.

### 2. Submit the file

```php
use ParseForArtisans\Facades\Parse;

$parse = Parse::file('contracts/foo.pdf')   // a path on your parse disk
    ->to('contracts/foo.md')                // optional: defaults to parsed/contracts/foo.md
    ->withMeta(['invoice_id' => $invoice->id])  // optional: your own context, returned later
    ->async();

$parse->id;      // a parse reference (uuid) the SDK tracks for you
$parse->status;  // 'pending'
```

`->async()` is just a small, fast API call (we receive a tiny JSON payload — no file bytes),
so **you can call it inline in a controller — you don't need Laravel's queue.** It returns a
`ParseRequest` model the SDK persists, so you never juggle the id or webhook secret yourself.

> Submitting **thousands** at once? That's the one time to reach for Laravel's queue —
> wrap the `->async()` call in your own job so the submissions run in parallel with retries.
> The SDK doesn't force a queue; it's your call.

#### Parse a whole batch

Got a pile of files? Hand `Parse::files()` an array (or a collection) of paths. They're
submitted as one batch and you get back a collection of `ParseRequest` models — one per file:

```php
use ParseForArtisans\Facades\Parse;

$paths = Storage::disk('s3')->files('contracts');   // ['contracts/a.pdf', 'contracts/b.docx', ...]

$batch = Parse::disk('s3')->files($paths)
    ->frontmatter(true)        // options apply to every file in the batch
    ->async();

$batch->count();               // how many were queued
$batch->pluck('id');           // the parse references the SDK is tracking
```

Each file fires its own `ParseCompleted` event as it finishes (see below) — so you handle
results the same way whether you submitted one file or ten thousand. For very large batches,
dispatch the `Parse::files(...)->async()` call from a queued job so it runs in the background.

### 3. Handle the result

The package registers its own signed webhook route, verifies the callback, matches it back to
your `ParseRequest`, and fires a Laravel event — write a listener, not a controller:

```php
use ParseForArtisans\Events\ParseCompleted;

class StoreParsedDocument
{
    public function handle(ParseCompleted $event): void
    {
        $request = $event->request;            // the ParseRequest, now 'completed'
        $request->meta['invoice_id'];          // your context, reunited
        $request->output_path;                 // 'parsed/contracts/foo.md' on your disk
        $request->page_count;

        $markdown = Storage::disk($request->disk)->get($request->output_path);
        // ...store, index, summarize
    }
}
```

The Markdown is already sitting in your bucket by the time the event fires — the callback
just tells you it's ready. (`ParseFailed` fires on errors, carrying `$request->error`.)

---

## Use it with the Laravel AI SDK

Parsed Markdown drops straight into a prompt:

```php
use Illuminate\Support\Facades\AI;

$markdown = $request->file('contract')->markdown();

$summary = AI::text("Summarize this contract:\n\n{$markdown}")->text();
```

---

## Limits & errors

| Aspect | Sync | Async |
|:--|:--|:--|
| **Max file size** | 20 MB | 200 MB |
| **Returns** | Markdown string, in the response | Webhook callback |
| **Best for** | One file, user waiting | Large files, bulk imports |

Failures throw `ParseForArtisans\Exceptions\ParseException` with a readable message
(unsupported type, file too large, parse error, quota exceeded).

> Size limits above are examples and not yet final.
