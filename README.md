# outbox

The transactional outbox pattern over `std.sql`: write a business row and the
message you owe the outside world - a webhook, an event - in one database
transaction, then deliver it with retries, backoff and a dead-letter state. No
capabilities of its own: every function takes a database handle you opened.

## Install

```bash
ecko get github.com/ecko-lang/outbox
```

```ecko
import outbox
```

## Usage

```ecko
import std.sql
import outbox

db = sql.open(":memory:")  # sql.open("app.db") in production
outbox.init(db)
sql.exec(db, "create table orders (id text primary key, total_cents integer)")

# Insert the order and queue its webhook in the same transaction: if either
# write fails, both roll back, and no message is queued for an unsaved order.
sql.transaction(db, fn(tx) {
    sql.exec(tx, "insert into orders (id, total_cents) values (?, ?)", ["o-1", 4200])
    outbox.add(tx, "order.created", { order_id: "o-1", total_cents: 4200 }, id: "msg-1")
})
print(len(outbox.pending(db)))  # 1

# Somewhere else - a worker loop, a cron tick - relay what is due. In real code
# deliver_fn makes the http.post and throws when the receiver says no.
summary = outbox.relay(db, fn(msg) print("deliver {msg.id} {msg.topic} {msg.payload.order_id}"))
# deliver msg-1 order.created o-1
print(summary)  # {dead: 0, delivered: 1, failed: 0}

# A message that keeps failing is dead-lettered after max_attempts.
outbox.add(db, "order.created", { order_id: "o-2" }, id: "msg-2", max_attempts: 1)
print(outbox.relay(db, fn(msg) error({ kind: "http", message: "503" })))  # {dead: 1, delivered: 0, failed: 0}
print(outbox.dead(db)[0].last_error)  # {kind: http, message: 503}
outbox.retry_dead(db, "msg-2")
print(len(outbox.pending(db)))  # 1
```

## API

Every function takes an already-open `db` (or `tx`, inside a transaction) as
its first argument.

| function | returns |
|---|---|
| `init(db)` | `null`, after creating the outbox table and its index if they do not exist |
| `add(db, topic, payload, id: null, max_attempts: 5, now: null)` | the new message's id, queued in the caller's transaction |
| `relay(db, deliver_fn, batch_size: 100, base_backoff_ms: 1000, max_backoff_ms: 60000, now: null)` | `{ delivered, failed, dead }` counts, after delivering the due messages oldest first |
| `pending(db)` | the messages awaiting delivery (including ones backing off), oldest first |
| `dead(db)` | the messages that reached `max_attempts`, oldest first |
| `retry_dead(db, id, now: null)` | `null`, after moving one dead message back to pending with its attempts reset |
| `prune(db, older_than)` | the number of delivered and dead messages removed that were last touched before `older_than` (Unix ms) |

`id` defaults to a fresh `uuid.v7()`; `now` defaults to `time.now()` (Unix ms),
and a fixed value makes tests independent of the clock. `max_attempts` is how
many delivery attempts a message gets before it is dead-lettered.

## Notes

### Errors

| kind | reason | when |
|---|---|---|
| `outbox` | `not_storable` | `add` was given a payload holding a decimal, record, secret, bytes or function |
| `outbox` | `duplicate_id` | `add` was given an `id` that is already queued; the error carries `id` |
| `outbox` | `not_found` | `retry_dead` was given an `id` that names no dead message; the error carries `id` |
| `bug` | `bad_argument` | an empty topic, a non-string `id`, a count or delay that is not a positive Int, a non-Int `now` or `older_than`, a `deliver_fn` that is not a function, or an options map |
| `sql` | | the database refused a read or write |

`deliver_fn`'s own errors are never re-thrown by `relay`: they are caught,
recorded in `last_error` as `{ kind, message }` (a thrown string becomes
`{ kind: null, message }`), and folded into the `failed`/`dead` counts.

### The message shape

`relay`'s `deliver_fn`, and `pending`/`dead`, receive:

```
{ id, topic, payload, status, attempts, max_attempts,
  created_at, next_attempt_at, delivered_at, last_error }
```

`payload` is exactly the value passed to `add`: a payload that JSON would not
give back unchanged (a decimal, record, secret, bytes or function, anywhere
inside it) is refused by `add` rather than delivered as something else.
Convert first - `string(amount)` for a decimal, a map for a record - and never
put a secret in a message.

### Delivery

**At-least-once, never exactly-once.** A crash between a successful
`deliver_fn` call and the row being marked delivered means the same message is
retried on the next `relay`. Every message carries a stable `id` for exactly
this reason - dedupe on it in `deliver_fn`.

**Run one relay at a time per table.** `relay` does not claim rows, so two
relays running at once read the same pending messages and both deliver them.

**One failing message never blocks the batch.** `relay` catches each
`deliver_fn` call on its own; a message that throws is recorded and skipped,
and the rest of the batch still runs.

**Backoff is exponential per message.** After attempt *n* fails, the message
is rescheduled `base_backoff_ms * 2^(n-1)` ms later, capped at
`max_backoff_ms`. `max_attempts` is set per message by `add`, so different
message types can carry different retry budgets on the same table.

**A caller-supplied `id` dedupes by refusal.** Adding the same `id` twice
throws `outbox`/`duplicate_id` instead of queuing it again, so a caller that
retries its own work can catch that reason and treat the message as queued.

## Testing

```bash
ecko test
```

Offline and deterministic - every test uses a `:memory:` SQLite database and
an injected `now`, never the wall clock.

## License

[MIT](LICENSE)
