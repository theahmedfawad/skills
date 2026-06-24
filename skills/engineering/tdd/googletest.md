# Google Test Framework Reference

## Table of Contents
1. [Setup](#setup)
2. [Test Structure](#test-structure)
3. [Assertions](#assertions)
4. [Test Fixtures](#test-fixtures)
5. [Mocking with Google Mock](#mocking-with-google-mock)
6. [Parameterized Tests](#parameterized-tests)
7. [Testing C Code](#testing-c-code)

## Setup

### Project Structure
```
project/
├── src/
│   └── module.c
├── test/
│   ├── test_module.cpp
│   └── main.cpp
└── CMakeLists.txt
```

### CMake Integration
```cmake
include(FetchContent)
FetchContent_Declare(
    googletest
    GIT_REPOSITORY https://github.com/google/googletest.git
    GIT_TAG release-1.12.1
)
FetchContent_MakeAvailable(googletest)

add_executable(tests test_module.cpp)
target_link_libraries(tests GTest::gtest_main GTest::gmock)
```

### Basic Test File
```cpp
#include <gtest/gtest.h>

extern "C" {
    #include "module_under_test.h"
}

TEST(ModuleName, FunctionName_Condition_ExpectedBehavior) {
    // Arrange
    int input = 5;

    // Act
    int result = function_under_test(input);

    // Assert
    EXPECT_EQ(10, result);
}
```

## Test Structure

### Naming Convention
```cpp
TEST(TestSuiteName, TestName)
// TestSuiteName: Usually the module or class being tested
// TestName: FunctionName_Scenario_ExpectedResult
```

Examples:
```cpp
TEST(Buffer, Init_SetsAllFieldsToZero)
TEST(StateMachine, ProcessEvent_InvalidInput_ReturnsError)
TEST(Timer, GetElapsed_AfterOverflow_CalculatesCorrectly)
```

## Assertions

### Basic Assertions
```cpp
// Fatal (stops test on failure)
ASSERT_EQ(expected, actual);
ASSERT_NE(val1, val2);
ASSERT_LT(val1, val2);
ASSERT_LE(val1, val2);
ASSERT_GT(val1, val2);
ASSERT_GE(val1, val2);

// Non-fatal (continues on failure)
EXPECT_EQ(expected, actual);
EXPECT_NE(val1, val2);
EXPECT_LT(val1, val2);
EXPECT_LE(val1, val2);
EXPECT_GT(val1, val2);
EXPECT_GE(val1, val2);
```

### Boolean
```cpp
EXPECT_TRUE(condition);
EXPECT_FALSE(condition);
ASSERT_TRUE(condition);
ASSERT_FALSE(condition);
```

### String
```cpp
EXPECT_STREQ(expected, actual);    // C strings equal
EXPECT_STRNE(str1, str2);          // C strings not equal
EXPECT_STRCASEEQ(exp, act);        // Case-insensitive
```

### Floating Point
```cpp
EXPECT_FLOAT_EQ(expected, actual);
EXPECT_DOUBLE_EQ(expected, actual);
EXPECT_NEAR(val1, val2, abs_error);
```

### Pointers
```cpp
EXPECT_EQ(nullptr, ptr);
EXPECT_NE(nullptr, ptr);
```

### Death Tests (for testing assertions/crashes)
```cpp
EXPECT_DEATH(function_that_crashes(), "expected error message");
ASSERT_DEATH(dangerous_call(), "");
```

### Custom Messages
```cpp
EXPECT_EQ(expected, actual) << "Custom failure message: " << context;
```

## Test Fixtures

### Basic Fixture
```cpp
class BufferTest : public ::testing::Test {
protected:
    void SetUp() override {
        buffer_init(&buffer_, storage_, sizeof(storage_));
    }

    void TearDown() override {
        // cleanup
    }

    circular_buffer_t buffer_;
    uint8_t storage_[64];
};

TEST_F(BufferTest, Write_IncreasesCount) {
    buffer_write(&buffer_, 0x42);
    EXPECT_EQ(1, buffer_.count);
}

TEST_F(BufferTest, Read_DecreasesCount) {
    buffer_write(&buffer_, 0x42);
    buffer_read(&buffer_);
    EXPECT_EQ(0, buffer_.count);
}
```

### Fixture with Helper Methods
```cpp
class StateMachineTest : public ::testing::Test {
protected:
    void SetUp() override {
        sm_init(&sm_);
    }

    void AdvanceToState(state_t target) {
        while (sm_get_state(&sm_) != target) {
            sm_process_event(&sm_, EVENT_NEXT);
        }
    }

    state_machine_t sm_;
};
```

## Mocking with Google Mock

### Interface Definition
```cpp
// For C code, create a C++ interface
class IHardwareTimer {
public:
    virtual ~IHardwareTimer() = default;
    virtual uint32_t GetTicks() = 0;
    virtual void Reset() = 0;
};
```

### Mock Class
```cpp
#include <gmock/gmock.h>

class MockHardwareTimer : public IHardwareTimer {
public:
    MOCK_METHOD(uint32_t, GetTicks, (), (override));
    MOCK_METHOD(void, Reset, (), (override));
};
```

### Using Mocks in Tests
```cpp
using ::testing::Return;
using ::testing::_;

TEST_F(TimerTest, Elapsed_ReturnsCorrectValue) {
    MockHardwareTimer mock_hw;

    EXPECT_CALL(mock_hw, GetTicks())
        .WillOnce(Return(100))
        .WillOnce(Return(150));

    Timer timer(&mock_hw);
    timer.Start();

    EXPECT_EQ(50, timer.GetElapsed());
}
```

### Mock Expectations
```cpp
// Exact call count
EXPECT_CALL(mock, Method()).Times(3);

// Call count ranges
EXPECT_CALL(mock, Method()).Times(::testing::AtLeast(1));
EXPECT_CALL(mock, Method()).Times(::testing::AtMost(5));
EXPECT_CALL(mock, Method()).Times(::testing::Between(2, 4));

// Argument matchers
EXPECT_CALL(mock, Method(::testing::Eq(42)));
EXPECT_CALL(mock, Method(::testing::_));  // Any argument
EXPECT_CALL(mock, Method(::testing::Gt(10)));

// Sequences
{
    ::testing::InSequence seq;
    EXPECT_CALL(mock, First());
    EXPECT_CALL(mock, Second());
}
```

## Parameterized Tests

### Value-Parameterized Tests
```cpp
class CrcTest : public ::testing::TestWithParam<std::tuple<
    std::vector<uint8_t>,  // input data
    uint16_t               // expected CRC
>> {};

TEST_P(CrcTest, CalculatesCorrectly) {
    auto [data, expected] = GetParam();
    uint16_t result = crc16(data.data(), data.size());
    EXPECT_EQ(expected, result);
}

INSTANTIATE_TEST_SUITE_P(
    KnownValues,
    CrcTest,
    ::testing::Values(
        std::make_tuple(std::vector<uint8_t>{}, 0x0000),
        std::make_tuple(std::vector<uint8_t>{0x41}, 0x30C0),
        std::make_tuple(std::vector<uint8_t>{0x31, 0x32, 0x33}, 0x5BCE)
    )
);
```

### Type-Parameterized Tests
```cpp
template <typename T>
class BufferTest : public ::testing::Test {
protected:
    T buffer_;
};

using BufferTypes = ::testing::Types<
    CircularBuffer<uint8_t, 64>,
    CircularBuffer<uint16_t, 32>,
    CircularBuffer<uint32_t, 16>
>;

TYPED_TEST_SUITE(BufferTest, BufferTypes);

TYPED_TEST(BufferTest, Init_IsEmpty) {
    EXPECT_TRUE(this->buffer_.IsEmpty());
}
```

## Testing C Code

### Extern C Linkage
```cpp
extern "C" {
    #include "c_module.h"
}
```

### Function Pointer Injection for C
```cpp
// In C header
typedef uint32_t (*timer_get_ticks_fn)(void);
void module_init(timer_get_ticks_fn get_ticks);

// In test
static uint32_t mock_ticks = 0;

uint32_t mock_get_ticks() {
    return mock_ticks;
}

TEST(Module, UsesInjectedTimer) {
    module_init(mock_get_ticks);
    mock_ticks = 1000;

    EXPECT_EQ(1000, module_get_current_time());
}
```

### Link-Time Substitution
```cpp
// Create mock implementation file
extern "C" {
    uint32_t hal_gpio_read(uint8_t pin) {
        // Mock implementation
        return mock_gpio_states[pin];
    }
}

// Link test with mock instead of real HAL
```

## Advanced Patterns

### Testing State Machines
```cpp
class StateMachineTest : public ::testing::Test {
protected:
    state_machine_t sm_;

    void SetUp() override {
        sm_init(&sm_);
    }

    void ExpectState(state_t expected) {
        EXPECT_EQ(expected, sm_get_state(&sm_));
    }

    void ProcessAndExpect(event_t event, state_t expected) {
        sm_process_event(&sm_, event);
        ExpectState(expected);
    }
};

TEST_F(StateMachineTest, FullTransitionSequence) {
    ExpectState(STATE_IDLE);
    ProcessAndExpect(EVENT_START, STATE_RUNNING);
    ProcessAndExpect(EVENT_PAUSE, STATE_PAUSED);
    ProcessAndExpect(EVENT_RESUME, STATE_RUNNING);
    ProcessAndExpect(EVENT_STOP, STATE_IDLE);
}
```

### Testing Timeouts
```cpp
TEST_F(TimerTest, Timeout_AfterDuration) {
    mock_ticks = 0;
    timer_start(&timer_, 100);

    mock_ticks = 99;
    EXPECT_FALSE(timer_expired(&timer_));

    mock_ticks = 100;
    EXPECT_TRUE(timer_expired(&timer_));
}
```
