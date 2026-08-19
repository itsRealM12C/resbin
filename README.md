# resbin
A res.bin (similar to GIF) extractor.

# Origins
`res.bin` originally came from a product called HONOR CHOICE CuBuds. I've
got file access and I've extracted 4 `res.bin`s. But there's more, but I
didn't wanted to waste my storage with these.

---

## Format background

`res.bin` is a **flat frame-table + palette-indexed bitmap** container —
structurally the closest comparison is GIF (palette-indexed frames), not a
transform-coded video format. There's no compression on the pixel data
itself; each frame is a fixed-size palette lookup table followed by raw
8-bit indexed pixels, one byte per pixel, no RLE/LZW packing at all. That's
the "similar to GIF" comparison: same *palette-indexed* idea, but even
simpler — GIF at least LZW-compresses the indexed data, this doesn't.

### Frame table

```
offset 0x00        : (unidentified, 0x10 bytes — not parsed by this tool)
offset 0x10        : frame table start (HEADER_OFFSET)

each frame table record is 28 bytes (RECORD_SIZE):
  +0x00  12 bytes   unidentified (not read by this tool — likely
                     timing/flags/name/reserved fields)
  +0x0C   4 bytes   frame data offset   (u32 LE, absolute file offset)
  +0x10   4 bytes   frame data size     (u32 LE, bytes)
  +0x14   8 bytes   (unread — record is 28 bytes total, only 20 are consumed)

51 records read sequentially from 0x10, i.e. records occupy
0x10 .. 0x10 + 51*28 = 0x10 .. 0x596
```

The table is walked positionally (`fi * 28` from `0x10`) rather than via any
in-file count field — `51` is a constant baked into the extractor from
observation of the sample files, not read from a header field. Worth
double-checking against new dumps in case some CuBuds variants ship a
different frame count.

### Frame data (at each record's offset/size)

```
+0x000            palette: 256 entries × 4 bytes = 1024 bytes total
                     each entry: [ unused, R, G, B ]  (byte 0 of each
                     4-byte entry is skipped/ignored — possibly an alpha
                     or index-echo byte, not used for color)
+0x400 (1024)     raw pixel data: 296 × 240 = 71,040 bytes,
                     1 byte per pixel = palette index (0-255), row-major,
                     no padding/stride between rows
```

Fixed frame dimensions (`296×240`) are hardcoded in the extractor rather
than read from any per-frame field — every frame in the observed samples is
this exact resolution, consistent with this being a small onboard display
(charging-case/earbuds status screen) rather than a general video format.

### Why "similar to GIF, but simpler"

- **Shared idea**: both are palette-indexed — a small color table (≤256
  entries) plus a per-pixel index into it, rather than storing full RGB per
  pixel. This is the classic space-saving trick for small embedded/LCD
  displays with limited color depth and limited flash storage.
- **Difference**: GIF's indexed pixel stream is LZW-compressed and frames
  can be sub-regions with disposal methods, transparency, interlacing, etc.
  `res.bin` has none of that — every frame is a full, uncompressed
  296×240 index buffer at a fixed offset/size the frame table already
  tells you, which is why extraction here is a straight `slice()` + palette
  lookup with no decompression step at all.

---

## Extraction pipeline

```
res.bin file
   │  <input type="file"> select
   ▼
ArrayBuffer → Uint8Array + DataView
   │
   for fi in 0..51:
     read record at 0x10 + fi*28
     frame_offset = u32LE @ record+0x0C
     frame_size   = u32LE @ record+0x10
     frame_bytes  = data.slice(frame_offset, frame_offset+frame_size)
     palette      = frame_bytes[0..1024), take bytes [1,2,3] of every 4 → RGB
     pixels       = frame_bytes[1024..)   (296×240 indices)
   │
   ▼
per frame: build ImageData by palette[pixels[i]] → RGBA, draw to <canvas>
   │
   ├─▶ Export All JPGs — canvas.toDataURL('image/jpeg', 0.8) per frame,
   │                      staggered download (100ms apart) to avoid the
   │                      browser blocking rapid multi-file downloads
   │
   └─▶ Export GIF — gif.js (CDN, cdnjs), quality:20 + dither:false for
                     fast encoding over accuracy, 100ms per-frame delay
                     → re-encodes the raw frames into an actual
                       (now LZW-compressed) animated GIF
```

The GIF export is genuinely re-encoding here — `res.bin`'s own frames
aren't GIF-compatible data, they're raw indexed buffers, so `gif.js` is
doing real work (palette quantization/LZW) rather than just repackaging
bytes, unlike some container-extraction tools where "export as X" is closer
to a raw byte copy.

## Files

| File          | Purpose |
|---------------|---------|
| `index.html`  | The tool: frame-table walk, palette+pixel decode, canvas render, JPG/GIF export. Single file; depends on `gif.js` from cdnjs for the GIF export path only. |

## Known gaps / things to revisit

- The 12 unread bytes at the start of each frame-table record and the 8
  unread trailing bytes are unidentified — likely candidates: per-frame
  duration/timing, a name/tag string, or flags, based on position and
  size, but not confirmed.
- The first byte of each 4-byte palette entry is discarded without a
  confirmed reason (assumed non-color/padding/alpha).
- `51` frames and `296×240` are hardcoded constants from the observed
  samples, not read from any in-file count/dimension field — a file with a
  different frame count or resolution would need those constants adjusted
  manually.
- Only 4 `res.bin` samples have been examined so far, per the Origins note
  above — constants above should be treated as "true for this device family
  so far," not as a confirmed spec.
