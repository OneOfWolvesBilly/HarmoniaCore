# HarmoniaCore Test Strategy

## Testing Philosophy

HarmoniaCore ensures cross-platform behavior parity between Swift (Apple) and C++20 (Linux) implementations through:

- **Dependency injection**: All platform-specific code isolated behind Port interfaces
- **Mock-based testing**: Business logic tested independently of adapters
- **Behavior contracts**: Both implementations must pass identical test vectors

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

**Current focus**:
- **Unit tests** — Models and Services with mock ports (in place)
- **Integration tests** — Adapters with real I/O (planned)
- **E2E tests** — Complete playback scenarios (planned)

---

## Coverage Goals

| Component | Target | Swift (current) | C++20 (planned) |
|-----------|--------|------------|------------|
| **Models** | 100% | ✅ 100% | 🎯 Planned |
| **Services** | 85%+ | ✅ 85% | 🎯 Planned |
| **Adapters** | 70%+ | 🔜 Planned | 🎯 Planned |
| **Overall** | 70%+ | 50% | 🎯 Planned |

---

## What We Test

### 1. Models
- Data initialization and validation
- Constraint enforcement
- Equality comparison
- Error case handling

### 2. Services
- State machine transitions
- Error handling and propagation
- Idempotent operations
- Concurrent access safety
- Business logic correctness

### 3. Adapters (planned)
- Platform API integration
- Real file I/O operations
- Audio hardware interaction
- Error mapping correctness

---

## Test Categories

### Positive Tests (Happy Path)
Verify normal operation sequences:
- Load valid file → Success
- Play after load → Playing state
- Seek to valid position → Position updated

### Negative Tests (Error Cases)
Verify error handling:
- Load missing file → `CoreError.notFound`
- Play without load → `CoreError.invalidState`
- Seek beyond duration → `CoreError.invalidArgument`

### Boundary Tests (Edge Cases)
Verify behavior at limits:
- Seek to 0.0 seconds
- Seek to exactly duration
- Files with zero duration
- Rapid state transitions

### Idempotency Tests
Verify operations can be called repeatedly:
- `play()` when already playing → No error
- `pause()` when already paused → No error
- `stop()` when already stopped → No error

### Concurrency Tests
Verify thread safety:
- Concurrent reads of `currentTime()`
- Background file loading
- State queries during playback

---

## Cross-Platform Parity (planned)

Swift and C++20 implementations **must produce identical observable behavior**:

| Behavior | Parity Requirement |
|----------|-------------------|
| **State transitions** | Exact same sequence |
| **Error types** | Same `CoreError` category |
| **Seek positions** | ±1ms tolerance |
| **Decoded waveforms** | ±1 sample tolerance |
| **Timing** | ±10ms tolerance |

Parity is verified through **test vectors** (JSON-based behavioral specifications).

See [`docs/specs/api-parity.md`](specs/api-parity.md) for detailed requirements.

---

## Quality Gates

All implementations must meet these thresholds before merge:

**Code Quality**:
- ✅ All unit tests pass (100% pass rate)
- ✅ Coverage thresholds met (see table above)
- ✅ No compiler warnings
- ✅ No static analysis issues

**Runtime Quality**:
- ✅ No memory leaks (verified by tooling)
- ✅ No race conditions (thread sanitizer clean)
- ✅ No undefined behavior (sanitizers clean)

**Behavioral Quality**:
- ✅ Test vectors pass (cross-platform parity)
- ✅ Integration tests pass (when adapter-level coverage is in place)
- ✅ Performance benchmarks within tolerance

---

## Testing Anti-Patterns to Avoid

### Don't Test Implementation Details
❌ Avoid testing internal state or private methods  
✅ Test observable behavior through public interfaces

### Don't Create Multi-Concern Tests
❌ Avoid tests that verify multiple unrelated behaviors  
✅ Each test should verify one specific concern

### Don't Write Flaky Tests
❌ Avoid timing-dependent assertions (`sleep(1)`)  
✅ Use mock clocks and deterministic control flow

### Don't Ignore Error Cases
❌ Avoid only testing happy paths  
✅ Negative tests are as important as positive tests

---

## Future Testing Roadmap

### Integration Tests (planned)
- Real audio file decoding
- Actual audio output verification
- Cross-platform behavior validation
- CC0 test corpus creation

### E2E Tests (planned)
- Complete playback scenarios
- Playlist handling
- Error recovery flows
- Performance benchmarking

---

## Implementation Guides

**Platform-specific testing details**:
- Swift: [`docs/impl/testing_swift.md`](impl/testing_swift.md)
- C++20: [`docs/impl/testing_cpp.md`](impl/testing_cpp.md)

**Cross-platform parity**:
- Behavior contracts: [`docs/specs/api-parity.md`](specs/api-parity.md)
- Test vectors: [`docs/specs/vectors/`](specs/vectors/)