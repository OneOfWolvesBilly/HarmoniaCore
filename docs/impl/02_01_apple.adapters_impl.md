# Apple Adapters Implementation (Swift)

This document describes Swift implementations of adapters for the Apple platform.

**Spec Reference:** [`specs/02_01_apple.adapters.md`](../specs/02_01_apple.adapters.md)

---

## OSLogAdapter : LoggerPort

Uses `os.Logger` for unified logging.

```swift
import OSLog

public final class OSLogAdapter: LoggerPort {
    private let logger: Logger
    
    public init(subsystem: String = "com.harmonia.core", 
                category: String = "default") {
        self.logger = Logger(subsystem: subsystem, category: category)
    }
    
    public func debug(_ msg: @autoclosure () -> String) {
        logger.debug("\(msg())")
    }
    
    public func info(_ msg: @autoclosure () -> String) {
        logger.info("\(msg())")
    }
    
    public func warn(_ msg: @autoclosure () -> String) {
        logger.warning("\(msg())")
    }
    
    public func error(_ msg: @autoclosure () -> String) {
        logger.error("\(msg())")
    }
}
```

**Thread Safety:** `os.Logger` is thread-safe by default.

---

## NoopLogger : LoggerPort

Zero-overhead no-op implementation for testing.

```swift
public final class NoopLogger: LoggerPort {
    public init() {}
    
    public func debug(_ msg: @autoclosure () -> String) {}
    public func info(_ msg: @autoclosure () -> String) {}
    public func warn(_ msg: @autoclosure () -> String) {}
    public func error(_ msg: @autoclosure () -> String) {}
}
```

---

## MonotonicClockAdapter : ClockPort

Uses `DispatchTime` for nanosecond precision monotonic time.

```swift
import Dispatch

public final class MonotonicClockAdapter: ClockPort {
    public init() {}
    
    public func now() -> UInt64 {
        return DispatchTime.now().uptimeNanoseconds
    }
}
```

**Precision:** Nanosecond resolution guaranteed.

---

## SandboxFileAccessAdapter : FileAccessPort

Wraps `FileHandle` with UUID-based tokens for sandbox-safe file access.

```swift
import Foundation

public final class SandboxFileAccessAdapter: FileAccessPort {
    private var handles: [FileHandleToken: FileHandle] = [:]
    private let lock = NSLock()
    
    public init() {}
    
    public func open(url: URL) throws -> FileHandleToken {
        do {
            let handle = try FileHandle(forReadingFrom: url)
            let token = FileHandleToken(id: UUID())
            lock.withLock {
                handles[token] = handle
            }
            return token
        } catch {
            throw mapToCorError(error, context: "Opening file: \(url.path)")
        }
    }
    
    public func read(_ token: FileHandleToken, 
                     into buffer: UnsafeMutableRawPointer, 
                     count: Int) throws -> Int {
        guard let handle = lock.withLock({ handles[token] }) else {
            throw CoreError.invalidState("File handle not found")
        }
        
        do {
            let data = try handle.read(upToCount: count) ?? Data()
            let bytesToCopy = min(data.count, count)
            data.copyBytes(to: buffer.assumingMemoryBound(to: UInt8.self), 
                          count: bytesToCopy)
            return bytesToCopy
        } catch {
            throw mapToCoreError(error, context: "Reading file")
        }
    }
    
    public func size(_ token: FileHandleToken) throws -> Int64 {
        guard let handle = lock.withLock({ handles[token] }) else {
            throw CoreError.invalidState("File handle not found")
        }
        
        do {
            let current = try handle.offset()
            try handle.seekToEnd()
            let size = try handle.offset()
            try handle.seek(toOffset: current)
            return Int64(size)
        } catch {
            throw mapToCoreError(error, context: "Getting file size")
        }
    }
    
    public func close(_ token: FileHandleToken) {
        lock.withLock {
            handles.removeValue(forKey: token)
        }
    }
    
    private func mapToCoreError(_ error: Error, context: String) -> CoreError {
        if let nsError = error as NSError? {
            switch nsError.code {
            case NSFileReadNoSuchFileError:
                return .notFound("\(context): File not found")
            case NSFileReadNoPermissionError:
                return .ioError(underlying: error)
            default:
                return .ioError(underlying: error)
            }
        }
        return .ioError(underlying: error)
    }
}
```

## AVAssetReaderDecoderAdapter : DecoderPort

Uses `AVAssetReader` to decode audio files to interleaved Float32 PCM.

**Note:** Full implementation requires significant AVFoundation integration code.  
See actual source code for complete implementation.

**Key Points:**
- Supports: MP3, AAC, ALAC, WAV, AIFF, CAF
- Output: Interleaved Float32 PCM in range [-1.0, 1.0]
- Thread-safe: Safe to use on background threads
- **Security-scoped resource handling**: Automatically manages file access permissions for sandboxed macOS apps

---

### Security-Scoped Resource Implementation

**Overview:**
On macOS with App Sandbox enabled, accessing files selected by the user through `.fileImporter` or `NSOpenPanel` requires explicit permission handling. The adapter automatically manages this through `startAccessingSecurityScopedResource()` and `stopAccessingSecurityScopedResource()`.

**Implementation Details:**

```swift
private struct State: Sendable {
    let url: URL
    let asset: AVAsset
    let track: AVAssetTrack
    let reader: AVAssetReader
    let output: AVAssetReaderTrackOutput
    let duration: Double
    let sampleRate: Double
    let channels: Int
    let bitDepth: Int
    let isAccessingSecurityScopedResource: Bool  // NEW: Track resource state
}
```

**In `open()` method:**

```swift
private func openAsync(asset: AVAsset, url: URL) async throws -> DecodeHandle {
    // Start accessing security-scoped resource (macOS sandbox)
    let didStartAccessing = url.startAccessingSecurityScopedResource()
    
    if !didStartAccessing {
        logger.warn("Failed to start accessing security-scoped resource: \(url.path)")
        // Continue anyway - may work in non-sandbox environments
    }
    
    do {
        // ... decode setup code ...
        
        let state = State(
            url: url,
            // ... other fields ...
            isAccessingSecurityScopedResource: didStartAccessing
        )
        
        return DecodeHandle(id: id)
        
    } catch {
        // IMPORTANT: Stop accessing on error
        if didStartAccessing {
            url.stopAccessingSecurityScopedResource()
        }
        throw error
    }
}
```

**In `close()` method:**

```swift
public func close(_ handle: DecodeHandle) {
    let removedState = lock.withLock {
        handles.removeValue(forKey: handle.id)
    }
    
    if let state = removedState {
        // Stop accessing security-scoped resource if we started it
        if state.isAccessingSecurityScopedResource {
            state.url.stopAccessingSecurityScopedResource()
            logger.debug("Stopped accessing security-scoped resource for [\(handle.id)]")
        }
        logger.debug("Decoder close [\(handle.id)]")
    }
}
```

**Why This Matters:**

1. **macOS Sandbox Requirement**: Without this, sandboxed apps get error `-12203` (permission denied) when trying to access user-selected files
2. **Cross-Environment Compatibility**: Works correctly on:
   - macOS with App Sandbox enabled (required)
   - macOS without App Sandbox (harmless no-op)
   - iOS (harmless no-op, as iOS handles this differently)
3. **Proper Resource Management**: Ensures resources are released in all code paths, including errors

**Testing Checklist:**

- ✅ User selects file via `.fileImporter` → File opens successfully
- ✅ Multiple files can be opened simultaneously
- ✅ `close()` properly releases file access
- ✅ Error during `open()` doesn't leak resource access
- ✅ Works on non-sandboxed macOS apps (backward compatible)

---

## AVAudioEngineOutputAdapter : AudioOutputPort

Uses `AVAudioEngine` and `AVAudioPlayerNode` for audio playback.

**Typical Usage:**
```swift
@MainActor
public final class AVAudioEngineOutputAdapter: AudioOutputPort {
    private let engine = AVAudioEngine()
    private let playerNode = AVAudioPlayerNode()
    
    public func configure(sampleRate: Double, channels: Int, framesPerBuffer: Int) {
        let format = AVAudioFormat(
            standardFormatWithSampleRate: sampleRate,
            channels: AVAudioChannelCount(channels)
        )!
        engine.attach(playerNode)
        engine.connect(playerNode, to: engine.mainMixerNode, format: format)
    }
    
    public func start() throws {
        try engine.start()
        playerNode.play()
    }
    
    public func stop() {
        playerNode.stop()
        engine.stop()
        engine.reset()
    }

    /// Clears in-flight buffers without stopping AVAudioEngine.
    ///
    /// Calls `playerNode.stop()` to cancel all scheduled buffers, signals
    /// the backpressure semaphore to unblock any thread waiting in `render()`,
    /// then restarts the player node for fresh output. The engine continues
    /// running throughout — only the player node is recycled.
    ///
    /// Must be called before submitting decoded audio from a new seek position.
    public func flush() {
        // Stop player node — cancels all scheduled buffers without callbacks.
        playerNode.stop()
        // Signal semaphore to unblock any thread blocked in render().
        for _ in 0..<maxInFlight { bufferSemaphore.signal() }
        // Reset semaphore and restart player node.
        bufferSemaphore = DispatchSemaphore(value: maxInFlight)
        if isStarted { playerNode.play() }
    }

    public func render(_ buffer: UnsafePointer<Float>, frameCount: Int) throws -> Int {
        // Uses DispatchSemaphore for double-buffer backpressure.
        // MUST NOT be called from a Swift async Task — use DispatchQueue instead.
        bufferSemaphore.wait()
        // Schedule PCM buffer to playerNode
        return frameCount
    }
}
```

**Backpressure mechanism:** `render()` uses a `DispatchSemaphore` with
`maxInFlight = 2` (double-buffering). The semaphore is signalled in the
AVAudioPlayerNode completion handler after each buffer finishes playing.
This ensures the decode loop does not outrun the audio engine.

**Why `DispatchQueue`, not `Task.detached`:** Calling `DispatchSemaphore.wait()`
from a Swift async context blocks a cooperative thread pool thread, causing
thread pool starvation and choppy audio. The playback loop MUST run on
`DispatchQueue.global(qos: .userInteractive)` instead.

**Thread Safety:** `configure()`, `start()`, `stop()`, and `flush()` are
protected by an internal `NSLock`. `render()` may be called from any thread.

---

## AVMetadataTagReaderAdapter : TagReaderPort

Reads metadata using `AVAsset` APIs.

**Why two metadata collections:**
`asset.load(.commonMetadata)` only returns cross-format common keys
(title, artist, album, artwork). Format-specific fields such as albumArtist,
genre, year, trackNumber, and discNumber are only accessible via
`asset.load(.metadata)`, which returns all available metadata items including
iTunes and ID3 format-specific identifiers.

**Async bridging pattern:**
`TagReaderPort.read(url:)` is synchronous. AVFoundation loading is async.
The pattern below wraps each `asset.load()` in a `Task { }` with a
`DispatchSemaphore` to bridge the two worlds without blocking the cooperative
thread pool.

```swift
import Foundation
@preconcurrency import AVFoundation

public final class AVMetadataTagReaderAdapter: TagReaderPort {

    public init() {}

    public func read(url: URL) throws -> TagBundle {
        let asset = AVURLAsset(url: url)

        // ── Step 1: Load commonMetadata (title, artist, album, artwork) ──

        var commonItems: [AVMetadataItem] = []
        var loadError: Error?

        let sema1 = DispatchSemaphore(value: 0)
        Task {
            do { commonItems = try await asset.load(.commonMetadata) }
            catch { loadError = error }
            sema1.signal()
        }
        sema1.wait()
        if let error = loadError { throw CoreError.ioError(underlying: error) }

        // ── Step 2: Load all metadata (albumArtist, genre, year, numbers) ─

        var allItems: [AVMetadataItem] = []
        let sema2 = DispatchSemaphore(value: 0)
        Task {
            do { allItems = try await asset.load(.metadata) }
            catch { /* non-fatal — extended fields will be nil */ }
            sema2.signal()
        }
        sema2.wait()

        // ── Step 3: Build TagBundle ───────────────────────────────────────

        var bundle = TagBundle()

        // Core fields from commonMetadata
        for item in commonItems {
            guard let key = item.commonKey else { continue }
            switch key {
            case .commonKeyTitle:
                bundle.title = loadString(from: item)
            case .commonKeyArtist:
                bundle.artist = loadString(from: item)
            case .commonKeyAlbumName:
                bundle.album = loadString(from: item)
            default:
                break
            }
        }

        // Artwork from commonMetadata
        if let artworkItem = AVMetadataItem.metadataItems(
            from: commonItems,
            filteredByIdentifier: .commonIdentifierArtwork
        ).first {
            bundle.artworkData = loadData(from: artworkItem)
        }

        // Extended fields from format-specific metadata
        bundle.albumArtist = readString(from: allItems, identifiers: [
            .iTunesMetadataAlbumArtist,
            .id3MetadataBand                    // TPE2
        ])

        bundle.genre = readString(from: allItems, identifiers: [
            .iTunesMetadataUserGenre,
            .iTunesMetadataPredefinedGenre,
            .id3MetadataContentType             // TCON
        ])

        bundle.year = readYear(from: allItems)

        bundle.trackNumber = readPartNumber(from: allItems, identifiers: [
            .iTunesMetadataTrackNumber,
            .id3MetadataTrackNumber             // TRCK
        ])

        bundle.discNumber = readPartNumber(from: allItems, identifiers: [
            .iTunesMetadataDiskNumber
            // ID3 TPOS has no named AVFoundation constant
        ])

        return bundle
    }

    // MARK: - Private helpers

    /// Synchronously loads the string value of a metadata item.
    private func loadString(from item: AVMetadataItem) -> String? {
        let sema = DispatchSemaphore(value: 0)
        var result: String?
        Task {
            result = try? await item.load(.stringValue)
            sema.signal()
        }
        sema.wait()
        return result
    }

    /// Synchronously loads the data value of a metadata item.
    private func loadData(from item: AVMetadataItem) -> Data? {
        let sema = DispatchSemaphore(value: 0)
        var result: Data?
        Task {
            result = try? await item.load(.dataValue)
            sema.signal()
        }
        sema.wait()
        return result
    }

    /// Returns the first non-empty string matching any of the given identifiers.
    private func readString(
        from items: [AVMetadataItem],
        identifiers: [AVMetadataIdentifier]
    ) -> String? {
        for id in identifiers {
            let matches = AVMetadataItem.metadataItems(from: items,
                                                       filteredByIdentifier: id)
            if let first = matches.first,
               let s = loadString(from: first),
               !s.isEmpty {
                return s
            }
        }
        return nil
    }

    /// Parses a year integer from release-date / year metadata identifiers.
    ///
    /// Handles ISO-8601 dates ("1977-05-25") and bare year strings ("1977").
    /// Takes the first 4 characters and converts to Int.
    private func readYear(from items: [AVMetadataItem]) -> Int? {
        let identifiers: [AVMetadataIdentifier] = [
            .iTunesMetadataReleaseDate,
            .id3MetadataRecordingTime,          // TDRC (ID3v2.4)
            .id3MetadataYear,                   // TYER (ID3v2.3)
            .commonIdentifierCreationDate
        ]
        for id in identifiers {
            let matches = AVMetadataItem.metadataItems(from: items,
                                                       filteredByIdentifier: id)
            if let first = matches.first,
               let s = loadString(from: first) {
                let yearStr = String(s.prefix(4))
                if let y = Int(yearStr), y > 0 { return y }
            }
        }
        return nil
    }

    /// Parses the first part of a "N/total" track or disc number string.
    ///
    /// Examples: "3/12" → 3, "3" → 3. Returns nil if value is 0 or negative.
    private func readPartNumber(
        from items: [AVMetadataItem],
        identifiers: [AVMetadataIdentifier]
    ) -> Int? {
        for id in identifiers {
            let matches = AVMetadataItem.metadataItems(from: items,
                                                       filteredByIdentifier: id)
            if let first = matches.first,
               let s = loadString(from: first) {
                let part = s.split(separator: "/").first.map(String.init) ?? s
                let trimmed = part.trimmingCharacters(in: .whitespaces)
                if let n = Int(trimmed), n > 0 { return n }
            }
        }
        return nil
    }
}
```

**Thread Safety:** All AVFoundation calls are bridged via `DispatchSemaphore`.
The adapter is safe to call from background threads.

---

## AVMutableTagWriterAdapter : TagWriterPort

Currently not functional on any Apple platform.

```swift
public final class AVMutableTagWriterAdapter: TagWriterPort {
    public init() {}
    
    public func write(url: URL, tags: TagBundle) throws {
        throw CoreError.unsupported(
            "Tag writing is not supported. " +
            "iOS: sandbox restrictions. macOS: deferred."
        )
    }
}
```

**Rationale:**
- iOS: Sandbox restrictions prevent file writes
- macOS: Support deferred to future version

---

## Error Mapping

All AVFoundation errors must be mapped to `CoreError`:

```swift
private func mapAVError(_ error: Error) -> CoreError {
    if let avError = error as? AVError {
        switch avError.code {
        case .fileNotFound:
            return .notFound("File not found: \(avError.localizedDescription)")
        case .unsupportedOutputSettings, .decoderNotFound:
            return .unsupported(avError.localizedDescription)
        case .decodeFailed:
            return .decodeError(avError.localizedDescription)
        default:
            return .ioError(underlying: error)
        }
    }
    return .ioError(underlying: error)
}
```