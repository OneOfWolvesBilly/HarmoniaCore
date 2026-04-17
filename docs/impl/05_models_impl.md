# Models Implementation

This document provides concrete Swift and C++ implementations of the data models.

**Spec Reference:** [`specs/05_models.md`](../specs/05_models.md)

---

## Swift: StreamInfo

```swift
public struct StreamInfo: Sendable, Equatable {
    public let duration: Double
    public let sampleRate: Double
    public let channels: Int
    public let bitDepth: Int
    
    public init(duration: Double, 
                sampleRate: Double, 
                channels: Int, 
                bitDepth: Int) {
        self.duration = duration
        self.sampleRate = sampleRate
        self.channels = channels
        self.bitDepth = bitDepth
    }
}
```

**Thread Safety:** Immutable struct, automatically `Sendable`.

---

## Swift: TagBundle

```swift
public struct TagBundle: Sendable, Equatable {

    /// Current schema version. HarmoniaPlayer compares this against its
    /// persisted `metadataVersion` to decide when to trigger a background
    /// metadata re-read for tracks saved by an older schema.
    ///
    /// History:
    /// - 0: legacy (no technical info fields)
    /// - 1: added duration, bitrate, sampleRate, channels, fileSize
    public static let currentSchemaVersion: Int = 1

    // MARK: - Tag Fields

    public var title: String?
    public var artist: String?
    public var album: String?
    public var albumArtist: String?
    public var composer: String?
    public var genre: String?
    public var year: Int?
    public var trackNumber: Int?
    public var trackTotal: Int?
    public var discNumber: Int?
    public var discTotal: Int?
    public var bpm: Int?
    public var replayGainTrack: Double?
    public var replayGainAlbum: Double?
    public var comment: String?
    public var artworkData: Data?

    // MARK: - Technical Info Fields
    //
    // These fields describe audio stream and file properties read alongside
    // tags from the same AVURLAsset. They are NOT tag metadata in the
    // ID3 / MP4 sense and are intentionally excluded from `isEmpty`.

    public var duration: TimeInterval?
    public var bitrate: Int?
    public var sampleRate: Double?
    public var channels: Int?
    public var fileSize: Int?

    public init() {}

    public init(
        title: String? = nil,
        artist: String? = nil,
        album: String? = nil,
        albumArtist: String? = nil,
        composer: String? = nil,
        genre: String? = nil,
        year: Int? = nil,
        trackNumber: Int? = nil,
        trackTotal: Int? = nil,
        discNumber: Int? = nil,
        discTotal: Int? = nil,
        bpm: Int? = nil,
        replayGainTrack: Double? = nil,
        replayGainAlbum: Double? = nil,
        comment: String? = nil,
        artworkData: Data? = nil,
        duration: TimeInterval? = nil,
        bitrate: Int? = nil,
        sampleRate: Double? = nil,
        channels: Int? = nil,
        fileSize: Int? = nil
    ) {
        self.title = title
        self.artist = artist
        self.album = album
        self.albumArtist = albumArtist
        self.composer = composer
        self.genre = genre
        self.year = year
        self.trackNumber = trackNumber
        self.trackTotal = trackTotal
        self.discNumber = discNumber
        self.discTotal = discTotal
        self.bpm = bpm
        self.replayGainTrack = replayGainTrack
        self.replayGainAlbum = replayGainAlbum
        self.comment = comment
        self.artworkData = artworkData
        self.duration = duration
        self.bitrate = bitrate
        self.sampleRate = sampleRate
        self.channels = channels
        self.fileSize = fileSize
    }
}

// Helper methods
extension TagBundle {
    /// Returns true if all tag fields are nil.
    ///
    /// Technical info fields (duration, bitrate, sampleRate, channels, fileSize)
    /// are excluded because they describe the audio stream, not user-facing tags.
    /// A file with no ID3 / MP4 tags but a valid duration is still considered
    /// "empty" from a tagging perspective.
    public var isEmpty: Bool {
        return title == nil &&
               artist == nil &&
               album == nil &&
               albumArtist == nil &&
               composer == nil &&
               genre == nil &&
               year == nil &&
               trackNumber == nil &&
               trackTotal == nil &&
               discNumber == nil &&
               discTotal == nil &&
               bpm == nil &&
               replayGainTrack == nil &&
               replayGainAlbum == nil &&
               comment == nil &&
               artworkData == nil
    }
}
```

---

## Swift: CoreError

```swift
public enum CoreError: Error, Sendable {
    case invalidArgument(String)
    case invalidState(String)
    case notFound(String)
    case ioError(underlying: Error)
    case decodeError(String)
    case unsupported(String)
}

// CustomStringConvertible for better debugging
extension CoreError: CustomStringConvertible {
    public var description: String {
        switch self {
        case .invalidArgument(let msg):
            return "Invalid argument: \(msg)"
        case .invalidState(let msg):
            return "Invalid state: \(msg)"
        case .notFound(let msg):
            return "Not found: \(msg)"
        case .ioError(let underlying):
            return "I/O error: \(underlying.localizedDescription)"
        case .decodeError(let msg):
            return "Decode error: \(msg)"
        case .unsupported(let msg):
            return "Unsupported: \(msg)"
        }
    }
}

// Equatable implementation
extension CoreError: Equatable {
    public static func == (lhs: CoreError, rhs: CoreError) -> Bool {
        switch (lhs, rhs) {
        case (.invalidArgument(let l), .invalidArgument(let r)):
            return l == r
        case (.invalidState(let l), .invalidState(let r)):
            return l == r
        case (.notFound(let l), .notFound(let r)):
            return l == r
        case (.ioError, .ioError):
            return true // Simplified: underlying errors not compared
        case (.decodeError(let l), .decodeError(let r)):
            return l == r
        case (.unsupported(let l), .unsupported(let r)):
            return l == r
        default:
            return false
        }
    }
}
```

---

## Swift: PlaybackState

```swift
public enum PlaybackState: Equatable {
    case stopped
    case playing
    case paused
    case buffering
    case error(CoreError)
}

// CustomStringConvertible for debugging
extension PlaybackState: CustomStringConvertible {
    public var description: String {
        switch self {
        case .stopped:
            return "stopped"
        case .playing:
            return "playing"
        case .paused:
            return "paused"
        case .buffering:
            return "buffering"
        case .error(let error):
            return "error(\(error))"
        }
    }
}
```

---

## C++20: StreamInfo

```cpp
struct StreamInfo {
    double duration;
    double sample_rate;
    int channels;
    int bit_depth;
    
    // Default comparison operator (C++20)
    bool operator==(const StreamInfo&) const = default;
};
```

---

## C++20: TagBundle

```cpp
#include <cstdint>
#include <optional>
#include <string>
#include <vector>

struct TagBundle {
    // Schema version for cross-session compatibility checks
    static constexpr int current_schema_version = 1;

    // Tag fields
    std::optional<std::string> title;
    std::optional<std::string> artist;
    std::optional<std::string> album;
    std::optional<std::string> album_artist;
    std::optional<std::string> composer;
    std::optional<std::string> genre;
    std::optional<int>         year;
    std::optional<int>         track_number;
    std::optional<int>         track_total;
    std::optional<int>         disc_number;
    std::optional<int>         disc_total;
    std::optional<int>         bpm;
    std::optional<double>      replay_gain_track;
    std::optional<double>      replay_gain_album;
    std::optional<std::string> comment;
    std::optional<std::vector<uint8_t>> artwork_data;

    // Technical info fields (excluded from is_empty)
    std::optional<double>      duration;      // seconds
    std::optional<int>         bitrate;       // kbps
    std::optional<double>      sample_rate;   // Hz
    std::optional<int>         channels;
    std::optional<int64_t>     file_size;     // bytes

    // Default comparison (C++20)
    bool operator==(const TagBundle&) const = default;

    // Returns true if all tag fields are nullopt.
    // Technical info fields are excluded — they describe the audio stream,
    // not user-facing tags.
    bool is_empty() const {
        return !title && !artist && !album && !album_artist &&
               !composer && !genre && !year &&
               !track_number && !track_total &&
               !disc_number && !disc_total &&
               !bpm && !replay_gain_track && !replay_gain_album &&
               !comment && !artwork_data;
    }
};
```

---

## C++20: CoreError

```cpp
#include 
#include 

enum class CoreErrorType {
    InvalidArgument,
    InvalidState,
    NotFound,
    IoError,
    DecodeError,
    Unsupported
};

class CoreError : public std::exception {
private:
    CoreErrorType type_;
    std::string message_;
    
public:
    CoreError(CoreErrorType type, std::string message)
        : type_(type), message_(std::move(message)) {}
    
    CoreErrorType type() const noexcept { return type_; }
    const char* what() const noexcept override { return message_.c_str(); }
    const std::string& message() const noexcept { return message_; }
    
    // Factory methods for cleaner usage
    static CoreError InvalidArgument(std::string msg) {
        return CoreError(CoreErrorType::InvalidArgument, std::move(msg));
    }
    
    static CoreError InvalidState(std::string msg) {
        return CoreError(CoreErrorType::InvalidState, std::move(msg));
    }
    
    static CoreError NotFound(std::string msg) {
        return CoreError(CoreErrorType::NotFound, std::move(msg));
    }
    
    static CoreError IoError(std::string msg) {
        return CoreError(CoreErrorType::IoError, std::move(msg));
    }
    
    static CoreError DecodeError(std::string msg) {
        return CoreError(CoreErrorType::DecodeError, std::move(msg));
    }
    
    static CoreError Unsupported(std::string msg) {
        return CoreError(CoreErrorType::Unsupported, std::move(msg));
    }
};

// Helper for string conversion
inline std::string to_string(CoreErrorType type) {
    switch (type) {
        case CoreErrorType::InvalidArgument: return "InvalidArgument";
        case CoreErrorType::InvalidState: return "InvalidState";
        case CoreErrorType::NotFound: return "NotFound";
        case CoreErrorType::IoError: return "IoError";
        case CoreErrorType::DecodeError: return "DecodeError";
        case CoreErrorType::Unsupported: return "Unsupported";
    }
    return "Unknown";
}
```

---

## C++20: PlaybackState

```cpp
enum class PlaybackState {
    Stopped,
    Playing,
    Paused,
    Buffering,
    Error
};

// For services that need to track error state
struct PlaybackStatus {
    PlaybackState state;
    std::optional error;
    
    bool isError() const {
        return state == PlaybackState::Error && error.has_value();
    }
};

// Helper for string conversion
inline std::string to_string(PlaybackState state) {
    switch (state) {
        case PlaybackState::Stopped: return "stopped";
        case PlaybackState::Playing: return "playing";
        case PlaybackState::Paused: return "paused";
        case PlaybackState::Buffering: return "buffering";
        case PlaybackState::Error: return "error";
    }
    return "unknown";
}
```

---

## Usage Examples

### Swift: Creating and Using Models

```swift
// Create StreamInfo
let info = StreamInfo(
    duration: 245.5,
    sampleRate: 44100.0,
    channels: 2,
    bitDepth: 16
)

// Create TagBundle
var tags = TagBundle()
tags.title = "Bohemian Rhapsody"
tags.artist = "Queen"
tags.year = 1975

// Throw errors
throw CoreError.notFound("File not found: \(url.path)")
throw CoreError.invalidArgument("Sample rate must be > 0")

// Handle errors
do {
    try decoder.open(url: url)
} catch let error as CoreError {
    switch error {
    case .notFound(let msg):
        print("File not found: \(msg)")
    case .unsupported(let msg):
        print("Format not supported: \(msg)")
    default:
        print("Error: \(error)")
    }
}
```

### C++20: Creating and Using Models

```cpp
// Create StreamInfo
StreamInfo info{
    .duration = 245.5,
    .sample_rate = 44100.0,
    .channels = 2,
    .bit_depth = 16
};

// Create TagBundle
TagBundle tags;
tags.title = "Bohemian Rhapsody";
tags.artist = "Queen";
tags.year = 1975;

// Throw errors
throw CoreError::NotFound("File not found: " + url);
throw CoreError::InvalidArgument("Sample rate must be > 0");

// Handle errors
try {
    decoder->open(url);
} catch (const CoreError& e) {
    switch (e.type()) {
        case CoreErrorType::NotFound:
            std::cerr << "File not found: " << e.message() << std::endl;
            break;
        case CoreErrorType::Unsupported:
            std::cerr << "Format not supported: " << e.message() << std::endl;
            break;
        default:
            std::cerr << "Error: " << e.what() << std::endl;
    }
}
```

---

## Validation

### Swift: Input Validation

```swift
extension StreamInfo {
    public func validate() throws {
        guard duration >= 0 else {
            throw CoreError.invalidArgument("Duration must be >= 0")
        }
        guard sampleRate > 0 else {
            throw CoreError.invalidArgument("Sample rate must be > 0")
        }
        guard channels >= 1 else {
            throw CoreError.invalidArgument("Channels must be >= 1")
        }
        guard bitDepth >= 8 else {
            throw CoreError.invalidArgument("Bit depth must be >= 8")
        }
    }
}

extension TagBundle {
    public func validate() throws {
        if let year = year, !(1000...9999).contains(year) {
            throw CoreError.invalidArgument("Year must be in range 1000-9999")
        }
        if let track = trackNumber, track < 1 {
            throw CoreError.invalidArgument("Track number must be >= 1")
        }
        if let disc = discNumber, disc < 1 {
            throw CoreError.invalidArgument("Disc number must be >= 1")
        }
    }
}
```

### C++20: Input Validation

```cpp
void validate(const StreamInfo& info) {
    if (info.duration < 0) {
        throw CoreError::InvalidArgument("Duration must be >= 0");
    }
    if (info.sample_rate <= 0) {
        throw CoreError::InvalidArgument("Sample rate must be > 0");
    }
    if (info.channels < 1) {
        throw CoreError::InvalidArgument("Channels must be >= 1");
    }
    if (info.bit_depth < 8) {
        throw CoreError::InvalidArgument("Bit depth must be >= 8");
    }
}

void validate(const TagBundle& tags) {
    if (tags.year && (*tags.year < 1000 || *tags.year > 9999)) {
        throw CoreError::InvalidArgument("Year must be in range 1000-9999");
    }
    if (tags.track_number && *tags.track_number < 1) {
        throw CoreError::InvalidArgument("Track number must be >= 1");
    }
    if (tags.disc_number && *tags.disc_number < 1) {
        throw CoreError::InvalidArgument("Disc number must be >= 1");
    }
}
```

---

## Cross-Platform Considerations

| Concern | Swift | C++20 |
|---------|-------|-------|
| **Optionals** | `String?` | `std::optional<std::string>` |
| **Binary data** | `Data` | `std::vector<uint8_t>` |
| **Immutability** | `let` for immutable | `const` for immutable |
| **Thread safety** | `Sendable` marker | Requires explicit synchronization if mutable |
| **Error comparison** | Pattern matching with `==` | `CoreErrorType` enum comparison |