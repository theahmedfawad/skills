# Edge Cases for Embedded Systems Testing

## Table of Contents
1. [Boundary Values](#boundary-values)
2. [Timing Edge Cases](#timing-edge-cases)
3. [Buffer and Memory](#buffer-and-memory)
4. [State Machine Edge Cases](#state-machine-edge-cases)
5. [Error Conditions](#error-conditions)
6. [Hardware-Specific Edge Cases](#hardware-specific-edge-cases)

## Boundary Values

### Integer Boundaries
Always test at and around type boundaries:

```c
// For uint8_t parameters
void test_function_atminboundary(void) { test_func(0); }
void test_function_atmaxboundary(void) { test_func(255); }
void test_function_justbelowmax(void) { test_func(254); }

// For int8_t parameters
void test_function_atnegativemax(void) { test_func(-128); }
void test_function_atpositivemax(void) { test_func(127); }
void test_function_atzero(void) { test_func(0); }

// For uint16_t/uint32_t
void test_function_at16bitmax(void) { test_func(0xFFFF); }
void test_function_at32bitmax(void) { test_func(0xFFFFFFFF); }
```

### Array/Buffer Index Boundaries
```c
#define BUFFER_SIZE 64

void test_buffer_writeatfirstindex(void) {
    buffer_write_at(&buf, 0, 0x42);
    TEST_ASSERT_EQUAL(0x42, buf.data[0]);
}

void test_buffer_writeatlastindex(void) {
    buffer_write_at(&buf, BUFFER_SIZE - 1, 0x42);
    TEST_ASSERT_EQUAL(0x42, buf.data[BUFFER_SIZE - 1]);
}

void test_buffer_writeatinvalidindex_returnserror(void) {
    error_t err = buffer_write_at(&buf, BUFFER_SIZE, 0x42);
    TEST_ASSERT_EQUAL(ERROR_OUT_OF_BOUNDS, err);
}
```

### Bit Position Boundaries
```c
void test_setbit_atposition0(void) {
    uint32_t val = 0;
    set_bit(&val, 0);
    TEST_ASSERT_EQUAL_HEX32(0x00000001, val);
}

void test_setbit_atposition31(void) {
    uint32_t val = 0;
    set_bit(&val, 31);
    TEST_ASSERT_EQUAL_HEX32(0x80000000, val);
}

void test_setbit_atinvalidposition(void) {
    uint32_t val = 0;
    error_t err = set_bit(&val, 32);
    TEST_ASSERT_EQUAL(ERROR_INVALID_BIT, err);
}
```

## Timing Edge Cases

### Timer Overflow/Rollover
Critical for 32-bit millisecond counters (rolls over at ~49.7 days):

```c
void test_timer_elapsedtime_nooverflow(void) {
    time_mock_set(1000);
    timer_start(&timer);
    time_mock_set(1500);
    TEST_ASSERT_EQUAL(500, timer_elapsed(&timer));
}

void test_timer_elapsedtime_withoverflow(void) {
    // Start near max value
    time_mock_set(0xFFFFFFFE);  // 2 ms before overflow
    timer_start(&timer);
    time_mock_set(3);  // 5 ms after overflow
    // Expected: 5 ms elapsed (wraps correctly)
    TEST_ASSERT_EQUAL(5, timer_elapsed(&timer));
}

void test_timer_timeout_acrossoverflow(void) {
    time_mock_set(0xFFFFFFF0);
    timer_start_timeout(&timer, 100);

    time_mock_set(0xFFFFFFFF);  // Still within timeout
    TEST_ASSERT_FALSE(timer_expired(&timer));

    time_mock_set(100 - 16);  // Exactly at timeout
    TEST_ASSERT_TRUE(timer_expired(&timer));
}
```

### Zero Duration
```c
void test_timer_zeroduration_expiresimmediately(void) {
    timer_start_timeout(&timer, 0);
    TEST_ASSERT_TRUE(timer_expired(&timer));
}

void test_delay_zeroms_returnsimmediately(void) {
    time_mock_set(1000);
    delay_ms(0);
    TEST_ASSERT_EQUAL(1000, time_mock_get());  // No time passed
}
```

### Maximum Duration
```c
void test_timer_maxduration(void) {
    timer_start_timeout(&timer, 0xFFFFFFFF);
    time_mock_advance(0xFFFFFFFE);
    TEST_ASSERT_FALSE(timer_expired(&timer));
}
```

### Simultaneous Events
```c
void test_twotimers_expireatsametime(void) {
    timer_start_timeout(&timer1, 100);
    timer_start_timeout(&timer2, 100);

    time_mock_advance(100);

    TEST_ASSERT_TRUE(timer_expired(&timer1));
    TEST_ASSERT_TRUE(timer_expired(&timer2));
}
```

## Buffer and Memory

### Empty Buffer
```c
void test_buffer_readfromempty_returnserror(void) {
    TEST_ASSERT_EQUAL(0, buffer_count(&buf));
    TEST_ASSERT_EQUAL(ERROR_EMPTY, buffer_read(&buf, &data));
}

void test_buffer_peekempty_returnserror(void) {
    TEST_ASSERT_EQUAL(ERROR_EMPTY, buffer_peek(&buf, &data));
}
```

### Full Buffer
```c
void test_buffer_writetofull_returnserror(void) {
    for (int i = 0; i < BUFFER_SIZE; i++) {
        buffer_write(&buf, i);
    }
    TEST_ASSERT_EQUAL(ERROR_FULL, buffer_write(&buf, 0xFF));
}

void test_buffer_writetofull_datapreserved(void) {
    for (int i = 0; i < BUFFER_SIZE; i++) {
        buffer_write(&buf, i);
    }
    buffer_write(&buf, 0xFF);  // Attempt overflow

    // Verify original data unchanged
    for (int i = 0; i < BUFFER_SIZE; i++) {
        TEST_ASSERT_EQUAL(i, buffer_read_at(&buf, i));
    }
}
```

### Circular Buffer Wraparound
```c
void test_circularbuffer_headwraparound(void) {
    // Fill buffer
    for (int i = 0; i < BUFFER_SIZE; i++) {
        buffer_write(&buf, i);
    }
    // Read half
    for (int i = 0; i < BUFFER_SIZE / 2; i++) {
        buffer_read(&buf, &data);
    }
    // Write more (wraps around)
    for (int i = 0; i < BUFFER_SIZE / 2; i++) {
        TEST_ASSERT_EQUAL(ERROR_NONE, buffer_write(&buf, 100 + i));
    }
    // Verify data integrity after wrap
    for (int i = 0; i < BUFFER_SIZE / 2; i++) {
        buffer_read(&buf, &data);
        TEST_ASSERT_EQUAL(BUFFER_SIZE / 2 + i, data);
    }
}
```

### Single Element
```c
void test_buffer_singleelement_writeread(void) {
    buffer_write(&buf, 0x42);
    TEST_ASSERT_EQUAL(1, buffer_count(&buf));

    buffer_read(&buf, &data);
    TEST_ASSERT_EQUAL(0x42, data);
    TEST_ASSERT_EQUAL(0, buffer_count(&buf));
}
```

## State Machine Edge Cases

### Invalid State Transitions
```c
void test_statemachine_invalidtransition_staysincurrentstate(void) {
    sm_init(&sm);
    TEST_ASSERT_EQUAL(STATE_IDLE, sm_get_state(&sm));

    // STOP event invalid in IDLE state
    sm_process_event(&sm, EVENT_STOP);
    TEST_ASSERT_EQUAL(STATE_IDLE, sm_get_state(&sm));
}

void test_statemachine_invalidtransition_returnserror(void) {
    sm_init(&sm);
    error_t err = sm_process_event(&sm, EVENT_STOP);
    TEST_ASSERT_EQUAL(ERROR_INVALID_TRANSITION, err);
}
```

### Repeated Events
```c
void test_statemachine_repeatedstartevent_ignored(void) {
    sm_init(&sm);
    sm_process_event(&sm, EVENT_START);
    TEST_ASSERT_EQUAL(STATE_RUNNING, sm_get_state(&sm));

    // Second START should be ignored or error
    sm_process_event(&sm, EVENT_START);
    TEST_ASSERT_EQUAL(STATE_RUNNING, sm_get_state(&sm));
}
```

### Self-Transitions
```c
void test_statemachine_selftransition_executesaction(void) {
    sm_init(&sm);
    sm_process_event(&sm, EVENT_START);

    int action_count_before = get_action_count();
    sm_process_event(&sm, EVENT_TICK);  // Self-transition in RUNNING

    TEST_ASSERT_EQUAL(STATE_RUNNING, sm_get_state(&sm));
    TEST_ASSERT_EQUAL(action_count_before + 1, get_action_count());
}
```

### Unknown Events
```c
void test_statemachine_unknownevent_returnserror(void) {
    sm_init(&sm);
    error_t err = sm_process_event(&sm, (event_t)999);
    TEST_ASSERT_EQUAL(ERROR_UNKNOWN_EVENT, err);
}
```

## Error Conditions

### NULL Pointer Inputs
```c
void test_function_nullpointer_returnserror(void) {
    TEST_ASSERT_EQUAL(ERROR_NULL_POINTER, module_init(NULL));
}

void test_function_nullconfig_returnserror(void) {
    module_t module;
    TEST_ASSERT_EQUAL(ERROR_NULL_POINTER, module_init(&module, NULL));
}

void test_Function_NullOutput_ReturnsError(void) {
    Test_assert_equal(ERROR_NULL_POINTER, module_get_value(NULL));
}
```

### Invalid Parameters
```c
void test_config_invalidrange_returnserror(void) {
    config_t cfg = { .timeout_ms = 0 };  // 0 not allowed
    TEST_ASSERT_EQUAL(ERROR_INVALID_CONFIG, module_init(&m, &cfg));
}

void test_config_abovemaximum_returnserror(void) {
    config_t cfg = { .timeout_ms = MAX_TIMEOUT + 1 };
    TEST_ASSERT_EQUAL(ERROR_INVALID_CONFIG, module_init(&m, &cfg));
}
```

### Resource Exhaustion
```c
void test_pool_allocationwhenexhausted_returnsnull(void) {
    void* ptrs[POOL_SIZE];

    // Allocate all
    for (int i = 0; i < POOL_SIZE; i++) {
        ptrs[i] = pool_alloc(&pool);
        TEST_ASSERT_NOT_NULL(ptrs[i]);
    }

    // Next allocation fails
    TEST_ASSERT_NULL(pool_alloc(&pool));
}
```

## Hardware-Specific Edge Cases

### Register Bit Manipulation
```c
void test_register_setbitinfullregister(void) {
    mock_reg_set(CTRL_REG, 0xFFFFFFFF);
    driver_set_enable_bit();
    TEST_ASSERT_EQUAL_HEX32(0xFFFFFFFF, mock_reg_get(CTRL_REG));
}

void test_register_clearbitinemptyregister(void) {
    mock_reg_set(CTRL_REG, 0x00000000);
    driver_clear_enable_bit();
    TEST_ASSERT_EQUAL_HEX32(0x00000000, mock_reg_get(CTRL_REG));
}
```

### ADC Conversion Boundaries
```c
void test_adc_convertminvalue(void) {
    mock_adc_set_value(0);
    TEST_ASSERT_EQUAL(0, adc_read_voltage_mv());
}

void test_adc_convertmaxvalue(void) {
    mock_adc_set_value(4095);  // 12-bit ADC
    TEST_ASSERT_EQUAL(3300, adc_read_voltage_mv());  // 3.3V reference
}

void test_adc_convertmidpoint(void) {
    mock_adc_set_value(2048);
    TEST_ASSERT_EQUAL(1650, adc_read_voltage_mv());
}
```

### PWM Duty Cycle Boundaries
```c
void test_pwm_setduty0percent(void) {
    pwm_set_duty(&pwm, 0);
    TEST_ASSERT_EQUAL(0, mock_pwm_get_compare());
}

void test_pwm_setduty100percent(void) {
    pwm_set_duty(&pwm, 100);
    TEST_ASSERT_EQUAL(PWM_PERIOD, mock_pwm_get_compare());
}

void test_pwm_setdutyabove100_clamped(void) {
    pwm_set_duty(&pwm, 150);
    TEST_ASSERT_EQUAL(PWM_PERIOD, mock_pwm_get_compare());
}
```

### Communication Timeouts
```c
void test_i2c_readtimeout(void) {
    mock_i2c_set_busy(true);  // Device doesn't respond
    time_mock_set(0);

    uint8_t data;
    error_t err = i2c_read(&data, I2C_TIMEOUT_MS);

    TEST_ASSERT_EQUAL(ERROR_TIMEOUT, err);
}

void test_i2c_readjustbeforetimeout(void) {
    mock_i2c_set_response_delay(I2C_TIMEOUT_MS - 1);
    mock_i2c_set_response_data(0x42);

    uint8_t data;
    error_t err = i2c_read(&data, I2C_TIMEOUT_MS);

    TEST_ASSERT_EQUAL(ERROR_NONE, err);
    TEST_ASSERT_EQUAL(0x42, data);
}
```

### Interrupt Edge Cases
```c
void test_interrupthandler_reentrancyprotection(void) {
    // First interrupt
    interrupt_mock_trigger(TIMER_IRQ);
    timer_isr();

    // Simulated nested interrupt (should be blocked or handled)
    interrupt_mock_trigger(TIMER_IRQ);
    timer_isr();

    // Verify no data corruption
    TEST_ASSERT_EQUAL(expected_count, timer_get_count());
}

void test_interrupthandler_rapidfiring(void) {
    for (int i = 0; i < 1000; i++) {
        interrupt_mock_trigger(TIMER_IRQ);
        timer_isr();
    }
    TEST_ASSERT_EQUAL(1000, timer_get_count());
}
```
