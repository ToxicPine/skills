# Example: an MCP server connected to the stream

This example sends JSON-RPC requests to a process through a duplex service. The
service records responses as frames. An actor reacts to response text.

The example assumes a running store and xs.nu loaded with `XS_ADDR` set. The xs
server's environment must provide `npx` and access to the example package. The
package version is fixed for this example's `echo` and `add` tools.

## 1. Observe the stream

Run this monitor in a separate Nushell session:

```nushell
.cat -f | each {|frame|
  if $frame.hash != null {
    $frame | insert content { .cas $frame.hash } | print
  } else {
    $frame | print
  }
}
```

The monitor shows frame metadata and fetches content when a hash exists.

## 2. Define the service

```nushell
r#'{
  run: {|| npx -y "@modelcontextprotocol/server-everything@2025.3.19" | lines }
  duplex: true
  return_options: {target: "cas"}
}'# | .append xs.service.mcp.create
```

Wait for `xs.service.mcp.active` before sending requests. `lines` separates
stdout into response frames on `mcp.recv`. The CAS target stores each response
as text. Content on `mcp.send` goes to the process's stdin. Each JSON-RPC
request needs a terminating newline.

## 3. List tools and call echo

```nushell
{jsonrpc: "2.0", id: 1, method: "tools/list", params: {}}
| to json -r | $in + "\n" | .append mcp.send
```

Wait for a `mcp.recv` response with JSON-RPC ID `1`. The response contains the
tool names and their input schemas.

```nushell
{jsonrpc: "2.0", id: 2, method: "tools/call", params: {
  name: "echo", arguments: {message: "Hello from cross.stream!"}
}}
| to json -r | $in + "\n" | .append mcp.send
```

After the echo response arrives, inspect its content:

```nushell
.last mcp.recv | .cas $in.hash | from json
```

The response ID associates the JSON-RPC result with its request. The frame ID
identifies the stored event.

## 4. React to a response

This actor emits a message when response text contains `42`:

```nushell
r#'{
  run: {|frame, state|
    if $frame.topic != "mcp.recv" { return {next: $state} }
    let content = .cas $frame.hash | from json | get -o result.content.0.text
    if ($content != null and "42" in $content) {
      {out: {message: "meaning of life detected!"}, next: $state}
    } else {
      {next: $state}
    }
  }
}'# | .append xs.actor.meaning.detector.create
```

Wait for `xs.actor.meaning.detector.active`, then call the addition tool:

```nushell
{jsonrpc: "2.0", id: 10, method: "tools/call", params: {
  name: "add", arguments: {a: 21, b: 21}
}}
| to json -r | $in + "\n" | .append mcp.send
```

The stream contains the request, the response with the sum, and
`meaning.detector.out`. The actor's message is in the output frame's metadata.
The substring check illustrates event processing; it matches any response text
containing `42`.

## 5. Inspect and stop

```nushell
.cat --topic 'mcp.*'
.last meaning.detector.out
.append xs.service.mcp.term
.append xs.actor.meaning.detector.term
```

The runtime acknowledges termination with each processor's `fin.term` event.
Separate service names provide separate `.send` and `.recv` channels for
multiple MCP servers. Further actors can process responses or call actions.
