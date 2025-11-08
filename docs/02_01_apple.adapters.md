
# Apple Adapters (AVFoundation / macOS + iOS)

## Overview
Apple adapters implement the common Ports using system frameworks:

| Port | Adapter | Framework | Platforms | Notes |
|------|----------|------------|------------|-------|
| AudioOutputPort | AVAudioEngineOutputAdapter | AVFoundation / AVFAudio | iOS / macOS | Uses AVAudioEngine + AVAudioPlayerNode. Handles sample rate via AVAudioFormat. |
| DecoderPort | AVAssetReaderDecoderAdapter | AVFoundation | iOS / macOS | Reads MP3/AAC/ALAC/WAV/AIFF/CAF. |
| DecoderPort (Pro) | FlacDecoderAdapter | Embedded C (dr_flac / libFLAC) | macOS Pro | Converts FLAC → Float32 PCM. |
| DecoderPort (Pro) | DsdDecoderAdapter | Embedded C (dsd2pcm) | macOS Pro | DSF / DFF → PCM. |
| FileAccessPort | SandboxFileAccessAdapter | Foundation / Security | iOS / macOS | Manages sandbox-scoped URLs. |
| TagReaderPort | AVMetadataTagReaderAdapter | AVFoundation | iOS / macOS | Reads ID3 / MP4 metadata. |
| TagWriterPort | AVMutableTagWriterAdapter | AVFoundation | macOS only | iOS: throws .operationNotSupported. |
| ClockPort | MonotonicClockAdapter | Dispatch / mach | iOS / macOS | Uses DispatchTime.now().uptimeNanoseconds. |
| LoggerPort | OSLogAdapter | os.log | iOS / macOS | Uses unified logging; fallback to no-op. |

## Platform Constraints
- iOS sandbox forbids arbitrary file writes. TagWriterPort must throw .operationNotSupported.
- macOS Pro builds may statically link libFLAC and dsd2pcm; ensure license headers preserved.
- AVFoundation automatically converts compressed → PCM for playback; ensure double decode avoided.

## Example Instantiation
```swift
let logger = OSLogAdapter()
let audio: AudioOutputPort = AVAudioEngineOutputAdapter(logger: logger)
let decoder: DecoderPort = AVAssetReaderDecoderAdapter(logger: logger)
let svc = PlaybackService(audio: audio, decoder: decoder, clock: MonotonicClockAdapter(), logger: logger)
```
