# Failure modes

Each window states what the client observed and where the only copies are.

## Crash windows

| When the origin dies | Client | WAL on origin | Peer | What to do |
|---|---|---|---|---|
| Before WAL commit | Error or timeout | Nothing | Nothing | Client retries with the same `client_id`. |
| After WAL commit, before handler | No success | `stored`, not pending | Nothing unless quorum already appended | On restart, either finish apply or delete the row if the handler never ran. Do not leave `stored` rows forever with no sweeper. |
| After local apply, before the response | Client may retry | `applied` | Maybe | Retry must be idempotent on `client_id`. |
| After success, peer has the WAL row, peer not yet applied | Success | `applied` | Row committed | Peer apply retries. Safe under the disk-loss rule. |
| After success, async mode, peer has nothing, origin disk then destroyed | Success | Gone | Nothing | **Acknowledged write lost.** This is wish-bank's default 60s push. Do not use it when the disk can disappear. |

Quorum-ack closes the last row: no success until a peer commit of the WAL row exists.

## Partition

Both sides keep accepting writes. Each push fails and retries. When the link returns, both origins push their own pending entries. LWW keeps one version per `(entity, client_id)`.

The losing write remains in `wal_entries` as `superseded`. Its business effect is gone on every node. Tell the operator that concurrent edits of one entity drop one side. Split settings and other compound records into per-field ids if both edits must survive.

There is no automatic primary. Do not add a failover flag that disables writes on one node unless the product explicitly wants read-only degraded mode. This design's availability comes from every survivor accepting writes.

## Peer apply failure

The peer stores the row, marks `failed`, and withholds the ack. The origin logs the error and retries. Ingest of an existing unapplied id deletes and re-appends, so a fixed payload (for example a body that used to be NULL) can replace the broken one.

A poison entry retries forever. Status must show `failed` and `apply_error`. Fix the handler or the payload, then let redelivery succeed. Do not delete a failed peer row by hand unless you also accept that the origin will redeliver it.

## Clock skew

`rev` uses wall time, then bumps by one if it would not exceed the entity's current `rev` on that origin. Two nodes can stamp the same real-world edit with very different `rev` values if one clock is ahead. The ahead clock wins even if its edit happened earlier in real life.

Run NTP on every node. Do not "fix" skew by taking the minimum clock. Do not compare arrival time.

## Id collisions

Local integer ids will diverge. A foreign key sent as only the integer applies to the wrong row on the peer, or to no row.

Missing `client_id` on a historical row: the reference backfill is `srv-` plus the local id. Those backfills are **not** equal across nodes. Seed `wal_id_map` after a single snapshot is copied to every node, so the backfilled ids match the copied rows. Do not backfill independently on two databases that were never the same file.

## Double apply

LWW does not stop two different `client_id`s. A weekly settle, a payment post, or any "once per natural key" operation needs a unique index on that natural key. The replay then hits the same constraint the original request would hit. Map that constraint violation to a successful idempotent replay, not to `failed`, or the origin will retry forever.

## Join and history

Entries are not forwarded. If origin A dies forever before B and C have acked, B and C do not fetch A's tail from each other unless they already stored it.

A new node started empty, with replication on, does not reconstruct old business rows. Those rows have no WAL. Copy a snapshot first.

## SQLite backup

The engine journal mode WAL splits a committed database across `<db>` and `<db>-wal`. `cp` of `<db>` alone can open and still be missing or mixing commits. That looks like a successful backup.

Use `VACUUM INTO` or the online backup API, which produces one consistent file. Restore by stopping the process, moving the db into place, and deleting leftover `-wal` and `-shm` files so SQLite does not replay a journal from a different generation.

`synchronous=NORMAL` plus a power cut can drop the last commits even though the application committed the transaction. Use `synchronous=FULL` when a lost tail is a data-loss bug.

`MaxOpenConns(1)` on SQLite avoids self-deadlock when code reads rows and then writes on the same connection. If you raise the pool, stop interleaving a read cursor with writes on one connection.

## Credentials

Ingest uses the application credential. A leaked credential lets a caller append WAL rows, not only read. Keep it in an environment file on the node, mode `600`, never in the skill, the repo, or a client-side public bundle.

Do not replicate the secret-bearing config through the WAL.
