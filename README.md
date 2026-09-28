# outbox - transactional outbox pattern over std.sql, written in Ecko

Write a business row and the message you owe the outside world - a webhook,
an event - in one database transaction, so both commit or neither does. A
separate `relay` call then delivers queued messages: retry with backoff on
failure, dead-letter after too many attempts, and never let one bad message
block the rest of the batch.

No native code: it composes `std.sql`, `std.json`, `std.uuid` and `std.time`
over a connection you already opened. This package never calls `sql.open` or
`sql.close` itself - you pass in the handle, so it needs no capabilities.

## Install

```bash
ecko get github.com/ecko-lang/outbox
```

```ecko
import outbox
```

`outbox` opens no connections and touches no filesystem or network itself, so
it needs no `grant` in your `ecko.json` dependency entry.

## Usage

```ecko
import std.sql
import outbox

db = sql.open("app.db")
outbox.init(db)

# Insert the order and queue its webhook in the same transaction: if either
# write fails, both roll back, and the outbox never queues a message for an
# order that was not actually saved.
sql.transaction(db, fn(tx) {
    sql.exec(tx, "insert into orders (id, total_cents) values (?, ?)", [order_id, total])
    outbox.add(tx, "order.created", { order_id: order_id, total_cents: total })
})

# Somewhere else - a worker loop, a cron tick - relay what is due.
summary = outbox.relay(db, fn(msg) {
    http.post("https://hooks.example.com/orders", json.encode(msg.payload))
})
# { delivered: 1, failed: 0, dead: 0 }
```

## API

Every function takes an already-open `db` (or `tx`, inside a transaction) as
its first argument.

| Function | Description |
|---|---|
| `init(db)` | Create the outbox table and its relay index if they do not exist. Safe to call on every startup. |
| `add(db, topic, payload, opts?)` | Queue a message. Call inside the same `sql.transaction` as the write it belongs to. Returns the message's `id`. |
| `relay(db, deliver_fn, opts?)` | Deliver due messages, oldest first. Returns `{ delivered, failed, dead }`. |
| `pending(db)` | Messages awaiting delivery (including ones backing off), oldest first. |
| `dead(db)` | Messages that hit `max_attempts` and stopped retrying, oldest first. |
| `retry_dead(db, id, opts?)` | Move one dead message back to pending with a reset attempt count. |
| `prune(db, older_than)` | Delete delivered/dead messages last touched before `older_than` (Unix ms). Returns rows removed. |

### `add` options

| Key | Default | Meaning |
|---|---|---|
| `id` | a fresh `uuid.v7()` | Use your own id (e.g. to make `add` idempotent on retry of the caller itself). |
| `max_attempts` | `5` | How many delivery attempts `relay` makes before dead-lettering this message. |
| `now` | `time.now()` | Unix ms to stamp the message with. Inject a fixed value in tests. |

### `relay` options

| Key | Default | Meaning |
|---|---|---|
| `batch_size` | `100` | Maximum messages read per call. |
| `base_backoff_ms` | `1000` | Delay after the first failure. |
| `max_backoff_ms` | `60000` | Cap on the backoff delay. |
| `now` | `time.now()` | Unix ms used to pick due messages and stamp attempts. Inject a fixed value in tests. |

### The message shape

`relay`'s `deliver_fn`, and `pending`/`dead`, hand back:

```
{ id, topic, payload, status, attempts, max_attempts,
  created_at, next_attempt_at, delivered_at, last_error }
```

`payload` is decoded back to the value passed to `add`. `last_error` is
`null`, or the `{ kind, message }` of the most recent failed attempt.

## Notes

**Delivery is at-least-once, never exactly-once.** A crash between a
successful `deliver_fn` call and the row being marked delivered means the
same message is retried on the next `relay`. Every message carries a stable
`id` for exactly this reason - dedupe on it in `deliver_fn`.

**One failing message never blocks the batch.** `relay` catches each
`deliver_fn` call individually; a message that throws is recorded and
skipped, and the rest of the batch still runs. `relay` itself never throws
because of a failed delivery.

**Backoff is exponential per message.** After attempt *n* fails, the message
is rescheduled `base_backoff_ms * 2^(n-1)` ms later, capped at
`max_backoff_ms`. `max_attempts` is set per message (via `add`'s `opts`, not
per `relay` call), so different message types can carry different retry
budgets on the same table.

**Error kinds this package throws:**

| `kind` | Where | Meaning |
|---|---|---|
| `not_found` | `retry_dead` | `id` does not name a message currently in the `dead` state (never existed, or is pending/delivered). |

`deliver_fn`'s own errors are never re-thrown by `relay` - they are caught,
recorded in `last_error`, and folded into the `failed`/`dead` counts.

**Time is always explicit.** Every function that needs "now" takes it as an
`opts.now` override; pass a fixed value in tests instead of depending on the
wall clock.

## Testing

```bash
ecko test
```

Offline and deterministic - every test uses a `:memory:` SQLite database and
an injected `now`, never the wall clock. Covers: a message added inside a
committed transaction is delivered; one added inside a rolled-back
transaction never appears; a failing delivery retries with backoff and then
dead-letters at `max_attempts`; one bad message does not block the rest of a
batch; a delivered message is never redelivered; delivery order; `retry_dead`
(including the not-found error); and `prune`.

## License

MIT - see [LICENSE](LICENSE).
