# HarmoniaCore

[![Swift](https://img.shields.io/badge/Swift-5.9+-orange.svg)](https://swift.org)
[![Platform](https://img.shields.io/badge/Platform-macOS%2013+%20%7C%20iOS%2016+-lightgrey.svg)](https://developer.apple.com)
[![License](https://img.shields.io/badge/License-MIT-blue.svg)](LICENSE.md)

Platform-independent audio playback core with Ports & Adapters architecture.

## What is HarmoniaCore?

**HarmoniaCore** is an open-source, architecture-first audio playback core designed for
**behavior parity across platforms**.

It provides a deterministic, testable audio domain model implemented independently on
multiple platforms without sharing source code, using **hexagonal (ports & adapters)
architecture**:

1. **Specification repository** — Source of truth for ports, services, models, and parity rules
2. **Swift reference implementation** — AVFoundation-backed adapter set for Apple platforms
3. **C++20 Linux implementation** — Planned parity target

HarmoniaCore is consumed as a Swift Package by [HarmoniaPlayer](https://github.com/OneOfWolvesBilly/HarmoniaPlayer), which serves as the reference application and parity harness.

### Architecture

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
|        Ports Layer      |  <-- DecoderPort, AudioOutputPort, ClockPort, ...
+-----------+-------------+
            |
+-----------v-------------+
|      Adapters Layer     |  <-- AVFoundation / PipeWire
+-------------------------+
```

**Design principles:**

- Core behavior is platform-agnostic and reusable
- All logic is fully testable through mock ports
- New platforms can be added without modifying domain logic
- Independent implementations can be validated for behavior parity

See [Architecture Overview](docs/specs/01_architecture.md) for the full system design.

## Implementation Status

| Component | Swift (Apple) | C++20 (Linux) |
|-----------|---------------|---------------|
| Ports | ✅ Implemented | 🚧 Planned |
| Adapters | ✅ Implemented | 🚧 Planned |
| Services | ✅ Implemented | 🚧 Planned |
| Tests | ✅ Available | 🚧 Planned |

Component-level status only. Version-level progress is tracked in [Roadmap](#roadmap).

For the current inventory of ports, adapters, services, and models, see the corresponding specification documents under [Documentation](#documentation) below.

## Installation (Swift)

```swift
// Package.swift
dependencies: [
    .package(url: "https://github.com/OneOfWolvesBilly/HarmoniaCore-Swift.git", from: "0.1.0")
]
```

> `HarmoniaCore-Swift` is a standalone Swift Package extracted from `apple-swift/` via `git subtree split`. See [Related Projects](#related-projects) for the relationship between the two repositories.

**Requirements:**
- Xcode 15.3+ (Xcode 26 beta supports Swift 6 strict concurrency)
- Swift 5.9+ (Swift 5.10+ recommended for `nonisolated(unsafe)`)
- macOS 13+ or iOS 16+

See [Services Implementation](docs/impl/04_services_impl.md) for service construction examples and [Testing Guide](docs/testing.md) for writing tests against mock ports.

## Building and Testing

```bash
# Build
swift build

# Run all tests
swift test

# Run with code coverage
swift test --enable-code-coverage
```

For comprehensive testing guidance — mock usage patterns, CI configuration, and troubleshooting — see [Testing Guide](docs/testing.md).

## Repository Structure

```
HarmoniaCore/
├── apple-swift/              # Swift reference implementation
│   ├── Sources/HarmoniaCore/
│   │   ├── Ports/            # Port protocols (LoggerPort, ClockPort, ...)
│   │   ├── Adapters/         # Apple platform adapters (AVFoundation, OSLog, ...)
│   │   ├── Services/         # PlaybackService protocol and default implementation
│   │   └── Models/           # StreamInfo, TagBundle, CoreError
│   └── Tests/                # XCTest suite with mock ports
├── docs/
│   ├── specs/                # Platform-agnostic specifications
│   ├── impl/                 # Platform-specific implementation notes
│   ├── specs_to_impl_map.md  # Navigation guide
│   └── testing.md            # Comprehensive testing overview
├── README.md
└── LICENSE.md
```

For the full file listing of any sub-directory, see the corresponding specification document under [Documentation](#documentation) below.

## Documentation

### Specifications (Platform-Agnostic)
- **[Architecture Overview](docs/specs/01_architecture.md)** — System design, layer responsibilities, data flow
- **[Adapters Specification](docs/specs/02_adapters.md)** — Cross-platform adapter contract
- **[Apple Adapters](docs/specs/02_01_apple.adapters.md)** — AVFoundation adapter behavioral specs
- **[Linux Adapters](docs/specs/02_02_linux.adapters.md)** — C++20 / PipeWire adapter specs (planned)
- **[Ports Specification](docs/specs/03_ports.md)** — Port protocols and semantics
- **[Services Specification](docs/specs/04_services.md)** — PlaybackService and high-level service contracts
- **[Models Specification](docs/specs/05_models.md)** — StreamInfo, TagBundle, CoreError, and validation rules
- **[Test Strategy](docs/specs/06_test_strategy.md)** — Testing philosophy, categories, coverage goals
- **[API Parity](docs/specs/07_api-parity.md)** — Cross-platform behavior validation rules

### Implementation Guides
- **[Apple Adapters Implementation](docs/impl/02_01_apple.adapters_impl.md)** — Swift adapter code patterns
- **[Linux Adapters Implementation](docs/impl/02_02_linux.adapters_impl.md)** — C++20 adapter patterns (planned)
- **[Ports Implementation](docs/impl/03_ports_impl.md)** — Concrete Swift / C++ port shapes
- **[Services Implementation](docs/impl/04_services_impl.md)** — Service wiring and usage examples
- **[Models Implementation](docs/impl/05_models_impl.md)** — Model definitions and validation logic
- **[Swift Testing Implementation](docs/impl/06_01_testing_swift.md)** — XCTest patterns and mock usage
- **[C++20 Testing Implementation](docs/impl/06_02_testing_cpp.md)** — Google Test patterns (planned)

### Navigation
- **[Spec → Impl Mapping](docs/specs_to_impl_map.md)** — How specifications map to implementation notes
- **[Testing Guide](docs/testing.md)** — Comprehensive testing overview

## Roadmap

- **v0.1** — Swift reference supporting HarmoniaPlayer Free (In development)
- **v0.2** — Swift extensions for HarmoniaPlayer Pro (In development)
- **v0.3+** — Advanced audio features (Planned)
- **Linux C++20 parity** — Cross-platform implementation (Deferred)

## Contributing

Contributions welcome. Please:

1. Follow the hexagonal architecture (Ports for abstractions, Adapters for implementations)
2. Write tests for all new code against mock ports
3. Update the relevant specification and implementation documents
4. Use conventional commit messages

See the [specification documents](#specifications-platform-agnostic) for detailed design guidelines.

## Related Projects

- **[HarmoniaPlayer](https://github.com/OneOfWolvesBilly/HarmoniaPlayer)** — Reference SwiftUI music player built on HarmoniaCore
- **[HarmoniaCore-Swift](https://github.com/OneOfWolvesBilly/HarmoniaCore-Swift)** — Standalone Swift Package (subtree split from `apple-swift/`). Tagged releases define the version HarmoniaPlayer pins for deployment.

## License

MIT License — see [LICENSE.md](LICENSE.md) for details.

Copyright (c) 2025 Chih-hao (Billy) Chen

## Contact

- **GitHub**: [@OneOfWolvesBilly](https://github.com/OneOfWolvesBilly)
- **Project**: [HarmoniaCore](https://github.com/OneOfWolvesBilly/HarmoniaCore)