---
name: bcd-format
description: Decode/encode ".bcd" clip files — an original format (by itsRealM13C) packing nibble-based RGB332 video plus trailing mu-law audio. Use when the user has a .bcd file, a file that starts with the magic bytes "BCD", or raw dumps matching the legacy headerless stream (bytes restricted to 0x00/0x01/0x10/0x11 with 0xFF/0xEE/0xAA markers).
---

# BCD clip format

An original format designed by itsRealM13C — not a CuBuds/hardware format;
the device name that came up earlier in discovery was unrelated. Two
variants exist: a legacy **headerless stream** (video only) and a
**headered** variant (magic `BCD`) that adds audio. Always check for the
magic bytes first.

## Headered variant (preferred)

Byte 0-2: ASCII `"BCD"`
Byte 3: version (seen: `0x02`)

All multi-byte header fields are **big-endian uint16**, starting at byte 4:

| Offset | Field                     | Notes                                  |
|-------:|---------------------------|-----------------------------------------|
| 4      | width                     | pixels (seen: 160)                      |
| 6      | height                    | pixels (seen: 284)                      |
| 8      | fps                       | video frame rate (seen: 10)             |
| 10     | reserved                  | seen as 0                               |
| 12     | audio sample rate (Hz)    | seen: 8000                              |
| 14     | reserved                  | seen as 0                               |
| 16     | frame count               | number of AA-terminated video frames    |
| 18-21  | audio length (bytes)      | **two u16 BE words, hi then lo** — combine as `(hi<<16)|lo` for the 32-bit byte length of the trailing audio block |
| 22-25  | reserved                  | seen as 0                               |
| 26     | sync byte                 | seen as `0xFF`, precedes pixel data     |

Video/pixel data starts at **byte offset 27** and runs until
`file_length - audio_length`. The trailing `audio_length` bytes at the
very end of the file are raw **8-bit G.711 mu-law** samples at the
header's sample rate — decode directly, no pixel unpacking involved.

## Legacy headerless variant

No magic bytes. Pixel data starts at byte 0. No audio is present —
every byte in the file belongs to the pixel/marker alphabet below.
Width/height are not stored; 160x284 has been the only size observed
so far. Treat any other size as unconfirmed.

## Pixel/video encoding (both variants)

- Each **pixel** is encoded as **4 consecutive bytes**, each byte one of
  `0x00, 0x01, 0x10, 0x11` only. Read each byte's two hex digits as two
  binary bits (`hi = byte>>4`, `lo = byte&0xF`, both always 0 or 1) to
  get a 2-bit value 0-3: `val = (hi<<1)|lo`.
- The four 2-bit values of a pixel's group pack MSB-first into one byte:
  `pixel = (v0<<6)|(v1<<4)|(v2<<2)|v3`.
- That packed byte is **RGB332**: bits 7-5 = red (0-7), bits 4-2 = green
  (0-7), bits 1-0 = blue (0-3). Scale red/green by `*255/7`, blue by
  `*85`.
- **Markers**, appearing immediately after the 4-byte group's position:
  - `0xFF` — ends a pixel group (always present, one per pixel).
  - `0xEE` — ends a row (appears right after that pixel's `0xFF`).
  - `0xAA` — ends a frame (appears right after that row's `0xEE`).
- A frame is `height` rows of `width` pixels. Frames seen so far have
  several blank (constant-value) rows top and bottom — crop per-file by
  finding rows/cols that are constant across every frame, don't hardcode
  a fixed crop.
- Known container quirk: a file can be a byte-for-byte duplicate of a
  prefix of another file (seen: an 8-frame file whose frames 2-7 exactly
  matched another file's frames 0-5). Check for this before treating two
  dumps as distinct clips.
- Known content quirk: within one AA-terminated frame, consecutive
  sub-regions can be near-exact horizontal mirrors of each other (~85%+
  pixel match when flipped vs ~55% unflipped). This looked like a
  dual-earbud mirrored display rather than real animation progression —
  verify with the same flip-vs-same pixel-match test before assuming
  extra sub-frames are distinct content.

## Audio decoding

Trailing `audio_length` bytes, standard G.711 mu-law, 8-bit, mono.
Decode with the standard ITU-T G.711 mu-law-to-linear table (do NOT
treat these bytes as part of the pixel alphabet — unlike the video
region, they span the full 0-255 byte range, that's the tell you're in
the audio block). Sample rate from header offset 12 (seen: 8000 Hz).
Package as 16-bit PCM WAV for playback.

Validate before trusting a decode: real audio here showed lag-1
autocorrelation ~0.85 decaying smoothly through longer lags. Noise or
a mis-aligned offset gives autocorrelation near 0 — check this before
reporting a successful audio extraction.

Playback direction (forward vs reversed) is a per-clip property, not
a format constant — don't assume every .bcd file needs the same
reversal. Check each clip individually.

## Quick identification checklist

1. Bytes 0-2 == `"BCD"` → headered variant, parse header fields above.
2. Otherwise, check if the whole file's byte alphabet is only
   `{0x00,0x01,0x10,0x11,0xFF,0xEE,0xAA}` (plus maybe `0xFF` padding at
   the very start, which is flash-erase filler, not data) → legacy
   headerless video-only variant.
3. Otherwise, it's not this format — don't force-fit it in.

## Known-working reference implementation

See `bcd_extractor.html` (sibling file, if present) for a complete,
self-contained parser: header parsing, pixel/frame decode, blank-border
crop, frame dedupe, and mu-law-to-WAV audio extraction, all in-browser.
