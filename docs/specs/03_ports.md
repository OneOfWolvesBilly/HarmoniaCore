# 03. Ports Specification

This document defines the abstract Port interfaces that form the boundary between
HarmoniaCore's platform-neutral logic and platform-specific adapters.

All content in this specification is written in English and is language-neutral.
Code samples are illustrative only; actual implementations MUST follow the semantics
defined here while using idiomatic syntax for their target language.

---

## Port Design Principles

1. **Platform Independence**  
   Ports MUST NOT reference any platform-specific types or APIs.  
   All types crossing the Port boundary MUST be defined in `05_models.md`.

2. **Thread Safety**  
   All Ports MUST be safe to call from any thread unless explicitly documented otherwise.  
   Implementations MUST provide appropriate synchronization.

3. **Error Handling**  
   All recoverable errors MUST be signaled via `CoreError` (defined in `05_models.md`).  
   Error mapping rules are defined in `01_architecture.md`.

4. **Minimal Surface**  
   Ports define only essential operations required for cross-platform behavior parity.

---

## Document Structure

Each Port specification contains:

### Core Contract (Normative)
- Method signatures and return values
- Error conditions and mapping  
- Thread safety requirements
- Observable behavior that MUST be identical across platforms

### Adapter Notes (Non-normative)
- Implementation guidance is provided in separate adapter implementation documents:
  - Apple adapters: `docs/impl/02_01_apple_adapters_impl.md`
  - Linux adapters: `docs/impl/02_02_linux_adapters_impl.md`

---

## LoggerPort

Provides structured logging for debugging and parity validation.

### Interface

```text
protocol LoggerPort {
    debug(msg: String)
    info(msg: String)
    warn(msg: String)
    error(msg: String)
}
```

### Semantics

- **Thread Safety:** MUST be safe to call from any thread concurrently.
- **Performance:** Implementations SHOULD use lazy evaluation to avoid string formatting overhead when logging is disabled.
- **Output:** Implementations MAY output to any destination (console, file, system logger, etc.).
- **Format:** Implementations SHOULD include timestamp and log level in output.

---

## ClockPort

Provides monotonic time for timing measurements and parity validation.

### Interface

```text
protocol ClockPort {
    now() -> UInt64  // monotonic nanoseconds since unspecified epoch
}
```

### Semantics

- **Monotonic Guarantee:** Returned values MUST NEVER decrease, even across system sleep/wake.
- **Precision:** MUST provide nanosecond resolution or better.
- **Epoch:** The epoch is unspecified and implementation-defined. Only relative differences between calls are meaningful.
- **Thread Safety:** MUST be safe to call from any thread concurrently without synchronization.
- **Real-Time Safety:** SHOULD be safe to call from real-time audio threads (no allocations, no blocking).

### Usage

```text
let start = clock.now()
// ... perform operation ...
let end = clock.now()
let elapsed_ns = end - start
```

---

## FileAccessPort

Provides platform-neutral file I/O.

### Types

```text
type FileHandleToken  // opaque handle (implementation-defined)

enum FileSeekOrigin {
    start    // Seek from beginning of file
    current  // Seek from current position
    end      // Seek from end of file
}
```

### Interface

```text
protocol FileAccessPort {
    open(url: String) throws -> FileHandleToken
    read(token: FileHandleToken, buffer: UnsafeMutableRawPointer, count: Int) throws -> Int
    seek(token: FileHandleToken, offset: Int64, origin: FileSeekOrigin) throws
    size(token: FileHandleToken) throws -> Int64
    close(token: FileHandleToken)
}
```

### Semantics

**open(url)**
- Opens file at `url` for reading.
- Returns opaque `FileHandleToken` on success.
- Throws `CoreError.notFound` if file does not exist.
- Throws `CoreError.ioError` for permission denied or other I/O errors.
- Thread Safety: Safe to call concurrently for different files.

**read(token, buffer, count)**
- Reads up to `count` bytes into `buffer` from file associated with `token`.
- Returns actual number of bytes read (may be less than `count` at EOF).
- Returns `0` at end of file.
- Throws `CoreError.invalidState` if `token` is invalid.
- Throws `CoreError.ioError` for I/O errors.
- Thread Safety: Safe to call concurrently with different tokens.

**seek(token, offset, origin)**
- Changes the current file position for subsequent read operations.
- Throws `CoreError.invalidState` if `token` is invalid.
- Throws `CoreError.invalidArgument` if resulting position is invalid.
- Throws `CoreError.ioError` for I/O errors.
- Thread Safety: Undefined if called concurrently with same token.

**size(token)**
- Returns total size of file in bytes.
- Throws `CoreError.invalidState` if `token` is invalid.
- Throws `CoreError.ioError` for I/O errors.
- Thread Safety: Safe to call concurrently.

**close(token)**
- Closes file associated with `token` and releases resources.
- MUST be idempotent (safe to call multiple times).
- MUST NOT throw exceptions.
- Thread Safety: Safe to call concurrently with different tokens.

---

## DecoderPort

Decodes audio files to interleaved Float32 PCM.

### Types

```text
type DecodeHandle  // opaque handle (implementation-defined)
```

### Interface

```text
protocol DecoderPort {
    open(url: String) throws -> DecodeHandle
    read(handle: DecodeHandle, pcmInterleaved: UnsafeMutablePointer<Float>, maxFrames: Int) throws -> Int
    seek(handle: DecodeHandle, toSeconds: Double) throws
    info(handle: DecodeHandle) throws -> StreamInfo
    close(handle: DecodeHandle)
}
```

### Semantics

**open(url)**
- Opens audio file at `url` and prepares for decoding.
- Returns opaque `DecodeHandle` on success.
- Throws `CoreError.notFound` if file does not exist.
- Throws `CoreError.unsupported` if file format or codec is not supported.
- Throws `CoreError.decodeError` if file is corrupted or invalid.
- Thread Safety: Safe to call concurrently for different files.

**read(handle, pcmInterleaved, maxFrames)**
- Decodes up to `maxFrames` audio frames into `pcmInterleaved` buffer.
- Output format: Interleaved Float32 PCM, range [-1.0, 1.0].
- Returns actual number of frames decoded (may be less than `maxFrames` at EOF).
- Returns `0` at end of stream.
- Throws `CoreError.invalidState` if `handle` is invalid.
- Throws `CoreError.decodeError` if decoding fails.

**seek(handle, toSeconds)**
- Seeks to position `toSeconds` in the audio stream.
- Throws `CoreError.unsupported` if seeking is not supported for this format.
- Throws `CoreError.invalidArgument` if `toSeconds` is negative or beyond stream duration.
- Throws `CoreError.invalidState` if `handle` is invalid.

**info(handle)**
- Returns `StreamInfo` describing the audio stream (see `05_models.md`).
- Throws `CoreError.invalidState` if `handle` is invalid.
- Thread Safety: Safe to call concurrently.

**close(handle)**
- Closes decoder and releases resources.
- MUST be idempotent (safe to call multiple times).
- MUST NOT throw exceptions.

### Format Requirements

- **Output:** Interleaved Float32 PCM, sample values in range [-1.0, 1.0].
- **Supported Formats:** Implementations SHOULD support: WAV, AIFF, MP3, AAC, FLAC.
- **Unsupported Formats:** MUST throw `CoreError.unsupported` with descriptive message.

---

## AudioOutputPort

Outputs interleaved Float32 PCM to system audio hardware.

### Interface

```text
protocol AudioOutputPort {
    configure(sampleRate: Double, channels: Int, framesPerBuffer: Int)
    start() throws
    stop()
    flush()
    render(interleavedFloat32: UnsafePointer<Float>, frameCount: Int) throws -> Int
}
```

### Semantics

**configure(sampleRate, channels, framesPerBuffer)**
- Configures audio output parameters. MUST be called before `start()`.
- Thread Safety: MUST be called on main thread.

**start()**
- Starts audio output.
- Throws `CoreError.invalidState` if not configured.
- Throws `CoreError.ioError` if audio device cannot be started.

**stop()**
- Stops audio output. MUST be idempotent. MUST NOT throw.

**flush()**
- Clears all queued audio buffers without stopping the audio engine.
- Use case: seek operations — call `flush()` before decoding from a new position.
- MUST be idempotent. MUST NOT throw.

**render(interleavedFloat32, frameCount)**
- Provides audio data to be played.
- Returns number of frames actually consumed.
- Throws `CoreError.invalidState` if output is not started.
- **Real-Time Safety:** MUST NOT allocate memory, block, or acquire locks.

---

## TagReaderPort

Reads metadata tags from audio files.

### Interface

```text
protocol TagReaderPort {
    read(url: String) throws -> TagBundle
}
```

### Semantics

**read(url)**
- Reads metadata tags from audio file at `url`.
- Returns `TagBundle` containing extracted metadata (see `05_models.md`).
- Fields not present in file are left as `nil`/`null`.
- Throws `CoreError.notFound` if file does not exist.
- Throws `CoreError.ioError` for I/O errors.
- Throws `CoreError.unsupported` if file format does not support metadata.
- Thread Safety: Safe to call concurrently.

### Supported Tag Formats

Implementations SHOULD support:
- **ID3v1, ID3v2** (MP3)
- **Vorbis Comments** (FLAC, Ogg Vorbis, Opus)
- **MP4 metadata** (M4A, AAC)
- **APEv2 tags** (APE, some MP3)

---

## TagWriterPort

Writes metadata tags to audio files.

### Interface

```text
protocol TagWriterPort {
    write(url: String, tags: TagBundle) throws
}
```

### Semantics

**write(url, tags)**
- Writes metadata tags from `tags` to audio file at `url`.
- Only writes fields present in `tags` (non-nil/non-null).
- Preserves existing tags not present in `tags`.
- Throws `CoreError.notFound` if file does not exist.
- Throws `CoreError.ioError` for I/O errors (including permission denied).
- Throws `CoreError.unsupported` if platform or file format does not support writing.
- Thread Safety: MUST synchronize file writes if called concurrently.

---

## CueSheetPort

Parses CUE sheet files and returns a structured `CueSheet` model.

### Interface

```text
protocol CueSheetPort {
    parse(url: String) throws -> CueSheet
}
```

### Semantics

**parse(url)**
- Reads and parses the CUE sheet file at `url`.
- Returns a fully populated `CueSheet` (see `05_models.md`).
- Resolves the `FILE` directive path relative to the directory containing the `.cue` file.
- MUST NOT verify that the referenced audio file exists.
- `CueSheet.tracks` MUST be non-empty and sorted ascending by `startTime`.
- `CueTrack.endTime` for the last track MUST be `nil`.
- `CueTrack.endTime` for all other tracks is derived from the next track's `startTime` minus one CUE frame (1/75 s).
- Throws `CoreError.notFound` if the `.cue` file does not exist.
- Throws `CoreError.ioError` for I/O errors.
- Throws `CoreError.decodeError` if the CUE sheet is malformed (no `FILE` directive, no tracks, invalid timestamps).
- Thread Safety: Safe to call concurrently.

### CUE Timestamp Conversion

CUE sheet timestamps use MM:SS:FF format where FF is frames (1/75 second).

```text
seconds = MM * 60.0 + SS + FF / 75.0
```

Implementations MUST use this formula exactly to ensure cross-platform parity.

### Supported CUE Sheet Variants

Implementations SHOULD support:
- Single-file CUE sheets (one `FILE` directive, multiple `TRACK` entries)
- Per-track `TITLE` and `PERFORMER` fields
- `INDEX 00` (pre-gap) and `INDEX 01` (track start); `INDEX 01` is used for `startTime`

Implementations MAY ignore:
- `REM` comment lines
- `CATALOG`, `ISRC`, `SONGWRITER` fields
- `INDEX` entries other than `INDEX 01`

### Error Cases

| Condition | Error |
|-----------|-------|
| File does not exist | `CoreError.notFound` |
| File is not readable | `CoreError.ioError` |
| No `FILE` directive | `CoreError.decodeError` |
| No `TRACK` entries | `CoreError.decodeError` |
| Invalid timestamp format | `CoreError.decodeError` |
| Multiple `FILE` directives | `CoreError.unsupported` |

---

## Port Implementation Guidelines

### For Implementers

1. **Follow Semantics Exactly**  
   Observable behavior MUST match this specification, regardless of internal implementation.

2. **Error Mapping**  
   All platform errors MUST be mapped to `CoreError` categories as defined in `01_architecture.md`.

3. **Thread Safety**  
   Meet the thread-safety requirements specified for each Port.

4. **Real-Time Safety**  
   Methods called from audio threads (e.g., `AudioOutputPort.render()`) MUST NOT:
   - Allocate memory
   - Block or wait
   - Acquire locks

5. **Testing**  
   Every Port implementation MUST pass behavior parity tests defined in `api-parity.md`.

### For Service Authors

1. **Depend Only on Ports**  
   Services MUST NOT reference platform-specific adapters or APIs directly.

2. **Use Dependency Injection**  
   Receive Port implementations via constructor injection or factory.

3. **Handle All Errors**  
   All methods that throw MUST be wrapped in appropriate error handling.