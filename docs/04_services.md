# 04 – Services

Services define the **public API surface** of HarmoniaCore.
Applications (e.g. HarmoniaPlayer) interact only with Services and Models,
never directly with Ports or Adapters.

Key goals:

- Stable, documented API.
- Platform-agnostic semantics.
- Implemented separately in Swift and C++20, but with identical observable behavior.

---

## 4.1 PlaybackService

Responsible for controlling playback of a single active stream.

### Required behaviors

- Load a Track.
- Play / Pause / Stop.
- Seek within the current Track.
- Expose current `PlaybackState`.
- Respect `DecoderPort`, `AudioOutputPort`, `ClockPort`, `LoggerPort`.

### Swift (conceptual)

```swift
public protocol PlaybackService: Sendable {
    func load(track: Track) async throws
    func play() async throws
    func pause() async throws
    func stop() async throws
    func seek(to seconds: Double) async throws

    func currentState() -> PlaybackState
}
```

Implementation will inject:

```swift
public struct PlaybackServiceImpl: PlaybackService {
    let decoder: any DecoderPort
    let output: any AudioOutputPort
    let clock: any ClockPort
    let logger: any LoggerPort
    // internal state machine...
}
```

### C++ (conceptual)

```cpp
class PlaybackService {
public:
    virtual ~PlaybackService() = default;

    virtual void load(const Track& track) = 0;
    virtual void play() = 0;
    virtual void pause() = 0;
    virtual void stop() = 0;
    virtual void seek(double seconds) = 0;

    virtual PlaybackState current_state() const = 0;
};
```

`PlaybackService` must:

- Only depend on Ports.
- Be deterministic and parity-tested across Swift/C++ implementations.

---

## 4.2 TagService (MetadataService)

Wrapper around TagReaderPort / TagWriterPort.

### Swift

```swift
public protocol TagService {
    func readTags(for track: Track) async throws -> TagBundle
    func writeTags(for track: Track, tags: TagBundle) async throws
}
```

### C++

```cpp
class TagService {
public:
    virtual ~TagService() = default;
    virtual TagBundle read_tags(const Track& track) = 0;
    virtual void write_tags(const Track& track, const TagBundle& tags) = 0;
};
```

- On platforms where writing is unsupported, `writeTags` must fail with `operationNotSupported`.

---

## 4.3 LibraryService (optional / future)

Abstracts library / playlist / browsing.
Exact design can evolve, but examples include:

```swift
public protocol LibraryService {
    func resolve(by id: TrackID) async throws -> Track?
    func search(query: String) async throws -> [Track]
}
```

```cpp
class LibraryService {
public:
    virtual ~LibraryService() = default;
    virtual std::optional<Track> resolve(const TrackId& id) = 0;
    virtual std::vector<Track> search(const std::string& query) = 0;
};
```

Implementation may delegate to:

- OS media library APIs (Apple).
- Local database (Linux).
- Remote providers (future plugins).

---

## 4.4 Service Construction (Builder)

To hide Adapters and wiring, each platform provides a small builder/factory:

### Swift example

```swift
public enum CoreBuilder {
    public static func makePlaybackService() -> PlaybackService {
        let logger: any LoggerPort = OSLogAdapter()
        let output: any AudioOutputPort = AVAudioEngineOutputAdapter(logger: logger)
        let decoder: any DecoderPort = AVAssetReaderDecoderAdapter(logger: logger)
        let clock: any ClockPort = MonotonicClockAdapter()
        return PlaybackServiceImpl(decoder: decoder, output: output, clock: clock, logger: logger)
    }
}
```

### C++ example

```cpp
std::unique_ptr<PlaybackService> make_playback_service() {
    auto logger = std::make_shared<SpdlogAdapter>();
    auto output = std::make_unique<PipeWireOutputAdapter>(logger);
    auto decoder = std::make_unique<LibSndFileDecoderAdapter>(logger);
    auto clock = std::make_unique<SteadyClockAdapter>();
    return std::make_unique<PlaybackServiceImpl>(
        std::move(decoder), std::move(output), std::move(clock), logger);
}
```

Applications call builders; they never construct Adapters directly.

---

These Service definitions, combined with Ports and Models, form the contract
that both Swift and C++ implementations must follow.
Further Services (e.g., playlist, DSP, plugins) can be added later under the same principles.
