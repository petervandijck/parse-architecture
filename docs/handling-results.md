# Handling Results

You never write webhook-handling code. The SDK ships the route, verifies the signed callback,
and matches it back to your `ParseRequest`, then fires a Laravel event. All you write is what
to *do* with a finished document.

And you don't even start from scratch: [`parse:install`](installation.md#4-run-the-installer)
dropped a listener into `app/Listeners/HandleParsedDocument.php`. Open it and fill in the body:

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
        $parse    = $event->request;          // the ParseRequest, now 'completed'
        $markdown = $parse->markdown();       // the parsed Markdown
        $document = $parse->parsable;         // the model you passed to ->for() (null if none)

        // Your logic: store it, index it, send it to an LLM, notify the user…
        // $parse->meta and $parse->page_count are here too.
    }
}
```

On **Laravel 12 and 13 this listener is auto-discovered**: it's live the moment the file
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
        // Your logic: flag the document, alert someone, queue a retry…
    }
}
```

> **Prefer to wire your own?** Skip the publish and write any listener for `ParseCompleted` /
> `ParseFailed`; they're auto-discovered the same way. The published stubs are just a head
> start; once published they're yours, so a later SDK update won't overwrite them.

> `$parse->markdown()` reads the result from your bucket (or fetches the managed copy). You
> can also go straight to the file: `Storage::disk($parse->disk)->get($parse->output_path)`.

---

## Checking status

The Markdown is always delivered by the [event](#handling-results). But you'll often want to
show the *end user* where their document is: "still parsing", "done", or "something went
wrong". That's what `->status()` is for.

It reads the local `parse_requests` row, which the SDK keeps current as results come in (the
webhook updates it in production; the background poll does locally). It's a plain DB read (no
API call), so it's cheap to hit on every page load:

```php
$parse = Parse::find($id);   // look up the row by id

$label = match ($parse->status()) {
    'pending'   => 'Still parsing…',
    'completed' => 'Done',
    'failed'    => 'Something went wrong: ' . $parse->error,
};
```

Drive a progress badge or a Livewire/polling spinner with it. `->status()` only tells you *where
the job is*; it never fetches the Markdown. The result still arrives through the
`ParseCompleted` event.

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
| **Max file size** | PDFs up to 1 GB; other formats lower | smaller, dev-sized cap |
| **Total storage** | your own | quota-limited |
| **Result retention** | yours to keep | ~1 day |
| **Best for** | production, bulk | local development |

Results are delivered by **event** (`ParseCompleted` / `ParseFailed`): parse-time problems
arrive as a `ParseFailed` event carrying `$request->error`, not as a thrown exception. Only
**submission** problems (bad key, unsupported type, quota exceeded) throw
`ParseForArtisans\Exceptions\ParseException` from `->parse()` itself.

> Size and quota numbers above are examples and not yet final.
