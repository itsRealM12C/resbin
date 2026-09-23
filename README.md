# resbin

A res.bin (similar to GIF) extractor.

# Origins
`res.bin` originally came from a product called HONOR CHOICE CuBuds. I've
got file access and I've extracted 4 `res.bin`s. But there's more, but I
didn't wanted to waste my storage with these.

---

## "1.00" — keyframe/delta boot animation

Verified against `res.bin` (3,099,664 bytes, `/POWER/` resource):
296×240 canvas, 54 frames, all decoded as keyframes.

This is a multi-frame animation container: same per-row RLE idea as
BR02, but pixels are **palette indices** (1 byte each) instead of
3-byte ARGB8565, and frames can be full keyframes or deltas against
the last keyframe.

### Header (16 bytes)

| Offset | Type | Field |
|--------|------|-------|
| 0x00 | char[4] | magic `"1.00"` |
| 0x04 | u16 | canvas width |
| 0x06 | u16 | canvas height |
| 0x08 | u16 | frame count |
| 0x0A | u16 | default delay (ms) |
| 0x0C | u32 | reserved |

### Frame record (28 bytes each, array starts at 0x10)

| Offset | Type | Field |
|--------|------|-------|
| +0x00 | u16 | frame width |
| +0x02 | u16 | frame height |
| +0x04 | u16 | frame number |
| +0x06 | u16 | flags |
| +0x08 | u32 | metadata — **type = `(meta >> 16) & 0xFF`** |
| +0x0C | u32 | payload offset (absolute, from file start) |
| +0x10 | u32 | payload length |
| +0x14 | u32 | decoded byte count |
| +0x18 | u32 | reserved |

### Frame types

| Type | Name | Meaning |
|------|------|---------|
| 0x0B | KEY | full keyframe; own 1024-byte palette |
| 0x0F | dF  | delta frame; own 1024-byte palette |
| 0x0E | dE  | delta frame; reuses last keyframe's palette |

### Frame payload layout

- **KEY / dF**: bytes `[0x000, 0x400)` = 1024-byte palette (256 × 4-byte
  BGRA entries), row table starts at `0x400`.
- **dE**: no palette block; row table starts at `0x000`.

Row table format is the same packed-u32 scheme as BR02 (21-bit offset
relative to the row table's own base + 11-bit length), but each row's
RLE stream decodes to **1-byte palette indices**, not 3-byte pixels.

### Row RLE (palette-index variant)

- high bit set → repeat run: `count = low 7 bits`, followed by **one
  index byte**, repeated `count` times.
- high bit clear → literal run: `count` raw index bytes follow.

### Compositing

- A keyframe (0x0B) starts from a blank `width × height` indexed
  buffer.
- A delta frame (0x0E / 0x0F) starts from a **copy of the most recent
  keyframe's** indexed buffer — never the previous delta.
- Within a delta row, an index value of **0xFF means "unchanged"**:
  skip writing that pixel, leaving the keyframe's value in place.
- A frame smaller than the canvas is **right-aligned**:
  `x_offset = canvas_width - frame_width`, `y_offset = 0`.

### Rendering a frame

```
rgba[i].a = palette[index*4 + 0]
rgba[i].r = palette[index*4 + 1]
rgba[i].g = palette[index*4 + 2]
rgba[i].b = palette[index*4 + 3]
```

(Palette entries are stored **A, R, G, B** in that byte order.)

### Verified stats

- `res.bin`: 296×240 canvas, 54 frames, all type 0x0B (KEY) — this
  particular boot animation doesn't use delta frames, so it's really
  54 independent full images played back at the header's default
  delay (70 ms here).

---
