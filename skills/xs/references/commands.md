# Commands

## CLI conventions

The command form is `xs COMMAND [OPTIONS]`. `ADDR` identifies a running store by
local path, TCP address, or `iroh://` ticket. Uppercase arguments below are
placeholders. Square brackets indicate optional arguments.

Commands that append content or import frames read standard input. Frame output
is JSON. Content output preserves raw bytes. `xs --version` reports the client
version; `xs version ADDR` reports the server version.

## Command catalog

| Command form                             | Operation                                                 |
| ---------------------------------------- | --------------------------------------------------------- |
| `xs serve PATH [OPTIONS]`                | Run the store server                                      |
| `xs cat ADDR [OPTIONS]`                  | Read historical frames and optionally follow live appends |
| `xs append ADDR TOPIC [OPTIONS]`         | Append content from stdin with optional metadata          |
| `xs cas ADDR HASH`                       | Retrieve content bytes                                    |
| `xs cas-post ADDR`                       | Store stdin bytes in CAS and return their hash            |
| `xs remove ADDR ID`                      | Remove a frame                                            |
| `xs last ADDR [TOPIC] [COUNT] [OPTIONS]` | Retrieve the latest frames                                |
| `xs get ADDR ID [--with-timestamp]`      | Retrieve one frame                                        |
| `xs import ADDR`                         | Import one JSON frame from stdin, preserving its ID       |
| `xs eval ADDR [FILE] [-c SCRIPT]`        | Evaluate Nushell on the target server                     |
| `xs version ADDR`                        | Retrieve server version information                       |
| `xs nu [OPTIONS]`                        | Print or manage the bundled xs.nu module                  |
| `xs scru128`                             | Generate a SCRU128 ID                                     |
| `xs scru128 unpack ID`                   | Decode an ID into JSON fields                             |
| `xs scru128 pack`                        | Encode JSON fields from stdin into an ID                  |

Import handles a single frame per invocation. Directory export/import is
provided by xs.nu. Content transfer and live-notification behavior are described
in [Between stores](between-stores.md).

## Read options

`xs cat` accepts these options:

| Option                            | Meaning                                                             |
| --------------------------------- | ------------------------------------------------------------------- |
| `--follow`, `-f`                  | Continue with live appends after history                            |
| `--pulse MS`, `-p MS`             | Add a heartbeat interval to a following read                        |
| `--new`, `-n`                     | Skip history                                                        |
| `--after ID`, `-a ID`             | Start after an ID, exclusive                                        |
| `--from ID`                       | Start at an ID, inclusive                                           |
| `--limit N`                       | Return at most N frames                                             |
| `--last N`                        | Return the latest N frames in chronological order                   |
| `--topic PATTERNS`, `-T PATTERNS` | Filter by exact topic, prefix wildcard, or comma-separated patterns |
| `--with-timestamp`                | Add an RFC3339 timestamp extracted from the frame ID                |
| `--sse`                           | Format output as Server-Sent Events                                 |

A pattern of `*` selects all topics. Overlapping patterns return each frame
once. Read boundaries, system markers, and recovery behavior are described in
[Store operations](store.md#read-history-then-follow).

`xs last` accepts `--follow`/`-f` and `--with-timestamp`. Its count defaults
to 1. The optional topic accepts the same pattern syntax. A count can appear
directly after the address when the topic is omitted.

**Example: latest-frame queries.** Each line is an independent query:

```sh
xs last ./store notes
xs last ./store notes 5
xs last ./store 10
xs last ./store 'meetings.*' --follow
```

## Append options

| Option             | Meaning                                                      |
| ------------------ | ------------------------------------------------------------ |
| `--meta JSON`      | Attach custom metadata                                       |
| `--ttl POLICY`     | Set `forever`, `ephemeral`, `time:MS`, or `last:N` retention |
| `--with-timestamp` | Include the frame timestamp in the response                  |

**Example: content with metadata.**

```sh
printf '%s' 'Discuss the budget' | xs append ./store notes --meta '{"author":"alice"}'
```

## Eval input and output

Use `eval` to filter, transform, or aggregate frames on the server and return
the results to the client.

`-c SCRIPT` and `--commands SCRIPT` supply an inline script. `FILE` supplies a
script file. A file argument of `-` reads the script from stdin. The script
executes on the target server with its store commands available.

**Example: input forms.** Each line is an independent invocation:

```sh
xs eval ./store -c '.cat --topic notes | length'
xs eval ./store report.nu
printf '%s' '.last notes' | xs eval ./store -
```

Eval returns raw text for scalar strings and numbers, JSON for structured
values, and JSONL for list streams. Byte streams return raw bytes. An empty
result produces no output. The HTTP representation is listed in
[Transport](transport.md#http-access).

## SCRU128 operations

`unpack` returns `ts_ms`, `counter_hi`, `counter_lo`, and `node`. `ts_ms`
is an integer Unix timestamp in milliseconds. The counters are 24-bit values;
`node` is a 32-bit value represented as a hexadecimal string. `pack` accepts
the same fields as a JSON object on stdin.

**Example: round trip.** The `-` argument makes `unpack` read stdin:

```sh
xs scru128 | xs scru128 unpack - | xs scru128 pack
```

## Server and module options

These options control the server process and the local xs.nu module.

| `xs serve PATH` option | Meaning                                                           |
| ---------------------- | ----------------------------------------------------------------- |
| `--expose LISTEN_ADDR` | Expose a TCP address, socket path, or `iroh://` endpoint          |
| `--fsync always`       | Sync every append before returning                                |
| `--fsync interval:MS`  | Sync on a timer when writes occurred; default `interval:1000`     |
| `--fsync never`        | Leave synchronization to the operating system                     |
| `--ttl-sweep MS`       | Set the expiry sweep interval; default 1000; 0 disables the sweep |
| `--plugin PATH`        | Load a Nushell plugin binary; repeat for additional plugins       |

Plugin commands are available to eval, actors, services, and actions. Use
plugins compatible with the embedded Nushell version. A plugin load failure
stops server startup and identifies the failed path.

With no options, `xs nu` prints the module contents.

| `xs nu` option         | Meaning                                          |
| ---------------------- | ------------------------------------------------ |
| `--install`            | Install the module into Nushell configuration    |
| `--clean`              | Remove previously installed files                |
| `--lib-path PATH`      | Set the library file path; requires `--install`  |
| `--autoload-path PATH` | Set the autoload stub path; requires `--install` |

## xs.nu helpers

External xs.nu helpers use `$env.XS_ADDR` and return structured Nushell values.
The following table lists their documented forms:

| Helper                                                            | Operation                                         |
| ----------------------------------------------------------------- | ------------------------------------------------- |
| `.append TOPIC [--meta RECORD] [--ttl POLICY] [--with-timestamp]` | Append pipeline content                           |
| `.cat [READ_OPTIONS] [--detail]`                                  | Read frames; `--detail`/`-d` includes all fields  |
| `.last [TOPIC] [COUNT] [--follow] [--with-timestamp]`             | Read latest frames                                |
| `.get ID [--with-timestamp]`                                      | Retrieve one frame                                |
| `.remove ID`                                                      | Remove a frame                                    |
| `.cas HASH`                                                       | Retrieve content                                  |
| `.cas-post`                                                       | Store pipeline content and return its hash        |
| `.eval [FILE] [--commands SCRIPT]`                                | Evaluate a file, inline script, or piped script   |
| `.export DIRECTORY`                                               | Export frames and content to a directory          |
| `.import DIRECTORY`                                               | Restore an exported directory                     |
| `.id`                                                             | Generate an ID                                    |
| `.id unpack [ID]`                                                 | Decode an argument or piped ID                    |
| `.id pack [RECORD]`                                               | Encode an argument or piped component record      |
| `.tmp-spawn CLOSURE [--interactive]`                              | Run a closure in a temporary store, then clean up |

`.cat` supports the read options above except CLI-only `--sse`. Its topic filter
also accepts a Nushell list. `.last` accepts the same topic patterns or a list.
Timestamp options produce native Nushell datetime values. `.eval` accepts `-c`
as the alias for `--commands`. `.id unpack` adds `when` as a native datetime;
`.id pack` ignores that display field and uses the integer `ts_ms` value.

`.tmp-spawn` sets `XS_ADDR` for the closure. Its `--interactive` option opens a
Nushell session after the closure. Cleanup removes the process and temporary
store when the operation finishes.

Inside eval and processor scripts, builtins bind directly to the executing
store. The builtin `.import` accepts one piped frame record or JSON value. The
external xs.nu `.import` accepts an export directory.
