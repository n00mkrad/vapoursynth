# VSPipe NUT Audio PoC Design (Single-Stream Audio + Video/Audio Mux)

## CLI and mode selection
- Existing `-o/--outputindex` remains the primary node selector.
- New option: `--audio-outputindex N`.
- `--audio-outputindex` is rejected unless `-c nut` is active.
- Dual-stream NUT mode is active only when:
  - `-c nut` is selected
  - `--audio-outputindex` is provided

### Dual-stream validation
- Primary output (`-o`) must be `mtVideo`.
- Secondary output (`--audio-outputindex`) must be `mtAudio`.
- Primary and secondary indices must differ.
- Alpha output is rejected in dual-stream NUT mode.
- `--start/--end` are rejected in dual-stream NUT mode.
- `--timecodes` and `--json` are rejected in dual-stream NUT mode.

### Single-stream behavior with `-c nut`
- Video-only still supported.
- Audio-only now supported.

## NUT writer model
- Writer now initializes from a stream descriptor list.
- Stream classes used:
  - `0` for video
  - `1` for audio
- Stream headers are written in list order.
- Per-frame API now accepts `stream_id`.

## Stream layout for this PoC
- Video+audio mode writes two streams:
  - Stream `0`: video
  - Stream `1`: audio
- Single-stream video mode uses stream `0`.
- Single-stream audio mode uses stream `0`.

## FourCC mappings
### Video
- Uses the existing `getVideoFourCC(...)` mapping logic from the prior NUT work.

### Audio (new)
- Signed 16-bit LE PCM: `PSD[16]`
- Signed 24-bit LE PCM: `PSD[24]`
- Signed 32-bit LE PCM: `PSD[32]`
- Float 32-bit LE PCM: `PFD[32]`

Unsupported audio formats are rejected with explicit error messages.

## Timestamping and interleave policy

### Video PTS
- Same as existing NUT video behavior:
  - Prefer `_AbsoluteTime` when present
  - Advance by `_DurationNum/_DurationDen` when present
  - Use CFR-derived fallback duration when properties are absent

### Audio PTS
- Derived from emitted sample count, not frame properties.
- Convert cumulative samples to NUT ticks using configured NUT timebase.
- Packet duration is implicitly represented by next packet timestamp progression.

### Packet order in dual-stream mode
- Dedicated synchronous mux loop fetches frames from both nodes and emits whichever stream has the earlier presentation time.
- Tie-breaker prefers video.
- This keeps output directly usable for mux/decode/playback pipelines without stream-sequential drift.

## Payload writing
- Frame headers are emitted immediately before payload.
- Existing payload writers are reused:
  - Video plane write path unchanged
  - Audio interleaving/packing path unchanged (same as WAV/W64 path)

## Error behavior matrix
- Invalid primary index: fail.
- Invalid secondary index: fail.
- Wrong media type on either selected index: fail.
- Unsupported audio format for NUT: fail.
- Alpha in dual-stream mode: fail.
- `--start/--end` in dual-stream mode: fail.

## Future extension points
- Subtitle stream class and packet emission.
- Wider audio format coverage (for example unsigned 8-bit).
- Optional index/trailer writing strategy for seek-focused workflows.
- Optional `--audio-start/--audio-end` semantics if dual-stream trimming is needed without script-side trim.
