# Changelog

## 0.56.0

### Breaking

- `add`, `relay` and `retry_dead` take their options as named arguments
  instead of an options map. Migration: `outbox.add(tx, topic, payload,
  { id: "m-1", max_attempts: 3 })` becomes `outbox.add(tx, topic, payload,
  id: "m-1", max_attempts: 3)`; `outbox.relay(db, f, { batch_size: 50 })`
  becomes `outbox.relay(db, f, batch_size: 50)`; `outbox.retry_dead(db, id,
  { now: t })` becomes `outbox.retry_dead(db, id, now: t)`. Passing the old
  map is refused with a message showing the new form.
- `add` refuses a payload that JSON would not give back unchanged (a decimal,
  record, secret, bytes or function, anywhere inside it) with
  `outbox`/`not_storable`. These used to reach `deliver_fn` silently changed:
  a decimal as a float, a record as a `__type__` map, bytes as base64 and a
  secret as the text `"[secret]"`. Migration: convert before adding
  (`string(amount)`, a record to a map), and keep secrets out of messages.
- `retry_dead` on an id that names no dead message throws kind `outbox` with
  `reason: "not_found"` and the `id`, instead of kind `not_found`.
  Migration: match `get(e, "reason") == "not_found"` instead of
  `get(e, "kind") == "not_found"`.

### Fixed

- `add` with an `id` that is already queued throws `outbox`/`duplicate_id`
  (carrying the `id`) instead of a raw `sql` UNIQUE constraint error.
- Options are validated where they are passed: `max_attempts: 0` or `-1`,
  a non-Int `now` (which made `relay(db, f, now: "x")` deliver everything), a
  bad `batch_size` or backoff, an empty topic and a non-function `deliver_fn`
  are refused with `bug`/`bad_argument` naming the function and the option.
- The README no longer claims that a caller-supplied `id` makes `add`
  idempotent (it dedupes by refusal, as above), and now says to run one relay
  at a time per table, since two concurrent relays deliver the same messages.
