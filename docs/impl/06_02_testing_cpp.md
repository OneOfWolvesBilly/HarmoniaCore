# C++20 Testing Implementation Guide

This document provides C++20-specific testing implementation details for HarmoniaCore on Linux platforms.

**See also**: [Test Strategy](../test_strategy.md) for platform-agnostic testing philosophy.

---

## Testing Framework

### Google Test (gtest + gmock)

HarmoniaCore uses **Google Test** with **Google Mock**:

**Why Google Test?**
- Industry standard for C++ testing
- Excellent CMake integration
- Rich assertion library
- Built-in mocking support (gmock)
- Active maintenance and documentation

**Alternatives considered**:
- Catch2: Simpler but less mature mocking
- Boost.Test: Heavy dependency
- doctest: Lightweight but limited features

---

## Project Structure

```
linux-cpp/
├── CMakeLists.txt               # Root CMake configuration
├── src/
│   └── harmonia_core/           # Production code
│       ├── models/
│       ├── ports/
│       ├── adapters/
│       └── services/
├── tests/
│   ├── CMakeLists.txt           # Test configuration
│   ├── test_support/            # Mock implementations
│   │   ├── mock_clock_port.hpp
│   │   ├── mock_decoder_port.hpp
│   │   ├── mock_audio_output_port.hpp
│   │   ├── mock_file_access_port.hpp
│   │   ├── mock_tag_reader_port.hpp
│   │   └── mock_tag_writer_port.hpp
│   ├── models/                  # Model tests
│   │   ├── stream_info_test.cpp
│   │   ├── tag_bundle_test.cpp
│   │   └── core_error_test.cpp
│   └── services/                # Service tests
│       └── default_playback_service_test.cpp
└── build/                       # Build artifacts (gitignored)
```

---

## CMake Configuration

### Root CMakeLists.txt

```cmake
cmake_minimum_required(VERSION 3.20)
project(HarmoniaCore VERSION 0.2.0 LANGUAGES CXX)

# C++20 required
set(CMAKE_CXX_STANDARD 20)
set(CMAKE_CXX_STANDARD_REQUIRED ON)
set(CMAKE_CXX_EXTENSIONS OFF)

# Compiler warnings
if(CMAKE_CXX_COMPILER_ID MATCHES "GNU|Clang")
    add_compile_options(-Wall -Wextra -Wpedantic -Werror)
endif()

# Enable testing
enable_testing()

# Dependencies
find_package(PkgConfig REQUIRED)
pkg_check_modules(PIPEWIRE REQUIRED libpipewire-0.3)
pkg_check_modules(FFMPEG REQUIRED libavformat libavcodec libavutil libswresample)
pkg_check_modules(TAGLIB REQUIRED taglib)

# Production library
add_subdirectory(src)

# Tests (optional, controlled by BUILD_TESTING)
option(BUILD_TESTING "Build tests" ON)
if(BUILD_TESTING)
    add_subdirectory(tests)
endif()
```

### tests/CMakeLists.txt

```cmake
# Fetch Google Test
include(FetchContent)
FetchContent_Declare(
    googletest
    GIT_REPOSITORY https://github.com/google/googletest.git
    GIT_TAG v1.14.0
)
FetchContent_MakeAvailable(googletest)

# Test executable
add_executable(harmonia_core_tests
    # Test support
    test_support/mock_clock_port.cpp
    test_support/mock_decoder_port.cpp
    test_support/mock_audio_output_port.cpp
    
    # Model tests
    models/stream_info_test.cpp
    models/tag_bundle_test.cpp
    models/core_error_test.cpp
    
    # Service tests
    services/default_playback_service_test.cpp
)

target_link_libraries(harmonia_core_tests
    PRIVATE
        harmonia_core
        GTest::gtest_main
        GTest::gmock
)

# Discover tests for CTest
include(GoogleTest)
gtest_discover_tests(harmonia_core_tests)

# Coverage (optional)
option(ENABLE_COVERAGE "Enable code coverage" OFF)
if(ENABLE_COVERAGE)
    target_compile_options(harmonia_core_tests PRIVATE --coverage)
    target_link_options(harmonia_core_tests PRIVATE --coverage)
endif()
```

---

## Mock Implementation Pattern

All mock ports use **virtual functions** for polymorphism and **Google Mock** for call verification.

### Mock Template

```cpp
#pragma once
#include <gmock/gmock.h>
#include "harmonia_core/ports/<port>.hpp"

namespace harmonia_core::testing {

class Mock<Port>Port : public <Port>Port {
public:
    // Mock each virtual method
    MOCK_METHOD(<return_type>, <method_name>, (<parameters>), (override));
    
    // For methods with complex signatures, use helper types:
    // MOCK_METHOD(DecodeHandle, open, (const std::string& url), (override));
    
    // For void methods:
    // MOCK_METHOD(void, close, (DecodeHandle handle), (override));
    
    // For const methods:
    // MOCK_METHOD(StreamInfo, info, (DecodeHandle handle), (const, override));
};

} // namespace harmonia_core::testing
```

### Example: MockDecoderPort

```cpp
#pragma once
#include <gmock/gmock.h>
#include "harmonia_core/ports/decoder_port.hpp"

namespace harmonia_core::testing {

class MockDecoderPort : public DecoderPort {
public:
    MOCK_METHOD(DecodeHandle, open, (const std::string& url), (override));
    
    MOCK_METHOD(int, read, 
                (DecodeHandle handle, float* buffer, int max_frames), 
                (override));
    
    MOCK_METHOD(void, seek, 
                (DecodeHandle handle, double seconds), 
                (override));
    
    MOCK_METHOD(StreamInfo, info, 
                (DecodeHandle handle), 
                (const, override));
    
    MOCK_METHOD(void, close, 
                (DecodeHandle handle), 
                (override));
};

} // namespace harmonia_core::testing
```

---

## Test Naming Convention

Follow Google Test naming conventions:

```
<TestSuite><Test><Scenario>_<ExpectedResult>
```

### Examples

**Positive Tests**:
```cpp
TEST(PlaybackServiceTest, LoadValidFile_Success)
TEST(PlaybackServiceTest, PlayAfterLoad_TransitionsToPlaying)
TEST(PlaybackServiceTest, SeekToValidPosition_UpdatesCurrentTime)
```

**Negative Tests**:
```cpp
TEST(PlaybackServiceTest, LoadMissingFile_ThrowsNotFound)
TEST(PlaybackServiceTest, PlayWithoutLoad_ThrowsInvalidState)
TEST(PlaybackServiceTest, SeekBeyondDuration_ThrowsInvalidArgument)
```

**Parameterized Tests**:
```cpp
TEST_P(PlaybackServiceParameterizedTest, SeekToPosition_UpdatesCorrectly)
```

---

## Test Example: Service Testing

### Basic Test Structure

```cpp
#include <gtest/gtest.h>
#include <gmock/gmock.h>
#include "harmonia_core/services/default_playback_service.hpp"
#include "test_support/mock_decoder_port.hpp"
#include "test_support/mock_audio_output_port.hpp"
#include "test_support/mock_clock_port.hpp"
#include "test_support/noop_logger.hpp"

using namespace harmonia_core;
using namespace harmonia_core::testing;
using ::testing::Return;
using ::testing::Throw;
using ::testing::_;

class DefaultPlaybackServiceTest : public ::testing::Test {
protected:
    void SetUp() override {
        mock_decoder = std::make_shared<MockDecoderPort>();
        mock_audio = std::make_shared<MockAudioOutputPort>();
        mock_clock = std::make_shared<MockClockPort>();
        logger = std::make_shared<NoopLogger>();
        
        service = std::make_unique<DefaultPlaybackService>(
            mock_decoder,
            mock_audio,
            mock_clock,
            logger
        );
    }
    
    void TearDown() override {
        service.reset();
        mock_decoder.reset();
        mock_audio.reset();
        mock_clock.reset();
        logger.reset();
    }
    
    std::shared_ptr<MockDecoderPort> mock_decoder;
    std::shared_ptr<MockAudioOutputPort> mock_audio;
    std::shared_ptr<MockClockPort> mock_clock;
    std::shared_ptr<NoopLogger> logger;
    std::unique_ptr<DefaultPlaybackService> service;
};

TEST_F(DefaultPlaybackServiceTest, StateTransitions_LoadPlayPauseStop) {
    // Initial state
    EXPECT_EQ(service->state(), PlaybackState::Stopped);
    
    // Setup mock expectations
    DecodeHandle mock_handle{1};
    StreamInfo mock_info{180.0, 44100.0, 2, 16};
    
    EXPECT_CALL(*mock_decoder, open("/test/audio.mp3"))
        .WillOnce(Return(mock_handle));
    EXPECT_CALL(*mock_decoder, info(mock_handle))
        .WillOnce(Return(mock_info));
    EXPECT_CALL(*mock_audio, configure(44100.0, 2, _))
        .Times(1);
    
    // Load file
    service->load("/test/audio.mp3");
    EXPECT_EQ(service->state(), PlaybackState::Paused);
    
    // Setup play expectations
    EXPECT_CALL(*mock_audio, start())
        .Times(1);
    
    // Play
    service->play();
    EXPECT_EQ(service->state(), PlaybackState::Playing);
    
    // Pause (stop is called on audio)
    EXPECT_CALL(*mock_audio, stop())
        .Times(1);
    service->pause();
    EXPECT_EQ(service->state(), PlaybackState::Paused);
    
    // Stop (close decoder)
    EXPECT_CALL(*mock_decoder, close(mock_handle))
        .Times(1);
    service->stop();
    EXPECT_EQ(service->state(), PlaybackState::Stopped);
}

TEST_F(DefaultPlaybackServiceTest, PlayWithoutLoad_ThrowsInvalidState) {
    EXPECT_THROW({
        try {
            service->play();
        } catch (const CoreError& e) {
            EXPECT_EQ(e.type(), CoreErrorType::InvalidState);
            EXPECT_THAT(e.what(), ::testing::HasSubstr("No track loaded"));
            throw;
        }
    }, CoreError);
}

TEST_F(DefaultPlaybackServiceTest, SeekBeyondDuration_ThrowsInvalidArgument) {
    // Setup: load a file
    DecodeHandle mock_handle{1};
    StreamInfo mock_info{180.0, 44100.0, 2, 16};  // 180 second duration
    
    EXPECT_CALL(*mock_decoder, open(_))
        .WillOnce(Return(mock_handle));
    EXPECT_CALL(*mock_decoder, info(_))
        .WillRepeatedly(Return(mock_info));
    
    service->load("/test/audio.mp3");
    
    // Test: seek beyond duration
    EXPECT_THROW({
        try {
            service->seek(200.0);  // Beyond 180 seconds
        } catch (const CoreError& e) {
            EXPECT_EQ(e.type(), CoreErrorType::InvalidArgument);
            throw;
        }
    }, CoreError);
}
```

### Error Injection Test

```cpp
TEST_F(DefaultPlaybackServiceTest, LoadDecoderThrows_PropagatesError) {
    // Configure mock to throw error
    EXPECT_CALL(*mock_decoder, open("/test/missing.mp3"))
        .WillOnce(Throw(CoreError::NotFound("File not found")));
    
    // Attempt to load
    EXPECT_THROW({
        try {
            service->load("/test/missing.mp3");
        } catch (const CoreError& e) {
            EXPECT_EQ(e.type(), CoreErrorType::NotFound);
            EXPECT_STREQ(e.what(), "File not found");
            
            // Verify service transitioned to error state
            // (implementation-specific; may need StatusObserver pattern)
            throw;
        }
    }, CoreError);
}
```

---

## Model Testing

### Value Type Tests

```cpp
#include <gtest/gtest.h>
#include "harmonia_core/models/stream_info.hpp"

using namespace harmonia_core;

TEST(StreamInfoTest, Validation_ValidValues_Success) {
    StreamInfo info{180.0, 44100.0, 2, 16};
    EXPECT_NO_THROW(validate(info));
}

TEST(StreamInfoTest, Validation_NegativeDuration_Throws) {
    StreamInfo info{-1.0, 44100.0, 2, 16};
    
    EXPECT_THROW({
        try {
            validate(info);
        } catch (const CoreError& e) {
            EXPECT_EQ(e.type(), CoreErrorType::InvalidArgument);
            EXPECT_THAT(e.what(), ::testing::HasSubstr("Duration"));
            throw;
        }
    }, CoreError);
}

TEST(StreamInfoTest, Validation_InvalidSampleRate_Throws) {
    StreamInfo info{180.0, 0.0, 2, 16};
    
    EXPECT_THROW({
        try {
            validate(info);
        } catch (const CoreError& e) {
            EXPECT_EQ(e.type(), CoreErrorType::InvalidArgument);
            EXPECT_THAT(e.what(), ::testing::HasSubstr("Sample rate"));
            throw;
        }
    }, CoreError);
}

TEST(StreamInfoTest, Equality_SameValues_AreEqual) {
    StreamInfo info1{180.0, 44100.0, 2, 16};
    StreamInfo info2{180.0, 44100.0, 2, 16};
    
    EXPECT_EQ(info1, info2);
}

TEST(StreamInfoTest, Equality_DifferentValues_AreNotEqual) {
    StreamInfo info1{180.0, 44100.0, 2, 16};
    StreamInfo info2{180.0, 48000.0, 2, 16};
    
    EXPECT_NE(info1, info2);
}
```

---

## Parameterized Tests

For testing multiple similar scenarios:

```cpp
#include <gtest/gtest.h>

struct SeekTestCase {
    double position;
    bool should_succeed;
    std::string description;
};

class SeekParameterizedTest : public DefaultPlaybackServiceTest,
                               public ::testing::WithParamInterface<SeekTestCase> {
};

TEST_P(SeekParameterizedTest, SeekToPosition_BehavesCorrectly) {
    auto test_case = GetParam();
    
    // Setup
    DecodeHandle mock_handle{1};
    StreamInfo mock_info{180.0, 44100.0, 2, 16};
    
    EXPECT_CALL(*mock_decoder, open(_)).WillOnce(Return(mock_handle));
    EXPECT_CALL(*mock_decoder, info(_)).WillRepeatedly(Return(mock_info));
    
    service->load("/test/audio.mp3");
    
    // Test
    if (test_case.should_succeed) {
        EXPECT_CALL(*mock_decoder, seek(_, test_case.position)).Times(1);
        EXPECT_NO_THROW(service->seek(test_case.position)) 
            << test_case.description;
    } else {
        EXPECT_THROW(service->seek(test_case.position), CoreError) 
            << test_case.description;
    }
}

INSTANTIATE_TEST_SUITE_P(
    SeekPositions,
    SeekParameterizedTest,
    ::testing::Values(
        SeekTestCase{0.0, true, "Seek to beginning"},
        SeekTestCase{90.0, true, "Seek to middle"},
        SeekTestCase{180.0, true, "Seek to end"},
        SeekTestCase{-1.0, false, "Negative position"},
        SeekTestCase{200.0, false, "Beyond duration"}
    )
);
```

---

## Running Tests

### Command Line (CMake + CTest)

```bash
# Configure build
cmake -S . -B build -DCMAKE_BUILD_TYPE=Debug -DBUILD_TESTING=ON

# Build tests
cmake --build build

# Run all tests
cd build && ctest --output-on-failure

# Run specific test
cd build && ctest -R StreamInfoTest --verbose

# Run tests in parallel
cd build && ctest -j8

# Generate coverage
cmake -S . -B build -DENABLE_COVERAGE=ON
cmake --build build
cd build && ctest
gcov *.gcda
```

### Direct Execution

```bash
# Run test binary directly
./build/tests/harmonia_core_tests

# Run specific test suite
./build/tests/harmonia_core_tests --gtest_filter=PlaybackServiceTest.*

# Run specific test
./build/tests/harmonia_core_tests --gtest_filter=PlaybackServiceTest.LoadValidFile_Success

# List all tests
./build/tests/harmonia_core_tests --gtest_list_tests

# Repeat tests (for finding flaky tests)
./build/tests/harmonia_core_tests --gtest_repeat=100
```

---

## Code Coverage

### Generating Coverage with gcov/lcov

```bash
# 1. Build with coverage enabled
cmake -S . -B build -DCMAKE_BUILD_TYPE=Debug -DENABLE_COVERAGE=ON
cmake --build build

# 2. Run tests
cd build && ctest

# 3. Generate coverage report
lcov --capture --directory . --output-file coverage.info

# 4. Filter out system/test files
lcov --remove coverage.info '/usr/*' '*/tests/*' '*/googletest/*' \
     --output-file coverage_filtered.info

# 5. Generate HTML report
genhtml coverage_filtered.info --output-directory coverage_html

# 6. View report
xdg-open coverage_html/index.html
```

### Coverage Thresholds (CI)

```bash
#!/bin/bash
# check_coverage.sh

THRESHOLD=70

# Extract coverage percentage
COVERAGE=$(lcov --summary coverage_filtered.info 2>&1 | \
           grep "lines" | \
           awk '{print $2}' | \
           sed 's/%//')

echo "Coverage: $COVERAGE%"

if (( $(echo "$COVERAGE < $THRESHOLD" | bc -l) )); then
    echo "ERROR: Coverage $COVERAGE% is below threshold $THRESHOLD%"
    exit 1
fi

echo "Coverage check passed"
```

---

## Continuous Integration

### GitHub Actions Workflow

`.github/workflows/cpp-tests.yml`:

```yaml
name: C++ Tests

on:
  push:
    branches: [ main, develop ]
  pull_request:
    branches: [ main, develop ]

jobs:
  test:
    name: Test on Ubuntu
    runs-on: ubuntu-22.04
    
    steps:
    - name: Checkout code
      uses: actions/checkout@v3
    
    - name: Install dependencies
      run: |
        sudo apt-get update
        sudo apt-get install -y \
          build-essential \
          cmake \
          ninja-build \
          libpipewire-0.3-dev \
          libavformat-dev \
          libavcodec-dev \
          libavutil-dev \
          libswresample-dev \
          libtag1-dev \
          lcov
    
    - name: Configure CMake
      run: |
        cmake -S . -B build \
          -GNinja \
          -DCMAKE_BUILD_TYPE=Debug \
          -DBUILD_TESTING=ON \
          -DENABLE_COVERAGE=ON
    
    - name: Build
      run: cmake --build build
    
    - name: Run tests
      run: |
        cd build
        ctest --output-on-failure
    
    - name: Generate coverage
      run: |
        cd build
        lcov --capture --directory . --output-file coverage.info
        lcov --remove coverage.info '/usr/*' '*/tests/*' '*/googletest/*' \
             --output-file coverage_filtered.info
    
    - name: Upload coverage to Codecov
      uses: codecov/codecov-action@v3
      with:
        files: ./build/coverage_filtered.info
        flags: cpp
        name: cpp-coverage
    
    - name: Check coverage threshold
      run: |
        cd build
        COVERAGE=$(lcov --summary coverage_filtered.info 2>&1 | \
                   grep "lines" | \
                   awk '{print $2}' | \
                   sed 's/%//')
        
        if (( $(echo "$COVERAGE < 70" | bc -l) )); then
          echo "Coverage $COVERAGE% is below threshold 70%"
          exit 1
        fi
```

---

## Memory Safety Testing

### AddressSanitizer (ASan)

Detects memory errors: use-after-free, buffer overflows, leaks.

```bash
# Build with ASan
cmake -S . -B build \
  -DCMAKE_BUILD_TYPE=Debug \
  -DCMAKE_CXX_FLAGS="-fsanitize=address -fno-omit-frame-pointer"

cmake --build build

# Run tests (ASan will report errors)
cd build && ctest
```

### ThreadSanitizer (TSan)

Detects data races and deadlocks.

```bash
# Build with TSan
cmake -S . -B build \
  -DCMAKE_BUILD_TYPE=Debug \
  -DCMAKE_CXX_FLAGS="-fsanitize=thread"

cmake --build build
cd build && ctest
```

### UndefinedBehaviorSanitizer (UBSan)

Detects undefined behavior: signed integer overflow, null pointer dereference, etc.

```bash
# Build with UBSan
cmake -S . -B build \
  -DCMAKE_BUILD_TYPE=Debug \
  -DCMAKE_CXX_FLAGS="-fsanitize=undefined"

cmake --build build
cd build && ctest
```

### Valgrind

Traditional memory checker (slower but comprehensive).

```bash
# Build with debug symbols
cmake -S . -B build -DCMAKE_BUILD_TYPE=Debug
cmake --build build

# Run tests under Valgrind
valgrind --leak-check=full --show-leak-kinds=all \
  ./build/tests/harmonia_core_tests
```

---

## Performance Testing

### Google Benchmark Integration

```cpp
#include <benchmark/benchmark.h>
#include "harmonia_core/services/default_playback_service.hpp"

static void BM_LoadLargeFile(benchmark::State& state) {
    auto service = CreateTestService();
    
    for (auto _ : state) {
        service->load("/test/large-audio.mp3");
        service->stop();  // Cleanup for next iteration
    }
}
BENCHMARK(BM_LoadLargeFile);

static void BM_DecodingSpeed(benchmark::State& state) {
    auto decoder = CreateTestDecoder();
    auto handle = decoder->open("/test/audio.mp3");
    
    std::vector<float> buffer(4096 * 2);
    
    for (auto _ : state) {
        decoder->read(handle, buffer.data(), 4096);
    }
    
    decoder->close(handle);
}
BENCHMARK(BM_DecodingSpeed);

BENCHMARK_MAIN();
```

---

## Best Practices

### Do's ✅

- **Use smart pointers**: Prevent memory leaks with `std::unique_ptr`, `std::shared_ptr`
- **RAII everywhere**: Resources managed by constructors/destructors
- **Prefer value semantics**: Pass by const reference, return by value (move-enabled)
- **Mock dependencies**: Use `EXPECT_CALL` for deterministic tests
- **Test move semantics**: Verify your move constructors/assignments work

### Don'ts ❌

- **Don't use raw pointers**: Unless interfacing with C APIs
- **Don't ignore warnings**: `-Wall -Wextra -Wpedantic -Werror`
- **Don't forget const-correctness**: Mark methods `const` when appropriate
- **Don't leak resources**: Always use RAII
- **Don't write non-deterministic tests**: Avoid timing assumptions

---

## Troubleshooting

### Tests Pass Locally but Fail in CI

**Possible causes**:
- Different compiler versions (GCC vs Clang)
- Different standard library implementations
- Missing dependencies
- Timing assumptions

**Solutions**:
- Use Docker to reproduce CI environment
- Check compiler and library versions
- Pin dependency versions in CMake
- Use mock clocks, not real time

### Slow Compilation

**Possible causes**:
- Template instantiation overhead
- Too many includes
- Missing precompiled headers

**Solutions**:
```cmake
# Enable ccache
find_program(CCACHE_PROGRAM ccache)
if(CCACHE_PROGRAM)
    set(CMAKE_CXX_COMPILER_LAUNCHER "${CCACHE_PROGRAM}")
endif()

# Use precompiled headers
target_precompile_headers(harmonia_core_tests PRIVATE
    <memory>
    <vector>
    <string>
    <gtest/gtest.h>
)

# Enable parallel builds
cmake --build build -j$(nproc)
```

---

## Resources

- [Google Test Documentation](https://google.github.io/googletest/)
- [Google Mock for Dummies](https://github.com/google/googletest/blob/main/docs/gmock_for_dummies.md)
- [CMake Testing Guide](https://cmake.org/cmake/help/latest/manual/ctest.1.html)
- [Modern CMake](https://cliutils.gitlab.io/modern-cmake/)
- [C++ Core Guidelines: Testing](https://isocpp.github.io/CppCoreGuidelines/CppCoreGuidelines#S-testing)
