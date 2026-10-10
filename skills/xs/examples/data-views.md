# Examples: relationships and derived views

These examples use xs.nu with a running store selected through `XS_ADDR`. The
examples are independent.

## Threaded conversations

Each turn stores its role and the ID of the turn it continues.

```nushell
let first = ("What's the capital of France?" | .append chat.turn --meta {role: user}).id
let second = ("The capital is Paris." | .append chat.turn --meta {
  role: assistant, continues: $first
}).id
let third = ("Tell me a famous landmark there." | .append chat.turn --meta {
  role: user, continues: $second
}).id
let fourth = ("The Eiffel Tower." | .append chat.turn --meta {
  role: assistant, continues: $third
}).id
```

This helper reconstructs the chain in conversation order:

```nushell
def thread [id] {
  if $id == null { return [] }
  let frame = .get $id
  thread ($frame.meta?.continues?) | append $frame
}

thread $fourth | each {|frame| .cas $frame.hash }
```

The result contains the four messages above. A new turn can continue an earlier
message to start another branch:

```nushell
let branch = ("What about Germany?" | .append chat.turn --meta {
  role: user, continues: $second
}).id
thread $branch | each {|frame| .cas $frame.hash }
```

This branch contains the first two messages and the question about Germany. The
landmark discussion remains available through `$fourth`.

## Task state from events

The stream records each change. A projection derives the current task list.

```nushell
let todo = (.append todos --meta {action: "add", text: "Learn event sourcing"})
.append todos --meta {action: "add", text: "Build a view"}
.append todos --meta {action: "toggle", id: $todo.id}
```

This projection applies each event to the list:

```nushell
.cat --topic todos | generate {|frame, todos = []|
  match $frame.meta.action {
    "add" => {
      let updated = ($todos | append {
        id: $frame.id, text: $frame.meta.text, completed: false
      })
      {out: $updated, next: $updated}
    }
    "toggle" => {
      let updated = ($todos | each {|todo|
        if $todo.id == $frame.meta.id {
          $todo | update completed (not $todo.completed)
        } else { $todo }
      })
      {out: $updated, next: $updated}
    }
    _ => {next: $todos}
  }
} | last
```

The first task is complete. The second task is incomplete.

For a live view, add `--follow` to `.cat` and replace the final `last` with
`each { print $in }`. New events update the displayed list. A server can render
these values as HTML and send interface updates over SSE.
