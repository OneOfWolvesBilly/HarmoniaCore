# 02 – Adapters Overview

Adapters are **platform-specific implementations** of the Ports defined in `03_ports.md`.

They:
- Live in per-platform source trees (`apple-swift/`, `linux-cpp/`).
- Import concrete frameworks (AVFoundation, PipeWire, TagLib, etc.).
- Must NOT leak platform types above the Port boundary.
- Can differ internally, but MUST honor the same observable behavior.

---

## 2.1 Mapping Table

| Port            | Apple Implementation(s)                          | Linux Implementation(s)                       |
|----------------|--------------------------------------------------|----------------------------------------------|
| AudioOutputPort| `AVAudioEngineOutputAdapter`                     | `PipeWireOutputAdapter`                      |
| DecoderPort    | `AVAssetReaderDecoderAdapter`, `FlacDecoderAdapter` (macOS Pro), `DsdDecoderAdapter` (macOS Pro) | `LibSndFileDecoderAdapter`, optional `FFmpegDecoderAdapter` |
| FileAccessPort | `SandboxFileAccessAdapter`                       | `PosixFileAccessAdapter`                     |
| TagReaderPort  | `AVMetadataTagReaderAdapter`                     | `TagLibReaderAdapter`                        |
| TagWriterPort  | `AVMutableTagWriterAdapter` (macOS only)         | `TagLibWriterAdapter`                        |
| ClockPort      | `MonotonicClockAdapter`                          | `SteadyClockAdapter`                         |
| LoggerPort     | `OSLogAdapter`, `NoopLogger`                     | `SpdlogAdapter`, `StdErrLogger`              |

---

## 2.2 Platform-specific Details

- Apple-specific details: see `docs/apple/apple.adapters.md`
- Linux-specific details: see `docs/linux/linux.adapters.md`

Each Adapter:

1. Implements exactly one Port interface (or composes several).
2. Handles platform errors and maps them to `CoreError` or equivalent.
3. Must be covered by tests verifying behavior against `06_api-parity.md`.

Adapters are owned by HarmoniaCore and are part of what you implement.
Underlying frameworks (AVFoundation, PipeWire, TagLib, etc.) are external dependencies.
