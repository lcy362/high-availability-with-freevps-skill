---
name: high-availability-with-freevps-skill
description: >-
  Design and implement N-master application WAL replication so two or more
  unreliable VPS keep acknowledged writes and converge after partitions.
  Use when the user asks for high availability on free or flaky VPS,
  dual-master or multi-master sync, application WAL replication, SQLite
  without a reliable primary, or how to avoid data loss when any node
  and its disk can disappear.
---

# High availability on unreliable VPS

Build a system where every node accepts writes, an acknowledged write is committed on more than one disk, and live nodes converge after a partition.

While using this skill, write schemas, configs, and decisions into the target repo as soon as they are settled. The chat is not the store of record.

## Outcome

- Two or more nodes, each with its own disk and a stable `origin` name.
- Clients send writes to any live node.
- A success response means the WAL row is committed on the origin and on at least `F` other nodes. Default `F=1`: one peer. This survives one lost disk only if a node that stored the row is still alive.
- Business rows are produced by replaying that same request through the normal handlers, in per-origin `seq` order.
- A node that was down, or that was not in the ack set, catches up because any node holding the entry relays it. Idempotent ingest makes a duplicate delivery harmless.
- After the network returns, every live node applies the same winning version per entity.

Reads are not linearizable. A node that has not applied the winning `rev` serves stale data. Read-your-writes holds on the node that accepted the write. It holds on other nodes only after they have applied that entry.

## Do not use this design when

- A lost concurrent update of the same entity is unacceptable. Conflict resolution is last-writer-wins, not a merge. Overwriting a counter or a balance loses increments.
- One request must atomically commit a multi-entity invariant across nodes beyond a single replayed request.
- A replicated database with quorum disk is available and operated. Prefer that. This skill is for small services on VPS that reboot, get reclaimed, or lose the network.

## Hard rules

1. **Log, then apply, then ack the client.** Commit the WAL row before the business handler runs. Return success only after the durability rule is met.
2. **One database apply path.** Store method, path, and body. Replay them through the same handlers. Do not keep a second per-table apply implementation.
3. **Replay must not append again.** An internal context mark, not a client header, tells capture to skip the replayed request.
4. **Stable identity.** The entity key is a client-generated id, unique per entity type. Local integer primary keys may differ per node. Never use those integers as the WAL identity.
5. **Separate the entity from the attempt.** `client_id` is the entity (LWW key). `op_id` is one client attempt. A retry reuses `op_id`. A later edit of the same entity uses a new `op_id`. Handlers, crash recovery, and replay are idempotent on `op_id`. Applying the same `op_id` twice is not a second operation. The same `client_id` with a new `op_id` is a new operation.
6. **Idempotent ingest.** WAL id is `origin:seq`. Redelivery of an applied or superseded id returns success and does not apply twice.
7. **Total order.** Compare `(rev, origin)`. Higher `rev` wins. Equal `rev`: the lexicographically greater `origin` wins. Compare origins as raw strings, case-sensitive.
8. **Same-origin order.** Apply one origin's entries in `seq` order. Commit a higher `seq` early if it arrives first, but do not apply it while a lower `seq` is missing, unapplied, or failed.
9. **Any holder relays.** The origin pushes first. A peer that has committed the row also pushes it to peers that lack it. Do not rely on the origin staying up to reach nodes outside the ack set.
10. **Empty body is empty bytes.** Persist `[]byte{}`, never SQL NULL.
11. **Fail closed on missing references.** If a stable foreign key is not in this node's id map yet, do not apply and do not ack `applied`. Durability of the bytes is still allowed.
12. **Do not capture non-business writes.** Skip reads, ingest, status, admin, and bulk import/export.
13. **A non-2xx handler committed nothing.** If a failed response can leave business rows behind, the node will diverge. Fix the handler.
14. **External side effects are not the business row.** Mail, webhooks, and payments must be idempotent on `op_id`, or written as an outbox row in the same transaction as the business write. Replay runs the handler on every node.
15. **Engine WAL is not this log.** Copying a SQLite database file while its `-wal` journal exists can open a silently wrong database. Back up with `VACUUM INTO` or the engine backup API.

## Durability rule

Default for this skill: before success, the origin has committed the entry and `F` peers have committed the same bytes (`F=1` unless the operator raises it). The peer's business apply may finish after that commit. The ingest response must say so separately: `durable` means the bytes are committed, `applied` means the handler finished or the entry lost LWW.

`ack=local` (success after the local commit only) is a deliberate downgrade. A client can be told success, and the only disk can then disappear. Do not describe a background push as protection against disk loss.

Wait for `F` peers. Do not wait for every peer. A node outside that set catches up by relay. If every node that stored the row disappears before relay, the acknowledged write is gone. Extra live nodes do not help unless they hold a copy.

SQLite: `synchronous=FULL` when a power cut must not drop a commit. `NORMAL` can drop the tail of the engine journal.

`rev` is `max(unix_millis, current_rev+1)`, assigned under the same lock as the entity read. A backward step of the local clock cannot lose to an older row on the same node. Across nodes, a fast clock wins. State that in the service docs. Run NTP. Do not resolve ties by arrival time.

## Write path

Serialize apply on a node so two requests cannot read the same entity `rev` and both proceed.

1. If the request is an internal replay, or not a business write, call the handler and stop.
2. Read the body once. Restore it for the handler. Attach stable foreign-key markers beside local ids.
3. Resolve `(entity, client_id, op_id, op)`. If the entity id or `op_id` is missing, reject the write. If this node already has that `op_id` in `applied` or `rejected`, return the stored result and do not allocate a new `seq`.
4. Under the apply lock: read entity state, set `rev = max(now_ms, state.rev+1)`, allocate the next `seq` for this origin, insert `status=stored`, commit.
5. Re-read entity state. If this `(rev, origin)` would lose, do not run the handler. Mark the entry `superseded`, return `409`, and still let peers learn the entry so their `seq` stream has no gap.
6. If it wins, run the handler.
7. On 2xx: record the local id map, upsert entity state, set `applied` and `applied_at`, commit that status.
8. On non-2xx: the handler must have committed nothing. Mark the entry `rejected` and set `applied_at`. Do not delete it. A deleted `seq` is a hole, and every higher `seq` from this origin waits forever. Peers store the rejection and advance past it without running the handler.
9. Ask peers to ingest this entry. Return success only after `F` peers report `durable`. On timeout return `503` and leave the local row `applied` so relay continues. The client retries the same `op_id`, to any node.
10. Keep pushing until every configured peer reports `applied` or the entry is `superseded` everywhere it was applied. A `durable` ack alone does not retire the entry.

Allocate `seq` with the insert in one immediate transaction (or a counter row). `SELECT MAX(seq)+1` outside that lock can collide. A unique index on `(origin, seq)` is the backstop, not the allocator.

## Ingest path

For each entry in the request, in `seq` order per origin:

1. Reject an empty id or origin.
2. If this id is already `applied`, `superseded`, or `rejected`, ack `durable` and `applied`. If this `op_id` is already done here under another WAL id, do not run the handler again. Keep the first body. An entry that arrives already `rejected` is stored and counts as applied for the `seq` cursor. Do not call the handler.
3. If this id exists and is not applied, replace the payload (redelivery repairs a bad body) or keep the committed row if the bytes match.
4. Commit the row before apply. That commit is `durable`.
5. If a lower `seq` from this origin is missing, unapplied, or `failed`, stop apply for this origin. Ack `durable` and not `applied`, with a gap error. A local sweep applies the held row once the gap closes. Do not apply later sequences in this batch.
6. If the entry loses `(rev, origin)`, mark `superseded`, ack both. The business handler does not run.
7. Translate stable foreign keys. If a marker is present and the local id is unknown, ack `durable`, not `applied`.
8. Replay with the internal mark. On 2xx: update entity state only if this entry still wins, record the id map, mark `applied`, ack both.
9. On handler failure: mark `failed`, ack `durable`, not `applied`. The holder retries apply. Do not advance past this `seq`.

A peer that has a durable copy relays the original entry to other peers. Relay is not a new capture.

## Startup recovery

A `stored` row means the process died after the log commit and before `applied`. Do not delete it. You cannot tell whether the handler finished. Re-run the handler (idempotent), then mark `applied`, then resume relay.

A row left `applied` with peers unacked is resumed by the push loop. Nothing else repairs it.

## Identity

- Capture keeps the caller's local foreign key and adds a stable marker (`parentId` plus `_parentClientId`).
- Replay replaces the local field from `wal_id_map`, then strips the marker.
- After a successful create, read the stable id and the local id from the handler response and upsert the map.
- Seed the map from existing rows at startup.
- A row that lacks a stable id gets one assigned in the snapshot, once, before that snapshot is copied. Do not mint ids independently on each node.
- A singleton edited field-by-field uses one stable id per field. One id for the whole row makes LWW drop the other field.
- "Once per natural key" (a daily close, a single ledger post) needs a unique constraint on that key. Two different client ids both win LWW. Map a constraint conflict on replay to idempotent success, not to `failed`, or the `seq` stream stalls.

## What stays local

Ingest, status, admin, export, import, and other node-local controls. A version stamp excluded from capture must be written on every node by the deploy step.

## Agent workflow

1. Name the threat. Default is disk loss, `F=1`. Choose `ack=local` only when the operator accepts a lost acknowledged tail.
2. List business writes: entity name, stable id field, foreign keys, routes excluded from capture, external side effects.
3. Add the tables in [references/protocol.md](references/protocol.md). Migrate old databases by column presence, not by a version number that drifts across restored snapshots.
4. Implement capture, the apply lock, ingest, the id map, relay, and the gap sweep.
5. Tests that must exist before calling it done:
   - Redelivery does not double-apply.
   - A lower `(rev, origin)` becomes `superseded` and is still acked `applied`.
   - Equal `rev`, greater origin string wins on every node.
   - Replay does not insert a second WAL row.
   - A non-2xx handler leaves no business row, and the WAL row stays as `rejected` so `seq` does not skip.
   - An empty delete body replicates.
   - A higher `seq` that arrives first is durable and unapplied until the gap closes, then applies once.
   - Missing foreign-key map does not ack `applied`.
   - Kill the peer before it answers: the client sees `503`, the entry exists on the origin, and a later push finishes it.
   - Kill the origin after one peer has `durable`: that peer applies alone, and a third node receives the entry by relay.
   - Two concurrent updates of one counter document the lost increment. Do not hide it.
6. Bring up with [references/bringup.md](references/bringup.md). Run the kill tests on the real nodes once.
7. Record the crash windows the service still accepts, using [references/failure-modes.md](references/failure-modes.md).

## Reference map

| Topic | File |
|---|---|
| Entry, acks, tables, order | [references/protocol.md](references/protocol.md) |
| Crash windows and backup | [references/failure-modes.md](references/failure-modes.md) |
| Bootstrap and checks | [references/bringup.md](references/bringup.md) |
