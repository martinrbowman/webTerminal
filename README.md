# webTerminal

A browser-based serial port monitor and terminal for embedded development — no install, no drivers, no backend. Point a Chromium browser at it, pick a port, and go.

## Requirements

- **Browser:** Chrome, Edge, or Opera (desktop). Firefox and Safari don't implement the Web Serial API and aren't supported — the app will show a warning banner if you open it in one.
- **Connection:** the page must be served over **HTTPS**, or from `localhost`. Plain `http://` on any other host will not work (the Web Serial API requires a secure context).
- Nothing to install. Everything runs client-side in the tab; no data leaves your machine.

## Connecting to a device

1. Open **Port** (top-left panel) and set baud rate, data bits, parity, stop bits, and flow control to match your device.
2. Click **Connect** — your browser's native device picker opens; choose the serial adapter (FTDI, CP210x, CH340, etc.) from the OS-level list.
3. Status shows `connected` once open. **Disconnect** closes the port cleanly.

Note: flow control only offers **None** and **Hardware (RTS/CTS)** — that's a Web Serial API limitation, not a bug; software XON/XOFF and DTR/DSR flow control aren't exposed by the browser.

## Capture view (center panel)

- **RX** (top, large): everything received from the device, live.
- **TX** (bottom, fixed height): everything you've sent.
- Both auto-scroll to the latest data as long as you're scrolled to the bottom. Scroll up to read back through history — auto-scroll pauses, and a **↓ Jump to latest** button appears.

### View modes (right panel → View)

- **ASCII** — plain text. Non-printable bytes are shown safely, never executed: `\r` → `␍` (if enabled), control codes → `^X`, byte codes above 0x7F → `\xNN`.
- **ASCII (ANSI colors)** — interprets `\x1B[...m` color/bold escape codes from devices with colored shell prompts (Zephyr, U-Boot, etc.) instead of showing raw escape noise.
- **Hex + ASCII** — classic hex dump: offset, hex bytes, ASCII column.

Other toggles in the same panel:
- **Show timestamps** — off by default. Turning it on prefixes each received chunk with its time, but since chunks don't align to logical lines, this can visually split a single line across several timestamped entries — that's expected, not a bug.
- **Show CR/LF symbols** — off by default (clean text). On shows `␍`/`␊` markers alongside the line break.
- **Buffer size** — circular buffer, 1–100MB (default 10MB). Oldest data drops once the limit is hit.
- **Clear buffer** — wipes the capture buffer and any bookmarks.

## Searching and highlighting

The search bar (above the capture panel) supports:
- **Text** mode — substring or **Regex**, with an optional **Case match** toggle.
- **Hex** mode — byte pattern, space-separated, no `0x` prefix needed (e.g. `41 42 FF`).
- **RX / TX** scope checkboxes — limit the search to one direction or both.

Matches highlight inline in whichever view mode is active (ASCII, ANSI, or Hex). Matching runs off the main thread with a timeout guard, so a bad regex can't freeze the UI.

## Sending data

### Manual transmit
The box under the capture panel sends arbitrary data:
- **ASCII** mode supports escape sequences: `\r`, `\n`, `\t`, `\0`, `\xNN`.
- **Hex** mode takes space-separated bytes, no `0x` prefix: `01 02 FF 0A`.
- **Append CR/LF** is on by default. Pressing Enter on an empty field sends a bare line terminator, same as hitting return on a real terminal.

### Macros (left panel, under Handshake Lines)
- Three one-click presets: **ASCII Letters** (A-Z/a-z), **ASCII Numbers** (0-9), and **BERT-511** — a real ITU-T O.153 PRBS9 pseudorandom bit-error-rate test pattern (511-bit period), not a placeholder.
- Two custom slots: pick ASCII or Hex, type your content, optional **Append CR/LF**, and hit Send. Saved with your config (see below).

## Handshake lines

Live status of **DCD, CTS, DSR, RI** (polled every 100ms — the Web Serial API doesn't offer a change event). **DTR** and **RTS** are outputs you control via the checkboxes. State transitions are logged with timestamps below.

**⟳ Reset device** pulses DTR low then high — the standard auto-reset trick most Arduino/ESP-style boards use (same mechanism as avrdude/esptool).

*Limitation: single port at a time — no simultaneous multi-port monitoring yet.*

## Bookmarks

In the default view (ASCII, timestamps off), hover an RX line and click the dot in the left gutter to bookmark it. The **Bookmarks** panel (right column) lists them with a text preview — click one to jump straight back to that line in the RX pane.

*Note: bookmarks track a line's position in the current buffer. If the circular buffer trims old data on a long session, a bookmark can drift to point at the wrong line.*

## Logging and export

**Log & Export** (right column, collapsed by default):

- **Format:** text, binary, CSV, or JSON.
- **Include timestamps** (text format only, off by default) — off gives you the raw captured text exactly as it streamed in; on prefixes each chunk with `[ms] DIRECTION`.
- **Start logging to file** — continuous write to disk via the File System Access API. You pick the file once; the browser needs a Chromium-based build (same requirement as Web Serial). At 5MB the log closes automatically and a banner asks you to start a new file — browsers require a click for `showSaveFilePicker`, so true silent rotation isn't possible.
- **Export buffer now** — one-shot download of everything currently in the buffer, no file picker needed.
- **Copy visible text** — copies the current RX view to the clipboard.

### Config save/load
**Save config** downloads a JSON file with your port settings, view preferences, and custom macro slots. **Load config** restores them. Nothing is auto-persisted — it's an explicit action, so your settings won't silently change between sessions.

## Appearance

The footer status bar shows connection state, buffer usage, app version, and a **☀ Light / 🌙 Dark** toggle — your choice is remembered across visits.

## Known limitations

- Chromium desktop browsers only (Web Serial + File System Access API support).
- One port at a time.
- No CLI/scripting or CI integration — this is a browser tool by design, with no backend. See `spec.md` §10 for a possible companion-CLI direction.
- Continuous log rotation requires a manual click when a file hits its size cap (browser security constraint, not a design choice).

For the full functional specification and the reasoning behind these constraints, see [`spec.md`](spec.md).
