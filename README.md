# HarmoniaCore

[![Swift](https://img.shields.io/badge/Swift-5.9+-orange.svg)](https://swift.org)
[![Platform](https://img.shields.io/badge/Platform-macOS%2013+%20%7C%20iOS%2016+-lightgrey.svg)](https://developer.apple.com)
[![License](https://img.shields.io/badge/License-MIT-blue.svg)](LICENSE.md)

**HarmoniaCore** is an open-source, architecture-first audio playback core designed for
**behavior parity across platforms**.

It provides a deterministic, testable audio domain model implemented independently on
multiple platforms without sharing source code, using **hexagonal (ports & adapters)
architecture**.

---

## Overview

HarmoniaCore is an open-source, architecture-first audio playback core designed for
**behavior parity across platforms**.

It provides a deterministic, testable audio domain model implemented independently on
multiple platforms without sharing source code, using **hexagonal (ports & adapters)
architecture**.

* **Swift (Apple platforms)**: complete and serves as the **reference implementation**
* **C++20 (Linux)**: planned parity implementation

---

## Architecture

HarmoniaCore follows **Ports & Adapters (Hexagonal Architecture)** to guarantee platform
independence and long-term maintainability.

```
+-------------------------+
|     Application / UI    |
+-----------+-------------+
            |
+-----------v-------------+
|      Services Layer     |  <-- PlaybackService
+-----------+-------------+
            |
+-----------v-------------+
|        Ports Layer      |  <-- DecoderPort, AudioOutputPort, ClockPort
+-----------+-------------+
            |
+-----------v-------------+
|      Adapters Layer     |  <-- AVFoundation / PipeWire
+-------------------------+
```

**Design rationale**:

* Core behavior is platform-agnostic and reusable
* All logic is fully testable via mock ports
* New platforms can be added without modifying domain logic
* Independent implementations can be validated for behavior parity

---

## Quick Start (Swift)

### Installation

```swift
// Package.swift
dependencies: [
    .package(url: "https://github.com/OneOfWolvesBilly/HarmoniaCore-Swift.git", from: "0.1.0")
]
```

### Basic Usage

```swift
import HarmoniaCore

// Create service
let service = DefaultPlaybackService(
    decoder: AVAssetReaderDecoderAdapter(logger: OSLogAdapter()),
    audio: AVAudioEngineOutputAdapter(logger: OSLogAdapter()),
    clock: MonotonicClockAdapter(),
    logger: OSLogAdapter()
)

// Playback control
try service.load(url: audioFileURL)
try service.play()
service.pause()
try service.seek(to: 30.0)
service.stop()

// Query state
print("Duration: \(service.duration())s")
print("Position: \(service.currentTime())s")
print("State: \(service.state)")
```

---

## Implementation Status

| Component | Swift (Apple) | C++20 (Linux) |
|-----------|---------------|---------------|
| Ports (7) | ✅ Complete | 🚧 Planned v0.2 |
| Adapters | ✅ 8 adapters | 🚧 Planned v0.2 |
| Services | ✅ PlaybackService | 🚧 Planned v0.2 |
| Tests | ✅ Comprehensive | 🚧 Planned v0.2 |

### Implemented Components (Swift)

**Ports**: LoggerPort, ClockPort, FileAccessPort, DecoderPort, AudioOutputPort, TagReaderPort, TagWriterPort

**Apple Adapters**: OSLogAdapter, MonotonicClockAdapter, SandboxFileAccessAdapter, AVAssetReaderDecoderAdapter, AVAudioEngineOutputAdapter, AVMetadataTagReaderAdapter, AVMutableTagWriterAdapter, NoopLogger

**Services**: PlaybackService protocol, DefaultPlaybackService implementation

**Tests**: MockDecoderPort, MockAudioOutputPort, MockClockPort, DefaultPlaybackServiceTests

---

## Development

### Requirements
- Xcode 15.0+ (Swift 5.9+)
- macOS 13.0+ or iOS 16.0+

### Building

```bash
# Build
swift build

# Release build
swift build -c release
```

### Testing with Mocks

HarmoniaCore provides comprehensive mock implementations for all ports:

```swift
import XCTest
@testable import HarmoniaCore

let mockDecoder = MockDecoderPort(duration: 10.0, sampleRate: 44100.0)
let mockAudio = MockAudioOutputPort()
let mockClock = MockClockPort()

let service = DefaultPlaybackService(
    decoder: mockDecoder,
    audio: mockAudio,
    clock: mockClock,
    logger: NoopLogger()
)

try service.load(url: testURL)
XCTAssertTrue(mockDecoder.openCalled)
XCTAssertEqual(service.state, .paused)
```

---

## Validation & Testing

HarmoniaCore provides comprehensive testing infrastructure to ensure reliability and enable cross-platform behavior validation.

### Available Mocks

* **`MockDecoderPort`** - Simulates audio decoding with configurable behavior
* **`MockAudioOutputPort`** - Captures rendered audio for verification
* **`MockClockPort`** - Allows manual time control for deterministic tests
* **`MockFileAccessPort`** - Simulates file I/O operations
* **`MockTagReaderPort` / `MockTagWriterPort`** - Simulates metadata operations

### Running Tests (Swift)
```bash
# Run all tests
swift test

# Run with code coverage
swift test --enable-code-coverage

# Run specific test suite
swift test --filter DefaultPlaybackServiceTests
```

**See [Testing Guide](docs/testing.md) for comprehensive documentation:**
- Test structure and organization
- Writing tests with mocks
- Test patterns and best practices
- CI/CD configuration
- Troubleshooting

**Implementation Guides:**
- [Swift Testing Implementation](docs/impl/06_01_testing_swift.md) - XCTest patterns and examples
- [C++20 Testing Implementation](docs/impl/06_02_testing_cpp.md) - Google Test patterns (planned)

---

## Roadmap

### v0.1 - Swift Reference Implementation ✅ (Current)

* Core hexagonal architecture
* Complete port and adapter set
* PlaybackService API
* Comprehensive unit tests

### v0.2 - Linux C++20 Implementation (Q1-Q2 2026)

Focus areas:

* C++20 domain model mirroring Swift reference
* PipeWire / FFmpeg adapters
* Cross-platform behavior parity validation

### v0.3+ - Advanced Features (Future)

* Gapless playback
* Real-time equalizer
* Playlist service
* Hi-Res audio support (96kHz/192kHz/384kHz)

---

## Documentation

### Specifications (Platform-Agnostic)
- [Architecture Overview](docs/specs/01_architecture.md)
- [Adapters Specification](docs/specs/02_adapters.md)
- [Ports Specification](docs/specs/03_ports.md)
- [Services Specification](docs/specs/04_services.md)
- [Models Specification](docs/specs/05_models.md)

### Implementation Guides
- [Apple Adapters Implementation](docs/impl/02_01_apple_adapters_impl.md)
- [Ports Implementation](docs/impl/03_ports_impl.md)
- [Services Implementation](docs/impl/04_services_impl.md)
- [Models Implementation](docs/impl/05_models_impl.md)

### Testing Documentation
- [Testing Guide](docs/testing.md) - Comprehensive testing guide
- [Swift Testing Implementation](docs/impl/06_01_testing_swift.md) - XCTest patterns
- [C++20 Testing Implementation](docs/impl/06_02_testing_cpp.md) - Google Test patterns (planned)

---

## Contributing

Contributions welcome! Please:

1. Follow the hexagonal architecture (Ports for abstractions, Adapters for implementations)
2. Write tests for all new code
3. Update relevant documentation
4. Use conventional commit messages

See the specification documents for detailed design guidelines.

---

## License

MIT License - see [LICENSE.md](LICENSE.md) for details.

Copyright (c) 2025 Chih-hao (Billy) Chen

---

## Contact

- GitHub: [@OneOfWolvesBilly](https://github.com/OneOfWolvesBilly)
- Project: [HarmoniaCore](https://github.com/OneOfWolvesBilly/HarmoniaCore)

---

**Building a music player?** Check out [HarmoniaPlayer](https://github.com/OneOfWolvesBilly/HarmoniaPlayer) - a reference SwiftUI app using HarmoniaCore.
