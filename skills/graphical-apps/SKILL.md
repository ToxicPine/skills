---
name: graphical-apps
description:
  Use when the user wants to launch a graphical desktop application, view or
  control it through a browser, or have the agent operate its interface. Covers
  sharing an application session and passing control between the user and agent.
  Do not use to operate a web browser or interact with websites; use dedicated
  browser automation tools for those tasks.
---

# Graphical applications

## How this works

The applications run on **X11**. The user views the applications in a browser
through **Xpra**. Both the user and the agent operate the same application
session.

The session has two modes:

| Mode           | Who controls the application?                                                                   | What happens when the browser changes size?         |
| -------------- | ----------------------------------------------------------------------------------------------- | --------------------------------------------------- |
| **User mode**  | The user controls the application through the browser.                                          | The remote display changes size to fit the browser. |
| **Agent mode** | The agent controls the application through local input tools. The browser permits viewing only. | The remote display keeps a fixed size.              |

User mode lets the user work directly. Agent mode keeps the coordinate grid
stable for screenshots, clicks, and other input. A browser resize must not move
the agent's targets during computer use.

Either mode can come first. The user can request direct interaction or ask the
agent to operate the application. **After computer use, always return a running
session to user mode.** This includes work that fails or remains incomplete.

Each session has a **time lease**. The default lease is **30 minutes**. A
systemd service runs the application. A separate systemd timer stops that
service when the lease expires. This prevents applications from running
indefinitely.

If work needs more time, renew the timer for a duration from now. For example,
renew for 10 minutes. Replace only the shutdown timer. The application keeps
running. A mode change does not renew the lease.

## Prepare the software

Obtain the application and supporting tools before you start the lease. Use
installed packages when available. If Nix is available, obtain missing packages
together with `nix shell`. Include `xpra`, `xauth`, and the input and capture
tools you need. Include `xrandr` and `libxcvt` if you need to change the
resolution. Some distributions package the Xpra browser client separately as
`xpra-html5`.

Resolve executable paths in the package environment. Pass commands directly to
`systemd-run`. Keep the required environment in the service. Use shell variables
for session settings.

## Start a session

1. Choose a free X display.
2. Choose an unused TCP port in **18080–18179**.
3. Prepare the Xpra desktop command and application command.
4. Pass the command directly to a transient systemd user service.
5. Arm a shutdown timer immediately after the service starts.
6. Select the mode that the user's request requires.
7. Verify the application and its mode.
8. Give the user the browser URL and lease expiry.

Direct port exposing on the system is sufficient. Bind to an address that the
user can reach. Use the established network access control. If the port is in
use, choose another port in the range.

Use a 30-minute lease unless the work clearly requires more time. You may renew
the lease without asking when work needs more time. Renew before expiry. Choose
a duration from now that covers the remaining work. Keep a finite deadline.
Update the recorded expiry after renewal.

Use `--daemon=no` so systemd controls Xpra and its child processes. Set
`TimeoutStopSec`, `Restart=no`, and `KillMode=control-group`. Use a separate
shutdown timer with `OnActiveSec` to enforce the lease. Use `--collect` for
transient units. If the timer cannot be armed, stop the application service.

Read [session.md](references/session.md) for launch commands and lease checks.
Record the application and timer unit names, X display, Xauthority path, port,
application command, and expiry.

## Select user mode

Enable browser input. Enable display resizing. Allow one browser controller at a
time, with takeover by the next client. Use these launch options:

```text
--readonly=no --resize-display=yes --sharing=no --lock=no
```

Hide the browser controls with this URL query:

```text
?floating_menu=false&reconnect=false&sound=false
```

Show the application without extra browser controls or a connection dashboard.
Maximize the application if appropriate. Use a minimal window manager if the
application needs one. The application continues to run when the browser
disconnects, until the lease expires or the session stops.

## Select agent mode

Before computer use, disable browser input:

```bash
xpra control "$session_display" readonly yes
```

Disable automatic display resizing:

```bash
xpra control "$session_display" toggle-feature randr no
```

These commands keep the current dimensions without replacing the server. They
were verified together in Xpra 6.4.4 desktop mode. For another version, check
`xpra control "$session_display" help`.

Read the actual display dimensions. Take a screenshot. Let any pending resize
finish. Verify that the dimensions remain stable.

Keep the current dimensions if the application looks usable. Check that text is
readable and important controls are visible. Avoid an extreme aspect ratio or an
unnecessarily large display. Make this assessment yourself; user confirmation is
not necessary.

If the dimensions are unsuitable, set **1440×960**, a 3:2 resolution. Verify the
new dimensions. Inspect a new screenshot. Read
[session.md](references/session.md) for resolution commands and version limits.

## Operate the application

Use **xdotool** for input on an isolated Xpra display. Use **dotool** when its
input devices already reach the intended X session. Dotool does not select an X
display through `DISPLAY`. Use **scrot** for X screenshots. Always save
screenshots in a temporary directory, never in the project or skill directory.

Read [input.md](references/input.md) for tool behavior specific to this setup.

## Return control to the user

After computer use, enable display resizing:

```bash
xpra control "$session_display" toggle-feature randr yes
```

Enable browser input:

```bash
xpra control "$session_display" readonly no
```

Always do this before you end the turn, even if you started the session for
agent use. Use a `finally` block or shell cleanup trap when you automate the
handoff. Verify that input and resizing are enabled. The display can keep its
current size until the next browser resize.

If restoration fails, retry within the remaining lease. Report the actual mode
if you cannot restore user mode. If the lease expired or the user requested
shutdown, leave the session stopped.

Report the work result, browser URL, remaining lease, and any material
limitation.

## Stop the session

If the user requests shutdown, stop the session's systemd unit. Verify that its
processes and port listener are gone. Remove only the session's temporary files
and stale sockets. Keep any requested output files. Read
[session.md](references/session.md) for the stop command.
