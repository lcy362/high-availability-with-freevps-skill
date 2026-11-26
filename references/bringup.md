# Bring-up

Do this once per service. Repeat the verification after any change to capture, ingest, or the ack rule.

## Layout

- One process per VPS, supervised so it restarts on boot.
- Unique `origin` per node, chosen once and never renamed. Renaming forks the seq space and the LWW tie-break.
- Peer list is every other node's ingest base URL. Trim trailing slashes.
- Same binary (compare a hash) and same schema on every node before enabling the push loop.
- Application credential only in the node environment file.
- Clocks stepped with NTP before the first write.

## First data

Pick one node as the snapshot source. Stop writes.

1. Take a consistent snapshot (`VACUUM INTO` or the engine backup API), not a file copy.
2. Install that single file on every other node. Delete any `-wal` and `-shm` beside the destination.
3. Start each process so it can seed `wal_id_map` from the copied rows.
4. Turn on the push loop only after every node is serving that same snapshot.

Two nodes that already diverged cannot be merged by turning WAL on. The log does not contain their past. Choose one snapshot, accept the other side's extra rows as lost, or generate synthetic WAL entries on purpose and review them one by one.

## Deploy

Build once. Ship that artifact to every node. Restart one node at a time. A restarted node keeps its `origin` and its seq counter (the counter is `MAX(seq)` in the table, so it survives restart).

After rollout, check each node:

- Health responds.
- Status shows the expected `origin` and every peer.
- A new write on node A appears in A's pending list, then disappears from pending after the peer acks, and the business row is readable on every peer.

A stamp that clients use to decide which build is current, and that is excluded from capture, must be written by the deploy script on every node. Verify by reading it back from each node.

## Kill tests

Run these against the real processes once.

1. Stop peer B. Write on A. Confirm the client outcome matches the durability rule (503 or timeout in quorum mode; 200 plus a growing pending list in async mode). Start B. Confirm the row arrives and pending drains.
2. Stop A after a successful quorum write. Read the entity on B. It must be there even if B has not finished apply yet; if apply is still pending, it must finish without A.
3. Write different values for the same `client_id` on A and B while the link is cut. Restore the link. Both nodes show the winner of `(rev, origin)`. The loser is `superseded`.
4. Restart a node in the middle of a push. Redelivery does not create a second business row.
5. Restore a backup onto one node using the snapshot procedure. Confirm the process does not replay a stray `-wal` file.

## Ongoing checks

- Pending age. An entry applied locally and unacked longer than a few intervals means the peer is down, rejecting, or failing apply.
- `failed` count and `apply_error`.
- Disk of the data directory. A full disk turns the durability rule into a lie.
- Origin names still match the configured peer acks after a rebuild.

## What you publish for operators

State in the service README, in this order:

- How many nodes must be up for a write to succeed (one peer commit, or local only).
- The maximum acknowledged-data loss if that rule is violated (async interval, or none under quorum).
- That concurrent edits of one `client_id` keep a single winner.
- The exact backup command and the exact restore steps, including deletion of `-wal` and `-shm`.
- Where the credential lives, and that it is not in the client bundle.
