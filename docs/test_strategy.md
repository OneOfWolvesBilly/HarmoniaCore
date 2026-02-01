# HarmoniaCore Test Strategy

## Testing Philosophy

HarmoniaCore's testing strategy ensures cross-platform behavior parity between Swift (Apple) and C++20 (Linux) implementations through comprehensive unit testing and mock-based dependency injection.

---

## Test Pyramid

```
                    /\
                   /  \
                  /    \
                 /  E2E \
                /--------\
               /          \
              / Integration\
             /--------------\
            /                \
           /     Unit Tests   \
          /____________________\
```

### Current Focus (v0.1): Unit Tests

- **62 unit tests** covering Models and Services
- **50% code coverage** (Models 100%, Services 85%)
- **6 mock implementations** for all ports
- **Zero integration tests** (planned for v0.2)

---

## Coverage Strategy

### What We Test

#### 1. Models (100% Coverage)

**Goal**: Verify data integrity and validation

**Test Cases**:
- ✅ Initialization with valid values
- ✅ Validation rules enforcement
- ✅ Equatable conformance
- ✅ Error case handling

**Example**:
```swift
// StreamInfoTests.swift
func testValidation_InvalidSampleRate() {
    let info = StreamInfo(duration: 0, sampleRate: -1, channels: 2, bitDepth: 16)
    XCTAssertThrowsError(try info.validate())
}
```

#### 2. Services (~85% Coverage)

**Goal**: Verify business logic and state management

**Test Cases**:
- ✅ State machine transitions
- ✅ Error handling and propagation
- ✅ Idempotent operations
- ✅ Concurrent access safety
- ✅ Mock dependency injection

**Example**:
```swift
// DefaultPlaybackServiceTests.swift
func testStateTransitions() {
    XCTAssertEqual(service.state, .stopped)
    try service.load(url: testURL)
    XCTAssertEqual(service.state, .paused)
    try service.play()
    XCTAssertEqual(service.state, .playing)
}
```

### What We Don't Test (Yet)

#### Adapters (0% Coverage)

**Why**: Adapters are thin wrappers around platform APIs (AVFoundation, FFmpeg, PipeWire). Testing them requires:
- Real audio hardware
- Real file I/O
- Platform-specific integration testing

**Plan**: v0.2 will add integration tests with real audio files and hardware validation.

---

## Mock Strategy

### Mock Port Pattern

All mock ports implement the same pattern:

1. **Call Tracking**: Boolean flags for each method
2. **Argument Capture**: Store parameters for verification
3. **Configurable Returns**: Public properties for test customization
4. **Error Injection**: Optional error throwing for negative tests

**Example**:
```swift
final class MockDecoderPort: DecoderPort {
    // 1. Call tracking
    var openCalled = false
    
    // 2. Argument capture
    var lastOpenedURL: URL?
    
    // 3. Configurable return
    var mockStreamInfo = StreamInfo(...)
    
    // 4. Error injection
    var shouldThrowOnOpen = false
    
    func open(url: URL) throws -> DecodeHandle {
        openCalled = true
        lastOpenedURL = url
        
        if shouldThrowOnOpen {
            throw CoreError.notFound("Mock error")
        }
        
        return DecodeHandle(id: UUID())
    }
}
```

### Mock Benefits

- ✅ **Fast**: No real I/O or hardware access
- ✅ **Deterministic**: No timing dependencies
- ✅ **Isolated**: Test business logic only
- ✅ **Flexible**: Easy to simulate error conditions

---

## Test Categories

### 1. Positive Tests (Happy Path)

**Goal**: Verify normal operation

**Examples**:
- Load valid file → Success
- Play after load → Playing state
- Seek to valid position → Position updated

### 2. Negative Tests (Error Cases)

**Goal**: Verify error handling

**Examples**:
- Load invalid file → CoreError.notFound
- Play without load → CoreError.invalidState
- Seek beyond duration → CoreError.invalidArgument

### 3. Boundary Tests (Edge Cases)

**Goal**: Verify behavior at limits

**Examples**:
- Seek to 0.0 seconds
- Seek to exactly duration
- Load file with 0 duration
- Play/pause rapidly

### 4. Idempotency Tests

**Goal**: Verify operations can be called multiple times safely

**Examples**:
- Call play() twice → Same result
- Call pause() when already paused → No error
- Call stop() when already stopped → No error

### 5. Concurrency Tests

**Goal**: Verify thread safety

**Examples**:
- Multiple threads read currentTime() → No crash
- Load on background thread → Safe
- State reads while playing → Consistent

---

## Test Organization

### Directory Structure

```
Tests/HarmoniaCoreTests/
├── TestSupport/          # Shared test infrastructure
│   ├── MockClockPort.swift
│   ├── MockDecoderPort.swift
│   ├── MockAudioOutputPort.swift
│   ├── MockFileAccessPort.swift
│   ├── MockTagReaderPort.swift
│   └── MockTagWriterPort.swift
├── Models/               # Model tests
│   ├── StreamInfoTests.swift
│   ├── TagBundleTests.swift
│   └── CoreErrorTests.swift
└── Services/             # Service tests
    └── DefaultPlaybackServiceTests.swift
```

### Test Naming Convention

```
test<Component><Scenario>_<ExpectedResult>()
```

**Examples**:
- `testLoad_ValidFile_Success()`
- `testPlay_WithoutLoad_ThrowsInvalidState()`
- `testSeek_BeyondDuration_ThrowsInvalidArgument()`

---

## Coverage Metrics

### Current Status (v0.1)

| Component | Lines | Coverage | Tests |
|-----------|-------|----------|-------|
| Models | 150 | 100% | 21 |
| Services | 450 | 85% | 41 |
| Ports | 100 | 0% | 0 (interfaces only) |
| Adapters | 800 | 0% | 0 (planned v0.2) |
| **Total** | **1,500** | **50%** | **62** |

### Coverage Goals

- ✅ **v0.1**: 50% overall (Models 100%, Services 85%)
- 🎯 **v0.2**: 70% overall (add integration tests)
- 🎯 **v0.3**: 80% overall (add E2E tests)

---

## Continuous Integration

### GitHub Actions Workflow

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
    
    - name: Build
      run: |
        cd apple-swift
        swift build
    
    - name: Run tests
      run: |
        cd apple-swift
        swift test --parallel
    
    - name: Generate coverage
      run: |
        cd apple-swift
        swift test --enable-code-coverage
```

### Quality Gates

Tests must pass before merge:
- ✅ All unit tests pass
- ✅ No new compiler warnings
- ✅ Code coverage doesn't decrease
- ✅ No memory leaks (Instruments)

---

## Testing Anti-Patterns (Avoid These)

### ❌ Testing Implementation Details

```swift
// BAD: Testing internal state
XCTAssertEqual(service.internalBuffer.count, 4096)

// GOOD: Testing observable behavior
XCTAssertEqual(service.state, .playing)
```

### ❌ Testing Multiple Concerns

```swift
// BAD: One test does too much
func testEverything() {
    try service.load(url: testURL)
    try service.play()
    service.pause()
    try service.seek(to: 30)
    service.stop()
    // ... 50 more lines
}

// GOOD: Focused tests
func testLoad_ValidFile_Success() { ... }
func testPlay_AfterLoad_TransitionsToPlaying() { ... }
```

### ❌ Flaky Tests (Timing-Dependent)

```swift
// BAD: Depends on timing
try service.play()
sleep(1)
XCTAssertGreaterThan(service.currentTime(), 0.5)

// GOOD: Use mock clock
mockClock.advanceBy(nanoseconds: 1_000_000_000)
XCTAssertEqual(service.currentTime(), 1.0)
```

---

## Future Testing Plans

### v0.2: Integration Tests

**Goal**: Verify real adapter behavior

**Test Cases**:
- Real audio file playback
- Actual audio output to speakers
- Cross-platform behavior parity

**Infrastructure**:
- CC0 test audio corpus
- Frame-by-frame comparison
- Waveform checksum validation

### v0.3: E2E Tests

**Goal**: Verify complete user scenarios

**Test Cases**:
- Full playback session (load → play → seek → stop)
- Playlist playback
- Error recovery scenarios

---

## Test Maintenance

### When to Update Tests

1. **New feature**: Add positive and negative tests
2. **Bug fix**: Add regression test first
3. **Refactoring**: Tests should still pass (behavior unchanged)
4. **Breaking change**: Update tests to match new behavior

### Test Review Checklist

- [ ] Test names clearly describe scenario and expectation
- [ ] Each test verifies one concern
- [ ] Mocks are properly isolated
- [ ] No timing dependencies (flaky tests)
- [ ] Error cases are covered
- [ ] Documentation updated if needed

---

## Resources

- [Testing Guide](TESTING.md) - How to run and write tests
- [Swift Testing Best Practices](https://developer.apple.com/documentation/xctest)
- [Hexagonal Architecture Testing](https://herbertograca.com/2017/11/16/explicit-architecture-01-ddd-hexagonal-onion-clean-cqrs-how-i-put-it-all-together/)
