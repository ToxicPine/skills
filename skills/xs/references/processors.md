# Services, actors, and actions

## Register a processor

A processor definition is Nushell source text. The script returns a
configuration record containing a `run` closure. The runtime calls that closure
when the processor executes.

1. Write the configuration as a Nushell script.
2. Append the source text to `xs.<kind>.<name>.create`.
3. Wait for the corresponding `active` or `invalid` acknowledgment.

`<kind>` is `service`, `actor`, or `action`. `<name>` identifies the processor.
The append stores the script in CAS. The server evaluates it and registers the
processor. The processor executes on that server.

The examples use xs.nu with `XS_ADDR` set. The `r#'...'#` syntax creates a raw
string containing the script. Piping that string into `.append` publishes the
source text. The code inside `run` executes later, under the processor's
lifecycle.

## Services: produce and connect

Services run pipelines that produce frames.

**Example: file monitoring.** This service writes each log line to CAS:

```nushell
r#'{
  run: {|| ^tail -F http.log | lines }
  return_options: {target: "cas"}
}'# | .append xs.service.logs.create
```

Outputs go to `<name>.recv`. The default output target expects records and
stores them in frame metadata; `target: "cas"` accepts text or binary content.
`duplex` defaults to false. With `duplex: true`, content appended to
`<name>.send` is fed to the process's stdin. The external tool determines its
message format, such as newline-delimited JSON. Service output streams as it is
produced. Actor output is buffered until the invocation completes. Action output
is collected into one response.

Explicit `.append` calls inside a service merge `service_id` into the supplied
metadata to identify the service that wrote the frame.

**Example: MCP integration.** A duplex service connects a stdio MCP server. It
carries newline-terminated JSON-RPC requests on `<name>.send` and response lines
on `<name>.recv`. Response IDs identify their requests. An actor can process the
responses and trigger further work. The [worked MCP example](../examples/mcp.md)
shows the full request, response, and actor sequence.

The default `restart: "on-success"` restarts a cleanly completed pipeline after
one second. `restart: "never"` makes clean completion terminal. A pipeline error
emits `fin.error` and stays down under either policy.

## Actors: react and carry state

An actor consumes frames sequentially. Its `run` closure receives exactly two
positional arguments: the incoming frame and the carried state. It returns
`{out?, next?}`. The question marks indicate optional fields:

| Return                      | Effect                                 |
| --------------------------- | -------------------------------------- |
| `{out: value, next: state}` | Emit output and continue               |
| `{next: state}`             | Continue without output                |
| `{out: value}` or nothing   | Stop, optionally emitting final output |

**Example: running total.** This actor adds retained sale amounts and publishes
the current total.

```nushell
r#'{
  topics: ["sale"]
  start: "first"
  initial: 0
  run: {|frame, sum|
    if $frame.topic != "sale" { return {next: $sum} }
    let sum = $sum + ($frame.meta.amount | into float)
    {out: {total: $sum}, next: $sum}
  }
  return_options: {suffix: ".total", ttl: "last:1"}
}'# | .append xs.actor.revenue.create
```

The output record is metadata on `revenue.total`. After
`xs.actor.revenue.active`, append two sale events:

```nushell
.append sale --meta {amount: 49.99}
.append sale --meta {amount: 12.50}
```

After the actor processes both events, read the total:

```nushell
.last revenue.total | get meta.total
```

With no earlier sale events, the result is `62.49`. **Example uses:** The same
pattern can maintain task lists, summaries, and personal views.

Initial state comes from `initial`, then the closure's second-parameter default,
then `null`. `start` defaults to `"new"`. `"first"` replays retained history. A
frame ID starts after that ID. An actor always follows live appends.

`topics` accepts a list of strings or a comma-separated string. Omit it to
receive all frames. The actor also receives its own lifecycle frames and
synthetic control frames regardless of this filter. `pulse` requests periodic
heartbeats in milliseconds.

**Recommendation:** Check the frame topic before processing application data.

`next` carries state between invocations.

**Recommendation:** Reconstruct required state from retained events or explicit
persisted state after a restart. Deduplicate external work when replaying
history.

Actors can also `.append` explicit events. Output frames carry `actor_id` and
`frame_id` provenance. An actor ignores frames marked with its own actor ID;
loops involving multiple processors remain possible.

## Actions: call an operation

Actions are stateless, independently invoked operations; multiple calls may run
in parallel. A call is an event, so either a person or another processor can
make it.

**Example: calculate a value.** The action returns twice the supplied argument.

```nushell
r#'{
  run: {|frame|
    {result: ($frame.meta.args.n * 2)}
  }
}'# | .append xs.action.double.create
```

After `xs.action.double.active`, call the action:

```nushell
.append double.call --meta {args: {n: 21}}
```

The response arrives on `double.response` with `meta.result` equal to `42`.

The closure receives the call frame. If the frame has content,
`.cas $frame.hash` retrieves it. Arguments are available through `meta.args`.
Output is collected into one `<name>.response` frame; errors produce
`<name>.error`. Both link to the invocation with `call_id`/`frame_id` and the
definition with `action_id`. A failed call leaves the action available. Error
frames have indefinite retention. After the action is undefined, calls are
ignored without a response or error.

**Example: agent execution.** A workflow derives a possible task from a note. It
emits a call when that task meets its execution rules. An action or connected
process can invoke an agent runner and record the result. The runner,
credentials, and rule for when to execute must be supplied by the integration.

### Example: processing and enrichment

An actor calls an action that uses an external language model to classify
meeting notes into predefined categories. The action extracts stated project
names and dates into defined fields. Missing or ambiguous values remain unknown.

The actor appends each result with the source note's frame ID and supporting
text. The original notes remain unchanged. Another actor groups the results by
project and category.

### Example: schema migration by replay

This example uses topic names to distinguish schemas: `notes.v1` and `notes.v2`.
These names are application conventions. A processor replays retained `notes.v1`
frames, converts their metadata to the new schema, and appends the results to
`notes.v2`. Each result includes its source frame ID. The processor uses that
reference to skip records already converted. The original frames remain
unchanged. After replay, consumers read `notes.v2`, and producers write new
records in that schema.

## Lifecycle and shared configuration

Append `xs.<kind>.<name>.create` to define or start a processor. Append
`xs.<kind>.<name>.term` to stop or undefine it. Check for the corresponding
`active` or `invalid` event to determine readiness.

| Event suffix | Meaning                                  | Actor | Service | Action |
| ------------ | ---------------------------------------- | ----- | ------- | ------ |
| `create`     | User requests definition or start        | Yes   | Yes     | Yes    |
| `term`       | User requests stop or removal            | Yes   | Yes     | Yes    |
| `active`     | Runtime confirms readiness               | Yes   | Yes     | Yes    |
| `invalid`    | Runtime rejects initialization           | Yes   | Yes     | Yes    |
| `fin.ok`     | Natural completion                       | Yes   | Yes     | No     |
| `fin.error`  | Runtime failure                          | Yes   | Yes     | No     |
| `fin.term`   | Runtime confirms termination             | Yes   | Yes     | Yes    |
| `replaced`   | A newer definition replaces the instance | Yes   | Yes     | No     |
| `stopped`    | Server shutdown stops the instance       | No    | Yes     | No     |

An actor completes when its closure omits `next`. Service completion follows the
`restart` setting described above.

The `active` acknowledgment identifies its originating create frame through
`meta.actor_id`, `meta.source_id` for a service, or `meta.action_id`. Actor
`active` metadata also records the start position. Service `replaced` metadata
includes `update_id`, which identifies the replacement create frame.

At most one actor or service instance runs for each processor name. Action
invocations can run concurrently.

- `fin.ok`, `fin.error`, and `fin.term` are terminal; restarting xs does not
  restart that prior definition. Append a new `create` to run it again.
- Services and actors support live replacement through a new `create`. An
  invalid replacement leaves the prior working version available. Services and
  actions keep the old version. The dispatcher restarts an actor from its last
  known-good definition. This can reset its carried state.
- Service `stopped` records server shutdown and permits resumption on startup.
- Redefining an action replaces its callable definition and emits `active`.

**Example: stop and restore an action.**

```nushell
.append xs.action.double.term
```

After `xs.action.double.fin.term`, new `double.call` frames receive no response.
A server restart keeps the action undefined. Repeating its create definition
registers it again.

**Example: replacement.** Appending another `xs.service.logs.create` with a new
script replaces `logs`. A successful replacement emits `replaced`, then
`active`. A parse failure emits `invalid` and leaves the previous service
running.

**Example: diagnosis.** This command reads the last five lifecycle events for
service `api`:

```nushell
.cat --topic 'xs.service.api.*' | last 5
```

`invalid` indicates initialization failure (`meta.reason`); `fin.error`
indicates a runtime failure (`meta.message`).

**Recommendation:** Match acknowledgments to the create frame being diagnosed.

Restart behavior follows two slots per processor name: the last confirmed
`create` (one that emitted `active`) and the latest pending `create`. At the end
of history, the dispatcher tries pending, falling back to confirmed on parse
failure. `term` and `fin.*` clear both slots; `replaced` and `stopped` do not. A
`term` remains effective even if xs stopped before emitting its acknowledgment.
This process selects definitions to run. An actor reconstructs its state as
described in the actor section.

`return_options` controls output `suffix`, `ttl`, and `target`. Default suffixes
are `.recv` for services, `.out` for actors, and `.response` for actions.
Include the leading dot in a custom suffix. Output TTL defaults to `forever`.
Default outputs are records in metadata; choose `target: "cas"` for other types.

## Modules

Shared Nushell modules live on `xs.module.<name>`. Processors resolve module
versions as of their own creation frame. Dots in the module name map to slashes
in its virtual filesystem directory.

**Example:** `xs.module.discord.api` maps to `discord/api/mod.nu`. A script
loads it with `use discord/api`. A flat module such as `xs.module.mymod` loads
with `use mymod`.

### Example: publish a module and use it in an actor

First, publish a module that exports a function:

```nushell
r#'export def greet [name: string] { $"hello ($name)" }'#
| .append xs.module.mymod
```

Then register an actor that imports the module:

```nushell
r#'{
  topics: ["greeting.request"]
  run: {|frame, state|
    if $frame.topic != "greeting.request" { return {next: $state} }
    use mymod
    {out: {greeting: (mymod greet $frame.meta.name)}, next: $state}
  }
}'# | .append xs.actor.greeter.create
```

After `xs.actor.greeter.active`, send a request:

```nushell
.append greeting.request --meta {name: "Sam"}
```

The actor emits `greeter.out` with `meta.greeting` equal to `hello Sam`. Its
create frame selects the module version available when the actor was registered.

### Procedure: update a module

1. Append the updated module.
2. Recreate its dependent processors.
3. Retain each module version still resolved by an existing processor's create
   ID.

**Recommendation:** Keep module retention at `forever` to preserve required
versions.
