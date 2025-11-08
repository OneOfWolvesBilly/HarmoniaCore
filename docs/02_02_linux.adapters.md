
# Linux Adapters (PipeWire / ALSA / libsndfile)

## Overview
Linux adapters implement the same Ports using native libraries:

| Port | Adapter | Library | Notes |
|------|----------|----------|-------|
| AudioOutputPort | PipeWireOutputAdapter | PipeWire | Real-time low-latency playback. Uses ring buffer to push PCM frames. |
| DecoderPort | LibSndFileDecoderAdapter | libsndfile / libFLAC | Supports WAV / AIFF / FLAC; can use mpg123 for MP3. |
| DecoderPort (optional) | FFmpegDecoderAdapter | FFmpeg | Optional; may require non-free license flag. |
| FileAccessPort | PosixFileAccessAdapter | POSIX syscalls | open, read, lseek, close. |
| TagReaderPort | TagLibReaderAdapter | TagLib | Reads ID3 / Vorbis / MP4 tags. |
| TagWriterPort | TagLibWriterAdapter | TagLib | Writes common tags. |
| ClockPort | SteadyClockAdapter | std::chrono | Provides monotonic time for latency metrics. |
| LoggerPort | SpdlogAdapter | spdlog / iostream | Structured logging or stdout fallback. |

## Platform Constraints
- Ensure PipeWire dev libraries installed (libpipewire-0.3-dev).
- If FFmpeg enabled, comply with distro packaging licenses.
- Prefer TagLib for tag consistency across formats.

## Example C++ Adapter Stub
```cpp
class PipeWireOutputAdapter : public AudioOutputPort {
public:
    void configure(double sampleRate, int channels, int framesPerBuffer) override;
    void start() override;
    void stop() override;
    int render(const float* interleaved, int frameCount) override;
};
```

## Notes
- Adapters must obey the same observable behavior spec in docs/api-parity.md.
- All timestamps derived from ClockPort for deterministic testing.
