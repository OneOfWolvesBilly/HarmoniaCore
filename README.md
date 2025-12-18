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
┌─────────────────────────┐
│     Application / UI    │
└───────────┬─────────────┘
            │
┌───────────▼─────────────┐
│      Services Layer     │  ◄── PlaybackService
└───────────┬─────────────┘
            │
┌───────────▼─────────────┐
│        Ports Layer      │  ◄── DecoderPort, AudioOutputPort, ClockPort
└───────────┬─────────────┘
            │
┌───────────▼─────────────┐
│      Adapters Layer     │  ◄── AVFoundation / PipeWire
└─────────────────────────┘
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

```

---

## Implementation Status

| Component | Swift (Apple)   | C++20 (Linux)   |
| --------- | --------------- | --------------- |
| Ports     | ✅ Complete      | 🚧 Planned v0.2 |
| Adapters  | ✅ Complete      | 🚧 Planned v0.2 |
| Services  | ✅ Complete      | 🚧 Planned v0.2 |
| Tests     | ✅ Comprehensive | 🚧 Planned      |

### Implemented Components (Swift)

**Ports**: LoggerPort, ClockPort, FileAccessPort, DecoderPort, AudioOutputPort, TagReaderPort, TagWriterPort
**Adapters**: OSLogAdapter, MonotonicClockAdapter, SandboxFileAccessAdapter, AVAssetReaderDecoderAdapter, AVAudioEngineOutputAdapter, AVMetadataTagReaderAdapter, AVMutableTagWriterAdapter, NoopLogger
**Services**: PlaybackService, DefaultPlaybackService

---

## Validation & Testing

* Mock-based unit tests for all ports
* Deterministic playback assertions
* Executable test vectors for future cross-platform parity
* Continuous validation via embedding applications

---

## Roadmap

### v0.1 — Swift Reference Implementation ✅

* Core hexagonal architecture
* Complete port and adapter set
* PlaybackService API
* Comprehensive unit tests

### v0.2 — Linux C++20 Implementation

**Planned execution: Q1–Q2 2026**

Focus areas:

* C++20 domain model mirroring Swift reference
* PipeWire / FFmpeg adapters
* Cross-platform behavior parity validation

---

## Documentation

### Specifications (Platform-Agnostic)
- [Architecture Overview](docs/specs/01_architecture.md)
- [Adapters Specification](docs/specs/02_adapters.md)
- [Ports Specification](docs/specs/03_ports.md)
- [Services Specification](docs/specs/04_services.md)
- [Models Specification](docs/specs/05_models.md)

---

## License

MIT License - see [LICENSE.md](LICENSE.md) for details.

Copyright (c) 2025 Chih-hao (Billy) Chen

---

## Related Projects

* **HarmoniaPlayer** — reference SwiftUI application embedding HarmoniaCore
  [https://github.com/OneOfWolvesBilly/HarmoniaPlayer](https://github.com/OneOfWolvesBilly/HarmoniaPlayer)