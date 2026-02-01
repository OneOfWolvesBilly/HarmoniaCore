# HarmoniaCore Testing Guide

## Quick Start

### 1. Navigate to Project

```bash
cd apple-swift
```

### 2. Run Tests

```bash
# Run all tests
swift test

# Or use Xcode
open Package.swift
# Press Cmd + U to run tests
```

---

## Test Structure

```
apple-swift/
└── Tests/
    └── HarmoniaCoreTests/
        ├── TestSupport/
        │   ├── MockClockPort.swift
        │   ├── MockDecoderPort.swift
        │   ├── MockAudioOutputPort.swift
        │   ├── MockFileAccessPort.swift
        │   ├── MockTagReaderPort.swift
        │   └── MockTagWriterPort.swift
        ├── Models/
        │   ├── StreamInfoTests.swift
        │   ├── TagBundleTests.swift
        │   └── CoreErrorTests.swift
        └── Services/
            └── DefaultPlaybackServiceTests.swift
```

---

## Running Tests

### Command Line

```bash
# Run all tests
swift test

# Run specific test suite
swift test --filter DefaultPlaybackServiceTests

# Run with verbose output
swift test --verbose

# Generate code coverage
swift test --enable-code-coverage

# Parallel execution (faster)
swift test --parallel
```

### Xcode

1. Open `Package.swift` in Xcode
2. Press `Cmd + U` to run all tests
3. View results in Test Navigator (`Cmd + 6`)
4. View coverage in Report Navigator (`Cmd + 9` → Coverage tab)

---

## Test Coverage

### Current Implementation (v0.1)

| Component | Test File | Coverage | Tests | Status |
|-----------|-----------|----------|-------|--------|
| **Models** | | | | |
| StreamInfo | StreamInfoTests.swift | 100% | 4 | ✅ Complete |
| TagBundle | TagBundleTests.swift | 100% | 3 | ✅ Complete |
| CoreError | CoreErrorTests.swift | 100% | 14 | ✅ Complete |
| **Services** | | | | |
| PlaybackService | DefaultPlaybackServiceTests.swift | ~85% | 41 | ✅ Complete |
| **Test Infrastructure** | | | | |
| Mock Ports | TestSupport/*.swift | - | 6 mocks | ✅ Complete |

**Total: 62 tests, 50% overall coverage**

---

## Writing Tests

### Model Tests

Model tests verify data structure integrity and validation:

```swift
import XCTest
@testable import HarmoniaCore

final class StreamInfoTests: XCTestCase {
    func testInitialization() {
        let info = StreamInfo(
            duration: 245.5,
            sampleRate: 44100.0,
            channels: 2,
            bitDepth: 16
        )
        
        XCTAssertEqual(info.duration, 245.5)
        XCTAssertEqual(info.sampleRate, 44100.0)
        XCTAssertEqual(info.channels, 2)
        XCTAssertEqual(info.bitDepth, 16)
    }
    
    func testValidation() throws {
        let info = StreamInfo(
            duration: 245.5,
            sampleRate: 44100.0,
            channels: 2,
            bitDepth: 16
        )
        
        // Should not throw
        try info.validate()
    }
}
```

### Service Tests with Mocks

Service tests use mock ports to verify business logic:

```swift
import XCTest
@testable import HarmoniaCore

final class PlaybackServiceTests: XCTestCase {
    var service: PlaybackService!
    var mockDecoder: MockDecoderPort!
    var mockAudio: MockAudioOutputPort!
    var mockClock: MockClockPort!
    var mockLogger: NoopLogger!
    
    override func setUp() {
        super.setUp()
        
        mockDecoder = MockDecoderPort()
        mockAudio = MockAudioOutputPort()
        mockClock = MockClockPort()
        mockLogger = NoopLogger()
        
        service = DefaultPlaybackService(
            decoder: mockDecoder,
            audio: mockAudio,
            clock: mockClock,
            logger: mockLogger
        )
    }
    
    func testLoadCallsDecoder() throws {
        let testURL = URL(fileURLWithPath: "/test/audio.mp3")
        
        try service.load(url: testURL)
        
        XCTAssertTrue(mockDecoder.openCalled)
        XCTAssertEqual(mockDecoder.lastOpenedURL, testURL)
    }
}
```

---

## Mock Implementations

### Creating Custom Mocks

All mock ports follow the same pattern:

```swift
final class MockDecoderPort: DecoderPort {
    // Track calls
    var openCalled = false
    var readCalled = false
    var seekCalled = false
    var closeCalled = false
    
    // Track arguments
    var lastOpenedURL: URL?
    var lastSeekPosition: Double?
    
    // Return values
    var mockStreamInfo = StreamInfo(
        duration: 245.5,
        sampleRate: 44100.0,
        channels: 2,
        bitDepth: 16
    )
    var mockFramesRead = 4096
    
    func open(url: URL) throws -> DecodeHandle {
        openCalled = true
        lastOpenedURL = url
        return DecodeHandle(id: UUID())
    }
    
    func read(_ handle: DecodeHandle, 
              into buffer: UnsafeMutablePointer<Float>, 
              maxFrames: Int) throws -> Int {
        readCalled = true
        return mockFramesRead
    }
    
    // ... other methods
}
```

### Mock Best Practices

1. **Track all calls**: Use boolean flags (`openCalled`, etc.)
2. **Record arguments**: Store parameters for verification
3. **Provide defaults**: Return sensible mock data
4. **Allow customization**: Public properties for test configuration

---

## Test Patterns

### 1. State Transition Testing

```swift
func testStateTransitions() throws {
    // Initial state
    XCTAssertEqual(service.state, .stopped)
    
    // Load → Paused
    try service.load(url: testURL)
    XCTAssertEqual(service.state, .paused)
    
    // Play → Playing
    try service.play()
    XCTAssertEqual(service.state, .playing)
    
    // Pause → Paused
    service.pause()
    XCTAssertEqual(service.state, .paused)
    
    // Stop → Stopped
    service.stop()
    XCTAssertEqual(service.state, .stopped)
}
```

### 2. Error Handling

```swift
func testLoadThrowsOnInvalidFile() {
    mockDecoder.shouldThrowOnOpen = true
    
    XCTAssertThrowsError(
        try service.load(url: invalidURL)
    ) { error in
        guard case CoreError.notFound = error else {
            XCTFail("Expected .notFound error")
            return
        }
    }
}
```

### 3. Idempotency Testing

```swift
func testPlayIsIdempotent() throws {
    try service.load(url: testURL)
    
    try service.play()
    let callCount1 = mockAudio.startCallCount
    
    try service.play()
    let callCount2 = mockAudio.startCallCount
    
    XCTAssertEqual(callCount1, callCount2, 
                   "play() should be idempotent")
}
```

### 4. Concurrent Access

```swift
func testConcurrentAccess() throws {
    try service.load(url: testURL)
    
    let group = DispatchGroup()
    
    for _ in 0..<100 {
        group.enter()
        DispatchQueue.global().async {
            _ = self.service.currentTime()
            group.leave()
        }
    }
    
    group.wait()
    // Should not crash
}
```

---

## Continuous Integration

### GitHub Actions Configuration

```yaml
name: Tests

on:
  push:
    branches: [ main, develop ]
  pull_request:
    branches: [ main, develop ]

jobs:
  test:
    runs-on: macos-13
    
    steps:
    - uses: actions/checkout@v3
    
    - name: Run tests
      run: |
        cd apple-swift
        swift test --parallel
    
    - name: Generate coverage
      run: |
        cd apple-swift
        swift test --enable-code-coverage
```

---

## Troubleshooting

### Common Issues

**Issue**: `error: no such module 'HarmoniaCore'`  
**Solution**: Ensure you're in `apple-swift/` directory and run `swift build` first

**Issue**: `error: no such module 'XCTest'`  
**Solution**: 
- Use Xcode to run tests (`Cmd + U`)
- Or reset Command Line Tools: `sudo xcode-select --switch /Applications/Xcode.app/Contents/Developer`

**Issue**: Tests fail with "file not found"  
**Solution**: Check that test files are in correct directories (see structure above)

**Issue**: Mock not recording calls  
**Solution**: Ensure mock properties are public and initialized correctly

**Issue**: Tests timeout  
**Solution**: Check for infinite loops in playback implementation

---

## Next Steps

### Planned Test Additions

1. **Integration Tests** (v0.2)
   - Real file playback with test audio files
   - Cross-adapter compatibility

2. **Performance Tests** (v0.2)
   - Decode speed benchmarks
   - Memory usage profiling

3. **Platform-Specific Tests** (v0.2)
   - iOS sandbox behavior
   - macOS security-scoped bookmarks

4. **Cross-Platform Parity** (v0.2)
   - Swift vs C++20 behavior verification
   - Identical output validation

---

## Resources

- [Test Strategy](TEST_STRATEGY.md) - Overall testing approach
- [Swift Testing Documentation](https://developer.apple.com/documentation/xctest)
- [Port Specifications](../specs/03_ports.md)
- [Service Specifications](../specs/04_services.md)
