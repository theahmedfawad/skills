# Unity Test Framework Reference

## Table of Contents
1. [Setup](#setup)
2. [Test Structure](#test-structure)
3. [Assertions](#assertions)
4. [Test Groups](#test-groups)
5. [Fixtures](#fixtures)
6. [Common Patterns](#common-patterns)

## Setup

### Minimal Project Structure
```
project/
├── src/
│   └── module.c
├── test/
│   ├── test_module.c
│   └── test_runners/
│       └── test_module_runner.c
└── unity/
    ├── unity.c
    ├── unity.h
    └── unity_internals.h
```

### Basic Test File Template
```c
#include "unity.h"
#include "module_under_test.h"

void setup(void) {
    // Runs before each test
}

void teardown(void) {
    // Runs after each test
}

void test_functionname_condition_expectedbehavior(void) {
    // Arrange
    int input = 5;

    // Act
    int result = function_under_test(input);

    // Assert
    TEST_ASSERT_EQUAL(10, result);
}

int main(void) {
    UNITY_BEGIN();
    RUN_TEST(test_functionname_condition_expectedbehavior);
    return UNITY_END();
}
```

## Test Structure

### Naming Convention
```
test_<functionname>_<scenario>_<expectedresult>
```

Examples:
```c
test_buffer_init_setsallfieldstozero
test_statemachine_invalidevent_returnserror
test_timer_overflow_wrapscorrectly
test_gpio_read_activelow_returnsinvertedvalue
```

### Arrange-Act-Assert Pattern
```c
void test_circularbuffer_write_incrementshead(void) {
    // Arrange
    circular_buffer_t buf;
    buffer_init(&buf);
    uint8_t data = 0x42;

    // Act
    buffer_write(&buf, data);

    // Assert
    TEST_ASSERT_EQUAL(1, buf.head);
    TEST_ASSERT_EQUAL(data, buf.data[0]);
}
```

## Assertions

### Equality
```c
TEST_ASSERT_EQUAL(expected, actual);           // int
TEST_ASSERT_EQUAL_UINT8(expected, actual);     // uint8_t
TEST_ASSERT_EQUAL_UINT16(expected, actual);    // uint16_t
TEST_ASSERT_EQUAL_UINT32(expected, actual);    // uint32_t
TEST_ASSERT_EQUAL_INT8(expected, actual);      // int8_t
TEST_ASSERT_EQUAL_HEX8(expected, actual);      // hex display
TEST_ASSERT_EQUAL_HEX32(expected, actual);     // hex display
TEST_ASSERT_EQUAL_PTR(expected, actual);       // pointers
```

### Arrays
```c
TEST_ASSERT_EQUAL_UINT8_ARRAY(expected, actual, len);
TEST_ASSERT_EQUAL_MEMORY(expected, actual, len);
```

### Floating Point
```c
TEST_ASSERT_FLOAT_WITHIN(delta, expected, actual);
TEST_ASSERT_DOUBLE_WITHIN(delta, expected, actual);
```

### Boolean and NULL
```c
TEST_ASSERT_TRUE(condition);
TEST_ASSERT_FALSE(condition);
TEST_ASSERT_NULL(pointer);
TEST_ASSERT_NOT_NULL(pointer);
```

### Bit Operations (embedded-specific)
```c
TEST_ASSERT_BITS(mask, expected, actual);
TEST_ASSERT_BITS_HIGH(mask, actual);
TEST_ASSERT_BITS_LOW(mask, actual);
TEST_ASSERT_BIT_HIGH(bit_number, actual);
TEST_ASSERT_BIT_LOW(bit_number, actual);
```

### String
```c
TEST_ASSERT_EQUAL_STRING(expected, actual);
TEST_ASSERT_EQUAL_STRING_LEN(expected, actual, len);
```

### Failure
```c
TEST_FAIL();
TEST_FAIL_MESSAGE("Custom failure message");
TEST_IGNORE();
TEST_IGNORE_MESSAGE("Reason for ignoring");
```

## Test Groups

### Organizing Related Tests
```c
// test_buffer_group.c
#include "unity.h"
#include "unity_fixture.h"

TEST_GROUP(buffer);

TEST_SETUP(buffer) {
    buffer_init(&test_buffer);
}

TEST_TEAR_DOWN(buffer) {
    // cleanup
}

TEST(Buffer, init_setscounttozero) {
    TEST_ASSERT_EQUAL(0, test_buffer.count);
}

TEST(Buffer, write_increasescount) {
    buffer_write(&test_buffer, 0x42);
    TEST_ASSERT_EQUAL(1, test_buffer.count);
}

TEST_GROUP_RUNNER(Buffer) {
    RUN_TEST_CASE(Buffer, init_setscounttozero);
    RUN_TEST_CASE(Buffer, write_increasescount);
}
```

## Fixtures

### Static Test Data
```c
static module_state_t test_state;
static const module_config_t test_config = {
    .param1 = 100,
    .param2 = 200,
    .enabled = true
};

void setup(void) {
    memset(&test_state, 0, sizeof(test_state));
    module_init(&test_state, &test_config);
}
```

### Multiple Configurations
```c
static const test_case_t test_cases[] = {
    { .input = 0,   .expected = 0 },
    { .input = 1,   .expected = 1 },
    { .input = 255, .expected = 255 },
};

void test_function_parameterizedcases(void) {
    for (size_t i = 0; i < ARRAY_SIZE(test_cases); i++) {
        int result = function_under_test(test_cases[i].input);
        TEST_ASSERT_EQUAL_MESSAGE(
            test_cases[i].expected,
            result,
            "Failed at index"
        );
    }
}
```

## Common Patterns

### Testing State Machines
```c
void test_statemachine_transitionsequence(void) {
    state_machine_t sm;
    sm_init(&sm);

    TEST_ASSERT_EQUAL(STATE_IDLE, sm_get_state(&sm));

    sm_process_event(&sm, EVENT_START);
    TEST_ASSERT_EQUAL(STATE_RUNNING, sm_get_state(&sm));

    sm_process_event(&sm, EVENT_STOP);
    TEST_ASSERT_EQUAL(STATE_IDLE, sm_get_state(&sm));
}
```

### Testing with Time
```c
static uint32_t mock_time_ms = 0;

uint32_t get_time_ms(void) {
    return mock_time_ms;
}

void test_timer_expiration(void) {
    timer_t timer;
    mock_time_ms = 0;
    timer_start(&timer, 100);

    mock_time_ms = 99;
    TEST_ASSERT_FALSE(timer_expired(&timer));

    mock_time_ms = 100;
    TEST_ASSERT_TRUE(timer_expired(&timer));
}
```

### Testing Error Returns
```c
void test_function_invalidinput_returnserror(void) {
    error_code_t result = function_under_test(NULL);
    TEST_ASSERT_EQUAL(ERROR_NULL_POINTER, result);
}

void test_function_validinput_returnssuccess(void) {
    valid_input_t input = {0};
    error_code_t result = function_under_test(&input);
    TEST_ASSERT_EQUAL(ERROR_NONE, result);
}
```

### Testing Callbacks
```c
static int callback_count = 0;
static int last_callback_value = 0;

void mock_callback(int value) {
    callback_count++;
    last_callback_value = value;
}

void setup(void) {
    callback_count = 0;
    last_callback_value = 0;
}

void test_module_callbackinvoked(void) {
    module_register_callback(mock_callback);
    module_trigger_event(42);

    TEST_ASSERT_EQUAL(1, callback_count);
    TEST_ASSERT_EQUAL(42, last_callback_value);
}
```
