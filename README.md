# dictate

Push-to-talk voice dictation. Press hotkey → speak → press again → text is typed into the focused window.

Uses Lemonade (Whisper) for transcription and ydotool to inject the result as keystrokes.

---

## How it works

1. First hotkey press: starts recording mic to `/tmp/dictate.wav` via `pw-record`
2. Second hotkey press: stops recording, sends WAV to Lemonade's Whisper API, types the transcription into the focused window via `ydotool`

---

## Dependencies

- `pw-record` — mic recording (PipeWire, installed by default on CachyOS)
- `curl` + `jq` — send audio to API, parse JSON response
- `notify-send` — desktop notifications for recording state
- `ydotool` — simulate Ctrl+V paste (kept as daemon for future use)
- `wtype` — type Unicode text directly into the focused Wayland window (layout-independent)
- Lemonade server running with `whisper-v3-turbo-FLM` loaded

---

## Setup

### 1. Install ydotool
```fish
paru -S ydotool
```

### 2. Allow your user to access /dev/uinput
ydotool needs `/dev/uinput` (the kernel's virtual input device) to fake keystrokes.

Create a udev rule so the `input` group owns it:
```fish
echo 'KERNEL=="uinput", GROUP="input", MODE="0660", OPTIONS+="static_node=uinput"' | sudo tee /etc/udev/rules.d/80-ydotool.rules
```

Add yourself to the `input` group:
```fish
sudo usermod -aG input $USER
```

Reload udev rules (or reboot):
```fish
sudo udevadm control --reload-rules && sudo udevadm trigger
```

### 3. Enable the ydotool daemon
ydotool needs a background daemon (`ydotoold`) that holds the `/dev/uinput` handle:
```fish
systemctl --user enable --now ydotool
```

### 4. Install Whisper in Lemonade
```fish
lemonade backends install whispercpp:vulkan
lemonade pull Whisper-Large-v3-Turbo
```

### 5. Symlink into PATH
```fish
ln -s ~/projects/dictate/dictate ~/.local/bin/dictate
```

### 6. Bind to a hotkey in COSMIC
Settings → Keyboard → Keyboard Shortcuts → Custom Shortcuts → Add:
- **Command:** `/home/<your-username>/.local/bin/dictate`
- **Shortcut:** Super+Space (remove the default language-input binding first)

Use a full absolute path, not just `dictate` and not `~` or `$USER` — COSMIC
launches shortcuts without a shell, so nothing gets expanded.

---

## Log out and back in after step 2
Group membership (`input`) only takes effect after a new login session.
