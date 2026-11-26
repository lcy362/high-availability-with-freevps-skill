# WAL protocol

The replication unit is one business request, stored as a row, replayed through the existing handler.

## Entry

| Field | Meaning |
|---|---|
| `id` | `origin + ":" + seq`. Primary key. |
| `origin` | Stable node name. Only that node allocates `seq` for this origin. |
| `seq` | Next integer for that origin, assigned in the same locked transaction as the insert. |
| `rev` | Conflict clock. `max(unix_millis, current_entity_rev+1)` under the apply lock. |
| `client_id` | Entity key for LWW. Many operations share one entity. |
| `op_id` | One client attempt. Retries reuse it. A new edit mints a new one. |
| `operator` | Caller identity header, restored on replay. |
| `entity` | Logical type. |
| `op` | `create`, `update`, or `delete`. Derived from the method. |
| `method`, `path` | Replay target. |
| `body` | Original body plus stable foreign-key markers. NOT NULL. |
| `status` | `stored`, then `applied`, `superseded`, `rejected`, or `failed`. |
| `applied_at` | Set when this node finishes apply, marks superseded, or marks rejected. Empty while only the bytes are stored. |
| `apply_error` | Last apply or gap error. Cleared on a successful apply. |
| `created_at` | Insert time on the node that first wrote the row. Preserved across relay. |

Ack, one row per holder per peer:

| Field | Meaning |
|---|---|
| `wal_id` | Entry id. |
| `peer` | Peer origin name. |
| `durable_at` | That peer committed the bytes. |
| `applied_at` | That peer finished the handler or marked the entry superseded. Empty if only durable. |

`durable_at` satisfies the client quorum. `applied_at` retires the push. A single flag cannot say both, and a push must keep going while apply is still failing or waiting on a `seq` gap.

## Tables

```sql
CREATE TABLE wal_entries (
  id            TEXT PRIMARY KEY,
  origin        TEXT NOT NULL,
  seq           INTEGER NOT NULL,
  rev           INTEGER NOT NULL,
  client_id     TEXT NOT NULL,
  op_id         TEXT NOT NULL,
  operator      TEXT NOT NULL DEFAULT '',
  entity        TEXT NOT NULL,
  op            TEXT NOT NULL,
  method        TEXT NOT NULL,
  path          TEXT NOT NULL,
  body          BLOB NOT NULL,
  status        TEXT NOT NULL DEFAULT 'stored',
  applied_at    TEXT NOT NULL DEFAULT '',
  apply_error   TEXT NOT NULL DEFAULT '',
  created_at    TEXT NOT NULL
);
CREATE UNIQUE INDEX idx_wal_origin_seq ON wal_entries(origin, seq);
CREATE INDEX idx_wal_op_id ON wal_entries(op_id);

CREATE TABLE wal_peer_ack (
  wal_id      TEXT NOT NULL,
  peer        TEXT NOT NULL,
  durable_at  TEXT NOT NULL DEFAULT '',
  applied_at  TEXT NOT NULL DEFAULT '',
  PRIMARY KEY (wal_id, peer)
);

CREATE TABLE wal_entity_state (
  entity    TEXT NOT NULL,
  client_id TEXT NOT NULL,
  rev       INTEGER NOT NULL,
  origin    TEXT NOT NULL,
  wal_id    TEXT NOT NULL,
  deleted   INTEGER NOT NULL DEFAULT 0,
  PRIMARY KEY (entity, client_id)
);

CREATE TABLE wal_id_map (
  entity    TEXT NOT NULL,
  client_id TEXT NOT NULL,
  local_id  INTEGER NOT NULL,
  PRIMARY KEY (entity, client_id)
);
CREATE INDEX idx_wal_id_map_local ON wal_id_map(entity, local_id);
```

`wal_entity_state` is the version truth. Business tables are the materialization of the winning entry.

## Who wins

```text
wins(rev, origin, current):
  if current is missing: true
  if rev != current.rev: return rev > current.rev
  return origin > current.origin
```

Run the business handler only when this is true at apply time, not only at capture time. A loss still becomes `superseded` and is acked `applied`, so holders stop retrying it. The row stays in the log. Later sequences from that origin must not see a hole.

Upsert entity state only inside that same winner check. A late, older entry must not move `rev` backward.

Same origin, same `rev` must not happen. The apply lock assigns a strictly newer `rev` before the next write starts. The string tie-break is for two origins.

## Apply order

For each origin, let `next_seq` be one more than the highest `seq` that is `applied`, `superseded`, or `rejected`.

- An entry with `seq > next_seq` may be committed. It stays unapplied.
- An entry with `seq = next_seq` is the only one that may be applied. `rejected` counts as applied for this cursor: record it and move on, with no handler call.
- A `failed` entry at `next_seq` blocks that origin. Fix the cause and retry the same row. Do not skip it.
- A local sweep watches committed rows and applies them when they become `next_seq`. Do not wait for a fresh network delivery.

Delete of a missing entity is successful and idempotent. Record entity state as deleted when this entry wins. A `404` from a blind delete must not become `failed`, or it blocks every higher `seq`.

## Who pushes

Push an entry this node has committed when any configured peer lacks `applied_at` for it. The origin usually pushes first. A peer that stored the bytes pushes too. Skip the peer that just delivered the same bytes in this request. Do not skip it forever: if that peer's other deliveries fail, the normal sweep still pushes.

Order a given origin's batch by `seq`. Stop that origin's batch at the first entry that is not yet `applied`, `superseded`, or `rejected`. Other origins in the same process continue.

A push error leaves acks unchanged. Batch about 200. Retry on an interval (60 seconds unless the product needs a shorter lag). Quorum wait during a client request is separate and uses a short timeout.

## Ingest HTTP

`POST /api/wal/ingest` with the same application credential as business writes.

`body` is base64 in JSON so an empty body, a missing field, and binary bytes stay distinct. Both sides use that encoding.

```json
{
  "entries": [
    {
      "id": "n1:1",
      "origin": "n1",
      "seq": 1,
      "rev": 1,
      "clientId": "c1",
      "opId": "op-1",
      "entity": "document",
      "op": "create",
      "method": "POST",
      "path": "/api/documents",
      "body": ""
    }
  ]
}
```

```json
{
  "acks": [
    { "id": "n1:1", "durable": true, "applied": true }
  ]
}
```

| `durable` | `applied` | Sender does |
|---|---|---|
| false | false | Keep pending. This attempt did not land. |
| true | false | Count it toward `F`. Keep pushing until apply or supersede. |
| true | true | Count it toward `F`. Stop pushing this id to that peer. |

`GET /api/wal/status` returns this node's origin, its peer list, the oldest unapplied `seq` per origin, and recent entries with per-peer `durable_at` and `applied_at`.

## Capture exclusions

Skip when any of these hold:

- Method is GET, HEAD, or OPTIONS.
- Path is the ingest or status route.
- Path is admin, export, import, or another node-local control.
- The request context carries the internal replay mark.

If the route is a business write but `client_id` cannot be resolved, fail the request.

## Foreign keys

```text
capture:  { "parentId": 4, "_parentClientId": "p-abc" }
replay:   lookup wal_id_map("document", "p-abc") -> local id on this node
          handler sees { "parentId": <local> }
```

Keep both fields in the stored body. Strip markers before the handler runs. If `_parentClientId` is present and the map has no row, the referenced create is not applied here yet. Leave this entry durable and unapplied.

## Replay request

- Method, path, and translated body from the entry.
- Application auth header, so the replay passes the same gate as a client.
- Operator header from the entry.
- Internal context mark set before the handler chain.
- Do not trust a client-supplied header as that mark.

## Status transitions

```text
insert committed                         -> stored          (durable for holders)
origin or peer handler 2xx, still wins   -> applied
loses LWW                                -> superseded      (applied ack, no handler)
handler non-2xx, no business commit      -> rejected        (applied ack, no handler, seq consumed)
handler non-2xx after the bytes were
  committed on a peer                    -> failed          (durable, not applied)
process restart on stored                -> rerun handler, then applied
```

A client retry reuses `op_id` and may contact any node. If that `op_id` is already `applied` or `rejected`, return the stored result and do not allocate another `seq`. A new edit of the same `client_id` uses a new `op_id` and does allocate a `seq`. Two nodes can both accept the same `op_id` during a partition. Apply runs the handler for the first copy only. The other copy converges on replay. Do not treat "this `client_id` already has an applied entry" as a retry. That would drop every later edit of the entity.
