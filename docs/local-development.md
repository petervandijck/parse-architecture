# Local Development

Two things differ on a laptop, and the SDK handles both so your code stays identical to
production: where results are stored, and how they're delivered back to your app.

## How results are delivered

"Delivery" is how a finished result travels from us back into your app as a `ParseCompleted`
event. There are two modes, and the default `auto` chooses between them per environment:

| Mode | What happens |
|:--|:--|
| `auto` *(default)* | `poll` when `APP_ENV=local`; `webhook` everywhere else (staging, production, …), keyed off `APP_ENV`. |
| `webhook` | We POST a signed callback to the SDK's route; it verifies the signature and fires the event. Reactive, no polling. |
| `poll` | The SDK checks our API from a queued job and fires the same event when the result lands. For local dev, where we can't reach an inbound webhook. |

Both modes fire the **same** `ParseCompleted` event, so your listener code is identical either
way; delivery only changes how the result gets *to* that listener. With `auto`, production uses
webhooks and your laptop polls, with nothing to set by hand.

## No bucket? Use ours.

Leave `parse.disk` unset and we write results to a **managed dev bucket**: zero setup, nothing
to configure. It's quota-limited (sized for development, not production) and results are kept
for **~1 day**, so fetch what you need promptly. For production, set `PARSE_DISK` to your own
bucket; your bytes then never transit us.

## No inbound webhook? We poll instead.

We can't reach a webhook on `localhost`, so with `delivery=auto` your local environment uses
**`poll`**: `->parse()` quietly dispatches a small background job that checks our API, then
updates the `parse_requests` row and fires the same `ParseCompleted` event when the result
lands. It rides the queue worker you're already running; if you use Laravel's `composer run
dev`, that includes `queue:listen`, so **events just work locally with nothing extra**.

```bash
composer run dev        # serve + queue:listen + vite, events fire locally
```

> That background poll is also what advances [`->status()`](handling-results.md#checking-status)
> locally (in production the webhook keeps the row current). So **run a worker locally to see
> status move or events fire**, and it needs a real queue driver (`database` or `redis`, the
> Laravel default), not `sync`.

Prefer real webhooks locally for true production parity? Run a tunnel (ngrok, Expose, Herd
Share), point `APP_URL` at it, and set `PARSE_DELIVERY=webhook`.
