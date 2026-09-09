# dictate

Push-to-talk voice dictation. Press hotkey → speak → press again → text is typed into the focused window.

Uses Lemonade (Whisper) for transcription and ydotool to inject the result as keystrokes.

---

## How it works

1. First hotkey press: starts recording mic to `/tmp/dictate.wav` via `pw-record`
2. Second hotkey press: stops recording, sends WAV to Lemonade's Whisper API, types the transcription into the focused window via `ydotool`

Transcription language is pinned to German (`-F language=de` in the script,
added 2026-07-06): Whisper's per-recording auto-detect flipped to English on
German speech with English loanwords and produced a poor translation instead
of a transcript. English terms inside German sentences still come out fine.
To dictate in another language (or restore auto-detect), edit/remove that
flag in `dictate`.

---

## Dependencies

- `pw-record` — mic recording (PipeWire, installed by default on CachyOS)
- `curl` + `jq` — send audio to API, parse JSON response
- `notify-send` — desktop notifications for recording state
- `ydotool` — simulate Ctrl+V paste (kept as daemon for future use)
- `wtype` — type Unicode text directly into the focused Wayland window (layout-independent)
- Lemonade server running with `Whisper-Large-v3-Turbo` loaded on the
  `whispercpp:vulkan` (GPU) backend — not the NPU/FLM build (it conflicts
  with the background agent and can evict all loaded models)

---

## Setup

### 1. Install ydotool
```fish
shelly install ydotool
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

---

## Troubleshooting

### "Nothing recognized" on every recording
Almost always Lemonade, not dictate. Check first:
```fish
systemctl status lemond
curl -s http://localhost:13305/api/v1/models | jq -r '.data[].id'
```

**Known cause (2026-08-14):** an `mbedtls` upgrade to 4.1.0 shipped
`libmbedtls.so.23`, but `lemonade-server` 11.5.2-1.1 is built against
`libmbedtls.so.21` from mbedtls 3.x. `lemond` then crash-loops with
`error while loading shared libraries` (exit 127) and dictate gets no
transcription back.

Fix — old libs from the pacman cache, for this one service only:
```fish
# extract mbedtls 3.6.5 (still in /var/cache/pacman/pkg/) somewhere temporary
tar --use-compress-program=unzstd -xf /var/cache/pacman/pkg/mbedtls-3.6.5-1-x86_64.pkg.tar.zst -C /tmp/mb usr/lib
sudo install -d /usr/lib/lemonade-compat
sudo cp -a /tmp/mb/usr/lib/libmbed{tls,x509,crypto}.so.* /usr/lib/lemonade-compat/
sudo install -d /etc/systemd/system/lemond.service.d
# /etc/systemd/system/lemond.service.d/mbedtls-compat.conf:
#   [Service]
#   Environment=LD_LIBRARY_PATH=/usr/lib/lemonade-compat
sudo systemctl daemon-reload && sudo systemctl restart lemond
```
The rest of the system stays on mbedtls 4. **Delete the drop-in and
`/usr/lib/lemonade-compat/` once a rebuilt `lemonade-server` (> 11.5.2-1.1)
lands in the repos** — check with `pacman -Si lemonade-server`.

Not done instead: downgrading `mbedtls` (breaks everything else built
against so.23) or symlinking so.23 → so.21 (mbedtls 3 → 4 is an ABI break,
TLS would crash).

### Satzende fehlt: letzter Buchstabe weg, kein Punkt

Aufgeklaert am 2026-09-09. Es waren zwei unabhaengige Fehler.

Das Werkzeug dafuer war eine Zeile, die jede Transkription mitschrieb. Sie ist
wieder heraus -- dictate laeuft ohne. Fuer den naechsten Verdachtsfall reicht
es, sie hinter der `set text`-Zeile wieder einzusetzen:
```fish
echo (date -Is)" | "$text >> /tmp/dictate.log
```
Sie protokolliert jedes Diktat im Klartext, gehoert also nur voruebergehend
ins Script.

**Fehler 1: zerschnittene Woerter (behoben).** Whisper bricht `.text` hart bei
~55 Zeichen um, auch mitten im Wort. In fish wird eine Kommandosubstitution an
Zeilenumbruechen in eine *Liste* zerlegt; `"$text"` fuegt diese Liste dann mit
Leerzeichen wieder zusammen. Aus `Klassen` wurde so `Kl assen`, aus
`Expertise` `Expert ise`. Das betraf jedes Fenster, nicht nur VS Code.
`tr -d '\n'` entfernt die Umbrueche jetzt ersatzlos, bevor fish splittet.

**Fehler 2: verschluckte Satzenden (behoben, ausserhalb von dictate).** Im
VS-Code-Terminal fehlte an *jedem* Punkt der Satz-Punkt plus der Buchstabe
davor (`gut.` -> `gu`, `worden.` -> `worde`). Im Log stand der Satz
vollstaendig, Audio und Whisper waren also unschuldig -- und deterministisch
an einem Zeichen statt zufaellig, ein Timing-Problem war es damit auch nicht.

Der Gegentest, mit dieser Zeile im Script, diktiert in ein Programm ohne TUI:
```fish
cat > /tmp/dictate-test.txt   # diktieren, Enter, Ctrl+D
diff (tail -1 /tmp/dictate.log | cut -d'|' -f2- | psub) /tmp/dictate-test.txt
```
Die Datei kam fehlerfrei an. `wtype` liefert also korrekt ab; kaputt war die
*Darstellung* in der TUI. Ursache ist VS Codes Local Echo: das integrierte
Terminal sagt Tastendrucke lokal voraus, um Latenz zu kaschieren, und
verrechnet sich mit einer TUI, die dieselbe Zeile selbst neu zeichnet.
Abgeschaltet am 2026-09-09 in `~/.config/Code/User/settings.json`:
```json
"terminal.integrated.localEchoLatencyThreshold": -1
```
Kehrt der Fehler zurueck, bleibt als naechster Schritt Einfuegen statt Tippen
(`wl-copy` + ein Ctrl+V-Anschlag statt hunderter Tastenevents) -- das umgeht
die Tastendruck-Verarbeitung der TUI komplett. Die Paste-Taste ist allerdings
je nach Terminal Ctrl+V oder Ctrl+Shift+V, deshalb erst bei Bedarf.

Nicht die Ursache: abgeschnittenes Audio, und auch kein Timing. Beides war
zwischenzeitlich im Script (`sleep 0.4` vor dem `kill`, `wtype -d 2`) und ist
wieder heraus -- es brachte keinen Fix und kostete nur Latenz.
