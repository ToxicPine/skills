# Store operations

## Frames and content

The event store maintains an append-only index of frame metadata in fjall.
Each frame has a topic, a time-ordered SCRU128 ID, optional metadata,
an optional content hash, and retention settings.

Content lives separately in content-addressable storage (CAS). Content is
immutable, referenced by hash, and deduplicated. Reading frames retrieves their
metadata. A separate CAS read retrieves content.

**Recommendation:** Use metadata for records, numbers, and small strings.
Use CAS for large text, images, and other blobs. Store application dates in
metadata. The frame's timestamp identifies its creation time.

**Example: task state.** A task-completion event refers to an earlier
creation event. A view combines these events to show the task's current state.

## Command surfaces

[Commands](commands.md) lists every documented CLI command and xs.nu helper,
including argument forms and options.

The CLI takes a store address explicitly: a local store path, TCP address, or
`iroh://` ticket. [Transport](transport.md) covers connections and the HTTP API.

### Example: CLI operations

These commands assume the target store is running. Each line shows an independent
operation. The command with `--follow` continues until interrupted:

```sh
xs cat ./store --topic notes
xs cat ./store --topic 'notes,meetings.*' --follow
printf '%s' 'Ask Sam about the budget' | xs append ./store notes
xs last ./store notes
xs get ./store FRAME_ID
xs cas ./store CONTENT_HASH
```

Capitalized IDs/hashes in examples stand for values obtained from frames.

### Example: Nushell operations

These commands load xs.nu, select a store, append a note, and read its topic:

```nushell
use xs.nu *
$env.XS_ADDR = "./store"
"Ask Sam about the budget" | .append notes --meta {author: "alice"}
.cat --topic notes
```

### Execution context

Inside `xs eval <addr> -c '<script>'` and processor scripts, store commands are
builtins bound to that server's store. They include `.append`, `.cat`, `.last`,
`.get`, `.cas`, `.cas-post`, `.remove`, and `.import`. External `xs.nu` wrappers
use `XS_ADDR`. The external `.import <directory>` restores an export directory.
Inside eval, `$frame | .import` imports one frame.
Remote eval runs on the remote server. External xs.nu commands return structured
Nushell values. CLI frame output is JSON. The xs.nu `.cat --detail` option includes
all frame fields, including TTL.

### Additional operations

`xs remove ADDRESS ID` (or `.remove ID`) deletes a frame.
`xs scru128` generates an ID. `xs scru128 unpack ID` inspects an ID.
The xs.nu equivalents `.id`, `.id unpack`, and `.id pack` support native Nushell
datetime values. `.tmp-spawn { ... }` runs a closure against an isolated temporary
store and cleans it up.

## Read history, then follow

Reads return historical frames in ID order. `--follow` continues with live
appends; `xs.threshold` marks the transition from history to live delivery.

| Option | Selection |
| --- | --- |
| `--from ID` | Start at ID, inclusive |
| `--after ID` | Start after ID, exclusive |
| `--new` | Skip history |
| `--limit N` | Stop after N frames |
| `--last N` | Most recent N, returned chronologically |
| `--topic PATTERNS` | Exact topic, prefix wildcard, or comma-separated list |
| `--with-timestamp` | Include timestamp extracted from the ID |
| `--follow --pulse MS` | Follow with periodic heartbeat frames |

Topics use letters, digits, `_`, `-`, and `.`. A topic starts with a letter or
`_`. It cannot end in `.` or contain `..`. The maximum length is 255 bytes.
The runtime reserves `xs.*` for lifecycle, modules, and system markers.

| System topic | Meaning |
| --- | --- |
| `xs.start` | Server initialization is complete |
| `xs.threshold` | A read has reached the end of historical replay |
| `xs.pulse` | Synthetic heartbeat for a following read |
| `xs.stopping` | The server is shutting down |

**Example: topic selection.** `user.*` selects descendants such as
`user.alice.notes`. Selecting both the parent and its descendants requires
`user,user.*`.

Following reads may deliver synthetic `xs.threshold` and `xs.pulse` frames even
with a topic filter. `--new` skips the historical replay and its threshold marker.
The server disconnects a slow reader that exhausts the 4,096-frame live buffer.
Recovery reads can retrieve retained frames after the last successfully handled
source ID. Expired and ephemeral frames are unavailable for recovery.

## Relationships and views

Metadata can link frames.

**Example: threaded conversations.** Each turn uses `meta.role` and `meta.continues`, where
`continues` holds the preceding turn's ID. These fields are application conventions.
Two turns can continue the same predecessor, creating branches.

Procedure to read a thread:

1. Read the final turn with `.get`.
2. Follow each `meta.continues` ID with `.get` until you reach the first turn.
3. Reverse the collected frames into conversation order.
4. Fetch each turn's content with `.cas`.

**Recommended pattern: interactive views.** Reduce events into current state.
Follow new frames to update that state. A processor can publish the state, or
a client can derive it during reads.

**Example: task-list interface.** A server renders the current task state as
HTML and streams updates to clients over SSE. New task events update the
projection and the displayed list.

[Worked examples](../examples/data-views.md) show thread reconstruction, branching,
and a task-state projection.

## Retention

| TTL | Meaning |
| --- | --- |
| `forever` | Retain indefinitely (default) |
| `ephemeral` | Broadcast only; no history |
| `time:<ms>` | Expire relative to the frame's creation timestamp |
| `last:<n>` | Retain the newest N on that topic; N must be at least 1 |

The `.append --ttl` and `xs append --ttl` options set retention.
A frame's TTL is fixed when it is appended.
`last:n` applies to all frames on the topic.
Time-expired frames are hidden immediately on reads; disk cleanup runs in the
background. `last:n` trimming is asynchronous, so reads may briefly see more
than N frames. Explicit removal and retention policies can delete history.

**Recommendation:** Retain original inputs when later reprocessing is needed.
Use `last:1` for a topic that holds one derived current-state value.

Appends commit to the journal before returning, so a process crash loses no
committed frames. Power loss or a kernel crash can lose writes that have not been
fsynced; recovery returns a stable prefix of the stream. The default fsync
interval is one second. `xs serve --fsync always` fsyncs each append before
returning.
