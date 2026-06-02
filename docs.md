# Parse for Artisans

Turn PDFs, Word docs, spreadsheets, and more into clean Markdown — with one method call.
Built for Laravel.

```php
$markdown = $request->file('document')->markdown();
```

That's the whole thing. Upload a file, get Markdown back.

---

## Installation

### 1. Sign up & get your API key

Create an account at [parseforartisans.com](https://parseforartisans.com), add billing,
and grab your API key from the dashboard. Keys look like:

```
pfa_live_a1b2c3d4e5f6g7h8i9j0
```

Keep it secret — it's the only credential you need.

### 2. Install the package

```bash
composer require parseforartisans/laravel
```

### 3. Add your key

Add the key to your `.env`:

```env
PARSE_API_KEY=pfa_live_a1b2c3d4e5f6g7h8i9j0
```

That's it — the package auto-registers. No config file required for the basics. To publish
the config (timeouts, default disk, async webhook route) run:

```bash
php artisan vendor:publish --tag=parse-config
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

Parse a real file from the command line to see the Markdown:

```bash
php artisan parse:file storage/app/contract.pdf
```

Add `--save=out.md` to write the result to a file instead of printing it.

---

## Usage

### The simplest case

You have a file upload in a request. Call `->markdown()` on it:

```php
use Illuminate\Http\Request;

public function store(Request $request)
{
    $markdown = $request->file('document')->markdown();

    return response($markdown)->header('Content-Type', 'text/markdown');
}
```

This is the **sync flow**: you post a file, you get the Markdown back in the same request.
Good for files up to **20 MB** (and a single document a user is waiting on).

### Parse a file from disk

Not from an upload? Use the `Parse` facade. It takes a path or an `UploadedFile`:

```php
use ParseForArtisans\Facades\Parse;

$markdown = Parse::file(storage_path('app/contract.pdf'))->markdown();
```

### Validate the upload first

Standard Laravel validation — nothing special:

```php
$request->validate([
    'document' => ['required', 'file', 'mimes:pdf,docx,xlsx', 'max:20480'], // 20 MB
]);

$markdown = $request->file('document')->markdown();
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

The sync flow blocks until the parse finishes, so it's capped (e.g. **20 MB**). For large
documents (up to **100 MB**) or when you're parsing hundreds of files at once, use the
**async flow**: dispatch the job, and we POST the result to your webhook when it's done.
No polling.

```php
Parse::file($path)->queue(webhook: route('parsed'));
```

Handle the result in your app. The package ships a verified-webhook helper:

```php
use ParseForArtisans\Http\ParseWebhook;

Route::post('/parsed', function (ParseWebhook $result) {
    // signature already verified
    $result->id;        // your job id
    $result->markdown;  // the parsed Markdown
})->name('parsed');
```

See [architecture.md](architecture.md) for how the async flow works end to end.

---

## Use it with the Laravel AI SDK

Parsed Markdown drops straight into a prompt:

```php
$markdown = $request->file('contract')->markdown();

$summary = Prism::text()
    ->using('anthropic', 'claude-opus-4-8')
    ->withPrompt("Summarize this contract:\n\n{$markdown}")
    ->generate();
```

---

## Limits & errors

| | Sync | Async |
|---|---|---|
| Max file size | 20 MB | 100 MB |
| Returns | Markdown string, in the response | Webhook callback |
| Best for | One file, user waiting | Large files, bulk imports |

Failures throw `ParseForArtisans\Exceptions\ParseException` with a readable message
(unsupported type, file too large, parse error, quota exceeded).

> Size limits above are examples and not yet final.
