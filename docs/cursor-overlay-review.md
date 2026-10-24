# Cursor overlay implementation review

## Overall

The implementation is sound and fixes the problem at the right layer. The important architectural choices are right:

- It sends the **actual X cursor bitmap, hotspot, and position**, not the guest scanout.
- It keeps the cursor on a **separate transparent layer above wmtr's game layers**.
- It excludes that layer from game-layer hiding and HUD policies.
- It detects visibility from the game's real X cursor state, which is what Wine changes when DayZ enters or leaves inventory.
- It keeps the Vulkan path untouched by requiring an actual remoted game layer.

It is more complex than it needs to be, though, and there are a few robustness issues worth knowing about.

## Main critique

### 1. A Metal layer is unnecessary for a static cursor image

The cursor overlay is just an image. It does not need:

- a `CAMetalLayer`,
- a render pipeline,
- command buffers,
- `nextDrawable()`,
- per-update Metal draws,
- premultiplied blend state written by hand.

A plain `CALayer` with `contents = CGImage` would do the same job: the compositor would scale and alpha-blend the cursor automatically. The image only changes when the cursor shape changes, and the layer only moves when the position changes.

That would remove most of `Renderer.setCursorImage` and `drawCursorOverlay`, and avoid the risk of `nextDrawable()` blocking the main thread when updates arrive faster than the display can present. The only care needed is the layer coordinate system: Core Animation on macOS is generally bottom-left-origin, unlike the Metal viewport path currently used.

This is the biggest simplification available.

### 2. The guest polls too much

Every 16 ms, `cursor.rs`:

- re-checks `_NET_ACTIVE_WINDOW`,
- reads `WM_CLASS` for each Xwayland display,
- calls `XFixesGetCursorImage`, which transfers the full cursor bitmap each time,
- then compares it to the previous image.

The bitmap is only sent on change, but it is **fetched** every frame. That is wasteful for an image that usually changes rarely.

A better split would be:

- use XFixes cursor-change notifications for `cursor-image`,
- use XInput2 motion events or `XQueryPointer` for position,
- fetch the full cursor image only when the cursor actually changes.

At minimum, the selected Xwayland display could be cached instead of re-querying the active window and `WM_CLASS` at 60 Hz.

### 3. The overlay is redrawn more often than necessary

`redraw()` calls `updateCursorLayer()` on every guest frame. That can create and present a cursor drawable even when the cursor has not moved and its image has not changed.

A dirty flag — redraw only when visibility, position, image, or layer attachment changes — would avoid a steady stream of unnecessary command buffers. This becomes less important if the overlay is changed to a plain `CALayer`.

### 4. Display discovery is not re-run after the first connection

`Cursor::connect()` only runs while `self.conns.is_empty()`. If:

- the game's Xwayland display appears after the agent first connects,
- an existing display dies,
- or the wrong XFixes display was selected initially,

the agent may never discover or recover the correct display. Failed `XFixesGetCursorImage` calls simply return; the stale connection is kept.

Periodic rescanning, or dropping failed connections and retrying, would make this more reliable.

The fallback in `select_index()` — "any XFixes-capable display" — can also select Steam's display before the game's active window is visible. Requiring `steam_app_` would be less surprising, though it may delay the first cursor update slightly.

### 5. Coordinate mapping assumes root == displayed game area

The host maps the X root position into the guest picture's fitted rectangle. That is correct when:

- the game is fullscreen,
- the X root matches the guest scanout,
- and wmtr's game layer fills the same rectangle.

It can drift if the game runs in a window, the game layer has a different viewport, or the remoted layer's scaling differs from the guest picture. More robust input would be the focused game window's geometry, or a normalized cursor position relative to the game layer.

### 6. Untrusted guest input needs tighter validation

The wmtr path already has a strong security caveat, but the cursor parser should still be hardened.

`Progress.swift` validates:

```swift
data.count == w * h * 4
```

where `w` and `h` come from guest text. A malformed line with very large values can overflow that multiplication and trap the host. `setCursorImage` performs the same multiplication.

There should be a conservative maximum, such as 512×512 or 1024×1024, checked before any multiplication or allocation. The hotspot should also be clamped or validated.

### 7. The text/base64 protocol is a little fragile for images

The cursor bitmap is sent as one long base64 line, which required raising the port buffer and line-splitter limits to 256 KB. That works for ordinary cursors, but:

- a maximum-size cursor can exceed the limit,
- a malformed or oversized line is silently dropped,
- base64 adds 33% overhead for every cursor change.

For occasional cursor changes this is fine. If animated cursors become common, chunked binary messages or a shared-memory path would be cleaner.

## Better or different designs

In rough order of effort:

1. **Replace the cursor `CAMetalLayer` with a plain `CALayer` carrying a `CGImage`.**  
   Simplest and probably best. Less code, no GPU draw per update, compositor handles scaling and blending.

2. **Make the guest event-driven.**  
   Use XFixes cursor notifications for image changes and XInput2/query-pointer for motion. Fetch the image only when it changes, and cache the active display.

3. **Composite the cursor inside wmtr/DXMT instead of as a separate host overlay.**  
   Architecturally the cleanest, because the cursor would be part of the remoted game layer and could never be misordered or misaligned. But it requires changes to wmtr or DXMT and a cursor metadata path, so it is much more invasive.

4. **Track cursor position on the host from input events.**  
   Tempting because the host already sends mouse movement, but it is not reliable: games warp, clamp, and hide the cursor, and relative input does not always correspond one-to-one with the displayed game cursor. The guest's X cursor position is the better source.

## Verdict

The solution is pragmatic and correct in principle. I would keep the protocol and the separate-overlay architecture, but simplify the overlay to a plain `CALayer`, reduce the polling cost, harden the size validation, and make the Xwayland connection lifecycle self-healing. Those changes would preserve the behaviour while making the implementation smaller and more robust.
