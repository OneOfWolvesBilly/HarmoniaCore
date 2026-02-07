# API Parity Specification

## 1. Overview

This document defines the **behavioral contract** between Swift (Apple) and C++20 (Linux) implementations of HarmoniaCore.

While the two implementations use different languages, frameworks, and idioms internally, they **MUST** produce identical observable behavior when given the same inputs.

**Scope**: This specification covers behavior that can be **objectively tested** through automated validation (test vectors).

**Non-goals**: This does not specify internal implementation details, performance characteristics, or platform-specific optimizations.

---

## 2. Core Principle

> **"Same Input → Same Output"**

For any valid sequence of operations:
- Swift implementation output = C++20 implementation output
- Within defined tolerance ranges (see §7)

**Verification Method**: Test vectors (JSON files) that both implementations execute.

---

## 3. Testable Categories

All cross-platform behavior falls into these categories:

| Category | What is Tested | Tolerance |
|----------|----------------|-----------|
| **State Transitions** | Service state changes from operations | Exact match |
| **Error Mapping** | Platform errors → CoreError categories | Exact match |
| **Seek Semantics** | Position changes from seek operations | ±1ms |
| **Metadata Round-trip** | TagBundle preservation through write/read | Exact (excludes artwork) |
| **Decode Consistency** | PCM waveform output from same audio file | ±1 sample |

---

## 4. Category Definitions

### 4.1 State Transitions

**Definition**: Given a service in state S, calling operation O must transition to state S' (or throw error E).

**What is tested**:
```
Initial State + Operation → Final State (or Error)
```

**Examples**:
```
stopped + load(valid_url) → paused
paused + play() → playing
playing + pause() → paused
stopped + play() → error(invalidState)
```

**Assertion Method**:
```
assert_state_transition(
    initial: "stopped",
    operation: "load",
    args: {"url": "test.mp3"},
    expected_final: "paused"
)
```

**Parity Requirements**:
- State names must match exactly
- Error categories must match exactly
- Transition sequence must be identical

---

### 4.2 Error Mapping

**Definition**: Platform-specific errors must map to the same CoreError category.

**What is tested**:
```
Platform Error → CoreError Category
```

**Examples**:
```
POSIX ENOENT → CoreError.notFound
AVError.fileNotFound → CoreError.notFound

Invalid file format → CoreError.unsupported
AAC decoder missing → CoreError.unsupported
```

**Assertion Method**:
```
assert_error_mapping(
    operation: "open",
    trigger: "nonexistent_file",
    expected_error: "notFound",
    expected_message_contains: "File not found"
)
```

**Parity Requirements**:
- Error category must match
- Error message should be descriptive (exact text not required)
- Associated data (if any) should be equivalent

---

### 4.3 Seek Semantics

**Definition**: Seeking to position T must place playback at position T (within tolerance).

**What is tested**:
```
seek(to: seconds) → currentTime() ≈ seconds
```

**Examples**:
```
seek(10.0) → currentTime() in [9.999, 10.001]
seek(0.0) → currentTime() ≈ 0.0
seek(duration - 0.1) → currentTime() ≈ duration - 0.1
```

**Assertion Method**:
```
assert_seek_position(
    target: 10.0,
    tolerance_ms: 1.0,
    expected_min: 9.999,
    expected_max: 10.001
)
```

**Parity Requirements**:
- Position accuracy: ±1ms
- Behavior while playing: continues from new position
- Behavior while paused: remains paused at new position
- Out-of-bounds seek: throws invalidArgument

---

### 4.4 Metadata Round-trip

**Definition**: Writing tags and reading them back must preserve all fields.

**What is tested**:
```
write(url, tags) → read(url) == tags
```

**Examples**:
```
Original: {title: "Song", artist: "Artist", year: 2025}
Write → Read
Result: {title: "Song", artist: "Artist", year: 2025}
```

**Assertion Method**:
```
assert_tag_roundtrip(
    original: {
        "title": "Test Song",
        "artist": "Test Artist",
        "year": 2025
    },
    expected: {
        "title": "Test Song",
        "artist": "Test Artist",
        "year": 2025
    }
)
```

**Parity Requirements**:
- All text fields must match exactly
- Numeric fields must match exactly
- Missing fields must be nil/null (not empty string)
- **Excluded from parity**: Artwork data (format conversion allowed)

---

### 4.5 Decode Consistency

**Definition**: Decoding the same audio file must produce identical PCM samples.

**What is tested**:
```
decode(file.mp3) → PCM waveform
```

**Examples**:
```
sine_440hz_1s.mp3 → [sample array]
Swift output ≈ C++ output (±1 sample)
```

**Assertion Method**:
```
assert_waveform_match(
    file: "sine_440hz_1s.mp3",
    tolerance_samples: 1,
    check_duration: true,
    check_sample_rate: true
)
```

**Parity Requirements**:
- Duration match: exact (from StreamInfo)
- Sample rate match: exact
- Channel count match: exact
- PCM samples: ±1 sample value difference allowed
- Seek-decode: samples after seek(T) must match

---

## 5. Test Vector Format

Test vectors are JSON files containing sequences of operations and assertions.

### 5.1 Vector Structure
```json
{
  "vector_name": "playback_state_transitions",
  "version": "1.0",
  "description": "Tests basic state machine transitions",
  "fixtures": {
    "audio_file": "tests/fixtures/sine_440hz_1s.mp3"
  },
  "test_cases": [
    {
      "name": "load_valid_file",
      "operations": [
        {
          "action": "load",
          "args": {"url": "{audio_file}"}
        }
      ],
      "assertions": [
        {
          "type": "state_equals",
          "expected": "paused"
        }
      ]
    }
  ]
}
```

### 5.2 Operation Types

| Action | Args | Description |
|--------|------|-------------|
| `load` | `{url}` | Load audio file |
| `play` | - | Start playback |
| `pause` | - | Pause playback |
| `stop` | - | Stop playback |
| `seek` | `{to: seconds}` | Seek to position |

### 5.3 Assertion Types

| Type | Args | Description |
|------|------|-------------|
| `state_equals` | `{expected}` | Assert current state |
| `error_thrown` | `{category, message_contains}` | Assert error was thrown |
| `position_near` | `{target, tolerance_ms}` | Assert playback position |
| `tags_equal` | `{expected}` | Assert tag bundle contents |
| `duration_equals` | `{expected}` | Assert stream duration |

---

## 6. Implementation Requirements

### 6.1 Vector Runner (Swift)

**Location**: `Tests/HarmoniaCoreTests/Parity/VectorRunnerTests.swift`

**Requirements**:
- Parse JSON test vectors
- Execute operations in sequence
- Evaluate assertions
- Report pass/fail for each test case

**Example**:
```swift
func testVector_playback_state_transitions() throws {
    let runner = VectorRunner(service: createTestService())
    let results = try runner.run(vector: "playback_state_transitions.json")
    XCTAssertTrue(results.allPassed)
}
```

### 6.2 Vector Runner (C++20)

**Location**: `tests/parity/vector_runner_test.cpp`

**Requirements**:
- Same as Swift runner
- Use nlohmann/json for parsing
- Use Google Test for assertions

**Example**:
```cpp
TEST(VectorRunner, PlaybackStateTransitions) {
    auto runner = VectorRunner(createTestService());
    auto results = runner.run("playback_state_transitions.json");
    EXPECT_TRUE(results.allPassed());
}
```

---

## 7. Tolerance Definitions

| Metric | Tolerance | Rationale |
|--------|-----------|-----------|
| **State transitions** | Exact match | States are discrete enum values |
| **Error categories** | Exact match | CoreError is well-defined enum |
| **Seek position** | ±1ms | Accounts for frame boundary rounding |
| **Tag text fields** | Exact match | UTF-8 strings must be identical |
| **Tag numeric fields** | Exact match | Integers must be identical |
| **PCM samples** | ±1 sample | Accounts for rounding in format conversion |
| **Duration** | Exact match | Derived from container metadata |
| **Sample rate** | Exact match | Derived from stream properties |

**Excluded from Parity**:
- Artwork image data (format conversion allowed: JPEG ↔ PNG)
- Artwork compression level
- Performance metrics (CPU, memory, latency)
- Log message text (only behavior is tested)

---

## 8. Parity Validation Rules

### 8.1 Continuous Integration

**Requirement**: Both Swift and C++20 implementations MUST pass all vectors before merge.

**Workflow**:
```yaml
# .github/workflows/parity.yml
name: Cross-Platform Parity

on: [push, pull_request]

jobs:
  swift-vectors:
    runs-on: macos-latest
    steps:
      - run: swift test --filter VectorRunnerTests
      
  cpp-vectors:
    runs-on: ubuntu-latest
    steps:
      - run: ctest -R VectorRunner
      
  parity-gate:
    needs: [swift-vectors, cpp-vectors]
    runs-on: ubuntu-latest
    steps:
      - name: Check both passed
        run: |
          if [ "${{ needs.swift-vectors.result }}" != "success" ]; then exit 1; fi
          if [ "${{ needs.cpp-vectors.result }}" != "success" ]; then exit 1; fi
```

### 8.2 Version Control

**Test Vectors**:
- Stored in: `tests/vectors/*.json`
- Version controlled in Git
- Any change requires review + approval

**Drift Prevention**:
- CI fails if Swift passes but C++ fails (or vice versa)
- No platform-specific exceptions allowed
- Failed vectors block merge

### 8.3 Exemption Process

If a vector cannot be implemented on one platform:

1. Document reason in `docs/impl/parity_exemptions.md`
2. Mark vector as `"skip_platform": "linux"` or `"skip_platform": "apple"`
3. Requires approval from project maintainer
4. Must include plan for future implementation

**Example**:
```json
{
  "test_cases": [
    {
      "name": "flac_decoding",
      "skip_platform": "apple",
      "skip_reason": "FLAC requires macOS Pro build (libFLAC)"
    }
  ]
}
```

---

## 9. Anti-Patterns

### ❌ Don't: Test implementation details
```json
// BAD: Testing internal class names
{
  "assertions": [
    {"type": "decoder_is", "expected": "AVAssetReaderDecoderAdapter"}
  ]
}
```

✅ **Instead**: Test observable behavior only.

### ❌ Don't: Platform-specific vectors
```json
// BAD: Swift-only vector
{
  "vector_name": "ios_sandbox_permissions",
  "test_cases": [...]
}
```

✅ **Instead**: Define behavior abstractly. Platform differences go in exemptions.

### ❌ Don't: Overly strict tolerances
```json
// BAD: Impossible precision
{
  "assertions": [
    {"type": "position_near", "target": 10.0, "tolerance_ms": 0.0}
  ]
}
```

✅ **Instead**: Use reasonable tolerances (±1ms for seek, ±1 sample for PCM).

---

## 10. Future Extensions

### Planned Categories (v0.3+)

- **Gapless Playback**: Frame-accurate transitions
- **Real-time Effects**: EQ parameter application
- **Playlist Queueing**: Multi-track state management
- **Hi-Res Audio**: 96kHz/192kHz/384kHz sample rates

### Vector Schema Evolution

When adding new categories:
1. Increment `"version"` in vectors
2. Document changes in `tests/vectors/CHANGELOG.md`
3. Maintain backward compatibility where possible

---

## 11. References

- Test Vectors: `tests/vectors/README.md`
- Vector Schema: `tests/vectors/schema.md`
- Parity Gate: `docs/impl/parity_gate.md`
- Swift Implementation: `docs/impl/02_01_apple_adapters_impl.md`
- C++20 Implementation: `docs/impl/02_02_linux_adapters_impl.md`