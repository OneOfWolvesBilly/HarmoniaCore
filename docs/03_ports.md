# 03 – Ports

Ports define the **platform-agnostic interfaces** that Services depend on.
They do not contain behavior themselves; they declare what capabilities are required.

**Rules**

1. Ports must not import or expose platform/framework types.
2. Ports must be small, stable, and behavior-focused.
3. Implementations (Adapters) may vary per platform, but must conform to these semantics.

---

## 3.1 AudioOutputPort

Low-level PCM sink.

### Responsibilities

- Configure output format.
- Start/stop device.
- Consume PCM frames (interleaved Float32).

### Semantics

- `configure` can be called once per playback session.
- `start` must be idempotent; `stop` must be safe to call multiple times.
- `render` returns the number of frames actually consumed (may be less on overload).

### Swift (interface)

```swift
package protocol AudioOutputPort {
    func configure(sampleRate: Double, channels: Int, framesPerBuffer: Int) throws
    func start() throws
    func stop() throws
    func render(_ interleavedFloat32: UnsafePointer<Float>, frameCount: Int) throws -> Int
}
```

### C++ (interface)

```cpp
class AudioOutputPort {
public:
    virtual ~AudioOutputPort() = default;
    virtual void configure(double sample_rate, int channels, int frames_per_buffer) = 0;
    virtual void start() = 0;
    virtual void stop() = 0;
    virtual int render(const float* interleaved, int frame_count) = 0;
};
```

---

## 3.2 DecoderPort

Abstract audio decoder.

### Responsibilities

- Open a source.
- Provide decoded PCM frames.
- Support seeking where possible.
- Report stream info.

### Swift

```swift
package struct DecodeHandle: Sendable { package let opaque: OpaquePointer }

package protocol DecoderPort {
    func open(url: URL) throws -> DecodeHandle
    func read(_ h: DecodeHandle,
              into pcmInterleaved: UnsafeMutablePointer<Float>,
              maxFrames: Int) throws -> Int
    func seek(_ h: DecodeHandle, toSeconds: Double) throws
    func info(_ h: DecodeHandle) throws -> StreamInfo
    func close(_ h: DecodeHandle)
}
```

### C++

```cpp
struct DecodeHandle {
    void* opaque;
};

class DecoderPort {
public:
    virtual ~DecoderPort() = default;
    virtual DecodeHandle open(const std::string& url) = 0;
    virtual int read(DecodeHandle, float* interleaved, int max_frames) = 0;
    virtual void seek(DecodeHandle, double seconds) = 0;
    virtual StreamInfo info(DecodeHandle) = 0;
    virtual void close(DecodeHandle) = 0;
};
```

---

## 3.3 FileAccessPort

Abstract file/sandbox access.

### Swift

```swift
package struct FileHandleToken: Sendable { package let opaque: OpaquePointer }

package protocol FileAccessPort {
    func open(url: URL) throws -> FileHandleToken
    func read(_ t: FileHandleToken,
              into buffer: UnsafeMutableRawPointer,
              count: Int) throws -> Int
    func size(_ t: FileHandleToken) throws -> Int64
    func close(_ t: FileHandleToken)
}
```

### C++

```cpp
struct FileHandleToken {
    int fd; // or void* for platform-specific handles
};

class FileAccessPort {
public:
    virtual ~FileAccessPort() = default;
    virtual FileHandleToken open(const std::string& url) = 0;
    virtual int read(FileHandleToken, void* buffer, int count) = 0;
    virtual long long size(FileHandleToken) = 0;
    virtual void close(FileHandleToken) = 0;
};
```

---

## 3.4 TagReaderPort & TagWriterPort

Metadata access.

```swift
package protocol TagReaderPort {
    func read(url: URL) throws -> TagBundle
}

package protocol TagWriterPort {
    func write(url: URL, tags: TagBundle) throws
}
```

```cpp
class TagReaderPort {
public:
    virtual ~TagReaderPort() = default;
    virtual TagBundle read(const std::string& url) = 0;
};

class TagWriterPort {
public:
    virtual ~TagWriterPort() = default;
    virtual void write(const std::string& url, const TagBundle& tags) = 0;
};
```

---

## 3.5 ClockPort

```swift
package protocol ClockPort {
    func now() -> UInt64 // monotonic ns
}
```

```cpp
class ClockPort {
public:
    virtual ~ClockPort() = default;
    virtual std::uint64_t now() = 0; // monotonic ns
};
```

---

## 3.6 LoggerPort

```swift
package protocol LoggerPort {
    func debug(_ msg: @autoclosure () -> String)
    func info(_ msg: @autoclosure () -> String)
    func warn(_ msg: @autoclosure () -> String)
    func error(_ msg: @autoclosure () -> String)
}
```

```cpp
class LoggerPort {
public:
    virtual ~LoggerPort() = default;
    virtual void debug(const std::string&) = 0;
    virtual void info(const std::string&) = 0;
    virtual void warn(const std::string&) = 0;
    virtual void error(const std::string&) = 0;
};
```

These Ports form the stable abstraction boundary between Services and platform implementations.
