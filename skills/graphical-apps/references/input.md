# Input tools

**dotool** uses Linux `uinput` and requires access to `/dev/uinput`. It does not
target a display through `DISPLAY`. Use it only when its devices reach the
intended session. For repeated actions, use `dotoold` and `dotoolc`. Commands
include `key ctrl+a`, `type TEXT`, and `click left`. `mouseto X Y` uses
coordinates from 0 to 1.

**xdotool** targets the X server selected by `DISPLAY`. It supports input,
window lookup, focus, and window sizing. Prefer it for an isolated Xpra display.
Set the session's actual X authority:

```bash
export DISPLAY="$session_display"
export XAUTHORITY="$session_xauthority"
```

Keyboard input normally goes to the focused window. Prefer focusing the window
before input; some applications reject events sent with `--window`. Use
`windowfocus` if the minimal desktop does not support `windowactivate`. Xpra
view-only mode does not block local X11 input.

Use `xwininfo` or `xprop` for window details. Neither dotool nor xdotool
captures screenshots. Local tools do not take over the user's browser
connection.

## X screenshots

Use **scrot**. It captures X pixels directly and writes PNG files. Capture the
full display by default. This includes separate menus and dialogs, and keeps
desktop coordinates unchanged.

Always create screenshot files in temporary storage:

```bash
capture_dir=$(mktemp -d /tmp/graphical-apps.XXXXXXXX)
scrot --silent --overwrite "$capture_dir/screen.png"
```

Open the PNG with the image viewer tool. Keep subsequent captures in the same
temporary directory. Remove that directory when the captures are no longer
needed.

For the focused window, add `--focused`. Window captures can omit separate menus
and dialogs. Map window coordinates to desktop coordinates before sending
desktop input.

The command uses the session's `DISPLAY` and `XAUTHORITY`. Keep capture
dimensions unchanged during computer use. Avoid browser screenshots for local X
input; browser scaling can change the coordinate mapping.

## References

- [dotool manual](https://git.sr.ht/~geb/dotool/tree/master/item/doc/dotool.1.scd)
- [xdotool](https://github.com/jordansissel/xdotool)
- [scrot manual](https://github.com/resurrecting-open-source-projects/scrot/blob/master/man/scrot.1)
