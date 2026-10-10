# Xpra session commands

These commands were checked with Xpra 6.4.4.

## Start the session

Choose a free X display. Choose an unused port in 18080–18179. Choose a unique
systemd unit name. Select an address that the user can reach. Keep these values
in shell variables.

Resolve executable paths in the package environment. Pass the Xpra command
directly to `systemd-run`. Use absolute paths for Xpra and the application.
Preserve PATH for Xpra and its child processes.

This example requires `xpra`, `xterm`, and `xauth`. Set the session variables
before you run it:

```bash
session_unit=graphical-example.service
session_display=:100
host_address=HOST_ADDRESS
session_port=18080
xpra_path=$(command -v xpra)
app_path=$(command -v xterm)

systemd-run --user --unit="$session_unit" --collect \
  --expand-environment=no --service-type=exec \
  --setenv="PATH=$PATH" --working-directory="$PWD" \
  -p TimeoutStopSec=10s -p Restart=no -p KillMode=control-group \
  -- "$xpra_path" start-desktop "$session_display" \
  --daemon=no \
  --env="PATH=$PATH" \
  --start-child="$app_path" \
  --bind-tcp="$host_address:$session_port" \
  --html=on \
  --readonly=no --resize-display=yes \
  --sharing=no --lock=no \
  --encoding=auto --mdns=no \
  --pulseaudio=no --speaker=off --microphone=off \
  --webcam=no --printing=no \
  --exit-with-children=yes
```

Replace the application command and session values as needed. The caller's shell
expands the variables before systemd receives the command.
`--expand-environment=no` prevents a second expansion by systemd.
`--service-type=exec` reports an error if the executable cannot start. These
commands were verified with Xpra 6.4.4 and systemd 260.

If the application needs a window manager, use `--start=ABSOLUTE_WM_PATH`. Use
`--start-child` for the application whose exit must stop the session. Read the
Xauthority path from the X server command or session data. Do not assume that it
is `~/.Xauthority`.

## Start the lease

Immediately arm a separate timer that stops the application service:

```bash
lease_unit="${session_unit%.service}-lease"
if ! systemd-run --user --unit="$lease_unit" --collect \
  --on-active=30min --timer-property=AccuracySec=1s \
  systemctl --user stop "$session_unit"; then
  systemctl --user stop "$session_unit"
  exit 1
fi
```

If timer creation fails, stop the application service immediately. The timer
enforces the lease. Do not set `RuntimeMaxSec` on the application service. That
limit would still stop the application at the original deadline. The stop
timeout permits 10 seconds for shutdown. Systemd then kills any remaining
processes in the service.

Check startup in the journal. Check the listener with `ss -ltn`. Check the
application with `xpra info` and a screenshot. Check the lease with
`systemctl --user list-timers --all "$lease_unit.timer"`. Use the timer's next
trigger time as the expiry. Do not calculate expiry from the time of your check.

Give the user this URL with the actual host and port:

```text
http://HOST:PORT/?floating_menu=false&reconnect=false&sound=false
```

Some browser decoding features require a secure context. If necessary, configure
Xpra to serve TLS directly with a suitable certificate. Alternatively,
[Tailscale Serve](https://tailscale.com/docs/features/tailscale-serve) can
provide an HTTPS URL for Xpra's local HTTP port. Direct port exposure is
sufficient when HTTPS is not needed.

## Renew the lease

Renew before expiry when work needs more time. No user confirmation is required.
Keep a finite deadline.

`OnActiveSec` counts from timer activation. Replace the shutdown timer to start
a new interval. Keep the application service running. A 10-minute renewal sets
expiry to 10 minutes from now. It does not add 10 minutes to the previous
deadline.

Before renewal, verify that the application is active and the timer has not
expired. Run the following commands together. This example renews for 10
minutes:

```bash
systemctl --user stop "$lease_unit.timer"
if ! systemd-run --user --unit="$lease_unit" --collect \
  --on-active=10min --timer-property=AccuracySec=1s \
  systemctl --user stop "$session_unit"; then
  systemctl --user stop "$session_unit"
  exit 1
fi
systemctl --user list-timers --all "$lease_unit.timer"
```

The old timer is removed when it stops. The new timer uses the same name. No
override files or manager reload are necessary. If timer creation fails, stop
the application service immediately.

This pattern was verified with systemd 260. The same application process
survived two renewals, including a duration shorter than the timer's elapsed
time. It survived the previous deadline and stopped at the final deadline.

Verify that the application is still active and the timer is armed. Update the
recorded expiry. If shutdown has already begun, renewal does not recover the
session.

## Keep the current dimensions

Use the live mode commands in [SKILL.md](../SKILL.md). Together, `readonly=yes`
and `randr=no` keep the existing dimensions. They do not replace the server or
change systemd process ownership.

`readonly=yes` blocks browser input and desktop-window size changes. `randr=no`
disables automatic display size negotiation. The `randr` toggle alone does not
block every desktop resize path.

Read the actual dimensions:

```bash
DISPLAY="$session_display" XAUTHORITY="$session_xauthority" xrandr --current
```

Verify that a browser resize does not change these dimensions during agent mode.

## Set a 3:2 resolution

Use an existing 1440×960 mode if available. Find the connected virtual output
with `xrandr`. Do not assume that its name is `DUMMY0` or `VFB-0`.

If the mode is absent, generate a modeline with `cvt 1440 960 60`. Add the mode
to the X server. Add it to the virtual output. Select the mode.

This example was verified on Xpra's Xorg dummy display:

```bash
export DISPLAY="$session_display" XAUTHORITY="$session_xauthority"
xrandr --newmode "1440x960_60.00" \
  113.75 1440 1520 1672 1904 960 963 973 996 -hsync +vsync
xrandr --addmode "$virtual_output" "1440x960_60.00"
xrandr --output "$virtual_output" --mode "1440x960_60.00"
```

Reuse the mode if it already exists. Read the dimensions again. Inspect a new
screenshot.

X backends have different RandR capabilities. If the command fails, inspect the
supported modes. Use another suitable 3:2 size, such as 1200×800 or 1920×1280.
If necessary, use a virtual X server that supports the required resolution.

## Start with a fixed size

For a new fixed-size desktop, use `--resize-display=no:1440x960`. This sets the
initial size and disables automatic resizing. `--resize-display=1440x960` sets
an initial size but still permits resizing.

## Avoid server replacement during a lease

Prefer live mode changes. `xpra upgrade-desktop --resize-display=no` does not
reliably keep the current size. In version 6.4.4, desktop initialization can
select the default resolution.

If an upgrade is necessary, first record the actual dimensions. Pass the
dimensions explicitly, for example `--resize-display=no:1200x800`. Keep the
listener settings and application state. An upgrade briefly disconnects clients.

An upgrade can replace the foreground process. It can also start the replacement
outside the original service process group. Use an upgrade only if the
supervisor keeps every replacement process under the original lease and cleanup.
Otherwise, use the live mode commands.

## Stop the session

Stop the timer and application service:

```bash
systemctl --user stop "$lease_unit.timer" "$session_unit"
```

Verify that its processes are gone. Verify that its port listener is gone.
Remove only its temporary captures and stale sockets. Keep requested output
files. Leave shared package caches in place.

## References

- [Xpra manual](https://xpra.org/manual)
- [Xpra HTML5 configuration](https://github.com/Xpra-org/xpra-html5/blob/master/docs/Configuration.md)
- [systemd service documentation](https://github.com/systemd/systemd/blob/main/man/systemd.service.xml)
- [systemd timer documentation](https://github.com/systemd/systemd/blob/main/man/systemd.timer.xml)
- [systemd-run documentation](https://github.com/systemd/systemd/blob/main/man/systemd-run.xml)
- [ASD-STE100, Issue 9](https://www.asd-ste100.org/assets/files/ASD-STE100_ISSUE9.pdf):
  short sentences, direct instructions, and consistent terms.
