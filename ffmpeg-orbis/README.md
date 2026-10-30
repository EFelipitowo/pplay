# PS4 hardware H.264 decoding for pplay

`h264_orbis` is a libavcodec decoder (patch for FFmpeg 5.0, the version PacBrew
ships) that drives the console's own video decoder, `libSceVideodec2`. mpv sees it
as the hwdec method **`orbis-copy`**, so the whole player (subtitles, OSD,
seeking, audio sync) stays unchanged: only the video decoding moves off the CPU.

## How it fits together

```
mpv (libmpv)  --hwdec=orbis-copy
   └─ libavcodec  h264_orbis            ffmpeg-orbis/0001-h264_orbis-decoder.patch
        ├─ h264_mp4toannexb bsf         MKV/MP4 (avcC) → Annex B access units
        ├─ SPS parser                   decides if the hardware can take the stream
        ├─ worker thread (16 MiB stack) every sceVideodec2* call runs here
        ├─ session (process wide)       compute queue + decoder + direct memory, never freed
        └─ NV12 → yuv420p copy          cropped, in ordinary memory
```

* **Subtitles / libass:** decoded pictures are copied into normal FFmpeg frames
  (`ff_get_buffer`) before mpv sees them. No GPU/Garlic/Onion buffer is ever shared
  with the renderer or libass, so subtitle drawing cannot collide with the decoder's
  frame buffers. Picture memory handed to `sceVideodec2Decode` is rotated
  (`dpb + 2` buffers) and read back before the next call.
* **Timestamps:** the decoder returns pictures in display order but without usable
  timestamps; input pts are kept in a sorted queue and the smallest is attached to
  every picture that comes out (correct with B-frames).
* **Memory:** all direct memory (compute queue, decoder CPU/GPU/CPU-GPU areas, frame
  pool, 3 × 4 MiB AU buffers) is allocated once and kept for the life of the process.
  A new file just `Reset`s the decoder. Releasing it is what makes the GPU fault.
* **Fallback:** only 8-bit 4:2:0 progressive Baseline/Main/High, level ≤ 5.1,
  ≤ 1920×1088 and ≤ `dpb` reference frames are accepted. Anything else (Hi10,
  4:2:2, interlaced, 4K...) is refused when the decoder opens and mpv decodes it in
  software automatically. A decoder error at the start of a file does the same.
  A call that hangs for 10 s marks the session broken and later files use software.

## Using it

`/data/pplay/pplay.cfg`:

| HWDEC     | meaning |
|-----------|---------|
| `"auto"`  | default. Hardware when possible, software otherwise, with the crash guard |
| `"no"`    | software only |
| `"force"` | like auto, but clears a previous automatic shutdown of the hardware decoder |
| other     | passed to mpv as is (`hwdec=<value>`), no guard |

**Crash guard.** While a file is being hardware decoded for the first seconds,
`/data/pplay/hwdec.guard` exists. If the console stops in that time, the next start
finds it, creates `hwdec.disabled` and uses software decoding. Delete
`hwdec.disabled` (or set `HWDEC="force"`) to try again.

Needs GoldHEN/HEN able to load `/system/common/lib/libSceVideodec2.sprx`. If a
plain module load fails, a GoldHEN jailbreak (syscall 500) is tried once.

## Build

`build-ffmpeg.sh` downloads FFmpeg 5.0, applies the patch, builds it with the
PacBrew configure line and installs it over the toolchain (the CI does this before
building pplay). `test/run.sh` builds the same patch for the PC and checks it against
a mock of libSceVideodec2 (bit-exact frames, pts order with B-frames, seeks,
re-opening, cropping, padding, errors, rejection of unsupported streams).

## Status

The decoder logic is tested on the PC against the mock. The real
`libSceVideodec2` could not be exercised from the build environment, so behaviour on
a console is based on the working code of Moonlight-ps4 and PS4 Cast (structure
layouts, memory types, call sequence). Treat the first runs as experimental.
