# HarmoniaCore Architecture

## 1. System Overview
HarmoniaCore is a cross-platform audio framework providing identical behavior on Apple (Swift) and Linux (C++20) platforms.
It defines a shared architecture centered on **Ports**, **Services**, and **Adapters** — separating abstract logic from platform implementation.

```
Core layers:
+-----------------------------+
|  Applications / UI Clients  |
+-------------+---------------+
              |
              v
+-----------------------------+
|        Services Layer       |  → PlaybackService, LibraryService, TagService
+-------------+---------------+
              |
              v
+-----------------------------+
|          Ports Layer        |  → AudioOutputPort, DecoderPort, TagReaderPort, ...
+-------------+---------------+
              |
              v
+-----------------------------+
|        Adapters Layer       |  → AVFoundation / PipeWire / TagLib
+-------------+---------------+
              |
              v
+-----------------------------+
|   System APIs / Hardware    |
+-----------------------------+
```

---

## 2. Core Layer Responsibilities

| Layer | Responsibility | Examples |
|-------|----------------|-----------|
| Services | Implements playback / metadata / library logic using Ports | `PlaybackService.play()`, `TagService.read()` |
| Ports | Defines abstract interfaces for I/O and timing | `AudioOutputPort`, `DecoderPort`, `ClockPort` |
| Adapters | Implements Ports using platform frameworks | `AVAudioEngineOutputAdapter`, `PipeWireOutputAdapter` |
| Models | Simple data structures for cross-platform state | `Track`, `StreamInfo`, `TagBundle` |
| Utils | Shared helpers (math, time, string) | `TimeFormatter`, `ErrorMapper` |

---

## 3. Language and Naming Conventions

HarmoniaCore implements identical behavior specifications in two languages.

| Layer | Apple (Swift) | Linux (C++20) |
|-------|----------------|---------------|
| Architecture | Modular package (SPM) | CMake-based project |
| API surface | Protocol-oriented (`protocol`, `struct`) | Class-based (`class`, pure virtual) |
| Naming style | `PascalCase` types, `camelCase` methods | `PascalCase` types, `snake_case` members |
| Error handling | `throws` / `try` | `try` / `catch` or `std::error_code` |
| Memory management | ARC (automatic) | RAII (`std::unique_ptr`, `std::shared_ptr`) |
| Asynchrony | `async/await` | `std::thread`, `std::future` |
| Documentation | SwiftDoc (`///`) | Doxygen (`/** */`) |

All platform-independent specifications (in `03_ports.md` and `05_services.md`) are written in a neutral style.  
Each platform implements its own idiomatic form while preserving identical observable behavior,  
as defined in `06_api-parity.md`.

---

## 4. Data Flow Overview
A typical playback pipeline (simplified):

```
Track → DecoderPort → AudioOutputPort → System Audio Device
                     ↘ TagReaderPort → Metadata UI
```

- **PlaybackService** coordinates decoding and output.  
- **DecoderPort** produces PCM frames.  
- **AudioOutputPort** pushes frames to hardware.  
- **ClockPort** tracks real-time position.  
- **LoggerPort** records events.

---

## 5. Error and Thread Model
HarmoniaCore isolates errors and concurrency:

| Concern | Description |
|----------|-------------|
| Error propagation | All recoverable errors use `CoreError` enumeration (cross-platform). |
| Threading model | Services are thread-safe; decoding and rendering may run on worker threads. |
| Timing consistency | All timestamps use `ClockPort.now()` for determinism. |

---

## 6. Extensibility & Plugin Design
Future versions of HarmoniaCore will support third-party Adapters and Services via a **Plugin Registry**.  
Plugins must conform to `AudioEffectPlugin` or `DecoderPlugin` protocol, verified at runtime.

---

## 7. Testing & Validation Overview
Each commit must pass behavior-parity tests:

- Swift vs C++ frame-by-frame output comparison.  
- Metadata extraction consistency.  
- Cross-platform file load/unload behavior.  
- CI integration via GitHub Actions and Linux/macOS runners.

---

## 8. Documentation Map

### Core Specifications
| File | Purpose |
|------|----------|
| docs/01_architecture.md | System architecture overview (this file) |
| docs/02_adapters.md | Platform adapter implementations |
| docs/02_01_apple.adapters.md | Apple-specific adapter implementations |
| docs/02_02_linux.adapters.md | Linux-specific adapter implementations |
| docs/03_ports.md | Interface definitions |
| docs/04_services.md | Public service APIs |
| docs/05_models.md | Shared data models |

### Extended Specifications
| File | Purpose |
| docs/api-parity.md | Behavior contract between Swift and C++ implementations |
| docs/behavior-flow.md | Visual runtime & data flow |
| docs/testing-strategy.md | Parity verification & CI |
