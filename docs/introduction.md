# Introduction

Turn PDFs, Word docs, spreadsheets, and more into clean Markdown, built for Laravel.

```php
use ParseForArtisans\Facades\Parse;

$parse = Parse::file('contract.pdf')->for($document)->parse();
$parse->status();  // 'pending', 'completed'
```

The result arrives via a Laravel event when it's ready. `->for($model)` ties the job to one of
your records, so the event hands that record straight back, with no ids to track. The SDK
publishes a listener that you can customize
```php
// app/Listeners/HandleParsedDocument.php:
use ParseForArtisans\Events\ParseCompleted;

class HandleParsedDocument
{
    // This event fires when parsing is finished
    public function handle(ParseCompleted $event): void
    {
        $document = $event->request->parsable;        // the model you passed to ->for()
        $document->update(['markdown' => $event->request->markdown()]);
        // ...store it, index it, send it to an LLM
    }
}
```

---

## How it works

1. **`->parse()`** hands us the job and returns a `ParseRequest` handle right away.
2. We parse the document and write the Markdown straight to your bucket (or our managed dev
   bucket if you haven't set one up).
3. We ping a webhook and you get a **`ParseCompleted`** Laravel event. The Markdown is already
   ready.

Our SDK handles webhooks, secrets, presigned URLs.

> For local development, if you're running a queue (**`composer dev`** works) it will just work.
> No need to set up webhook endpoints etc.

---

## What it supports

You call `->parse()` on the file; we detect the type and route it for you, with nothing to
configure per format:

- **PDF** (with automatic OCR for scanned files)
- **Word** (`.docx`, `.doc`)
- **PowerPoint** (`.pptx`, `.ppt`)
- **Spreadsheet** (`.xlsx`, `.xls`, `.csv`)
- **Email** (`.eml`, `.msg`)

See [Supported formats](supported-formats.md) for the full list, what each produces, and the
parsing features (multi-column layouts, tables, frontmatter, page ranges, and more).
