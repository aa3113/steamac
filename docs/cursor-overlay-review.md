# Cursor overlay

The DXMT path presents the game in a host `CAMetalLayer` above the guest scanout, so the guest
compositor's cursor is covered. The overlay is a separate transparent `CALayer` above the remoted
game layers; it carries the game's real XFixes cursor bitmap, hotspot, position and visibility.

## Data path

The guest agent (`guest/progress-agent/src/cursor.rs`) finds the Xwayland display whose active
window's `WM_CLASS` starts with `steam_app_`, follows the pointer with `QueryPointer` every 16 ms,
and fetches `XFixesGetCursorImage` every 50 ms. It sends only changes:

```text
cursor 0|1
cursor-image <w> <h> <xhot> <yhot> <base64 BGRA>
cursor-pos <x> <y> <win_w> <win_h>
```

`cursor-pos` is in the focused game's X **window client** pixels, from `QueryPointer` on that
window, not the X root. The client size is refreshed with `GetGeometry` when the target is
resolved. A 1×1 or fully transparent cursor is hidden, as Wine uses for a game without a cursor.

The launcher parses the messages in `Progress.swift` and forwards them through `Window.swift` to
`VMView` in `Presenter.swift`, which owns the transparent `CALayer` named
`steamac.cursor-overlay` at `zPosition` 1000. The cursor layer is excluded from the game-layer and
Metal HUD policies and is hidden unless a game layer is visible and the guest cursor is visible.

## Coordinate mapping

The cursor must follow the remoted game layer, not the guest scanout. The game layer's drawable is
aspect-fit into the layer (`contentsGravity = .resizeAspect`), so the overlay computes:

```swift
let fit = Renderer.fit(content: game.drawableSize, in: game.bounds)
let shown = game.convert(fit, to: metalLayer)
let sx = shown.width / CGFloat(window.width)
let sy = shown.height / CGFloat(window.height)
```

It then places the cursor image at `(x - xhot, y - yhot)` in that rect, scaled by the same factors.
Using `game.drawableSize` rather than the guest scanout fixes drift when the remoted swapchain's
size or aspect differs from the guest picture. Using window-relative coordinates fixes drift when a
game runs windowed. `CALayer.convert` keeps the mapping correct if wmtr ever places a game layer
somewhere other than the full VM view.

## Validation

Both ends cap cursor dimensions at 256×256 and check multiplication overflow before allocating or
creating the image. The hotspot is clamped to the image. The guest's cursor line buffer and the
host parser allow a maximum-size bitmap (about 350 KB of base64) within their 512 KB pending-line
limits.

## Remaining limitations

- The position is still polled at 16 ms rather than driven by XInput2/XFixes events. The active
  window/geometry are re-resolved every 250 ms, and displays are rescanned every 5 s.
- With several visible game layers, the overlay uses the topmost one. There is no explicit binding
  between the focused X window and a particular wmtr `CAMetalLayer`; a title with multiple
  swapchains could need wmtr to mark the active layer.
- `sx` and `sy` are separate. This is exact when the swapchain matches the X client area (the
  normal case); a swapchain with a different aspect than the window would stretch the cursor.
- The bitmap is base64 in one text line. Animated cursors would benefit from XFixes change
  notifications and a binary/chunked path.
- The cursor is not part of the game drawable, so it is not in game screenshots or captures. This
  is a consequence of keeping cursor updates independent of the game's present rate.

## Alternative not taken

Drawing the cursor into DXMT's game drawable before present would remove the coordinate mapping
entirely and make the cursor part of the game image. It would also tie cursor motion to the game's
present rate (a paused or static game would freeze the cursor), require format/colour-space-aware
blending in the wmtr/DXMT presentation path, and need a way to identify the correct swapchain.
The separate layer keeps cursor updates independent and leaves the vendored DXMT path untouched.
