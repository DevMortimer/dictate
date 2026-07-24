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
- Whisper runs on the GPU (Vulkan) by default. On Guix with the nonguix
  NVIDIA driver, the Vulkan loader can discover NVIDIA ICDs from both the
  system and home profiles; when the two profiles carry different builds
  of the driver, both copies get dlopened into one process and it
  segfaults (`vulkaninfo` crashes the same way — it was never a ggml
  bug). The script pins the loader to the system profile's ICD around
  `whisper-cli` (`/run/current-system` is a stable indirection, so the
  pin survives reconfigures); `DICTATE_VK_ICD` selects another driver,
  `DICTATE_GPU=0` forces CPU, and a missing ICD file falls back to CPU.
- Latency knobs: on GPU the model is nearly free (`base.en` encodes a
  5-second take in ~20 ms on an RTX 3060 Ti vs ~8.5 s on CPU), so
  `small.en` is an affordable accuracy upgrade; on CPU `base.en` is the
  sweet spot. whisper.cpp also ships `whisper-server` — a future upgrade
  is keeping the model warm in a daemon and pointing this script at it.
