# Parsing Documents

Use the `Parse` facade. It takes a path on a Laravel filesystem disk (resolved like
`Storage::get()`, so you never hand-build an OS path):

```php
use ParseForArtisans\Facades\Parse;

$parse = Parse::file('contracts/foo.pdf')->parse();           // default disk
$parse = Parse::disk('s3')->file('contracts/foo.pdf')->parse();
```

Tie the job to one of your models with `->for($model)`. The event hands that record straight
back, so you never store or match an id yourself:

```php
$parse = Parse::file('contracts/foo.pdf')
    ->for($document)                            // associate with your model
    ->to('contracts/foo.md')                    // optional: defaults to parsed/contracts/foo.pdf.md
    ->parse();

$parse->id;        // uuid
$parse->status();  // 'pending'
```

By default the output mirrors the source path under `parsed/` (so `contracts/foo.pdf` becomes
`parsed/contracts/foo.md`). Override per-file with `->to(...)`.

## Correlating results with your models

Parsing is async: the Markdown shows up later, in a `ParseCompleted` event. So you need a way
to know *which* of your records a finished document belongs to. `->for($model)` handles this:
it records the association on the SDK's own `parse_requests` row (a polymorphic relation, so
nothing to add to your schema), and the event gives the model back:

```php
Parse::file('contracts/foo.pdf')->for($document)->parse();

// later, in your listener:
$document = $event->request->parsable;          // your Document model, typed
```

Want the reverse link on your model? Add one line, no migration:

```php
class Document extends Model
{
    public function parse()
    {
        return $this->morphOne(\ParseForArtisans\Models\ParseRequest::class, 'parsable');
    }
}

$document->parse?->status();   // 'completed'
```

**No model to point at?** (a one-off script, a batch of loose files, or a record you only
create *after* parsing). Skip `->for()` and attach your own context with `->withMeta()`:

```php
$parse = Parse::file('contracts/foo.pdf')
    ->withMeta(['tenant_id' => $tenant->id, 'source' => 'bulk-import'])  // any scalars you like
    ->parse();

// in your listener:
$tenantId = $event->request->meta['tenant_id'];
```

`->for()` and `->withMeta()` compose: use the relation for the model, meta for extra context.

## Parse a URL

Already have the document at a public URL? Skip the upload; we fetch it. Try it right now in
`php artisan tinker` with our sample file:

```php
$parse = Parse::url('https://parseforartisans.com/samples/invoice.pdf')->parse();
```

## From a file upload

There's no inline Markdown to return, and parsing happens in your bucket, so the idiomatic
Laravel move is **store the upload, then parse the stored path**:

```php
use Illuminate\Http\Request;

public function store(Request $request)
{
    $request->validate([
        'document' => ['required', 'file', 'mimes:pdf,docx,xlsx', 'max:51200'],
    ]);

    $path     = $request->file('document')->store('uploads', 's3');  // your disk
    $document = Document::create(['source_path' => $path]);
    $parse    = Parse::disk('s3')->file($path)->for($document)->parse();

    // Hand the id to your frontend so it can poll ->status(), or just wait for the event.
    return response()->json(['parse_id' => $parse->id]);
}
```

You don't tell us the file type; we detect it (PDF, Word, PowerPoint, Excel, email, and more)
and route it automatically.

## Batch processing

Hand `Parse::files()` an array (or collection) of paths instead of a single file. They're
submitted as one batch and you get back a collection of `ParseRequest` models, one per file:

```php
$paths = Storage::disk('s3')->files('contracts');

$batch = Parse::disk('s3')->files($paths)
    ->frontmatter(true)        // options apply to every file in the batch
    ->parse();

$batch->count();               // how many were submitted
$batch->pluck('id');           // the parse references the SDK is tracking
```

Each file fires its own `ParseCompleted` event as it finishes, so your published listener
handles results the same way whether you submitted one file or ten thousand.

> Submitting **thousands** at once? Wrap the `->parse()` call in your own queued job so the
> submissions run in the background with retries. The SDK doesn't force a queue; it's your call.

## Options

Chain options before `->parse()`:

```php
$parse = Parse::file($path)
    ->for($document)         // associate with one of your models, handed back in the event
    ->withMeta([...])        // attach arbitrary scalar context, handed back in the event
    ->ocr(true)              // force OCR (auto-detected by default for scanned PDFs)
    ->pages('1-20')          // only these pages
    ->frontmatter(true)      // prepend YAML frontmatter (author, dates, page count)
    ->to('out/foo.md')       // override the output path (default: parsed/<source>.md)
    ->parse();
```

> The exact option set is still being finalized; treat this list as the direction, not a
> contract.
