# Between stores

## Example: shared inputs and local execution

A local service follows selected events from an organization-wide store and
writes them to a personal store. Local actors derive views or trigger tools on
that device. A separate workflow can publish results back to the organization-wide store.

Processors execute on the server where their definitions are registered.

**Recommendation:** Register the bridge and processors on the personal server
when the work requires that person's device.

## Select source frames

[Transport](transport.md) describes addresses, connections, and Iroh ticket discovery.
The source and destination each have their own skill catalog.

**Example:** The source is `organization-host:3021`; the destination is
`./personal-store`. This command reads selected history and then follows live
appends:

```sh
xs cat organization-host:3021 --topic 'notes,meetings.*' --follow
```

**Recommendation:** Select application topics for routine data intake. Apply
additional filters to metadata or content as needed. Treat runtime definitions
and store skills as separate publication decisions.

## Import and append behavior

| Operation | Result | Example purpose |
| --- | --- | --- |
| Import | Preserves the original ID, metadata, topic, and hash | Restore selected history |
| Append | Creates a local ID and notifies subscribers | Trigger a local workflow |

Import stores a frame without notifying live subscribers. Existing local actors
continue waiting for appends. An explicit historical read or replay can process
the imported frames. A separate appended event can also announce completed intake.

**Recommendation:** Use append for routine reactive intake. Preserve source
identity in the local frame's metadata. Use a stable source label without
credentials. Preserve the original metadata alongside that identity.

**Example provenance field:**
`source: {store: "organization", id: <original-id>, topic: <original-topic>}`.
The application defines this field. Source IDs and local IDs identify positions
in their respective streams.

## Procedure: transfer a frame with its original ID

A frame's hash identifies content in CAS. The destination needs those bytes to
resolve the hash. Metadata-only frames require only the frame transfer.

For each selected frame:

1. If the frame has a hash, fetch its bytes with `xs cas SOURCE HASH`.
2. If the frame has content, post the exact bytes with `xs cas-post DESTINATION`.
3. If you posted content, verify that the returned hash matches the source hash.
4. Pipe the frame's JSON into `xs import DESTINATION`.

The import endpoint accepts one JSON object per invocation.

## Procedure: append a local copy

1. Choose the local topic and retention policy.
2. Prepare the metadata, including the source identity.
3. If the source frame has a hash, fetch its bytes with `xs cas SOURCE HASH`.
4. Pass the bytes to `xs append DESTINATION TOPIC` with the prepared metadata and retention policy.

For a metadata-only frame, omit the content input in step 4. Append stores any
content in the destination CAS and creates its reference. Text conversions can
change raw bytes and their hashes.

## Whole-store transfer

External xs.nu `.export <directory>` writes `frames.jsonl` and `cas/`.
`.import <directory>` restores both. Selective transfers use the individual read,
content, and write operations described above.

## Recovery behavior

A disconnected reader can resume from retained history with
`--after SOURCE_ID --follow`. Expired and ephemeral events are unavailable for
recovery. A crash between a local append and a cursor update can cause redelivery.
Re-importing the same ID updates the stored frame. External effects require
separate duplicate handling.

An imported frame with an ID at or before the cursor falls outside an `--after`
read. Import also produces no live notification. Such frames require a historical
rescan or a fresh appended notification. Imported timestamps retain their original
values, so time-based expiry remains relative to source creation.

### Recommended recovery procedure

1. Keep a durable cursor for each source and selection.
2. Exclude synthetic `xs.threshold` and `xs.pulse` frames from copying and cursor updates.
3. Store each selected frame's content, then write its destination frame.
4. After the write succeeds, save the source frame ID as the cursor.
5. On reconnection, read with `--after SOURCE_ID --follow`.
6. Deduplicate intake and external work by `(source store, source frame ID)`.
7. If the selection expands to older data, perform a historical read for that data.
