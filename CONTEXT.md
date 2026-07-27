# Dictate

Push-button dictation: one keypress records the microphone, the next turns
what was said into text and hands it to the focused window. Runs on Linux
(niri/Wayland) and macOS. This glossary pins down the vocabulary of the
pipeline so the script, the docs and the prompt agree on what each stage
is called.

Where a term is platform-specific it says so. The two platforms share the
recording contract, the transcription, and the cleanup pass; delivery is
the one stage where they deliberately differ.

## Language

### The pipeline

**Take**:
One recording session, from the press that starts it to the press that
ends it, along with everything derived from it. The unit the tool is
built around: a take is what gets lost when something fails, which is why
the clipboard is written before anything riskier is attempted.
_Avoid_: recording, clip, session, capture

**Transcript**:
The raw text whisper.cpp produces from a take, before any cleanup. Always
exists once transcription succeeds, and is what gets delivered if the
cleanup pass is skipped or fails.
_Avoid_: raw text, output, whisper output

**Cleanup**:
The optional LLM pass that turns a transcript into finished text —
executing what was said to the typist and writing down what was said for
the page. Skipped when no API key is configured, and any failure falls
back to the transcript, so cleanup is never load-bearing.
_Avoid_: post-processing, correction, polish, LLM pass

**Delivery**:
Getting the finished text out of the tool and into the user's hands. The
one stage that differs by platform: on Linux it means clipboard *and*
synthesized typing; on macOS it means clipboard only, and the user presses
Cmd-V. The term names the *goal*, not the mechanism, precisely because the
mechanism is not shared.
_Avoid_: injection, typing, paste, output

### Speech

**For the page**:
Speech that is content — the words the user wants written down, including
questions and instructions aimed at whoever reads the result. The default
reading: when a phrase could be either, it is for the page.
_Avoid_: content, dictation, body text

**To the typist**:
Speech that is an instruction about the writing itself — spoken spellings,
corrections, retractions, formatting commands. Executed and then dropped,
never written down. The single judgment the cleanup pass makes is which of
the two a phrase is.
_Avoid_: command, meta, directive

**Hallucination**:
Text whisper.cpp invents from silence or room noise — "Thank you.",
"Thanks for watching!", "[BLANK_AUDIO]". Filtered by exact match before
delivery, because a take with no speech in it should deliver nothing
rather than something plausible.
_Avoid_: noise, artifact, false positive

### Control

**Toggle**:
The single action that both starts and ends a take, depending on whether
one is live. Not hold-to-talk: niri keybinds fire on press only, and a
toggle survives a long dictation without holding a key down.
_Avoid_: start/stop, push-to-talk, record button

**Cancel**:
Ending a take and discarding it — no transcription, no delivery. Distinct
from a toggle that ends a take, which is the whole point of having it.
_Avoid_: abort, stop, discard
