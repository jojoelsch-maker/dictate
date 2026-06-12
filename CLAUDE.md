# dictate

Push-to-talk dictation: hotkey → record mic → Lemonade Whisper API →
ydotool types the text. Everything is in the `dictate` script; docs in
README.md.

- Whisper runs on GPU via whispercpp (NOT the NPU/FLM variant — that
  combination overloads the NPU and evicts all models).
- Update README.md after any change (jsnd's docs rule).
