# Orca OCR

NVDA-style content recognition for the [Orca screen reader](https://wiki.gnome.org/Projects/Orca).

Press `Orca+R` on any window — accessible or not — and Orca
captures its pixels, runs them through Tesseract, and drops you
into a virtual buffer you can navigate with NumPad keys. Selection,
copy-to-clipboard, and synthesized mouse clicks all work. The
buffer never appears on screen; the application underneath stays
fully interactive.

## Why this exists

A lot of Linux applications are partly or entirely inaccessible
to AT-SPI. Some terminals (GNOME Console / VTE 4), most Electron
apps, games, kiosk UIs, scanned PDFs in document viewers — none
of these expose their text in a way a screen reader can read.
Sighted users see the text just fine because it's rendered to
pixels. This extension closes that gap by reading the pixels.

It's the screen-reader equivalent of NVDA's "Content Recognition"
feature, ported to Orca and adapted to the user-extension
framework that landed in Orca 51.

## Status

**0.2.0**, May 2026. Working and stable on the author's setup
(Fedora 44, MATE, X11, Voxin TTS). Verified against:
- Orca's own preferences dialog (clicked categories)
- GNOME Calculator
- Terminal output (mate-terminal, gnome-terminal, kitty)
- Web browsers (Firefox, Chrome — for sites where AT-SPI is
  unreliable)
- The MATE panel and desktop

## Requirements

- **Orca screen reader** with the user-extensions framework. As
  of this writing the framework's controller API is incomplete
  upstream; this extension needs four controller methods
  (`get_active_window`, `get_active_window_screen_rect`,
  `set_clipboard_text`, `synthesize_mouse_event`) plus the
  `enter_modal_mode` / `exit_modal_mode` pair, all of which are
  proposed as upstream issues but currently only exist in the
  [orca-perf branch](https://github.com/churst90/orca-perf). See
  "Compatibility" below for details.
- **Tesseract OCR** plus at least one language data pack:
  - Fedora / RHEL: `sudo dnf install tesseract tesseract-langpack-eng`
  - Debian / Ubuntu: `sudo apt install tesseract-ocr tesseract-ocr-eng`
  - Arch: `sudo pacman -S tesseract tesseract-data-eng`
- **Python 3.11+** (for `tomllib` in the loader; Orca itself
  bumped to this in late 2025).

## Install

```sh
git clone https://github.com/churst90/orca-ocr.git
cd orca-ocr
orca --install-extension ocr.orca-ext
```

That copies the package to `~/.local/share/orca/extensions/ocr/`,
records its SHA256 in dconf, and auto-approves it for loading.
Restart Orca:

```sh
pkill orca
orca --replace &
```

Then press `Orca+R` on any window with text.

## Uninstall

```sh
orca --uninstall-extension ocr
```

Removes the directory, revokes approval, and clears the
disabled-extensions entry if present.

## Usage

Press `Orca+R`. Orca says "Recognizing." while Tesseract works
(typically 0.3–2 seconds — the GLib main loop stays responsive
so other Orca commands continue to work during recognition). On
success Orca announces "OCR mode on. *N* lines. *first word*"
and the NumPad keys are temporarily remapped to navigate the
recognized text.

### Navigation

| Key | Action |
|---|---|
| `NumPad 8` | previous line |
| `NumPad 2` | next line |
| `NumPad 4` | previous word |
| `NumPad 6` | next word |
| `NumPad 1` | previous character |
| `NumPad 3` | next character |
| `NumPad 7` | first word of buffer |
| `NumPad 9` | last word of buffer |
| `NumPad 5` | re-speak the current word |

### Selection and copy

| Key | Action |
|---|---|
| `NumPad .` | set selection anchor at the cursor |
| `Shift +` *any nav key* | extend selection in that direction |
| `NumPad +` | copy selection to clipboard (or current line if no selection) |

Paste anywhere with the usual `Ctrl+V`.

### Click pass-through

| Key | Action |
|---|---|
| `NumPad /` | left-click in the source window at the cursor's word |
| `NumPad *` | right-click at the same position |
| `NumPad Enter` | left-click (alias of NumPad /) |

The click is a real synthesized mouse event (XTest on X11,
xdg-desktop-portal on Wayland) so the target application reacts
exactly as it would to a real click. No accessibility tree
required.

### Exit

| Key | Action |
|---|---|
| `NumPad -` | exit OCR mode |
| `Orca+R` (again) | exit OCR mode |

While the OCR cursor is active, the NumPad keys that normally
drive flat-review are temporarily suspended. On exit they're
restored exactly as they were.

## How it works

The extension is a single package (`~/.local/share/orca/extensions/ocr/`)
broken into focused modules:

| File | Purpose |
|---|---|
| `manifest.toml` | Extension metadata (name, version, entry point, compat) |
| `ocr.py` | `OcrExtension` class — entry module, all the commands |
| `buffer.py` | `OCRWord` / `OCRLine` / `OCRBuffer` dataclasses |
| `capture.py` | Three screen-capture backends: Gdk (X11), ImageMagick (X11), xdg-desktop-portal (Wayland) |
| `engine.py` | Tesseract subprocess wrapper + TSV parser |

Every Orca-internal capability the extension uses goes through
the controller API — `self.controller.get_active_window()`,
`self.controller.set_clipboard_text()`,
`self.controller.synthesize_mouse_event()`,
`self.controller.enter_modal_mode()`, etc. There are no direct
imports of `focus_manager`, `clipboard`, `ax_device_manager`,
or `command_manager`. This makes the extension a clean reference
for what's possible with the user-extensions framework.

## Compatibility

The user-extension framework's controller API has gaps that
this extension fills with proposals filed against upstream Orca.
None are merged yet; until they are, this extension only works
on the perf branch.

| Controller method | Upstream status |
|---|---|
| `present_message_internal` | ✓ in main |
| `get_active_window` / `_screen_rect` | proposed, not merged |
| `set_clipboard_text` / `get_clipboard_text` | proposed, not merged |
| `synthesize_mouse_event` | proposed, not merged |
| `enter_modal_mode` / `exit_modal_mode` | proposed, not merged |

If you're on stock upstream Orca, the extension will load
(the loader is forgiving) but pressing `Orca+R` will raise
`AttributeError` on the first controller call.

Use [orca-perf](https://github.com/churst90/orca-perf) in the
meantime, or wait for the upstream issues to land.

## Building from source

If you've changed any of the `.py` files and want to repackage:

```sh
./build-orca-ext.sh . ocr.orca-ext
# wrote ./ocr.orca-ext (10924 bytes)
```

The script zips the current directory into a deterministic
archive (sorted entries, no extended attributes) suitable for
distribution.

## License

LGPL-2.1-or-later. Same as Orca itself. See [LICENSE](LICENSE).

## Author and contact

Cody Hurst &lt;codythurst@gmail.com&gt;.

Bug reports and feature requests: [github issues](https://github.com/churst90/orca-ocr/issues).

Related projects:
- [orca-perf](https://github.com/churst90/orca-perf) — the perf
  branch of Orca that this extension is developed against, plus
  the proposed upstream API additions this extension uses.
- [GNOME Orca upstream](https://gitlab.gnome.org/GNOME/orca) —
  where the extension framework lives.
