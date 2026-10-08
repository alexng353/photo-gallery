# photo-gallery

A keyboard-driven culling gallery for camera shoots, served over HTTP from one
Python file and one HTML page. Marks go straight into darktable XMP sidecars,
so darktable picks them up on import.

- Each frame shows once: the camera JPEG when there is one, otherwise the RAW's
  embedded preview.
- Near-identical bursts (within 4 s, dHash within 12 bits) collapse into one
  stacked tile.
- Full-screen view with 1:1 zoom at the clicked point, drag to pan, and an
  EXIF panel.
- A darktable-style rating strip (✕ ○ 1–5): click one value or drag across a
  range. Colour chips cycle *only* → *except* → off.
- Shoots are sections you can fold.
- Sidecars changed by something else (darktable, a script) are re-read on page
  load.

## Requirements

Python 3.9+, [Pillow](https://pypi.org/project/pillow/), `exiftool`, and
`vipsthumbnail` (libvips) on `PATH`.

## Run

```bash
./photo-gallery ~/Pictures/shoot-a ~/Pictures/shoot-b      # http://127.0.0.1:19000/
./photo-gallery --host tailscale --port 19000              # folders from the shoots file
```

With no folders on the command line it reads `~/.config/photo-gallery/shoots`:
one folder per line, `#` comments allowed. Missing folders are skipped; if none
exist it exits non-zero, so a supervisor can retry until the disk is mounted.
`--host tailscale` binds the machine's tailnet IPv4.

The first run on a shoot builds 480 px thumbnails and an EXIF cache into
`~/.cache/photo-gallery/<hash of folder>/`; later starts read only new or
changed files and take about a second. Files added to a shoot appear after a
restart.
`gallery.html` is re-read on every request, so page edits need only a reload.

To run it permanently as a systemd user service, see the comments in
[`photo-gallery.service`](photo-gallery.service).

There is no authentication: anyone who can reach the port can mark frames.
Bind it to localhost or a private network such as a tailnet, never a public
address.

## Keys

| Key | Action |
| --- | --- |
| `r` | reject (again to un-reject) |
| `1`–`5` · `0` | set stars · clear rating |
| `d` `f` `g` | red · green · no colour label |
| `Ctrl+Z` · `Ctrl+Shift+Z` | undo · redo |
| arrows / `hjkl` | move |
| `Space` / `Enter` / click | full screen |
| click / `z` in full screen | toggle fit ↔ 100%, drag to pan |
| `i` | EXIF panel in full screen |
| `e` · `Shift+E` | expand/collapse this stack · all stacks |
| `x` | clear every filter |
| `?` | help |

## Sidecars

- The target is `<RAW>.xmp` when a RAW exists, else `<JPG>.xmp`.
- Rating `-1` is darktable's reject; `0`–`5` are stars. Colour labels use
  darktable's indices (0 red, 2 green) in `darktable:colorlabels`.
- An existing sidecar is edited in place: only the rating and the colour-label
  block change (plus the pre-cull tag below), and `darktable:history` is left
  alone. A missing sidecar gets a minimal one that darktable fills in on import.
- darktable reads sidecars only on import, or on startup with "look for
  updated XMP files" enabled.

## Pre-cull suggestions

A vision pre-cull can leave its verdict as the start of `dc:description`:
`[tag] keep|maybe|reject | reasons ...`. Until you mark that frame yourself,
the gallery treats the verdict as a suggestion:

- A suggested reject shows a dashed red outline and an `AI ✕?` badge instead
  of the greyed-out look of a real reject. The header counts these separately.
- Press `r` on it to confirm the reject. Any other mark overrides it.
- When you mark a frame, the gallery appends `, reviewed` to the tag. If the
  rating already differs from the suggestion, the frame counts as reviewed
  anyway.
- Filter chips under *pre-cull* show only, or hide, suggested rejects and
  unreviewed suggestions. Full screen shows the model's reasoning.
