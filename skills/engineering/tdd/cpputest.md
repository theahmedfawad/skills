# CppUTest Framework Reference

## Table of Contents
1. [Setup](#setup)
2. [Test Structure](#test-structure)
3. [Assertions](#assertions)
4. [Mocking with CppUMock](#mocking-with-cppumock)
5. [Memory Leak Detection](#memory-leak-detection)
6. [Testing C Code from C++](#testing-c-code-from-c)

## Setup

### Project Structure
```
project/
├── src/
│   └── module.c
├── test/
│   ├── test_module.cpp
│   └── AllTests.cpp
└── cpputest/
```

### Basic Test File
```cpp
#include "CppUTest/TestHarness.h"

extern "C" {
    #include "module_under_test.h"
}

TEST_GROUP(ModuleName) {
    void setup() override {
        // Runs before each test
    }

    void teardown() override {
        // Runs after each test
    }
};

TEST(ModuleName, FunctionName_Condition_ExpectedBehavior) {
    // Arrange
    int input = 5;

    // Act
    int result = function_under_test(input);

    // Assert
    LONGS_EQUAL(10, result);
}
```

### Test Runner (AllTests.cpp)
```cpp
#include "CppUTest/CommandLineTestRunner.h"

int main(int argc, char** argv) {
    return CommandLineTestRunner::RunAllTests(argc, argv);
}
```

## Test Structure

### Naming Convention
```cpp
TEST(GroupName, FunctionName_Scenario_ExpectedResult)
```

Examples:
```cpp
TEST(Buffer, Init_SetsAllFieldsToZero)
TEST(StateMachine, InvalidEvent_ReturnsError)
TEST(Timer, Overflow_WrapsCorrectly)
TEST(GPIO, ReadActiveLow_ReturnsInvertedValue)
```

### Test Groups with Shared State
```cpp
TEST_GROUP(CircularBuffer) {
    circular_buffer_t buffer;
    uint8_t storage[64];

    void setup() override {
        buffer_init(&buffer, storage, sizeof(storage));
    }

    void teardown() override {
        // cleanup if needed
    }
};
```

## Assertions

### Numeric Comparisons
```cpp
LONGS_EQUAL(expected, actual);           // long
UNSIGNED_LONGS_EQUAL(expected, actual);  // unsigned long
BYTES_EQUAL(expected, actual);           // single byte
SHORTS_EQUAL(expected, actual);          // short
POINTERS_EQUAL(expected, actual);        // pointers
DOUBLES_EQUAL(expected, actual, tolerance);
```

### Boolean and String
```cpp
CHECK(condition);
CHECK_TRUE(condition);
CHECK_FALSE(condition);
STRCMP_EQUAL(expected, actual);
STRNCMP_EQUAL(expected, actual, length);
```

### Memory
```cpp
MEMCMP_EQUAL(expected, actual, size);
```

### Failure
```cpp
FAIL("message");
CHECK_TEXT(condition, "message");
LONGS_EQUAL_TEXT(expected, actual, "message");
```

### Bits (useful for embedded)
```cpp
BITS_EQUAL(expected, actual, mask);
```

## Mocking with CppUMock

### Basic Mock Setup
```cpp
#include "CppUTest/TestHarness.h"
#include "CppUTestExt/MockSupport.h"

extern "C" {
    #include "module_under_test.h"
}

// Mock implementation
uint32_t hal_read_register(uint32_t address) {
    return mock().actualCall("hal_read_register")
                 .withParameter("address", address)
                 .returnUnsignedLongIntValue();
}

void hal_write_register(uint32_t address, uint32_t value) {
    mock().actualCall("hal_write_register")
          .withParameter("address", address)
          .withParameter("value", value);
}

TEST_GROUP(Driver) {
    void teardown() override {
        mock().clear();
    }
};

TEST(Driver, Init_WritesConfigRegister) {
    mock().expectOneCall("hal_write_register")
          .withParameter("address", CONFIG_REG_ADDR)
          .withParameter("value", DEFAULT_CONFIG);

    driver_init();

    mock().checkExpectations();
}

TEST(Driver, ReadStatus_ReturnsRegisterValue) {
    mock().expectOneCall("hal_read_register")
          .withParameter("address", STATUS_REG_ADDR)
          .andReturnValue(0x42);

    uint32_t status = driver_read_status();

    LONGS_EQUAL(0x42, status);
    mock().checkExpectations();
}
```

### Mock with Output Parameters
```cpp
void spi_transfer(uint8_t* tx, uint8_t* rx, size_t len) {
    mock().actualCall("spi_transfer")
          .withMemoryBufferParameter("tx", tx, len)
          .withOutputParameter("rx", rx)
          .withParameter("len", len);
}

TEST(SPI, Transfer_ReturnsExpectedData) {
    uint8_t tx_data[] = {0x01, 0x02};
    uint8_t rx_expected[] = {0xAA, 0xBB};
    uint8_t rx_actual[2];

    mock().expectOneCall("spi_transfer")
          .withMemoryBufferParameter("tx", tx_data, 2)
          .withOutputParameterReturning("rx", rx_expected, 2)
          .withParameter("len", 2);

    spi_transfer(tx_data, rx_actual, 2);

    MEMCMP_EQUAL(rx_expected, rx_actual, 2);
}
```

### Ignoring Calls
```cpp
mock().expectOneCall("function").ignoreOtherParameters();
mock().ignoreOtherCalls();
```

## Memory Leak Detection

CppUTest automatically detects memory leaks. Use `MemoryLeakWarningPlugin`:

```cpp
// In main
#include "CppUTest/MemoryLeakDetectorNewMacros.h"
#include "CppUTest/MemoryLeakDetectorMallocMacros.h"

TEST(Memory, NoLeaksAfterOperation) {
    char* buffer = (char*)malloc(100);
    // ... use buffer ...
    free(buffer);  // Must free or test fails
}
```

### Disabling for Specific Tests
```cpp
TEST(Module, TestWithIntentionalLeak) {
    MemoryLeakWarningPlugin::turnOffNewDeleteOverloads();
    // ... test code ...
    MemoryLeakWarningPlugin::turnOnNewDeleteOverloads();
}
```

## Testing C Code from C++

### Extern C Wrapper
```cpp
extern "C" {
    #include "c_module.h"

    // Include any C helper functions needed for testing
    #include "test_helpers.h"
}
```

### Linking Considerations
Ensure C code is compiled with a C compiler, test code with C++ compiler. Use `extern "C"` for all C headers in test files.

### Mock C Functions
```cpp
// In test file or separate mock file
extern "C" {
    // Replace the real implementation
    uint32_t hardware_timer_get_ticks(void) {
        return mock().actualCall("hardware_timer_get_ticks")
                     .returnUnsignedLongIntValue();
    }
}
```

## Advanced Patterns

### Test Doubles for Hardware
```cpp
// test_doubles/fake_gpio.cpp
#include "CppUTestExt/MockSupport.h"

static uint32_t gpio_states[8] = {0};

extern "C" {
    void gpio_write(uint8_t pin, uint8_t value) {
        gpio_states[pin] = value;
        mock().actualCall("gpio_write")
              .withParameter("pin", pin)
              .withParameter("value", value);
    }

    uint8_t gpio_read(uint8_t pin) {
        mock().actualCall("gpio_read")
              .withParameter("pin", pin);
        return gpio_states[pin];
    }
}

// Helper for tests
void fake_gpio_set_state(uint8_t pin, uint8_t value) {
    gpio_states[pin] = value;
}
```

### Parameterized Tests
```cpp
TEST_GROUP(Checksum) {
    struct TestCase {
        const uint8_t* data;
        size_t len;
        uint16_t expected_crc;
    };

    static const TestCase test_cases[];
};

const Checksum::TestCase Checksum::test_cases[] = {
    { (const uint8_t*)"", 0, 0x0000 },
    { (const uint8_t*)"A", 1, 0x30C0 },
    { (const uint8_t*)"123456789", 9, 0x29B1 },
};

TEST(Checksum, KnownValues) {
    for (const auto& tc : test_cases) {
        uint16_t result = crc16(tc.data, tc.len);
        LONGS_EQUAL(tc.expected_crc, result);
    }
}
```
