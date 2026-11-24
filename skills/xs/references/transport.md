# Transport and remote access

## Addresses and execution

Store clients accept a local store path, a TCP address, or an Iroh ticket. The
CLI takes the address as an argument. External xs.nu helpers use `XS_ADDR`. Eval
and processor scripts execute on the server at that address.

| Address form     | Transport                                         |
| ---------------- | ------------------------------------------------- |
| `./store`        | HTTP over the store's Unix socket, `./store/sock` |
| `127.0.0.1:3021` | HTTP over TCP                                     |
| `iroh://...`     | Peer-to-peer QUIC using a connection ticket       |

Iroh supports connections across NATs and firewalls. The full ticket identifies
the connection. An abbreviated ticket is only a placeholder.

## Example: TCP access

This server exposes its API on a loopback address:

```sh
xs serve ./store --expose 127.0.0.1:3021
```

In another terminal, clients can read, append, and evaluate against that
address:

```sh
xs cat 127.0.0.1:3021 --topic notes
printf '%s' 'Meeting note' | xs append 127.0.0.1:3021 notes
xs eval 127.0.0.1:3021 -c '.last notes'
```

For remote clients, the server's exposed host address must be reachable from the
client.

## Example: discover and use an Iroh ticket

This server exposes an Iroh endpoint:

```sh
xs serve ./store --expose iroh://
```

In a Nushell session with xs.nu loaded, read the startup frame through the local
store address:

```nushell
$env.XS_ADDR = "./store"
let ticket = (.last xs.start).meta.expose
```

The `expose` field contains the complete `iroh://` ticket. The same session can
use it through either command interface:

```nushell
xs cat $ticket --topic notes
$env.XS_ADDR = $ticket
"hello over Iroh" | .append notes
.last notes
```

A client on another machine can use the same ticket as its address.
[Between stores](between-stores.md) describes transferring selected frames after
connecting.

## Example: HTTP append and subscription

The append endpoint receives content in the request body. The `xs-meta` header
contains standard Base64-encoded JSON metadata.

```sh
xs_meta=$(printf '%s' '{"author":"alice"}' | base64 | tr -d '\n')
curl --fail --unix-socket ./store/sock \
  -H "xs-meta: $xs_meta" \
  --data-binary 'Discuss the budget' \
  'http://localhost/append/notes?ttl=forever'
```

This request reads notes and follows new appends as SSE:

```sh
curl --fail --no-buffer --unix-socket ./store/sock \
  -H 'Accept: text/event-stream' \
  'http://localhost/?topic=notes&follow=true'
```

For a TCP endpoint, use its HTTP URL and omit `--unix-socket`.

## HTTP access

The HTTP API exposes store operations to clients:

| Request                         | Operation                                                                                                   |
| ------------------------------- | ----------------------------------------------------------------------------------------------------------- |
| `GET /`                         | Read frames with `topic`, `from`, `after`, `new`, `limit`, `last`, `follow`, `with-timestamp` query options |
| `POST /append/{topic}`          | Append request-body content; `xs-meta` is standard Base64-encoded JSON metadata; `ttl` is a query option    |
| `GET /{id}` / `DELETE /{id}`    | Retrieve/remove a frame                                                                                     |
| `GET /last[/{topic}]`           | Latest frame, or latest N using `last=N`                                                                    |
| `POST /cas` / `GET /cas/{hash}` | Store/retrieve raw content                                                                                  |
| `POST /append-batch`            | Append up to 1024 NDJSON frames in one commit                                                               |
| `POST /gc`                      | Delete unreferenced CAS content older than one hour                                                        |
| `POST /import`                  | Import one JSON frame verbatim                                                                              |
| `POST /eval`                    | Evaluate the UTF-8 Nushell script in the request body                                                       |
| `GET /version`                  | Server version                                                                                              |

Stream reads return newline-delimited JSON, or SSE with
`Accept: text/event-stream`. HTTP `follow` accepts `true` or a heartbeat
interval in milliseconds. `GET /last` returns one JSON frame by default and
NDJSON for `last` greater than 1. It also accepts `follow` and `with-timestamp`.
`GET /{id}` and append responses accept `with-timestamp`.

A following HTTP read that falls behind sends `xs.lagged` with `meta.skipped`
as its last frame, then closes. A request with `TE: trailers` also receives an
`xs-lagged: N` trailer. Resume from the last handled source frame, as described
in [read recovery](store.md#read-history-then-follow).

`POST /append-batch` takes one JSON object per line. Each object requires
`topic` and can include `meta`, `ttl`, and `hash`. Store content with `POST /cas`
first, then put its hash in the corresponding object. Do not supply frame IDs.
The server assigns consecutive IDs and commits the batch together. A malformed
line or more than 1024 frames returns 400. An invalid topic also rejects the
whole batch; xs 0.16.0 returns 500 for that validation failure. These rejected
batches store no frames. The response is NDJSON in append order.
`with-timestamp` is also supported.

**Example: append two metadata-only frames.**

```sh
printf '%s\n' \
  '{"topic":"notes","meta":{"text":"Discuss the budget"}}' \
  '{"topic":"notes","meta":{"text":"Confirm the date"}}' |
  curl --fail --unix-socket ./store/sock --data-binary @- \
    'http://localhost/append-batch'
```

`POST /gc` returns `in_use`, `recent`, `deleted`, and `bytes_freed` counts.
It keeps content referenced by a stored frame's `hash`, and unreferenced content
written within the last hour. Metadata hashes do not retain content. The server
also runs this cleanup at startup. See [CAS retention](store.md#frames-and-content).

Successful reads return 200. Removal returns 204. Malformed requests return 400;
missing frames return 404; server errors return 500. In xs 0.16.0, a missing
CAS object can close the connection without an HTTP response.

| Eval result                | HTTP representation |
| -------------------------- | ------------------- |
| String, number, or boolean | Plain text          |
| Other single value         | JSON                |
| List stream                | NDJSON              |
| Byte stream                | Raw bytes           |
| No output                  | 204                 |
| Evaluation error           | 500                 |
