# Failure modes

Each window states what the client observed and where the copies are. `F=1` unless noted.

## Crash windows

| When the origin dies | Client | What remains | What to do |
|---|---|---|---|
| Before WAL commit | Error or timeout | Nothing | Client retries with the same `op_id`. |
| After WAL commit, before `applied` | No success | `stored` on the origin. A peer may already be `durable` if the wait was reordered badly. Do not wait for peers before the local commit. | On startup, rerun the handler and mark `applied`. Do not delete `stored`. Deleting drops a write the handler may already have committed. |
| After local `applied`, before the client response | Client may retry | Origin `applied`. Zero or more peers `durable`. | The same `op_id` returns the stored result on any node that already has it. A different `op_id` is a new edit. |
| After success | Success | Origin plus at least `F` peers have the bytes | A peer applies from its own row if the origin never returns. Holders relay to everyone else. |
| `ack=local`, then the only disk is destroyed | Success | Nothing | Acknowledged write lost. That mode is outside the default contract. |

A peer can commit and lose the HTTP response. The origin then returns `503` even though two disks have the row. That is a failed ack, not lost data, provided the client retries the same `op_id` and the handler is idempotent on that id.

Deleting a WAL row after `seq` was allocated opens a hole. Rejection marks the row `rejected` and replicates that mark. Peers advance `next_seq` without calling the handler.

## Which failures actually survive

`F=1` stores two copies. The write survives the death of one of those two disks. It does not survive the death of both. A third live node that has not received the relay does not have the write.

Raise `F` to require more copies before success. Availability drops: the client waits for more peers to be reachable. Relay still has to finish for nodes outside the ack set. There is no forwarding-free layout where every live node is caught up after the origin dies.

## Partition

Both sides keep accepting writes. Pushes fail and retry. When the link returns, holders exchange missing entries. LWW keeps one version per `(entity, client_id)`.

The losing write remains as `superseded`. Its business effect is gone on every node. Concurrent increments of one counter keep one value and drop the other. Model an increment as its own idempotent operation with its own id if both must survive.

There is no automatic primary. Do not disable writes on a minority unless the product explicitly wants a read-only degraded mode.

## Seq gaps and poison entries

A hole, or a `failed` head, blocks every higher `seq` from that origin on that node. Independent entities behind the hole wait. Status must show the blocking id and `apply_error`.

Do not delete the blocking row to "unstick" the node. The sender will redeliver it, or the sweep will see a hole again. Fix the handler or the payload and redeliver.

A handler that returns 2xx while ignoring a missing foreign key acks a partial apply. The skill forbids that. Fail closed and leave the entry unapplied.

## Clock skew

`rev` uses wall time, then adds one when the result would not exceed the entity's current `rev`. Two nodes can stamp the same real-world edit with very different `rev` values. The ahead clock wins even if its edit happened earlier.

Run NTP. Do not take the minimum of the two clocks. Do not compare arrival time.

## Identity

Local integer ids diverge. A foreign key sent as only the integer hits the wrong row, or no row.

Assign missing stable ids once, in the snapshot, before copying that file. Two nodes that invent their own fallback ids for the same logical row will treat them as different entities.

## Double apply

LWW does not collapse two client ids. A one-shot operation needs a unique constraint on its natural key. On replay, a constraint conflict is idempotent success. Marking it `failed` blocks the origin's `seq` stream forever.

## External side effects

Replay executes the handler on each node. A handler that sends mail or takes payment will do it once per node, and again on client retry, unless the effect is idempotent or stored as an outbox row committed with the business write.

If the origin dies after the database commit and before it performs the side effect, a peer still applies the database row. The side effect has to be recoverable from replicated data, not from the origin's memory.

## Join and history

Entries that were never logged are not reconstructed. Start a new node from a consistent snapshot, then let relay fill only the WAL tail from after the snapshot. A divergent database cannot be merged by turning capture on.

## SQLite backup

Engine journal mode WAL keeps commits in `<db>` and `<db>-wal`. Copying `<db>` alone can open and still be wrong. That looks like a successful backup.

Use `VACUUM INTO` or the online backup API. Restore by stopping the process, installing that one file, and deleting leftover `-wal` and `-shm` so the engine does not replay a journal from another generation.

`synchronous=NORMAL` plus a power cut can drop commits the application believed finished. Use `synchronous=FULL` when that tail is a data-loss bug.

One writer at a time for the apply path. If several connections interleave a read cursor with writes, SQLite can deadlock or assign the same `rev` twice. A single connection, or one apply mutex around the whole write path, is the intended shape.

## Credentials

Ingest uses the application credential. A leaked credential can append WAL rows. Keep it in an environment file on the node, mode `600`, never in the skill, the repo, or a public client bundle.

Do not replicate secret-bearing config through the WAL.
