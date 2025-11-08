# 05 – Models

HarmoniaCore models are **platform-agnostic data structures** shared by all implementations.
They are the only types exposed by public Services APIs.

## Design Principles

- No platform-specific types (no AVFoundation, no PipeWire types here).
- Serializable / comparable where practical.
- Simple enough to be mirrored in both Swift and C++20.

---

## 5.1 Track Identity & Location

Represents a playable audio resource.

### Swift (conceptual)

```swift
public struct TrackID: Hashable, Sendable {
    public let rawValue: String
}

public enum TrackLocation: Sendable {
    case fileURL(URL)
    case libraryID(String)      // system / player library reference
    case external(String)       // streaming or plugin-specific locator
}

public struct Track: Sendable {
    public let id: TrackID
    public let location: TrackLocation
    public let title: String?
    public let artist: String?
    public let album: String?
    public let durationHint: Double? // seconds, optional
}
```

### C++ (conceptual)

```cpp
struct TrackId {
    std::string value;
};

enum class TrackLocationKind { FileUrl, LibraryId, External };

struct TrackLocation {
    TrackLocationKind kind;
    std::string value;
};

struct Track {
    TrackId id;
    TrackLocation location;
    std::string title;
    std::string artist;
    std::string album;
    std::optional<double> duration_hint; // seconds
};
```

---

## 5.2 StreamInfo

Technical properties reported by DecoderPort.

```swift
public struct StreamInfo: Sendable {
    public let duration: Double      // seconds
    public let sampleRate: Double
    public let channels: Int
    public let bitDepth: Int
}
```

```cpp
struct StreamInfo {
    double duration;     // seconds
    double sample_rate;
    int channels;
    int bit_depth;
};
```

---

## 5.3 TagBundle

Metadata read/written via TagReaderPort / TagWriterPort.

```swift
public struct TagBundle: Sendable {
    public var title: String?
    public var artist: String?
    public var album: String?
    public var artworkData: Data? // small preview blob
}
```

```cpp
struct TagBundle {
    std::string title;
    std::string artist;
    std::string album;
    std::vector<std::uint8_t> artwork_data; // optional, may be empty
};
```

---

## 5.4 PlaybackState

Observable state published by PlaybackService.

```swift
public enum PlaybackStatus {
    case idle
    case loading
    case playing
    case paused
    case stopped
    case error(CoreError)
}

public struct PlaybackState: Sendable {
    public let status: PlaybackStatus
    public let trackID: TrackID?
    public let position: Double      // seconds
    public let bufferedUntil: Double // seconds
}
```

```cpp
enum class PlaybackStatus {
    Idle,
    Loading,
    Playing,
    Paused,
    Stopped,
    Error
};

struct PlaybackState {
    PlaybackStatus status;
    TrackId track_id;
    double position;       // seconds
    double buffered_until; // seconds
    // error details resolved via CoreError mapping if needed
};
```

These models define the common language between Services and their clients,
and are shared across all platforms.
