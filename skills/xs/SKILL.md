---
name: xs
description:
  "Work with cross.stream (xs), an event store for capturing information and
  running reactive workflows. Load when a task involves an xs store or
  evaluating xs for a proposed workflow. For example, xs can collect notes,
  messages, and updates from external sources into a shared history. Workflows
  can derive task lists or personal views from that history and trigger tools or
  agents when relevant events arrive. Covers reading and writing frames,
  services, actors, actions, transfers between stores, and store-hosted skills."
---

# cross.stream

cross.stream is a local-first event store for reactive workflows and automation.
It records frames in an ordered stream. Clients can read history and subscribe
to new arrivals. Processors can transform recorded information and append
results as new events. Queries and processors can also derive views from
recorded events.

A network can expose multiple stores belonging to different people and
organizations. The examples often pair a personal store with an
organization-wide store to illustrate how stores work together. Actual
arrangements can vary.

## Files and reading order

General reference material lives under `references/`. Worked examples live under
`examples/`. For a general introduction, read [store.md](references/store.md)
and [processors.md](references/processors.md).
[commands.md](references/commands.md) provides the command catalog.

1. Identify the target stores and the machine that will execute the work.
2. **Important:** Read [references/store-skills.md](references/store-skills.md)
   before accessing each store.
3. Read each store's skill catalog and load the applicable skill bodies.
4. Use the following table to select the remaining references.

An empty catalog permits work to continue. Catalog discovery is the first store
operation under this skill's recommended convention.

| Task                                         | Reading order                                                                                            | File responsibility                                               |
| -------------------------------------------- | -------------------------------------------------------------------------------------------------------- | ----------------------------------------------------------------- |
| Connect to a store or use HTTP               | [transport.md](references/transport.md)                                                                  | Local sockets, TCP, Iroh tickets, and the HTTP API                |
| Work through an MCP integration              | [processors.md](references/processors.md), then [mcp.md](examples/mcp.md)                                | Duplex requests, responses, and an actor reaction                 |
| Build threads or derived views               | [store.md](references/store.md), then [data-views.md](examples/data-views.md)                            | Executable conversation and task-state examples                   |
| Look up commands or options                  | [commands.md](references/commands.md)                                                                    | CLI command forms, options, and xs.nu helpers                     |
| Capture, query, or interpret information     | [store.md](references/store.md)                                                                          | Frames, content, topics, command interfaces, views, and retention |
| Implement or diagnose a workflow             | [store.md](references/store.md), then [processors.md](references/processors.md)                          | Processor definitions, state, calls, modules, and lifecycle       |
| Transfer selected information between stores | [store.md](references/store.md), then [between-stores.md](references/between-stores.md)                  | Selection, content transfer, import/append behavior, and recovery |
| Automate a transfer or execute work locally  | Transfer route above, then [processors.md](references/processors.md)                                     | Bridge execution and local reactions                              |
| Publish or update store skills               | [store.md](references/store.md), then [store-skills.md](references/store-skills.md), Publication section | Catalog format, file manifests, versions, and withdrawal          |

## Processing model

| Component   | Technical role                                 | Example uses                                                                                        |
| ----------- | ---------------------------------------------- | --------------------------------------------------------------------------------------------------- |
| Event store | Ordered frame history with live subscriptions  | Save notes, observations, and meeting details for later use                                         |
| Service     | Long-running producer or process connection    | Capture incoming chat messages or continuously copy selected events from an organization-wide store |
| Actor       | Event-driven closure with state between frames | Group incoming notes by project, update a task list, or trigger work when a matching event arrives  |
| Action      | Stateless operation invoked by a call frame    | Fetch a document, summarize supplied text, or run a tool when called                                |

A service can feed an actor. The actor can call an action and process its
result. Processor scripts define these connections.

**Examples: capture first, organize later.** These independent examples describe
two uses of recorded information:

- [Processing and enrichment](references/processors.md#example-processing-and-enrichment):
  an external language model classifies notes and extracts stated project names.
  A processor appends new frames with category and project fields in their
  metadata. Queries can then filter or group those frames by category or
  project.
- [Derived views](references/store.md#relationships-and-views): trace task
  dependencies to show blockers and affected tasks.

**Example: external integration.** A service can connect a task runner or a
Slack/WhatsApp connector to the stream. The integration supplies the external
program and its access credentials.

If deployed behavior differs, check `xs --version` and `xs version <addr>`.
