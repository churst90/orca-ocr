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

**0.3.0**, November 2026. Runs on stock upstream Orca built from
the main branch; no patched Orca build is needed any more. Working and
stable on the author's setup (Gentoo, MATE, X11, Voxin TTS).
Earlier versions were verified against:
- Orca's own preferences dialog (clicked categories)
- GNOME Calculator
- Terminal output (mate-terminal, gnome-terminal, kitty)
- Web browsers (Firefox, Chrome — for sites where AT-SPI is
  unreliable)
- The MATE panel and desktop

## Requirements

- **Orca built from upstream main, November 2026 or later**, which has the
  user-extensions framework, extension approval, and the
  controller and modal-input APIs this extension uses.
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
mkdir -p ~/.local/share/orca/extensions/ocr
cp __init__.py ocr.py buffer.py capture.py engine.py manifest.toml \
   ~/.local/share/orca/extensions/ocr/
orca --approve-extension ocr
orca --replace &
```

Orca only loads a user extension after it is approved, and the
approval is pinned to a hash of the package's files. After you
change or update any of the files, run
`orca --approve-extension ocr` again and restart Orca.

Then press `Orca+R` on any window with text.

## Uninstall

```sh
orca --revoke-extension ocr
rm -rf ~/.local/share/orca/extensions/ocr
```

Then restart Orca.

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

Orca-internal capabilities go through public APIs: the
extension controller (`present_message_internal`,
`get_active_window_internal`, `set_clipboard_text_internal`),
`command_manager` for the OCR-mode keys, and AT-SPI directly for
window extents and mouse clicks.

## How OCR mode claims the NumPad

The OCR-mode keys are NumPad keys, which flat review already
uses. Orca removes a user extension's binding when a built-in
command owns the same key, so the extension can't simply bind
them. Instead, while OCR mode is on, it registers itself as
Orca's modal input handler and claims those keys before flat
review sees them. Mode commands whose keys nothing else uses
stay registered in an `ocr-mode` activation group, which is only
active during OCR mode. Leaving OCR mode (`KP_Subtract`) hands
the NumPad back to flat review.

## License

LGPL-2.1-or-later. Same as Orca itself. See [LICENSE](LICENSE).

## Author and contact

Cody Hurst &lt;codythurst@gmail.com&gt;.

Bug reports and feature requests: [github issues](https://github.com/churst90/orca-ocr/issues).

Related projects:
- [GNOME Orca upstream](https://gitlab.gnome.org/GNOME/orca) —
  where the extension framework lives.
