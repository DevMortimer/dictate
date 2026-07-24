# dictate

Push-button dictation for Wayland (niri). One keypress records, the next
one types what you said into whatever window has focus — terminal, browser,
Emacs, anything.

The pipeline, one keybind end to end:

1. **Record** — `pw-record` captures the default mic (16 kHz mono WAV,
   whisper's native format).
2. **Transcribe** — `whisper-cli` (whisper.cpp) runs locally; the ggml
   model is auto-downloaded to `~/.cache/dictate/models` on first use.
3. **Clean** — the raw transcript goes to DeepSeek `/chat/completions`
   with a system prompt that fixes punctuation, drops filler and false
   starts, and applies spoken commands like "new line". Skipped when no
   API key is configured; any failure falls back to the raw transcript,
   so dictation works offline.
4. **Inject** — the text is copied to the clipboard (backup), then typed
   into the focused window with `wtype`, which uses the
   `zwp_virtual_keyboard_manager_v1` protocol niri exposes — so it works
   in terminals exactly as if you typed it.

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

## Config

`~/.config/dictate/config` — a POSIX sh fragment, sourced if present:

```sh
DICTATE_DEEPSEEK_API_KEY=sk-...        # empty/absent = no LLM cleanup
DICTATE_DEEPSEEK_MODEL=deepseek-v4-flash
DICTATE_WHISPER_MODEL=base.en          # any ggml model name, e.g. small.en
DICTATE_LANG=en
# DICTATE_PROMPT="..."                 # override the cleanup prompt
```

Keep the key file out of any public dotfiles repo.

## niri integration

```kdl
Mod+V { spawn "dictate" "toggle"; }
Mod+Shift+V { spawn "dictate" "cancel"; }
```

Progress is shown as a single dunst notification that updates in place
(dunst stack tag).

## Dependencies

`pw-record` (pipewire), `whisper-cli` (whisper.cpp), `curl`, `jq`,
`wtype`, `wl-copy` (wl-clipboard), `notify-send` (libnotify). The Guix
package (`mort packages dictate` in the dotfiles repo) wraps the script
with all of these on PATH.

## Notes

- Newlines are typed as Return. In a terminal that executes a line, so
  only say "new line" when you mean it.
- If typing fails (e.g. focus landed somewhere odd), the take is still on
  the clipboard.
- Latency knobs: `base.en` is the speed/accuracy sweet spot on CPU;
  `small.en` is noticeably better and still tolerable. whisper.cpp also
  ships `whisper-server` — a future upgrade is keeping the model warm in
  a daemon and pointing this script at it.
