# Swift Testing Implementation Guide

This document provides Swift-specific testing implementation details for HarmoniaCore on Apple platforms (macOS, iOS).

**See also**: [Test Strategy](../test_strategy.md) for platform-agnostic testing philosophy.

---

## Testing Framework

### XCTest
HarmoniaCore uses Apple's native **XCTest** framework:

- Built into Xcode
- Native Swift support
- Excellent IDE integration
- No external dependencies

**Alternative considered**: Swift Testing (iOS 16+, macOS 13+)  
**Reason for XCTest**: Wider platform support, more mature tooling.

---

## Project Structure

```
apple-swift/
├── Package.swift                 # SPM package definition
├── Sources/
│   └── HarmoniaCore/            # Production code
│       ├── Models/
│       ├── Ports/
│       ├── Adapters/
│       └── Services/
└── Tests/
    └── HarmoniaCoreTests/       # Test code
        ├── TestSupport/         # Mock implementations
        │   ├── MockClockPort.swift
        │   ├── MockDecoderPort.swift
        │   ├── MockAudioOutputPort.swift
        │   ├── MockFileAccessPort.swift
        │   ├── MockTagReaderPort.swift
        │   └── MockTagWriterPort.swift
        ├── Models/              # Model tests
        │   ├── StreamInfoTests.swift
        │   ├── TagBundleTests.swift
        │   └── CoreErrorTests.swift
        └── Services/            # Service tests
            └── DefaultPlaybackServiceTests.swift
```

---

## Mock Implementation Pattern

All mock ports follow a consistent pattern for testability:

### Mock Template

```swift
import Foundation
@testable import HarmoniaCore

final class Mock<Port>Port: <Port>Port {
    // MARK: - Call Tracking
    
    /// Tracks whether method was called
    var <method>Called = false
    var <method>CallCount = 0
    
    // MARK: - Argument Capture
    
    /// Captures last arguments passed to method
    var last<Parameter>: <Type>?
    
    // MARK: - Configurable Returns
    
    /// Configurable return value for testing
    var mock<Return>: <Type> = <default>
    
    // MARK: - Error Injection
    
    /// Set to true to throw error
    var shouldThrowOn<Method> = false
    var errorToThrow: Error = CoreError.ioError(underlying: NSError(...))
    
    // MARK: - Port Implementation
    
    func <method>(<parameters>) throws -> <Return> {
        <method>Called = true
        <method>CallCount += 1
        last<Parameter> = <parameter>
        
        if shouldThrowOn<Method> {
            throw errorToThrow
        }
        
        return mock<Return>
    }
}
```

### Example: MockDecoderPort

```swift
import Foundation
@testable import HarmoniaCore

final class MockDecoderPort: DecoderPort {
    // MARK: - Call Tracking
    
    var openCalled = false
    var readCalled = false
    var seekCalled = false
    var closeCalled = false
    
    // MARK: - Argument Capture
    
    var lastOpenedURL: URL?
    var lastSeekPosition: Double?
    
    // MARK: - Configurable Returns
    
    var mockStreamInfo = StreamInfo(
        duration: 180.0,
        sampleRate: 44100.0,
        channels: 2,
        bitDepth: 16
    )
    
    var mockDecodeHandle = DecodeHandle(id: UUID())
    var mockFramesRead = 4096
    
    // MARK: - Error Injection
    
    var shouldThrowOnOpen = false
    var shouldThrowOnRead = false
    var shouldThrowOnSeek = false
    var errorToThrow: Error = CoreError.notFound("Mock error")
    
    // MARK: - DecoderPort Implementation
    
    func open(url: URL) throws -> DecodeHandle {
        openCalled = true
        lastOpenedURL = url
        
        if shouldThrowOnOpen {
            throw errorToThrow
        }
        
        return mockDecodeHandle
    }
    
    func read(
        _ handle: DecodeHandle,
        into buffer: UnsafeMutablePointer<Float>,
        maxFrames: Int
    ) throws -> Int {
        readCalled = true
        
        if shouldThrowOnRead {
            throw errorToThrow
        }
        
        // Fill buffer with silence for testing
        for i in 0..<(maxFrames * 2) {
            buffer[i] = 0.0
        }
        
        return mockFramesRead
    }
    
    func seek(_ handle: DecodeHandle, toSeconds seconds: Double) throws {
        seekCalled = true
        lastSeekPosition = seconds
        
        if shouldThrowOnSeek {
            throw errorToThrow
        }
    }
    
    func info(_ handle: DecodeHandle) throws -> StreamInfo {
        return mockStreamInfo
    }
    
    func close(_ handle: DecodeHandle) {
        closeCalled = true
    }
}
```

---

## Test Naming Convention

Follow this pattern for clear, self-documenting tests:

```
test<Component><Scenario>_<ExpectedResult>()
```

### Examples

**Positive Tests**:
```swift
func testLoad_ValidFile_Success()
func testPlay_AfterLoad_TransitionsToPlaying()
func testSeek_ToValidPosition_UpdatesCurrentTime()
```

**Negative Tests**:
```swift
func testLoad_MissingFile_ThrowsNotFound()
func testPlay_WithoutLoad_ThrowsInvalidState()
func testSeek_BeyondDuration_ThrowsInvalidArgument()
```

**Boundary Tests**:
```swift
func testSeek_ToZero_Success()
func testSeek_ToExactDuration_Success()
func testLoad_EmptyFile_ThrowsDecodeError()
```

**Idempotency Tests**:
```swift
func testPlay_WhenAlreadyPlaying_NoError()
func testPause_WhenAlreadyPaused_NoError()
func testStop_WhenAlreadyStopped_NoError()
```

---

## Test Example: Service Testing

### Basic State Transition Test

```swift
import XCTest
@testable import HarmoniaCore

final class DefaultPlaybackServiceTests: XCTestCase {
    var service: DefaultPlaybackService!
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
    
    override func tearDown() {
        service = nil
        mockDecoder = nil
        mockAudio = nil
        mockClock = nil
        mockLogger = nil
        
        super.tearDown()
    }
    
    func testStateTransitions_LoadPlayPauseStop() throws {
        // Initial state
        XCTAssertEqual(service.state, .stopped)
        
        // Load file
        let testURL = URL(fileURLWithPath: "/test/audio.mp3")
        try service.load(url: testURL)
        XCTAssertEqual(service.state, .paused)
        XCTAssertTrue(mockDecoder.openCalled)
        
        // Play
        try service.play()
        XCTAssertEqual(service.state, .playing)
        XCTAssertTrue(mockAudio.startCalled)
        
        // Pause
        service.pause()
        XCTAssertEqual(service.state, .paused)
        XCTAssertTrue(mockAudio.stopCalled)
        
        // Stop
        service.stop()
        XCTAssertEqual(service.state, .stopped)
        XCTAssertTrue(mockDecoder.closeCalled)
    }
    
    func testPlay_WithoutLoad_ThrowsInvalidState() {
        XCTAssertThrowsError(try service.play()) { error in
            guard case CoreError.invalidState(let msg) = error else {
                XCTFail("Expected CoreError.invalidState, got \(error)")
                return
            }
            XCTAssertTrue(msg.contains("No track loaded"))
        }
    }
    
    func testSeek_BeyondDuration_ThrowsInvalidArgument() throws {
        // Setup: load a file
        let testURL = URL(fileURLWithPath: "/test/audio.mp3")
        mockDecoder.mockStreamInfo = StreamInfo(
            duration: 180.0,
            sampleRate: 44100.0,
            channels: 2,
            bitDepth: 16
        )
        try service.load(url: testURL)
        
        // Test: seek beyond duration
        XCTAssertThrowsError(try service.seek(to: 200.0)) { error in
            guard case CoreError.invalidArgument = error else {
                XCTFail("Expected CoreError.invalidArgument, got \(error)")
                return
            }
        }
    }
}
```

### Error Injection Test

```swift
func testLoad_DecoderThrows_PropagatesError() {
    // Configure mock to throw error
    mockDecoder.shouldThrowOnOpen = true
    mockDecoder.errorToThrow = CoreError.notFound("File not found")
    
    // Attempt to load
    let testURL = URL(fileURLWithPath: "/test/missing.mp3")
    
    XCTAssertThrowsError(try service.load(url: testURL)) { error in
        guard case CoreError.notFound(let msg) = error else {
            XCTFail("Expected CoreError.notFound, got \(error)")
            return
        }
        XCTAssertEqual(msg, "File not found")
    }
    
    // Verify service transitioned to error state
    guard case .error(let stateError) = service.state else {
        XCTFail("Expected .error state, got \(service.state)")
        return
    }
    
    guard case CoreError.notFound = stateError else {
        XCTFail("Expected CoreError.notFound in state, got \(stateError)")
        return
    }
}
```

---

## Model Testing

### Validation Tests

```swift
import XCTest
@testable import HarmoniaCore

final class StreamInfoTests: XCTestCase {
    
    func testValidation_ValidValues_Success() throws {
        let info = StreamInfo(
            duration: 180.0,
            sampleRate: 44100.0,
            channels: 2,
            bitDepth: 16
        )
        
        XCTAssertNoThrow(try info.validate())
    }
    
    func testValidation_NegativeDuration_Throws() {
        let info = StreamInfo(
            duration: -1.0,
            sampleRate: 44100.0,
            channels: 2,
            bitDepth: 16
        )
        
        XCTAssertThrowsError(try info.validate()) { error in
            guard case CoreError.invalidArgument(let msg) = error else {
                XCTFail("Expected CoreError.invalidArgument")
                return
            }
            XCTAssertTrue(msg.contains("Duration"))
        }
    }
    
    func testValidation_InvalidSampleRate_Throws() {
        let info = StreamInfo(
            duration: 180.0,
            sampleRate: 0.0,
            channels: 2,
            bitDepth: 16
        )
        
        XCTAssertThrowsError(try info.validate()) { error in
            guard case CoreError.invalidArgument(let msg) = error else {
                XCTFail("Expected CoreError.invalidArgument")
                return
            }
            XCTAssertTrue(msg.contains("Sample rate"))
        }
    }
    
    func testEquatable_SameValues_AreEqual() {
        let info1 = StreamInfo(duration: 180.0, sampleRate: 44100.0, channels: 2, bitDepth: 16)
        let info2 = StreamInfo(duration: 180.0, sampleRate: 44100.0, channels: 2, bitDepth: 16)
        
        XCTAssertEqual(info1, info2)
    }
    
    func testEquatable_DifferentValues_AreNotEqual() {
        let info1 = StreamInfo(duration: 180.0, sampleRate: 44100.0, channels: 2, bitDepth: 16)
        let info2 = StreamInfo(duration: 180.0, sampleRate: 48000.0, channels: 2, bitDepth: 16)
        
        XCTAssertNotEqual(info1, info2)
    }
}
```

---

## Running Tests

### Command Line (Swift Package Manager)

```bash
# Build and run all tests
swift test

# Run tests in parallel
swift test --parallel

# Run specific test
swift test --filter DefaultPlaybackServiceTests

# Generate code coverage
swift test --enable-code-coverage

# View coverage report (requires xcov or similar)
xcov --scheme HarmoniaCore
```

### Xcode

1. **Open Package**: `File > Open > Package.swift`
2. **Run Tests**: `⌘U` or `Product > Test`
3. **View Coverage**: `⌘9` (Show Report Navigator) > Coverage tab

### Watch Mode (for TDD)

```bash
# Install nodemon
npm install -g nodemon

# Watch for changes and rerun tests
nodemon --watch Sources --watch Tests --ext swift --exec 'swift test || true'
```

---

## Code Coverage

### Generating Coverage Reports

```bash
# 1. Run tests with coverage enabled
swift test --enable-code-coverage

# 2. Find coverage data (varies by platform)
# macOS:
find .build -name '*.profdata'

# 3. Generate human-readable report using xcov or llvm-cov
llvm-cov show \
  .build/debug/HarmoniaCorePackageTests.xctest/Contents/MacOS/HarmoniaCorePackageTests \
  -instr-profile .build/debug/codecov/default.profdata \
  -format=html \
  -output-dir=coverage
```

### Coverage Thresholds

Set in `Package.swift` (future enhancement):

```swift
// Not yet supported by SPM, but planned
.target(
    name: "HarmoniaCore",
    coverage: .minimum(70)
)
```

For now, enforce manually in CI.

---

## Continuous Integration

### GitHub Actions Workflow

`.github/workflows/swift-tests.yml`:

```yaml
name: Swift Tests

on:
  push:
    branches: [ main, develop ]
  pull_request:
    branches: [ main, develop ]

jobs:
  test:
    name: Test on macOS
    runs-on: macos-13
    
    steps:
    - name: Checkout code
      uses: actions/checkout@v3
    
    - name: Select Xcode version
      run: sudo xcode-select -s /Applications/Xcode_15.0.app
    
    - name: Build
      run: swift build -v
    
    - name: Run tests
      run: swift test --parallel
    
    - name: Generate coverage
      run: |
        swift test --enable-code-coverage
        xcrun llvm-cov export -format=lcov \
          .build/debug/HarmoniaCorePackageTests.xctest/Contents/MacOS/HarmoniaCorePackageTests \
          -instr-profile .build/debug/codecov/default.profdata > coverage.lcov
    
    - name: Upload coverage to Codecov
      uses: codecov/codecov-action@v3
      with:
        files: ./coverage.lcov
        flags: swift
        name: swift-coverage
    
    - name: Check coverage threshold
      run: |
        COVERAGE=$(xcrun llvm-cov report \
          .build/debug/HarmoniaCorePackageTests.xctest/Contents/MacOS/HarmoniaCorePackageTests \
          -instr-profile .build/debug/codecov/default.profdata \
          | grep TOTAL | awk '{print $NF}' | sed 's/%//')
        
        if (( $(echo "$COVERAGE < 50" | bc -l) )); then
          echo "Coverage $COVERAGE% is below threshold 50%"
          exit 1
        fi
```

---

## Memory Leak Detection

### Using Xcode Instruments

1. **Build for Profiling**: `⌘I` or `Product > Profile`
2. **Select Leaks Template**
3. **Run tests**
4. **Check for leaks in Memory Graph**

### Using Malloc Stack Logging

```bash
# Enable malloc stack logging
export MallocStackLogging=1

# Run tests
swift test

# Check for leaks (on macOS)
leaks --atExit -- swift test
```

### Common Leak Patterns to Avoid

```swift
// ❌ BAD: Retain cycle
class Service {
    var callback: (() -> Void)?
    
    func setup() {
        callback = {
            self.doSomething()  // Captures self strongly
        }
    }
}

// ✅ GOOD: Weak capture
class Service {
    var callback: (() -> Void)?
    
    func setup() {
        callback = { [weak self] in
            self?.doSomething()
        }
    }
}
```

---

## Performance Testing

### Measuring Execution Time

```swift
func testPerformance_LoadLargeFile() {
    let testURL = URL(fileURLWithPath: "/test/large-audio.mp3")
    
    measure {
        try? service.load(url: testURL)
    }
    
    // XCTest will report average time and standard deviation
}
```

### Benchmarking with XCTMetric

```swift
func testPerformance_DecodingSpeed() {
    let metrics: [XCTMetric] = [
        XCTClockMetric(),
        XCTMemoryMetric(),
        XCTStorageMetric()
    ]
    
    let options = XCTMeasureOptions()
    options.iterationCount = 10
    
    measure(metrics: metrics, options: options) {
        // Decode 1 second of audio
        try? mockDecoder.read(handle, into: buffer, maxFrames: 44100)
    }
}
```

---

## Best Practices

### Do's ✅

- **Test behavior, not implementation**: Focus on what the code does, not how
- **One assertion per test**: Makes failures easier to diagnose
- **Use descriptive names**: Test names should explain the scenario
- **Mock all dependencies**: Isolate the system under test
- **Clean up in tearDown**: Prevent test pollution

### Don'ts ❌

- **Don't test private methods directly**: Test through public interface
- **Don't use real file I/O in unit tests**: Use mocks for speed and isolation
- **Don't share state between tests**: Each test should be independent
- **Don't use sleep() for timing**: Use mock clocks instead
- **Don't ignore failing tests**: Fix or document immediately

---

## Troubleshooting

### Tests Pass Locally but Fail in CI

**Possible causes**:
- File path differences (absolute vs relative)
- Timing assumptions (CI may be slower)
- Missing test resources
- Platform-specific behavior

**Solutions**:
- Use `Bundle.module.url(forResource:)` for test files
- Use mock clocks instead of real time
- Commit test resources to repo
- Add platform-specific conditional compilation

### Tests Are Slow

**Possible causes**:
- Real I/O operations
- Network calls
- Thread.sleep() usage
- Too many integration tests

**Solutions**:
- Use mocks for all external dependencies
- Use `--parallel` flag
- Move slow tests to integration test suite
- Profile with Instruments to find bottlenecks

---

## Resources

- [XCTest Documentation](https://developer.apple.com/documentation/xctest)
- [Swift Package Manager Testing](https://github.com/apple/swift-package-manager/blob/main/Documentation/Usage.md#testing)
- [WWDC: Testing in Xcode](https://developer.apple.com/videos/play/wwdc2019/413/)
- [Ray Wenderlich Testing Tutorial](https://www.raywenderlich.com/21020457-ios-unit-testing-and-ui-testing-tutorial)
