# 05. Models Specification

This document defines the platform-neutral data models used throughout HarmoniaCore.
These models cross Port boundaries and must be representable in both Swift and C++20.

All content in this specification is written in English and is language-neutral.
Code samples are illustrative only; actual implementations MUST preserve the semantics
defined here while using idiomatic constructs for their target language.

---

## Design Principles

1. **Platform Independence**  
   Models MUST NOT reference platform-specific types (e.g., `NSError`, `AVAudioFormat`).

2. **Immutability Preferred**  
   Models SHOULD be immutable where possible to ensure thread safety.

3. **Simple Data Structures**  
   Models are pure data - no business logic, no methods beyond basic accessors.

4. **Optional Fields**  
   Use optional/nullable types for fields that may not always be present.

---

## StreamInfo

Describes an audio stream's format and duration.

### Fields

| Field | Type | Description | Constraints |
|-------|------|-------------|-------------|
| `duration` | `Double` | Duration in seconds | `≥ 0.0`, may be `INFINITY` for streams |
| `sampleRate` | `Double` | Sample rate in Hz | Common: 44100.0, 48000.0, 88200.0, 96000.0, 192000.0 |
| `channels` | `Int` | Number of audio channels | `≥ 1`, typically 1 (mono) or 2 (stereo) |
| `bitDepth` | `Int` | Bit depth per sample | Common: 16, 24, 32 (for PCM formats) |

### Semantics

- **duration:**  
  Total playable duration. For finite files, this is exact. For streams, may be `INFINITY`.

- **sampleRate:**  
  Native sample rate of the audio. Decoders MUST output at this rate (no resampling at decode stage).

- **channels:**  
  Channel count. Decoders output interleaved PCM with this many channels.

- **bitDepth:**  
  Original bit depth. For compressed formats, this represents the target PCM bit depth.  
  Note: HarmoniaCore always uses Float32 internally, so this is informational.

### Illustrative Shapes

**Swift:**
```swift
public struct StreamInfo: Sendable, Equatable {
    public let duration: Double
    public let sampleRate: Double
    public let channels: Int
    public let bitDepth: Int

    public init(duration: Double, sampleRate: Double, channels: Int, bitDepth: Int) {
        self.duration = duration
        self.sampleRate = sampleRate
        self.channels = channels
        self.bitDepth = bitDepth
    }
}
```

**C++:**
```cpp
struct StreamInfo {
    double duration;
    double sample_rate;
    int channels;
    int bit_depth;

    bool operator==(const StreamInfo&) const = default;
};
```

### Parity Requirements

**Exact Match Required:**
- All fields MUST be identical across platforms for the same file
- `duration` MUST match within ±0.001 seconds (1ms tolerance for rounding)
- `sampleRate`, `channels`, `bitDepth` MUST match exactly

**Test Vector Example:**
```json
{
  "operations": [
    {"type": "load", "args": {"url": "{fixture_dir}/sine_440hz_1s.mp3"}}
  ],
  "assertions": [
    {"type": "duration_equals", "expected": 1.0, "tolerance_ms": 1},
    {"type": "sample_rate_equals", "expected": 44100.0},
    {"type": "channels_equals", "expected": 2}
  ]
}
```

---

## TagBundle

Contains metadata tags extracted from or to be written to an audio file.

### Fields

| Field | Type | Description | Example |
|-------|------|-------------|---------|
| `title` | `String?` | Track title | "Bohemian Rhapsody" |
| `artist` | `String?` | Primary artist | "Queen" |
| `album` | `String?` | Album name | "A Night at the Opera" |
| `albumArtist` | `String?` | Album artist (may differ from artist) | "Queen" |
| `genre` | `String?` | Musical genre | "Rock" |
| `year` | `Int?` | Release year | 1975 |
| `trackNumber` | `Int?` | Track number on album | 11 |
| `discNumber` | `Int?` | Disc number in multi-disc set | 1 |
| `artworkData` | `Data?` / `ByteArray?` | Embedded cover art (raw image bytes) | JPEG/PNG data |

### Semantics

- **Optional Fields:**  
  All fields are optional. `nil`/`null`/empty indicates the field is not present in the source file.

- **Nil vs Empty String Rule (CRITICAL FOR PARITY):**  
  Missing tags MUST be represented as `nil`/`null`, NOT empty string `""`.  
  Empty string `""` is a valid tag value (e.g., intentionally blank title).

- **String Encoding:**  
  All strings MUST be UTF-8 encoded.

- **artworkData:**  
  Raw image bytes (typically JPEG or PNG). Implementations SHOULD detect image format from magic bytes.  
  Size limit: Recommended ≤ 10 MB per file format specifications.

### Cross-Platform Consistency

- Tag names MUST map consistently across platforms:
  - ID3v2 `TIT2` → `title`
  - ID3v2 `TPE1` → `artist`
  - Vorbis `TITLE` → `title`
  - Vorbis `ARTIST` → `artist`
  - MP4 `©nam` → `title`
  - MP4 `©ART` → `artist`

- Missing tags MUST result in `nil`/`null` fields, NOT empty strings.

### Illustrative Shapes

**Swift:**
```swift
public struct TagBundle: Sendable, Equatable {
    public var title: String?
    public var artist: String?
    public var album: String?
    public var albumArtist: String?
    public var genre: String?
    public var year: Int?
    public var trackNumber: Int?
    public var discNumber: Int?
    public var artworkData: Data?

    public init() {}
}
```

**C++:**
```cpp
struct TagBundle {
    std::optional<std::string> title;
    std::optional<std::string> artist;
    std::optional<std::string> album;
    std::optional<std::string> album_artist;
    std::optional<std::string> genre;
    std::optional<int> year;
    std::optional<int> track_number;
    std::optional<int> disc_number;
    std::optional<std::vector<uint8_t>> artwork_data;

    bool operator==(const TagBundle&) const = default;
};
```

### Parity Requirements

**Exact Match Required:**
- Text fields (title, artist, album, etc.) MUST match exactly after normalization
- Numeric fields (year, trackNumber, discNumber) MUST match exactly
- `nil` vs non-nil status MUST match

**Excluded from Parity:**
- `artworkData`: Image format conversion allowed (JPEG ↔ PNG)

**Normalization Rules:**
- Trim leading/trailing whitespace: `" Title "` → `"Title"`
- Normalize Unicode: NFD ↔ NFC acceptable
- Case: Preserve as-is (do NOT normalize case)

**Test Vector Example:**
```json
{
  "operations": [
    {"type": "load", "args": {"url": "{fixture_dir}/tagged.mp3"}},
    {"type": "read_tags"}
  ],
  "assertions": [
    {"type": "tags_equal", "expected": {
      "title": "Test Track",
      "artist": "Test Artist",
      "album": "Test Album",
      "year": 2025
    }}
  ]
}
```

---

## CueTrack

Represents a single track entry parsed from a CUE sheet.

### Fields

| Field | Type | Description | Constraints |
|-------|------|-------------|-------------|
| `index` | `Int` | Track number as declared in the CUE sheet | `≥ 1` |
| `title` | `String?` | Track title from `TITLE` field | Optional |
| `performer` | `String?` | Track performer from `PERFORMER` field | Optional |
| `startTime` | `Double` | Track start position in seconds | `≥ 0.0` |
| `endTime` | `Double?` | Track end position in seconds | `> startTime` if present; `nil` means play to EOF |

### Semantics

- **index:**  
  The `TRACK nn AUDIO` number from the CUE sheet. Starts at 1. Sequential but may not be contiguous.

- **title / performer:**  
  Sourced from per-track `TITLE` / `PERFORMER` fields in the CUE sheet.  
  Fall back to sheet-level values if per-track values are absent.

- **startTime:**  
  Derived from the `INDEX 01` timestamp of this track, converted from CUE MM:SS:FF format
  (frames = 1/75 second) to seconds.  
  Formula: `seconds = MM * 60 + SS + FF / 75.0`

- **endTime:**  
  Derived from the `INDEX 01` timestamp of the next track, minus one frame (1/75 s).  
  For the last track, `endTime` is `nil` (play to EOF of the audio file).

### Illustrative Shapes

**Swift:**
```swift
public struct CueTrack: Sendable, Equatable {
    public let index: Int
    public let title: String?
    public let performer: String?
    public let startTime: Double
    public let endTime: Double?

    public init(index: Int, title: String?, performer: String?,
                startTime: Double, endTime: Double?) {
        self.index = index
        self.title = title
        self.performer = performer
        self.startTime = startTime
        self.endTime = endTime
    }
}
```

**C++:**
```cpp
struct CueTrack {
    int index;
    std::optional<std::string> title;
    std::optional<std::string> performer;
    double start_time;
    std::optional<double> end_time;

    bool operator==(const CueTrack&) const = default;
};
```

### Parity Requirements

**Exact Match Required:**
- `index`, `startTime`, `endTime` MUST be identical across platforms for the same `.cue` file
- `startTime` / `endTime` MUST be computed from MM:SS:FF using the formula above; no rounding

**Test Vector Example:**
```json
{
  "operations": [
    {"type": "parse_cue", "args": {"url": "{fixture_dir}/album.cue"}}
  ],
  "assertions": [
    {"type": "track_count_equals", "expected": 3},
    {"type": "track_equals", "track_index": 1, "expected": {
      "index": 1,
      "startTime": 0.0,
      "endTime": 185.906
    }},
    {"type": "track_equals", "track_index": 3, "expected": {
      "index": 3,
      "startTime": 420.0,
      "endTime": null
    }}
  ]
}
```

---

## CueSheet

Represents a fully parsed CUE sheet file.

### Fields

| Field | Type | Description |
|-------|------|-------------|
| `fileURL` | `String` | Absolute path to the audio file referenced by the CUE sheet |
| `title` | `String?` | Album title from sheet-level `TITLE` field |
| `performer` | `String?` | Album performer from sheet-level `PERFORMER` field |
| `tracks` | `[CueTrack]` | Ordered list of tracks; sorted ascending by `startTime` |

### Semantics

- **fileURL:**  
  Resolved from the `FILE` directive in the CUE sheet.  
  Implementations MUST resolve relative paths against the directory containing the `.cue` file.  
  Implementations MUST NOT verify that the audio file actually exists at parse time.

- **tracks:**  
  MUST be non-empty. A CUE sheet with zero tracks MUST cause `CueSheetPort.parse()` to throw
  `CoreError.decodeError`.  
  MUST be sorted by `startTime` ascending.

### Illustrative Shapes

**Swift:**
```swift
public struct CueSheet: Sendable, Equatable {
    public let fileURL: String
    public let title: String?
    public let performer: String?
    public let tracks: [CueTrack]

    public init(fileURL: String, title: String?, performer: String?,
                tracks: [CueTrack]) {
        self.fileURL = fileURL
        self.title = title
        self.performer = performer
        self.tracks = tracks
    }
}
```

**C++:**
```cpp
struct CueSheet {
    std::string file_url;
    std::optional<std::string> title;
    std::optional<std::string> performer;
    std::vector<CueTrack> tracks;

    bool operator==(const CueSheet&) const = default;
};
```

### Parity Requirements

**Exact Match Required:**
- `fileURL`, `tracks` (count and all fields) MUST be identical across platforms for the same file
- Track ordering MUST be ascending by `startTime`

---

## CoreError

Unified error enumeration for all recoverable errors in HarmoniaCore.

### Categories

| Category | Description | When to Use |
|----------|-------------|-------------|
| `invalidArgument(String)` | Invalid parameter value | Null/empty URL, negative seek position, invalid buffer size |
| `invalidState(String)` | Operation invalid in current state | Play with no track loaded, configure while playing |
| `notFound(String)` | Resource not found | File does not exist, invalid file path |
| `ioError(underlying?)` | I/O operation failed | Permission denied, disk read error, network failure |
| `decodeError(String)` | Audio decode failed | Corrupted file, unsupported codec variant, malformed CUE sheet |
| `unsupported(String)` | Feature/format not supported | FLAC on standard build, DSD without Pro license, write on iOS |

### Semantics

- **invalidArgument:**  
  User provided invalid input. Should include parameter name and reason.  
  Example: `"Invalid seek position: -5.0 (must be ≥ 0)"`

- **invalidState:**  
  Operation is valid but not allowed in current state.  
  Example: `"Cannot play: no track loaded. Call load() first."`

- **notFound:**  
  Requested resource does not exist.  
  Example: `"File not found: /path/to/track.mp3"`

- **ioError:**  
  Low-level I/O operation failed. MAY wrap underlying platform error for debugging.

- **decodeError:**  
  Audio decoding or CUE sheet parsing failed.  
  Example: `"Malformed CUE sheet: missing FILE directive"`

- **unsupported:**  
  Feature or format is not available on this platform/build.

### Error Recovery

- All `CoreError` values are **recoverable** - they should not crash the application.
- Services SHOULD transition to `error` state and allow recovery via `load()` or other operations.

### Illustrative Shapes

**Swift:**
```swift
public enum CoreError: Error, Sendable {
    case invalidArgument(String)
    case invalidState(String)
    case notFound(String)
    case ioError(underlying: Error)
    case decodeError(String)
    case unsupported(String)
}
```

**C++:**
```cpp
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

    CoreErrorType type() const { return type_; }
    const char* what() const noexcept override { return message_.c_str(); }

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
```

### Parity Requirements

**Exact Match Required:**
- Error **type** (category) MUST be identical for same inputs across platforms

**Flexible:**
- Error **message** content (can vary by platform)

**Test Vector Example:**
```json
{
  "operations": [
    {"type": "load", "args": {"url": "{fixture_dir}/sine_440hz_1s.mp3"}},
    {"type": "seek", "args": {"seconds": -1.0}}
  ],
  "assertions": [
    {
      "type": "error_thrown",
      "expected": {
        "type": "invalidArgument",
        "message_contains": "negative"
      }
    }
  ]
}
```

---

## Additional Types

### FileHandleToken (Opaque)

Used by `FileAccessPort` to track open file handles.

**Requirements:**
- MUST be hashable/comparable for use in collections.
- MUST be unique per open file operation.

**Swift:**
```swift
public struct FileHandleToken: Hashable, Sendable {
    let id: UUID
    public init(id: UUID) { self.id = id }
}
```

**C++:**
```cpp
struct FileHandleToken {
    std::string id;
    bool operator==(const FileHandleToken&) const = default;
};
```

### DecodeHandle (Opaque)

Used by `DecoderPort` to track open decode sessions.

**Requirements:**
- Similar to `FileHandleToken` - opaque, unique, hashable.
- Implementation may wrap native decoder handles (e.g., `AVAssetReader*`, `AVFormatContext*`).

---

## Thread Safety Considerations

### Immutable Models
- `StreamInfo`, `CueTrack`, `CueSheet` are immutable → naturally thread-safe.
- `TagBundle` fields are independent → safe to read concurrently if not mutated.

### Mutable Models
- `TagBundle` when used for writing → caller must synchronize.

### Error Handling
- `CoreError` values are immutable → safe to share across threads.

---

## Validation Rules

### StreamInfo Validation
```text
duration ≥ 0.0
sampleRate > 0.0 (typically 8000.0 .. 384000.0)
channels ≥ 1 (typically 1 or 2, max 8)
bitDepth ≥ 8 (typically 16, 24, 32)
```

### TagBundle Validation
```text
year: if present, 1000 ≤ year ≤ 9999
trackNumber: if present, trackNumber ≥ 1
discNumber: if present, discNumber ≥ 1
artworkData: if present, size ≤ 10 MB (recommended)
```

### CueTrack Validation
```text
index ≥ 1
startTime ≥ 0.0
endTime > startTime (if present)
```

### CueSheet Validation
```text
tracks.count ≥ 1
tracks sorted ascending by startTime
fileURL must be non-empty
```

---

## Future Extensions (Reserved)

The following models may be added in future versions:

### PlaybackOptions
- Repeat mode (none, one, all)
- Shuffle mode
- Crossfade settings
- Gapless playback settings

### EqualizerBand
- Frequency, gain, Q-factor
- For parametric EQ support

### PlaylistEntry
- Track reference
- Position in queue
- User metadata (play count, rating, etc.)

Any future model MUST:
1. Be platform-neutral.
2. Be representable in both Swift and C++20.
3. Have clear semantics documented.
4. Pass cross-platform serialization tests if persisted.