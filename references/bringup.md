# Bring-up

Do this once per service. Repeat the verification after any change to capture, ingest, relay, or the ack rule.

## Layout

- One process per VPS, supervised so it restarts on boot.
- Unique `origin` per node, chosen once and never renamed. Renaming forks the `seq` space and the LWW tie-break.
- Peer list is every other node's base URL. Trim trailing slashes.
- Same binary (compare a hash) and same schema on every node before enabling push.
- Application credential only in the node environment file.
- `synchronous=FULL` on SQLite.
- Clocks stepped with NTP before the first write.
- `F` chosen and written down. Default is 1.

## First data

Pick one node as the snapshot source. Stop writes.

1. Take a consistent snapshot (`VACUUM INTO` or the engine backup API), not a file copy.
2. If any business row has no stable id, assign those ids in this one file before copying it.
3. Install that file on every other node. Delete any `-wal` and `-shm` beside the destination.
4. Start each process so it seeds `wal_id_map` from the copied rows.
5. Turn on push only after every node is serving that same snapshot.

Two nodes that already diverged cannot be merged by turning the log on. The log does not contain their past. Choose one snapshot and accept the other side's extra rows as lost, or write synthetic entries on purpose and review them one by one.

## Deploy

Build once. Ship that artifact to every node. Restart one node at a time. A restarted node keeps its `origin`. The next `seq` is the max already stored, so it survives restart. Startup reruns leftover `stored` rows before accepting new writes.

After rollout, check each node:

- Health responds.
- Status shows the expected `origin`, every peer, and no `seq` gap.
- A new write is unacked on the origin until `F` peers are `durable`, then the client receives success.
- Every peer eventually shows `applied`, including a peer that was not required for the quorum, by relay.
- The business row is readable on each node after that node's `applied` transition, not merely after `durable`.

A stamp excluded from capture must be written by the deploy script on every node. Read it back from each node.

## Kill tests

Run these against the real processes once.

1. Stop every peer. A write returns `503` (or times out). The origin keeps an `applied` row. Start one peer. That peer becomes `durable` and then `applied`. Retrying the same `op_id` does not create a second business row. A second `op_id` for the same `client_id` does append a new entry.
2. Complete a write with `F=1` so peer B has the bytes. Stop the origin before it can talk to peer C. B applies without the origin. C receives the entry from B. C's business row matches B after C applies.
3. Write different values for the same `client_id` on A and B while the link is cut. Restore the link. Both nodes show the winner of `(rev, origin)`. The loser is `superseded` and does not leave a `seq` hole.
4. Deliver `seq` 2 to a peer before `seq` 1. Status shows 2 durable and unapplied. After 1 arrives, both apply once, in order.
5. Restart a node during a push. Redelivery does not create a second business row.
6. Restore a backup with the snapshot procedure. The process does not replay a stray `-wal` file.

## Ongoing checks

- Age of the oldest entry that is locally `applied` and missing a peer `applied_at`.
- The blocking `seq` per origin, and any `failed` error.
- Disk free on the data directory. A full disk makes the durability rule false.
- Origin names still match the peer list after a rebuild.

## What you publish for operators

State in the service README, in this order:

- `F`, and that success means the origin plus `F` peers have committed the WAL bytes.
- That any node outside those copies can be missing the write until relay, and that the write is lost if every copy disappears first.
- That `ack=local` is not in use, or, if it is, the maximum acknowledged tail that one disk loss can erase.
- That concurrent edits of one `client_id` keep a single winner, and that counter fields lose increments.
- That reads from a node which has not applied the entry are stale.
- The backup command and the restore steps, including deletion of `-wal` and `-shm`.
- Where the credential lives, and that it is not in the client bundle.
