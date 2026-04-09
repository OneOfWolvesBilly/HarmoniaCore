# 04. Services Specification

This document defines the HarmoniaCore service layer interfaces that are exposed
to applications. The Services constitute the primary cross-platform contract
and are built on:

- Ports defined in `03_ports.md`
- Models defined in `05_models.md`
- Platform-specific adapters (Apple / Linux) implementing those Ports

All content in this specification is written in English and is language-neutral.
Code samples are illustrative only.

---

## 4.1 Goals

1. **Cross-platform consistency**  
   The same service APIs MUST expose consistent semantics on all supported platforms.

2. **Ports-driven design**  
   Services MUST depend only on Ports and Models, never directly on AVFoundation,
   FFmpeg, PipeWire, TagLib, or any other platform-specific API.

3. **Minimal stable API surface**  
   Only core functionality is specified here (v0.1).
   Additional services will be specified in separate documents.

4. **Verifiable behavior**  
   Implementations MUST be testable via shared cross-platform test suites that
   exercise the same Service contracts.

---

## 4.2 PlaybackService

`PlaybackService` is the primary playback control service.
All platform implementations MUST conform to the behavior defined in this section.

### 4.2.1 PlaybackState

`PlaybackState` represents the lifecycle of playback.

Normative semantics:

- **`stopped`** — No active playback; resources are released. Position is treated as 0.
- **`playing`** — Audio is being rendered.
- **`paused`** — Playback is suspended; position is retained.
- **`buffering`** *(Optional)* — Transient state during EOF drain. MUST transition to `stopped` once drain completes.
- **`error(CoreError)`** — An error state associated with a `CoreError` value.

**State Transition Rules:**

```
[stopped] --load()--> [paused]
[paused]  --play()--> [playing]
[playing] --pause()--> [paused]
[playing/paused] --stop()--> [stopped]
[any] --error--> [error]
[error]  --load()--> [paused]  (recovery)
```

**Detailed State Transition Table:**

| Current State | Operation | Valid? | Next State | Notes |
|--------------|-----------|--------|------------|-------|
| stopped | load() | ✅ | paused | File opened, ready to play |
| stopped | play() | ❌ | — | Throws `invalidState` |
| stopped | pause() | ✅ | stopped | Idempotent (no-op) |
| stopped | stop() | ✅ | stopped | Idempotent (no-op) |
| stopped | seek() | ❌ | — | Throws `invalidState` |
| paused | load() | ✅ | paused | Replaces current file |
| paused | play() | ✅ | playing | Starts playback |
| paused | pause() | ✅ | paused | Idempotent (no-op) |
| paused | stop() | ✅ | stopped | Releases resources |
| paused | seek() | ✅ | paused | Remains paused |
| playing | load() | ✅ | paused | Stops current, loads new |
| playing | play() | ✅ | playing | Idempotent (no-op) |
| playing | pause() | ✅ | paused | Suspends playback |
| playing | stop() | ✅ | stopped | Stops and releases |
| playing | seek() | ✅ | playing | Continues playing |
| error | load() | ✅ | paused | Recovery path |
| error | play() | ❌ | — | Throws `invalidState` |
| error | pause() | ❌ | — | Throws `invalidState` |
| error | stop() | ✅ | stopped | Cleanup |
| error | seek() | ❌ | — | Throws `invalidState` |

---

### 4.2.2 Required API Surface

The following members are REQUIRED for `PlaybackService`:

1. `load(url: String)`
2. `play()`
3. `pause()`
4. `stop()`
5. `seek(to seconds: Double)`
6. `currentTime() -> Double`
7. `duration() -> Double`
8. `state: PlaybackState` (read-only)
9. `setVolume(volume: Float)`
10. `setPlaybackRange(start: Double, end: Double?)`
11. `clearPlaybackRange()`

---

#### 4.2.2.1 `load(url: String)`

- Prepares audio output and sets `state = paused`.
- Resets `currentTime()` to `0`.
- Resets any active playback range set by `setPlaybackRange()`.
- Initializes `duration()` from `StreamInfo`.

**Error Cases:**
- File not found → `CoreError.notFound`
- Unsupported format → `CoreError.unsupported`
- Corrupted file → `CoreError.decodeError`

---

#### 4.2.2.2 `play()`

- If `state` is `paused`: starts playback, sets `state = playing`.
- If `state` is already `playing`: no-op.
- If no track loaded: throws `CoreError.invalidState`.

---

#### 4.2.2.3 `pause()`

- If `state` is `playing`: suspends playback, retains position, sets `state = paused`.
- If already `paused` or `stopped`: no-op.

---

#### 4.2.2.4 `stop()`

- Stops playback, releases decoder/output resources, resets position to `0`.
- Clears any active playback range.
- If already `stopped`: no-op.

---

#### 4.2.2.5 `seek(to seconds: Double)`

- Moves playback position to the requested time.
- MUST call `AudioOutputPort.flush()` before submitting new decoded audio.
- `seek()` MUST NOT modify the active playback range.
- Throws `CoreError.invalidArgument` if `seconds` is negative or beyond `duration()`.
- Throws `CoreError.unsupported` if seeking is not supported.
- Throws `CoreError.invalidState` if no track is loaded.

**Seek Accuracy (Parity Requirement):**
- Uncompressed formats (WAV, AIFF): within ±1ms
- Compressed formats (MP3, AAC): ±100ms tolerance

---

#### 4.2.2.6 `currentTime() -> Double`

- `playing`: returns current rendering position in seconds (continuously advancing).
- `paused`: returns last known position (frozen).
- `stopped`: returns `0`.
- Range: `[0.0, duration()]`.

---

#### 4.2.2.7 `duration() -> Double`

- Returns total duration of loaded track, or `0` when no track is loaded.

---

#### 4.2.2.8 `state: PlaybackState`

- Exposes current playback state. MUST be updated atomically. Thread-safe for concurrent reads.

---

#### 4.2.2.9 `setVolume(volume: Float)`

- Sets the playback volume.
- `volume` range: `[0.0, 1.0]`. Values outside this range MUST be clamped.
- Safe to call in any state, including `stopped`.
- Thread Safety: MAY be called from any thread.

---

#### 4.2.2.10 `setPlaybackRange(start: Double, end: Double?)`

Sets a time-bounded playback region. When active, playback is confined to
`[start, end]` within the loaded file.

**Parameters:**
- `start`: Start position in seconds. MUST be `≥ 0.0` and `< duration()`.
- `end`: End position in seconds. If not `nil`, MUST be `> start` and `≤ duration()`.
  `nil` means play from `start` to EOF.

**Behavior:**
- `setPlaybackRange()` MUST be called after `load()`. It MUST NOT be called when
  `state = stopped` with no file loaded.
- When a range is active:
  - `seek()` to a position outside `[start, end]` MUST throw `CoreError.invalidArgument`.
  - When `currentTime()` reaches `end`, the service MUST automatically transition to
    `state = paused` and set `currentTime()` to `start`.
  - `currentTime()` MUST remain within `[start, end]`.
  - `duration()` continues to return the full file duration (unchanged).
- Calling `load()` clears any active range.

**Error Cases:**
- `start < 0` or `start ≥ duration()` → `CoreError.invalidArgument`
- `end ≤ start` or `end > duration()` → `CoreError.invalidArgument`
- No track loaded → `CoreError.invalidState`

**Illustrative Swift shape:**
```swift
func setPlaybackRange(start: Double, end: Double?) throws
```

**Illustrative C++ shape:**
```cpp
virtual void setPlaybackRange(double start, std::optional<double> end) = 0;
```

---

#### 4.2.2.11 `clearPlaybackRange()`

- Removes any active playback range previously set by `setPlaybackRange()`.
- After clearing, the service plays the full file from the current position.
- Safe to call when no range is active (no-op).
- MUST NOT throw.

---

### 4.2.3 Buffer Size

Implementations SHOULD calculate `framesPerBuffer` dynamically from the stream's
sample rate, targeting approximately 100ms of audio per buffer, rounded to the
nearest power of two.

Examples:
- 44100 Hz → 4096 frames (~93ms)
- 48000 Hz → 4096 frames (~85ms)
- 96000 Hz → 8192 frames (~85ms)

---

### 4.2.4 Reference Interface Shapes

**Swift example:**

```swift
public protocol PlaybackService: AnyObject {
    func load(url: URL) throws
    func play() throws
    func pause()
    func stop()
    func seek(to seconds: Double) throws

    func currentTime() -> Double
    func duration() -> Double
    var state: PlaybackState { get }

    func setVolume(_ volume: Float)
    func setPlaybackRange(start: Double, end: Double?) throws
    func clearPlaybackRange()
}
```

**C++ example:**

```cpp
class PlaybackService {
public:
    virtual ~PlaybackService() = default;

    virtual void load(const std::string& url) = 0;
    virtual void play() = 0;
    virtual void pause() = 0;
    virtual void stop() = 0;
    virtual void seek(double seconds) = 0;

    virtual double currentTime() const = 0;
    virtual double duration() const = 0;
    virtual PlaybackState state() const = 0;

    virtual void setVolume(float volume) = 0;
    virtual void setPlaybackRange(double start, std::optional<double> end) = 0;
    virtual void clearPlaybackRange() = 0;
};
```

---

## 4.3 CueSheetService

`CueSheetService` provides CUE sheet parsing as a composable service.
It wraps `CueSheetPort` and exposes the result to the application layer.

### 4.3.1 Goals

- Parse a `.cue` file and return a `CueSheet` with a list of `CueTrack` entries.
- Decouple the application from the raw `CueSheetPort` interface.
- Remain independently testable via a mock `CueSheetPort`.

### 4.3.2 Required API Surface

```text
1. parse(url: String) throws -> CueSheet
```

---

#### 4.3.2.1 `parse(url: String) throws -> CueSheet`

- Delegates to the injected `CueSheetPort.parse()`.
- Returns a fully populated `CueSheet` (see `05_models.md`).
- Throws `CoreError.notFound` if the `.cue` file does not exist.
- Throws `CoreError.ioError` for I/O errors.
- Throws `CoreError.decodeError` if the CUE sheet is malformed.
- Thread Safety: Safe to call concurrently.

---

### 4.3.3 Relationship to PlaybackService

`CueSheetService` and `PlaybackService` are independent services.
The application layer orchestrates them together:

```text
1. cueSheetService.parse(url) → CueSheet
2. User selects a CueTrack
3. playbackService.load(audioURL)
4. playbackService.setPlaybackRange(start: track.startTime, end: track.endTime)
5. playbackService.play()
```

`CueSheetService` has no dependency on `PlaybackService` and vice versa.

---

### 4.3.4 Reference Interface Shapes

**Swift example:**

```swift
public protocol CueSheetService: AnyObject {
    func parse(url: URL) throws -> CueSheet
}

final class DefaultCueSheetService: CueSheetService {
    private let port: CueSheetPort

    init(port: CueSheetPort) {
        self.port = port
    }

    func parse(url: URL) throws -> CueSheet {
        try port.parse(url: url.path)
    }
}
```

**C++ example:**

```cpp
class CueSheetService {
public:
    virtual ~CueSheetService() = default;
    virtual CueSheet parse(const std::string& url) = 0;
};

class DefaultCueSheetService : public CueSheetService {
    std::shared_ptr<CueSheetPort> port_;
public:
    explicit DefaultCueSheetService(std::shared_ptr<CueSheetPort> port)
        : port_(std::move(port)) {}

    CueSheet parse(const std::string& url) override {
        return port_->parse(url);
    }
};
```

---

## 4.4 Relationship to Ports

### PlaybackService Port Dependencies

| Port | Required | Purpose |
|------|----------|---------|
| `DecoderPort` | Required | Decodes audio files to PCM |
| `AudioOutputPort` | Required | Outputs PCM to audio hardware |
| `ClockPort` | Required | Provides timing for position tracking |
| `LoggerPort` | Required | Logs events for debugging |
| `FileAccessPort` | Optional | Direct file access if needed |

### CueSheetService Port Dependencies

| Port | Required | Purpose |
|------|----------|---------|
| `CueSheetPort` | Required | Parses `.cue` files |
| `LoggerPort` | Optional | Logs parsing events |

**Constraints:**
- Services MUST NOT reference AVFoundation, FFmpeg, PipeWire, TagLib, or any
  other platform-specific APIs directly.
- Platform-specific adapters MUST be provided via dependency injection.

---

## 4.5 Composition / Factory

**Swift example (updated):**

```swift
public enum CoreFactory {
    public static func makeDefaultPlaybackService() -> PlaybackService {
        let logger  = OSLogAdapter(subsystem: "HarmoniaCore", category: "Playback")
        let clock   = MonotonicClockAdapter()
        let decoder = AVAssetReaderDecoderAdapter(logger: logger)
        let audio   = AVAudioEngineOutputAdapter(logger: logger)
        return DefaultPlaybackService(
            decoder: decoder,
            audio:   audio,
            clock:   clock,
            logger:  logger
        )
    }

    public static func makeDefaultCueSheetService() -> CueSheetService {
        let port = NativeCueSheetAdapter()
        return DefaultCueSheetService(port: port)
    }
}
```

---

## 4.6 State Machine Diagram (PlaybackService)

```
                    ┌─────────┐
                    │ stopped │ (initial state)
                    └────┬────┘
                         │ load()
                         ▼
                    ┌─────────┐
              ┌────▶│ paused  │◀─────┐
              │     └────┬────┘      │
        pause()│         │ play()    │ pause()
              │         ▼           │
              │     ┌─────────┐     │
              └─────│ playing │─────┘
                    └────┬────┘
                         │ stop()
                         ▼
                    ┌─────────┐
                    │ stopped │
                    └─────────┘

                         ┌─────────┐
              (any) ────▶│ error   │
                         └────┬────┘
                              │ load() (recovery)
                              ▼
                         ┌─────────┐
                         │ paused  │
                         └─────────┘
```

**Notes:**
- `setPlaybackRange()` does not change state; it constrains the active playback window.
- `seek()` respects active playback range; seeks outside the range throw `invalidArgument`.
- `load()` always clears any active playback range.

---

## 4.7 Error Handling Strategy

1. **Graceful Degradation** — Errors SHOULD NOT crash the application.
2. **Clear Error Messages** — All `CoreError` values MUST include descriptive messages.
3. **Recovery Support** — After `error` state, calling `load()` with a valid file SHOULD allow recovery.
4. **Thread Safety** — Error handling MUST be thread-safe.

---

## 4.8 Performance Considerations

### Real-Time Audio Thread
- `AudioOutputPort.render()` is called from a real-time audio thread.
- Services MUST ensure the audio callback remains real-time safe (no allocations, no blocking).

### Decoder Threading
- Decoding SHOULD occur on a background thread.
- Services SHOULD maintain a decode-ahead buffer (2–5 seconds recommended).

### Position Tracking
- `currentTime()` SHOULD reflect actual render position, not decode position.

---

## 4.9 Future Services (Reserved)

The following services are candidates for future specifications.
Each MUST follow the same principles (cross-platform, ports-driven, testable):

### LibraryService
- Library scanning, indexing, and queries

### TagEditingService
- Cross-platform metadata editing using `TagWriterPort`

### QueueService
- Playlist / playback queue management with shuffle and repeat

### EqualizerService
- Real-time audio effects and preset management

---

## 4.10 Testing Requirements

### PlaybackService Tests
- All valid state transitions work correctly
- Idempotent operations remain idempotent
- `setPlaybackRange` constrains position correctly
- Automatic pause at range end, position resets to range start
- `seek()` outside active range throws `invalidArgument`
- `load()` clears active range

### CueSheetService Tests
- Correctly parses a well-formed CUE sheet
- Returns correct `startTime` / `endTime` for each track
- Last track has `endTime = nil`
- Throws `decodeError` for malformed input
- Throws `notFound` for missing file