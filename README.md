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
3. **C++20 Linux implementation** — Deferred parity target

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
|        Ports Layer      |  <-- DecoderPort, AudioOutputPort, MonotonicTimePort, ...
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
| Ports | ✅ Implemented | 🚧 Deferred |
| Adapters | ✅ Implemented | 🚧 Deferred |
| Services | ✅ Implemented | 🚧 Deferred |
| Tests | ✅ Available | 🚧 Deferred |

Component-level status only. Version-level progress is tracked in [Roadmap](#roadmap).

For the current inventory of ports, adapters, services, and models, see the corresponding specification documents under [Documentation](#documentation) below.

## Installation (Swift)

```swift
// Package.swift
dependencies: [
    .package(url: "https://github.com/OneOfWolvesBilly/HarmoniaCore-Swift.git", revision: "<tagged revision>")
]
```

> `HarmoniaCore-Swift` is a standalone Swift Package extracted from `apple-swift/` via `git subtree split`. Its releases are tagged `vX.Y.Z-swift`; pin the tagged revision rather than a `from:` range (see [Versioning Policy](#versioning-policy)). See [Related Projects](#related-projects) for the relationship between the two repositories.

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
│   │   ├── Ports/            # Port protocols (LoggerPort, MonotonicTimePort, ...)
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
- **[Linux Adapters](docs/specs/02_02_linux.adapters.md)** — C++20 / PipeWire adapter specs (deferred)
- **[Ports Specification](docs/specs/03_ports.md)** — Port protocols and semantics
- **[Services Specification](docs/specs/04_services.md)** — PlaybackService and high-level service contracts
- **[Models Specification](docs/specs/05_models.md)** — StreamInfo, TagBundle, CoreError, and validation rules
- **[Test Strategy](docs/specs/06_test_strategy.md)** — Testing philosophy, categories, coverage goals
- **[API Parity](docs/specs/07_api-parity.md)** — Cross-platform behavior validation rules

### Implementation Guides
- **[Apple Adapters Implementation](docs/impl/02_01_apple.adapters_impl.md)** — Swift adapter code patterns
- **[Linux Adapters Implementation](docs/impl/02_02_linux.adapters_impl.md)** — C++20 adapter patterns (deferred)
- **[Ports Implementation](docs/impl/03_ports_impl.md)** — Concrete Swift / C++ port shapes
- **[Services Implementation](docs/impl/04_services_impl.md)** — Service wiring and usage examples
- **[Models Implementation](docs/impl/05_models_impl.md)** — Model definitions and validation logic
- **[Swift Testing Implementation](docs/impl/06_01_testing_swift.md)** — XCTest patterns and mock usage
- **[C++20 Testing Implementation](docs/impl/06_02_testing_cpp.md)** — Google Test patterns (deferred)

### Navigation
- **[Spec → Impl Mapping](docs/specs_to_impl_map.md)** — How specifications map to implementation notes
- **[Testing Guide](docs/testing.md)** — Comprehensive testing overview

## Roadmap

HarmoniaCore versioning is independent of any consumer application's release cadence. See [Versioning Policy](#versioning-policy) below.

- **Old line — v0.1.x** — Swift 5.9 language mode, macOS 13 / iOS 16. PlaybackService, EQ, TagReader/Writer, and the full Apple adapter set. Receives bug fixes only; no API additions. Anchored by `v0.1.1` (monorepo) / `v0.1.1-swift` (HarmoniaCore-Swift), the last revision before the Swift 6 baseline.
- **New line — v0.2.x** — Swift 6 language mode and `Sendable` port contracts, macOS 15 / iOS 18. Baselined on `main` once the Swift 6 branch merges; `v0.2.0` is cut from that baseline.
- **Package split (planned)** — Tag reading/writing and the shared foundation (logging, time, device route) move to sibling packages. Each package versions independently; the split specifications fix each package's first version before any code moves.
- **v1.0** — Cut when the public API is judged ready for long-term backward-compatibility guarantees. Not before the synchronous port surface is redesigned.
- **Linux C++20 parity** — Cross-platform implementation (Deferred).

### Versioning Policy

Versions are per package and follow the package's **consumer-visible contract**: its public API declarations and the behaviour specified in `docs/specs`. The count of ports, adapters, services, or models is never the test.

- **patch** — the contract is unchanged: bug fixes within specified behaviour, documentation, internal refactoring, dependency bumps that do not alter the package's own API.
- **minor** — the contract grows: new public API, or new specified behaviour that existing callers can ignore.
- **major** — the contract shrinks or changes: removed or renamed public API, a new requirement on a port protocol, a changed platform or language-mode floor, a behavioural change to specified semantics.

**While the major version is 0**, the minor digit plays the major role: breaking changes bump minor, additive changes and fixes bump patch. Standard semantics apply from 1.0.

**Restructuring.** Moving a type between packages is not a feature. For the package it leaves, the move is breaking; for the package it enters, it is part of that package's first release or an addition, as that package's own specification states. A move that leaves consumer source unchanged (e.g. via a re-export) is not a bump driver for the consumer-facing package.

Consumer applications, their release cadence, and their feature tiers do not trigger version bumps. `TagBundle.currentSchemaVersion` is a model-level schema marker, not the package version.

**Tags.** The monorepo tags `vX.Y.Z`; HarmoniaCore-Swift, the subtree split of `apple-swift/`, tags the same release `vX.Y.Z-swift`. Pin by exact revision or tag: `from:` ranges do not resolve `-swift` tags, because the suffix is a pre-release identifier.

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