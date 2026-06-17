# Installation

## 1. Sign up & get your keys

Create an account at [parseforartisans.com](https://parseforartisans.com), and
grab two values from the dashboard: your **API key** and your **webhook signing secret**.

```
pfa_...      # API key
whsec_...    # webhook signing secret (verifies result callbacks)
```

Keep them secret.

## 2. Install the package

```bash
composer require parseforartisans/laravel
```

## 3. Add your keys

```env
PARSE_API_KEY=pfa_...
PARSE_WEBHOOK_SECRET=whsec_...
```

## 4. Run the installer

```bash
php artisan parse:install
```

One command does it all: publishes `config/parse.php`, drops the event listeners into
`app/Listeners/` (`HandleParsedDocument`, `HandleFailedParse`), and runs the migration that adds
a small `parse_requests` table the SDK uses to track submissions and match results back to them,
so you never handle ids or secrets yourself. Customize the listeners later (see
[Handling Results](handling-results.md)).

`config/parse.php`:

```php
'disk'     => env('PARSE_DISK'),               // your bucket. Leave unset to use our managed dev bucket.
'output'   => 'parsed',                        // prefix where Markdown is written (default: parsed/)
'delivery' => env('PARSE_DELIVERY', 'auto'),   // auto | webhook | poll  (see Local Development)
```

`delivery` decides how finished results reach your app: webhooks in production, polling on your
local machine. The default `auto` picks the right one per environment, so you rarely touch it.
Full details in [Local Development](local-development.md).

> **You don't need a bucket to start.** Leave `PARSE_DISK` unset and results go to our managed
> dev bucket, with nothing to provision. Point it at your own bucket when you're ready for
> production. See [Local Development](local-development.md).

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
