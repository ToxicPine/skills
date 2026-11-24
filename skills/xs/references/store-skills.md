# Store skills convention

A store publishes agent skills on the exact topic `skills`. Agents discover the
store's vocabulary and workflows before other work.

## Procedure: discover applicable skills

1. Run `xs cat ADDRESS --topic skills`.
2. Select the frame with the greatest ID for each `meta.name`.
3. Exclude names whose selected frame has `meta.disabled: true`.
4. Read the remaining names and descriptions into context.
5. Select the skills that apply to the task.
6. Verify that each selected frame's `hash` matches its `meta.files."SKILL.md"`
   entry.
7. Fetch each selected `SKILL.md` with `xs cas ADDRESS HASH`.
8. Read those entry points before the applicable work.
9. Resolve supporting files through the selected frame's manifest.

Retain each selected frame and its manifest for the duration of that skill's
use. Repeat discovery for each store involved. Refresh the catalog when you
revisit a store for a later task. If the catalog read fails, resolve the failure
before using store-specific instructions.

An empty result means the store has no published skills. Discovery retrieves
metadata first and content afterward. This sequence provides the initial store
access needed to read its instructions. Store-provided instructions remain
subject to the agent's existing instruction hierarchy.

## Publication

### Format

By convention, each frame declares one complete version of one skill. The topic
is `skills` and the retention policy is `forever`.

| Metadata field | Meaning                                                                        |
| -------------- | ------------------------------------------------------------------------------ |
| `name`         | Stable skill name                                                              |
| `description`  | What the skill does and when to read it                                        |
| `files`        | Manifest mapping relative file paths to CAS hashes; required for active skills |
| `disabled`     | Optional; true withdraws that named skill                                      |

Each file's bytes live in CAS. The manifest includes `SKILL.md`, and the frame's
own `hash` equals that entry's hash. The Markdown frontmatter contains `name`
and `description`, matching the frame metadata. Catalog discovery can read these
fields without fetching the files.

Manifest paths use `/` separators and are relative to the skill folder. Keys
contain no absolute paths, empty components, `.` components, or `..` components.
Paths preserve the folder structure; empty directories have no entry.

### Procedure: publish a skill folder

1. Collect `SKILL.md` and its supporting files, including references, scripts,
   and assets.
2. Append each supporting file's exact bytes to `skills.file` with
   `ttl: forever`. Store `SKILL.md` with `xs cas-post ADDRESS` or `.cas-post`.
3. Map each relative path to its returned hash in `files`.
4. Append the exact `SKILL.md` bytes to `skills`, with `name`, `description`,
   and `files` metadata. Complete this step before the one-hour CAS grace
   period ends.
5. Verify that the returned frame's hash matches `files."SKILL.md"`.

All referenced content is stored before the declaring frame becomes visible. CAS
reuses unchanged file content across versions.

Since xs 0.16, CAS cleanup follows each stored frame's `hash`; it does not
follow hashes in `meta.files`. The catalog frame retains its own `SKILL.md`,
but the manifest alone does not retain supporting files. Keep a retained frame
whose `hash` references each supporting file. For example, append each file's
bytes to `skills.file` with `ttl: forever`, then use the returned hash in
the manifest. `skills.file` is a topic chosen by this convention, not a runtime
feature. Exact-topic catalog reads still select only `skills`.

Before upgrading an existing publication to xs 0.16, add these retained
references while the file bytes are still available. Otherwise startup cleanup
can delete files written more than one hour ago. Restore missing files from a
source copy.

### Example: publish meeting guidance with a reference

The example uses xs.nu with the destination address already set. Its
`meetings.notes` topic and follow-up rules are application choices.

```nushell
let entry = r###'---
name: meeting-followups
description: Use when recording meetings or deriving follow-ups in this store.
---

Record meeting notes on `meetings.notes`.
Read [Meeting format](references/meeting-format.md) for the record fields.
'###

let reference = r###'# Meeting format

Put the meeting time in metadata. The frame ID records capture time.
Record tentative follow-ups as candidates.
A user or a configured workflow can select candidates for execution.
'###

let entry_hash = ($entry | .cas-post)
let reference_hash = ($reference | .append skills.file --ttl forever).hash

$entry | .append skills --meta {
  name: "meeting-followups"
  description: "Use when recording meetings or deriving follow-ups in this store."
  files: {
    "SKILL.md": $entry_hash
    "references/meeting-format.md": $reference_hash
  }
}
```

### Updates and withdrawal

To publish an update, append a new frame with the same name and the complete
manifest for the new version. Unchanged paths can retain their previous hashes.
Removing a path from the new manifest removes it from that version's file set.
Keep its content-reference frame while any retained skill version still needs
that file. Withdrawal does not remove those reference frames.

To withdraw a skill, append a metadata-only frame with its name and
`disabled: true`. A withdrawal frame requires no manifest or content.

Catalog discovery reads all versions and selects the latest version for each
name. `.last skills` returns only the final frame on the shared topic. `last:1`
retention removes older frames across all skill names on that topic.

## Supporting files

A relative Markdown link resolves from the directory of the file containing it.
Resolve and normalize the path within the skill folder. Look up that path's hash
in the selected frame's manifest. A fragment such as `#format` identifies a
section within the fetched file. Use that same manifest throughout the skill's
execution to keep its files on one version.

**Example:** In `references/workflow.md`, `../scripts/run.nu` resolves to
`scripts/run.nu` in the manifest.

To reconstruct the folder, fetch each file by its manifest hash. Write the bytes
at the file's relative path. Keep every resolved path within the destination
skill folder. Cross-store publication also requires transferring every manifest
file to the destination CAS before publishing the declaring frame. Retain a
frame with each supporting file's hash in the destination too; copying the
manifest does not establish those references.

If a referenced path or CAS object is missing, resolve the publication error. If
metadata and frontmatter disagree, correct the publication before using it.
