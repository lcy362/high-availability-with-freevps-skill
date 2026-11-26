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

Build a system where every node accepts writes, the write is durable on more than one machine before the client is told it succeeded, and nodes converge after a partition. The reference implementation is wish-bank (`backend/internal/store/wal.go`, `backend/internal/httpapi/handlers_wal.go`). Copy its replay model. Do not copy its async-only ack if the disk can vanish.

While using this skill, write schemas, configs, and decisions into the target repo as soon as they are settled. The chat is not the store of record.

## Outcome

- Two or more nodes, each with its own disk and a stable `origin` name.
- Clients send writes to any live node.
- A success response means the WAL row is committed on the origin and on at least one peer.
- Business state is produced by replaying that same request through the normal handlers.
- After the network returns, every node that is still alive reaches the same entity version.

## Do not use this design when

- A lost concurrent update of the same entity is unacceptable. The conflict rule is last-writer-wins, not a merge.
- One request must atomically commit a multi-entity invariant across nodes beyond a single replayed request.
- A real replicated database with quorum disk is available and operated. Prefer that. This skill is for small services on VPS that reboot, get reclaimed, or lose the network.

## Hard rules

1. **Log, then apply, then ack.** Insert the WAL row and commit it before the business handler runs. Return success only after the durability rule below is met.
2. **One apply path.** Store the original method, path, and body. Replay them through the same HTTP handlers. Do not write a second per-table apply function.
3. **Replay must not append again.** Mark internally replayed requests so the capture middleware skips them. An external client cannot set that mark.
4. **Stable identity.** The replication key is a client-generated id, unique per entity. Local integer primary keys may differ per node. Never use those integers as the WAL identity.
5. **Idempotent ingest.** WAL id is `origin:seq`. A redelivery of an already-applied id returns success and does not apply twice.
6. **Total order.** Compare `(rev, origin)`. Higher `rev` wins. Equal `rev`: the lexicographically greater `origin` wins. Every node computes the same winner.
7. **Empty body is empty bytes.** A missing body is `[]byte{}`, never SQL NULL. NULL breaks the NOT NULL body column on the peer.
8. **Push only your own origin.** Do not forward another node's entries. The origin retries until each peer acks. That avoids loops.
9. **Do not capture non-business writes.** Skip reads, the ingest and status endpoints, admin routes, and bulk import/export.
10. **Engine WAL is not this log.** SQLite's `-wal` file is the database journal. Copying the db file while that journal exists can open a database that is silently wrong. Back up with `VACUUM INTO` or the engine backup API.

## Durability rule

| Threat | Required ack |
|---|---|
| Process crash or reboot, disk survives | Commit locally, then success. Peer push may be async. |
| The VPS and its disk can disappear (free-tier reclaim, no volume) | Success only after the origin commit **and** at least one peer has committed the same WAL row. Business apply on the peer may finish after the ack. |

wish-bank today is the first row: local apply, then a background push every `WISHBANK_WAL_INTERVAL_SEC` (default 60). A client can receive 200, and the only copy dies with that disk before the next push. For the second row, add a quorum wait on ingest-append. Do not describe the async loop as protection against disk loss.

SQLite: set `synchronous=FULL` when the goal is no lost commits on power loss. The reference uses `NORMAL`, which can drop the tail of a WAL on power loss.

`rev` is `max(unix_millis, current_rev+1)` on the origin so a backward step of the local clock cannot lose to an older row on the same node. Across nodes, a fast clock wins ties of real time. State that limit in the service docs.

## Write path

1. If the request is an internal replay, or not a business write, call the handler and stop.
2. Read the body once. Restore it for the handler.
3. Resolve `(entity, client_id, op)`. If this route has no stable id, reject the write. Do not invent a server id as the only key.
4. `rev = max(now_ms, entity.rev+1)`.
5. Insert the WAL row (`status=stored`) and commit.
6. Run the real handler on the origin.
7. On 2xx: record the local id map, upsert entity state only if this `(rev, origin)` wins, set `status=applied` and `applied_at`.
8. On non-2xx: delete that WAL row. A rejected business write must not ship.
9. Durability gate, then return the handler's status and body to the client.
   - Disk-loss threat: block until one peer's ingest has committed this id (apply may still be pending there). Timeout: return 503. Leave the local row applied so a later push can finish.
   - Reboot-only threat: return after step 8. A background loop pushes.

## Ingest path

For each entry, in order:

1. Reject an entry with an empty id or origin.
2. If this id is already `applied` or `superseded`, ack success.
3. If this id exists but is not applied, delete it and append the new payload. Redelivery repairs a bad body.
4. Append and commit **before** apply. That commit is what a quorum ack is allowed to wait on.
5. If `WALWins(rev, origin, current)` is false, mark `superseded`, ack success. The sender must stop retrying.
6. If it wins, replay the request with the internal mark. Translate stable foreign keys to this node's local ids first.
7. On 2xx: update entity state, mark `applied`, record the id map, ack success.
8. On failure: mark `failed` with the error text and do not ack. The origin keeps the entry pending and retries.

## N nodes

Configure every node with the full peer list. Track acks per peer (`wal_id`, `peer`), not one `peer_applied_at` column. wish-bank's single column is correct only for exactly one peer.

Pending means: `applied_at` set, status is not `failed`, and at least one configured peer has no ack.

A node that joins later does not receive history that was never in the WAL. Bootstrap from a consistent snapshot, then start capture. See [references/bringup.md](references/bringup.md).

## Identity translation

Business tables may use a local integer primary key. The WAL carries `client_id`.

- On capture, keep the local foreign-key fields and add stable markers (`memberId` plus `_memberClientId`).
- On replay, replace the local field with this node's id from `wal_id_map`, then strip the marker.
- After a successful create, read `(client_id, id)` from the handler response and upsert the map.
- Seed the map from existing rows at startup so old rows can be referenced.

A singleton row edited field-by-field (settings) must use one `client_id` per field. One id for the whole row makes an LWW update drop the other field.

Operations that must not double-apply (a weekly bonus, a ledger post) need a business unique key, not only LWW. Two creates with different client ids and different revs both win.

## What stays local

Replication endpoints, admin, export, import, lock-verify, and failover switches. A version stamp that clients read from "the" server must be written on every node by the deploy step. It will not arrive through the WAL.

## Agent workflow

1. Name the threat: reboot with disk, or disk gone. Pick the ack rule before writing code.
2. List business writes. For each: entity name, stable id field, foreign keys that need markers, routes excluded from capture.
3. Add the three tables in [references/protocol.md](references/protocol.md). Migrate old databases by column, not by a version number that drifts across restored snapshots.
4. Implement capture, ingest, id map, and the push loop. For disk-loss, implement quorum wait.
5. Tests that must exist before calling it done:
   - Redelivered ingest does not double-apply.
   - Lower `(rev, origin)` becomes `superseded` and is still acked.
   - Equal `rev`, higher origin string wins on both nodes.
   - Replay does not insert a second WAL row.
   - Failed handler deletes the origin WAL row.
   - Empty delete body replicates.
   - Kill the peer, write, revive the peer, the row appears.
   - Disk-loss mode: peer append committed before the client sees success.
6. Bring up with [references/bringup.md](references/bringup.md). Run the kill tests on the real nodes once.
7. Record crash windows you still accept in the service README, using [references/failure-modes.md](references/failure-modes.md).

## Reference map

| Topic | File |
|---|---|
| Entry, states, tables, wins | [references/protocol.md](references/protocol.md) |
| Crash windows and backup | [references/failure-modes.md](references/failure-modes.md) |
| Bootstrap and checks | [references/bringup.md](references/bringup.md) |
| Working code | wish-bank `backend/internal/store/wal.go`, `handlers_wal.go`, `store.go` schema `wal_*` |
