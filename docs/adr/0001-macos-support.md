# macOS support in one script, with clipboard-only delivery

## Status

accepted

## Context

The tool was Linux-only: `pw-record` for capture, `wtype` and `wl-copy` for
delivery, `notify-send` for progress, and a Vulkan ICD pin to work around a
Guix driver-duplication segfault. The goal is the same CLI and the same
one-keypress workflow on macOS, with Linux/niri support preserved untouched.

## Decision

Keep `dictate` a single POSIX `sh` script and branch on `uname` inside the
handful of functions that genuinely diverge, rather than splitting into
per-platform files. The divergence is four short functions in a ~300-line
script whose whole character is being one file you can read top to bottom.

| Stage | Linux/niri | macOS |
|---|---|---|
| Record | `pw-record` | `ffmpeg -f avfoundation -i ":default"` |
| Transcribe | `whisper-cli`, Vulkan | `whisper-cli`, Metal |
| Cleanup | DeepSeek — shared, no branch | same |
| Delivery | `wl-copy` + `wtype` | `pbcopy` only |
| Notifications | `notify-send`, dunst stack tag | `osascript display notification` |
| Keybinds | niri KDL | `skhd` |

**Delivery stops at the clipboard on macOS.** Synthesizing a paste would
require driving System Events, which needs an Accessibility grant *and* a
separate Automation grant attributed to whichever process launched the
script — under skhd that is a launchd daemon, where the consent dialog is
easily missed and a silently denied grant is indistinguishable from the tool
being broken. Copying needs no permission, cannot fail, and leaves the user
choosing the destination window. The cost is a second gesture: Cmd-V.

**macOS ignores `XDG_RUNTIME_DIR`.** It is not a system facility there, only
a convention a shell profile may invent, and skhd runs under launchd, which
reads no profile. Honouring it would put the hotkey's pid file in one
directory and a terminal's in another: a take started with the hotkey would
read as idle from a shell, and toggling there would start a second recorder
instead of stopping the first. macOS always uses `/tmp/dictate-$(id -u)`.

**`DICTATE_GPU` and `DICTATE_VK_ICD` are ignored on macOS.** Metal has no
loader to confuse and no ICD to pin, so inference is always accelerated and
neither knob has an honest meaning.

## Considered options

- **Per-platform files sourced by a core script** — rejected. It is what the
  sibling `screencast` project does, but that is C with genuinely disjoint
  capture backends; here it would add indirection, a second file to install,
  and complexity to the Guix package wrapper, all to isolate about fifty
  lines.
- **A separate `dictate-mac` script** — rejected. It would duplicate the
  pipeline and the typist prompt, which is where the actual thinking lives,
  and the two would drift.
- **Synthesized Cmd-V on macOS** (`osascript ... keystroke "v" using command
  down`) — rejected for the permission reasons above. A fallback-on-failure
  variant was considered and rejected as well: it buys parity only when the
  grants happen to be in place, at the cost of a failure mode that is
  invisible until you notice the text merely sat on the clipboard.
- **Synthesized typing** (the true `wtype` analogue) — rejected. Visibly slow
  on a long take, and unreliable with non-ASCII.
- **`terminal-notifier -group dictate`** for in-place progress, matching the
  dunst stack tag — rejected. A Homebrew dependency plus another permission
  grant is too much for progress text on a two-second operation; the four
  banners simply stack on macOS.

## Consequences

- macOS gains `ffmpeg` as a dependency; `whisper-cli` comes from
  `brew install whisper-cpp`, which is built with Metal.
- The one-keypress property is Linux-only. On macOS dictation is two
  gestures, and the finish notification carries a preview of the take so it
  can be confirmed before pasting.
- Progress notifications accumulate in Notification Center on macOS.
- The `/tmp` runtime-dir fallback is now per-uid on both platforms; Linux
  with `XDG_RUNTIME_DIR` set is unaffected.
- The Vulkan/Guix workaround stays exactly as it was, exercised only on
  Linux.
