# Changelog

## 0.2.0 — 2026-05-20

Initial public release as a standalone repo, split out from the
`orca-perf` branch where this feature was developed.

### Highlights

- Single-keybinding (`Orca+R`) entry into a virtual OCR cursor over
  the focused window's recognized text.
- Character / word / line navigation (NumPad arrows + `KP_End` /
  `KP_Page_Down`).
- Text selection (Shift + NumPad nav) and copy-to-clipboard (`KP_+`).
- Click pass-through (`KP_/`, `KP_*`, `KP_Enter`) — synthesized
  mouse events delivered to the source window with no on-screen
  overlay.
- Async Tesseract pipeline (Orca stays responsive during the
  0.3–2 s recognition).
- Three screen-capture backends auto-selected at runtime:
  Gdk (X11), ImageMagick (X11), xdg-desktop-portal (Wayland).
- Modal-key discipline: while OCR mode is active, the NumPad keys
  that normally drive flat-review are cleanly suspended and
  restored on exit.
- Package layout (`manifest.toml` + `*.py`) suitable for installation
  via `orca --install-extension foo.orca-ext`.

### Distribution

- Ships as `ocr.orca-ext` (10.9 KB, 6 files) installable via
  `orca --install-extension`.
- Source available as the package directory (this repo).
- `build-orca-ext.sh` rebuilds the archive from source.

### Known limitations

- Requires the controller-API extensions proposed for upstream
  Orca but not yet merged. See README "Compatibility" for the
  full list. Works on
  [orca-perf](https://github.com/churst90/orca-perf) today.
- Capture is X11-direct or via xdg-desktop-portal on Wayland.
  Pure-Wayland portal path is implemented but untested by the
  author (no Wayland session in regular use).
- Tesseract recognition quality depends on Tesseract itself and
  on the rendered text. Small antialiased UI text gets ~95% with
  the 2x pre-OCR upscale; pathological inputs can be much worse.

## Pre-history

This feature was developed iteratively on the
[orca-perf](https://github.com/churst90/orca-perf) branch from
mid-May 2026 through several design iterations:

1. Modal `Gtk.Dialog` with TreeView (discarded — click pass-through
   broken by modal grab).
2. Non-modal dialog with `Atspi.Action.do_action` (discarded —
   only works for accessible targets).
3. "Invisible" `Gtk.Window` (discarded — `opacity=0` ignored on
   compositor-less MATE; window clamped on-screen by WM).
4. Pure-keybinding `Orca+arrow` cursor (discarded — clashed with
   Orca's existing `Orca+Down` binding).
5. **Current design**: pure-virtual cursor with NumPad-key mode
   discipline; no on-screen widget; clicks pass through cleanly.

See the orca-perf branch's `BRANCH_INFO.md` "Phase 4 OCR" section
for the full design history.
