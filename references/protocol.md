# WAL protocol

The replication unit is one business request, stored as a row, replayed through the existing handler.

## Entry

| Field | Meaning |
|---|---|
| `id` | `origin + ":" + seq`. Primary key. |
| `origin` | Stable node name. Only that node allocates `seq` for this origin. |
| `seq` | `MAX(seq)+1` for that origin, assigned inside the insert transaction. |
| `rev` | Conflict clock. `max(unix_millis, current_entity_rev+1)` at capture. |
| `client_id` | Stable entity key. For an op stream this is the op id, not a row client id. |
| `operator` | Caller identity header, restored on replay. |
| `entity` | Logical type: `member`, `txn`, `settings`, … |
| `op` | `create`, `update`, or `delete`. Derived from the HTTP method. |
| `method`, `path` | Replay target. |
| `body` | Original body plus stable foreign-key markers. NOT NULL. Use empty bytes when absent. |
| `status` | `stored`, then `applied`, `superseded`, or `failed`. |
| `applied_at` | Set when this node finishes apply or marks superseded. |
| `apply_error` | Last apply failure. Cleared on a successful apply. |
| `created_at` | Insert time on the node that wrote the row. |

Per-peer ack (required for three or more nodes; use it for two as well):

| Field | Meaning |
|---|---|
| `wal_id` | Entry id. |
| `peer` | Peer origin name. |
| `acked_at` | When that peer reported the id applied or superseded. |

wish-bank uses a single `peer_applied_at` on the entry instead of this table. That matches one peer only.

## Tables

```sql
CREATE TABLE wal_entries (
  id            TEXT PRIMARY KEY,
  origin        TEXT NOT NULL,
  seq           INTEGER NOT NULL,
  rev           INTEGER NOT NULL,
  client_id     TEXT NOT NULL,
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

CREATE TABLE wal_peer_ack (
  wal_id   TEXT NOT NULL,
  peer     TEXT NOT NULL,
  acked_at TEXT NOT NULL,
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

Apply the business handler only when this returns true. A loss still becomes `superseded` and is acked, so the origin stops pushing it.

Upsert entity state only inside that same winner check. A late, older entry must not move `rev` backward.

## Pending query

Push entries where:

- `origin` is this node
- `applied_at` is set
- `status` is not `failed`
- a configured peer has no `wal_peer_ack` row

Order by `seq` ascending. Batch about 200. Retry the batch on the next interval (60 seconds unless product latency needs faster). A push error leaves acks unset.

## Ingest HTTP

`POST /api/wal/ingest` with the same application credential as business writes.

```json
{ "entries": [ { "id": "n1:1", "origin": "n1", "seq": 1, "rev": 1, "clientId": "c1", "entity": "member", "op": "create", "method": "POST", "path": "/api/members", "body": "" } ] }
```

`body` is raw bytes. Encode it the same way on both sides (base64 in JSON, or a byte-preserving encoding). A text JSON body is fine if both nodes agree.

```json
{ "acks": [ { "id": "n1:1", "applied": true } ] }
```

`applied: false` or a missing id keeps the entry pending. Log `error` on the sender. Do not mark the peer acked.

`GET /api/wal/status` returns this node's origin, its peer list, and recent entries including status and which peers have acked. Use it to see which id is stuck.

## Capture exclusions

Skip when any of these hold:

- Method is GET, HEAD, or OPTIONS.
- Path is the ingest or status route.
- Path is admin, export, import, or another node-local control.
- The request context carries the internal replay mark.

If the route is a business write but `client_id` cannot be resolved, fail the request. Shipping it without an entity key makes LWW undefined.

## Foreign keys

```text
capture:  { "memberId": 4, "_memberClientId": "m-abc" }
replay:   lookup wal_id_map("member", "m-abc") -> local id on this node
          handler sees { "memberId": <local> }
```

Keep both the local field and the marker in the stored body. Strip markers before the handler runs.

## Replay request

- Method, path, and translated body from the entry.
- Application auth header, so the replay passes the same gate as a client.
- Operator header from the entry.
- Internal context mark set before the handler chain.
- Do not trust a client-supplied header as that mark. Use a context value that only this process can set.

## Status transitions

```text
origin insert                         -> stored
origin handler 2xx + state updated    -> applied
origin handler not 2xx                -> row deleted
peer lost LWW                         -> superseded   (ack)
peer replay 2xx                       -> applied      (ack)
peer replay failed                    -> failed       (no ack; redelivery replaces the row)
```

A `stored` row whose handler never finished (process crash) is not pending, because `applied_at` is empty. The client did not get success. A client retry allocates a new seq.
