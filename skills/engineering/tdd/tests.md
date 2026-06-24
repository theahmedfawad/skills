# Good and Bad Tests

## Good Tests

**Behavior through the public interface**: Test what a caller observes, not how the module does it.

```c
// GOOD: Tests observable behavior through the module's API
void test_ringbuffer_writethenread_returnssamebyte(void) {
    ring_buffer_t buf;
    ring_buffer_init(&buf);

    ring_buffer_write(&buf, 0x42);
    uint8_t out;

    TEST_ASSERT_EQUAL(ERROR_NONE, ring_buffer_read(&buf, &out));
    TEST_ASSERT_EQUAL_HEX8(0x42, out);
}
```

Characteristics:

- Tests behavior callers care about
- Uses the public API only
- Survives internal refactors (rename a static helper, change the storage layout — test still passes)
- Describes WHAT, not HOW
- One logical assertion per test

### Structure every test with Arrange-Act-Assert

```c
void test_buffer_write_increasescount(void) {
    // Arrange
    buffer_t buf;
    buffer_init(&buf);

    // Act
    buffer_write(&buf, 0x42);

    // Assert
    TEST_ASSERT_EQUAL(1, buf.count);
}
```

### Naming convention

```
test_<module>_<function/scenario>_<expectedbehavior>
```

Examples:

- `test_timer_start_setsrunningflag`
- `test_buffer_writewhenfull_returnserror`
- `test_statemachine_invalidevent_staysincurrentstate`
- `test_gpio_readactivelow_returnsinvertedvalue`

A good test name reads like a one-line specification of a capability.

## Bad Tests

**Implementation-detail tests**: Coupled to internal structure.

```c
// BAD: Asserts on an internal call instead of observable behavior
void test_checkout_callshalwriteregister(void) {
    checkout(&ctx);
    TEST_ASSERT_EQUAL(1, mock_hal_get_write_count(CTRL_REG));
}
```

Red flags:

- Asserting on call counts/order of internal collaborators
- Testing static (private) functions directly
- Reaching past the interface to inspect internal state that the API doesn't expose
- Test breaks when refactoring without a behavior change
- Test name describes HOW, not WHAT

```c
// BAD: Bypasses the interface to verify internal storage
void test_config_storestimeoutinfield(void) {
    config_set_timeout(&cfg, 100);
    TEST_ASSERT_EQUAL(100, cfg.internal.raw_timeout_ticks);  // implementation detail
}

// GOOD: Verifies through the interface
void test_config_timeoutisreadback(void) {
    config_set_timeout(&cfg, 100);
    TEST_ASSERT_EQUAL(100, config_get_timeout(&cfg));
}
```

Note the difference from **hardware mocking**: replacing a register or peripheral at the boundary
so code can run on the host is legitimate (see [mocking.md](mocking.md)). Asserting on *how often*
your module poked that register, when the caller can't observe it, is testing implementation.
