# High availability on unreliable VPS

A skill for building a small multi-master service on two or more VPS whose disks can disappear. Every node accepts writes. A successful response means the write is already committed on more than one disk. After a partition, live nodes converge.

This is an application write-ahead log, not database engine replication. The log stores the original request. Another node replays that request through the same handlers.

## Logic

1. The client sends a write to any live node. `client_id` names the entity. `op_id` names this attempt. A retry reuses `op_id`. A later edit of the same entity uses a new `op_id`.
2. The node appends a log row (`origin:seq`) and commits it before running the business handler.
3. The handler updates local state. The same `op_id` run again is the same operation.
4. The node waits until `F` other nodes have committed that log row. The default is `F=1`, so one peer is enough. Then it returns success. The peer may finish applying the business handler after that.
5. Any node that holds the row relays it to nodes that do not. A node outside the first ack set catches up this way.
6. Each node applies one origin's rows in `seq` order. A newer row may be stored early, and it waits until the gap in front of it closes.
7. Two updates of the same entity use `(rev, origin)`. The higher revision wins. Equal revisions use the origin name, compared as a raw string. Both sides compute the same winner.

A node that has not applied the winning row still serves older data. Read-your-writes holds on the node that accepted the write.

`F=1` keeps two copies. The write survives the loss of one of those disks. It does not survive the loss of both. An extra node that has not received the relay does not hold the write.

Concurrent edits of one entity keep a single version. An overwrite of a counter drops the other increment. Model an increment as its own operation when both must survive.

## Usage

Point an agent at this directory and tell it to follow [SKILL.md](SKILL.md). The agent uses that file as the procedure: pick `F`, list the business writes, add the tables, implement capture and ingest, then run the kill tests.

Read the references only for the part you are touching:

| File | Read it for |
|---|---|
| [SKILL.md](SKILL.md) | Rules, write path, ingest path, tests |
| [references/protocol.md](references/protocol.md) | Tables, acks, sequence cursor, HTTP |
| [references/failure-modes.md](references/failure-modes.md) | Crash windows, clocks, backups |
| [references/bringup.md](references/bringup.md) | First snapshot, deploy, checks |

Do not start replication on two databases that have already diverged. Copy one consistent snapshot to every node, then turn the log on. History that was never logged is not reconstructed.

If a managed replicated database is available, use that instead.
