# PostgreSQL LISTEN/NOTIFY: A Minimal Queue Experiment

Watch PostgreSQL's notification queue fill up, reject new notifications, and
recover once a listener ends its open transaction.

**Requirements:** Docker with Compose. Run the commands from this directory.
The lab uses PostgreSQL 17 with a 64-page queue (512 KiB with 8 KiB pages).
It does not publish ports or require Python.

Validated with PostgreSQL 17.11: initial queue usage was `0`, a full queue
reported `1` and produced `too many notifications in the NOTIFY queue` errors,
and usage returned to `0` after the listener's `COMMIT`. The next send succeeded.

Read the [full experiment and results](Resultado.md) for a detailed explanation.

## 1. Start PostgreSQL

```sh
docker compose up -d --wait
```

## 2. Terminal A: Hold the Queue

```sh
docker compose exec postgres psql -X -U postgres
```

Execute these statements separately and leave the session open:

```sql
LISTEN demo;
BEGIN;
```

`LISTEN` commits before the transaction starts. While that transaction remains
open, the listener cannot advance and retains pending notifications.

## 3. Terminal B: Fill the Queue

```sh
docker compose exec postgres psql -X -U postgres
```

```sql
SHOW max_notify_queue_pages;
SELECT pg_notification_queue_usage() AS initial_usage;

-- Each generated statement commits separately (autocommit).
-- Continue after expected errors so we can measure queue usage afterward.
\set ON_ERROR_STOP off
SELECT format('SELECT pg_notify(%L, %L);', 'demo', repeat('x', 7000) || g)
FROM generate_series(1, 100) g
\gexec

SELECT round((100 * pg_notification_queue_usage())::numeric, 2) AS usage_percent;
```

Expect `too many notifications in the NOTIFY queue` errors and high queue usage.
The measurement does not need to reach exactly 100%: allocation happens in pages
and segments. Do not wrap the sends in a single `BEGIN`: we want successful sends
to remain committed before the first failure.

Optionally, inspect the segments backing the queue from another terminal:

```sh
docker compose exec postgres sh -c 'ls -lh "$PGDATA/pg_notify"'
```

The file sizes do not exactly match the bytes of pending messages because of
caching and segment allocation.

## 4. Release the Queue and Verify Recovery

In **terminal A**:

```sql
COMMIT;
```

`psql` will print the pending notifications, including their large payloads.
Wait for the prompt to return. In **terminal B**:

```sql
SELECT pg_notification_queue_usage() AS usage_after_commit;
SELECT pg_notify('demo', 'it works again');
```

Queue usage should drop, and sending should work again. If the measurement has
not dropped yet, query it again after a short delay: cleanup is not instantaneous.

## 5. Clean Up

Exit both sessions with `\q`, then remove the container and its test data:

```sh
docker compose down -v
```

## What This Demonstrates

A listener inside a transaction can retain the queue. When the queue fills up,
transactions sending notifications fail at commit. Ending the listener's
transaction allows the queue to be released and notification delivery to recover.

The queue is shared across the instance, although notifications are delivered
within each database. This test uses one database: it does not independently
demonstrate interference between databases or the historical wraparound limit.
It is not a RAM limit either: the queue is backed by `pg_notify/` with an SLRU
cache. A disconnected client does not retain the queue or receive missed messages
for replay when it reconnects.

In PostgreSQL 17, the default limit remains 8 GiB with 8 KiB pages; it is
configurable through `max_notify_queue_pages`.

Official references: [NOTIFY](https://www.postgresql.org/docs/17/sql-notify.html),
[max_notify_queue_pages](https://www.postgresql.org/docs/17/runtime-config-resource.html#GUC-MAX-NOTIFY-QUEUE-PAGES),
and the [queue implementation](https://github.com/postgres/postgres/blob/REL_17_STABLE/src/backend/commands/async.c).
