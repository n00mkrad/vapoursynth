# VSPipe NUT Audio Feasibility (Audio + A/V PoC)

## Goal
Enable NUT output in `vspipe` for:
- Single-stream video
- Single-stream audio
- Combined video+audio muxing into one NUT stream

## Result
Feasible with moderate integration complexity and low format risk.

The existing `vspipe` NUT implementation already provides:
- NUT packet helpers (varint, CRC, packet/syncpoint writing)
- Stable per-frame video PTS handling, including VFR property support
- Existing packed audio payload write path used by WAV/W64

The main missing piece was orchestration of two output nodes into one interleaved output stream.

## Chosen design
- Keep an in-tree writer, extend it to multi-stream headers and per-stream frame headers.
- Keep stream count limited to 1 or 2 for this phase.
- Use explicit index pairing for A/V mode:
  - `-o` remains primary output index
  - `--audio-outputindex` selects secondary audio output for NUT mux mode
- Use synchronous `getFrame()` in the dedicated dual-stream path for deterministic packet interleave in timestamp order.

## Why explicit dual-stream indexing
- `vspipe` traditionally targets one selected output index.
- Auto-picking audio from all script outputs introduces ambiguity and brittle policy.
- Explicit mapping is predictable and script-author controlled.

## Why synchronous dual-stream path
- Existing async path is built around one monotonically indexed node and one reorder map.
- Extending that queueing model to two independent timelines is higher risk and harder to reason about.
- A dedicated synchronous mux loop is simpler for v1 and isolates behavior to `-c nut` + `--audio-outputindex`.

## Scope constraints in this phase
- Windows-first implementation target.
- No alpha with NUT A/V mode.
- No subtitles yet.
- No trailing index writing (pipe-oriented behavior retained).
- Dual-stream mode rejects `--start/--end` to avoid implicit A/V desync semantics; trimming should be done in script.
- Audio format scope intentionally limited to:
  - Signed LE PCM 16/24/32
  - Float LE 32

## Interop notes
- NUT raw audio FourCCs follow `P<type><interleaving><bits>` little-endian PCM convention.
- For this PoC we use default interleaved (`D`) raw PCM tags:
  - `PSD[16]`, `PSD[24]`, `PSD[32]`, `PFD[32]`

## Risk notes
- Timestamp comparisons are made in NUT ticks using integer arithmetic for stream ordering.
- This avoids floating-point drift in long-running sessions and keeps mux ordering deterministic for equal-time boundaries.

## Relationship to earlier video-only notes
- `notes/vspipe-nut-feasibility.md` and `notes/vspipe-nut-poc-design.md` describe the original video-focused NUT implementation.
- This file documents the additional audio and dual-stream feasibility decisions.
