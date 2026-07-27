# dictate

Push-button dictation for Wayland (niri) and macOS. One keypress records,
the next turns what you said into text and hands it to the focused window.

On Linux that means it is *typed* for you — terminal, browser, Emacs,
anything. On macOS it stops at the clipboard and you press ⌘V; macOS has no
way to synthesize input without an Accessibility grant, and one that is
silently denied looks exactly like the tool being broken. See
[ADR 0001](docs/adr/0001-macos-support.md).

The pipeline, one keybind end to end:

1. **Record** — the default mic at 16 kHz mono WAV, whisper's native
   format: `pw-record` on Linux, `ffmpeg -f avfoundation -i ":default"` on
   macOS (`:default` follows the system input the way `pw-record` does).
2. **Transcribe** — `whisper-cli` (whisper.cpp) runs locally; the ggml
   model is auto-downloaded to `~/.cache/dictate/models` on first use.
   Vulkan on Linux, Metal on macOS.
3. **Clean** — the raw transcript goes to DeepSeek `/chat/completions`
   with a typist system prompt: phrase by phrase it judges whether you
   were speaking *for the page* or *to the typist*, so spoken spellings
   ("D-O-T files, one word"), corrections ("no, wait, I mean…"),
   retractions ("scratch that") and formatting commands ("new line") are
   executed rather than transcribed, while filler and false starts are
   dropped. Skipped when no API key is configured; any failure falls
   back to the raw transcript, so dictation works offline.
4. **Deliver** — on Linux the text is copied to the clipboard (backup),
   then typed into the focused window with `wtype`, which uses the
   `zwp_virtual_keyboard_manager_v1` protocol niri exposes — so it works
   in terminals exactly as if you typed it. On macOS it is copied with
   `pbcopy` and the notification tells you it is ready to paste.

## Usage

```
dictate            # same as: dictate toggle
dictate toggle     # start recording / stop + transcribe + type
dictate cancel     # abort the current recording, type nothing
dictate status     # "recording" or "idle" (for bars/widgets)
```

Toggle, not hold-to-talk: niri keybinds fire on key press only (no
release binds), and a toggle also survives long dictations without a
cramped pinky.

## Install

The script is self-contained; put it on `PATH` wherever you like. On macOS
it must be an absolute path for skhd (see below), so:

```sh
ln -s "$PWD/dictate" ~/.local/bin/dictate
```

A symlink rather than a copy: the repo stays the single source of truth,
so an edit is live and the keybind exercises the file you are editing.

## Config

`~/.config/dictate/config` — a POSIX sh fragment, sourced if present
(see `config.example`):

```sh
DICTATE_DEEPSEEK_API_KEY=sk-...        # empty/absent = no LLM cleanup
DICTATE_DEEPSEEK_MODEL=deepseek-v4-flash
DICTATE_WHISPER_MODEL=base.en          # any ggml model name, e.g. small.en
DICTATE_LANG=en
# DICTATE_PROMPT="..."                 # override the cleanup prompt
```

If `DICTATE_DEEPSEEK_API_KEY` is empty, the script falls back to reading
the bare key from `~/.config/dictate/deepseek-key` (chmod 600). Use that
file when the config itself is tracked in dotfiles — on Guix Home,
deployed dotfiles pass through the world-readable store, so the secret
must stay in the untracked key file.

## niri integration

```kdl
Mod+V { spawn "dictate" "toggle"; }
Mod+Shift+V { spawn "dictate" "cancel"; }
```

Progress is shown as a single dunst notification that updates in place
(dunst stack tag).

## macOS integration

Keybinds go through [skhd](https://github.com/koekeishiya/skhd). The full
path is required: skhd inherits launchd's environment, which does not
carry `~/.local/bin`.

```
cmd - d       : "$HOME/.local/bin/dictate" toggle
cmd + alt - d : "$HOME/.local/bin/dictate" cancel
```

Note that a global `cmd - d` shadows the application-level ⌘D (bookmark in
browsers, Duplicate in Finder).

Notification Center has no equivalent of the dunst stack tag, so the four
progress notifications stack rather than replacing one another.

Because skhd reads no shell profile, `dictate` keeps its runtime state in
`/tmp/dictate-$(id -u)` on macOS regardless of `XDG_RUNTIME_DIR` — the
hotkey and a terminal would otherwise disagree about whether a take is
live.

## Dependencies

**Linux**: `pw-record` (pipewire), `whisper-cli` (whisper.cpp), `curl`,
`jq`, `wtype`, `wl-copy` (wl-clipboard), `notify-send` (libnotify). The
Guix package (`mort packages dictate` in the dotfiles repo) wraps the
script with all of these on PATH.

**macOS**: `ffmpeg`, `whisper-cli`, `curl`, `jq`, `skhd` — `pbcopy` and
`osascript` ship with the system.

```sh
brew install ffmpeg whisper-cpp jq skhd
```

## Notes

- On Linux, newlines are typed as Return. In a terminal that executes a
  line, so only say "new line" when you mean it. On macOS you paste, and
  a terminal's bracketed paste does not execute the lines.
- If typing fails (e.g. focus landed somewhere odd), the take is still on
  the clipboard.
- On macOS, `DICTATE_GPU` and `DICTATE_VK_ICD` are ignored: Metal has no
  loader to confuse and no ICD to pin, so inference is always accelerated.
  The rest of this note is Linux-only.
- Whisper runs on the GPU (Vulkan) by default. On Guix with the nonguix
  NVIDIA driver, the Vulkan loader can discover NVIDIA ICDs from both the
  system and home profiles; when the two profiles carry different builds
  of the driver, both copies get dlopened into one process and it
  segfaults (`vulkaninfo` crashes the same way — it was never a ggml
  bug). On Guix the script therefore pins the loader to the system
  profile's ICD around `whisper-cli` (`/run/current-system` is a stable
  indirection, so the pin survives reconfigures); on other distros, which
  keep a single `icd.d`, the loader's stock discovery is left untouched.
  `DICTATE_VK_ICD` pins one explicit ICD on any distro, `DICTATE_GPU=0`
  forces CPU.
- Latency knobs (figures measured on Linux/Vulkan; macOS/Metal is
  unbenchmarked): on GPU the model is nearly free (`base.en` encodes a
  5-second take in ~20 ms on an RTX 3060 Ti vs ~8.5 s on CPU), so
  `small.en` is an affordable accuracy upgrade; on CPU `base.en` is the
  sweet spot. whisper.cpp also ships `whisper-server` — a future upgrade
  is keeping the model warm in a daemon and pointing this script at it.
