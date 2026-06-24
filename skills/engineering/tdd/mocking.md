# Mocking

## When to Mock

Mock at **system boundaries** only — the places where your code meets hardware or the outside world:

- Hardware registers and memory-mapped peripherals
- Peripherals and buses (GPIO, SPI, I2C, UART, ADC, PWM, CAN)
- Time and clocks (SysTick, timers, RTC)
- Interrupts and callbacks
- The RTOS / OS (queues, mutexes, delays) and the file system

Don't mock:

- Your own modules, drivers, or state machines
- Internal collaborators
- Anything you control and could test directly

The goal of mocking on embedded targets is to run the code under test on a **host machine** by
replacing hardware-dependent operations with controllable test doubles — not to verify that one
internal function called another.

## Designing for Mockability

At hardware boundaries, design interfaces that are easy to substitute.

**1. Use dependency injection.** Pass the hardware interface in rather than reaching for a global
register inside the module:

```c
// Easy to substitute: HAL passed in
void driver_init(driver_t* d, const hal_interface_t* hal);

// Hard to substitute: hardware baked in
void driver_init(driver_t* d) {
    PERIPHERAL->CTRL = CONFIG_VALUE;  // can't run on host
}
```

**2. Prefer specific accessors over one generic register poke.** A handful of named operations
(`reg_set_bits`, `gpio_write`, `spi_transfer`) are each independently mockable; a single
`reg_write(addr, val)` used everywhere forces conditional logic into the mock and hides which
peripheral a test actually exercises.

---

# Mocking Hardware Dependencies

Strategies for replacing hardware so embedded code runs under test on a host:

1. **Dependency Injection** — pass hardware interfaces as parameters
2. **Function Pointers** — configurable function tables
3. **Link-Time Substitution** — replace entire modules at link time
4. **Register Abstraction** — abstract register access behind functions

## Dependency Injection in C

### Function Pointer Structs (Interface Pattern)
```c
// hal_interface.h
typedef struct {
    uint32_t (*read_register)(uint32_t addr);
    void (*write_register)(uint32_t addr, uint32_t value);
    uint32_t (*get_time_ms)(void);
    void (*delay_ms)(uint32_t ms);
} hal_interface_t;

// module.h
typedef struct {
    const hal_interface_t* hal;
    // ... other state
} module_t;

void module_init(module_t* m, const hal_interface_t* hal);
```

### Production Implementation
```c
// hal_production.c
static uint32_t prod_read_register(uint32_t addr) {
    return *((volatile uint32_t*)addr);
}

static void prod_write_register(uint32_t addr, uint32_t value) {
    *((volatile uint32_t*)addr) = value;
}

static uint32_t prod_get_time_ms(void) {
    return SYSTICK->VAL / (SystemCoreClock / 1000);
}

const hal_interface_t hal_production = {
    .read_register = prod_read_register,
    .write_register = prod_write_register,
    .get_time_ms = prod_get_time_ms,
    .delay_ms = HAL_Delay
};
```

### Test Implementation
```c
// test_hal_mock.c
static uint32_t mock_registers[256];
static uint32_t mock_time_ms = 0;

static uint32_t mock_read_register(uint32_t addr) {
    return mock_registers[addr];
}

static void mock_write_register(uint32_t addr, uint32_t value) {
    mock_registers[addr] = value;
}

static uint32_t mock_get_time_ms(void) {
    return mock_time_ms;
}

const hal_interface_t hal_mock = {
    .read_register = mock_read_register,
    .write_register = mock_write_register,
    .get_time_ms = mock_get_time_ms,
    .delay_ms = NULL  // or stub
};

// Helper functions for tests
void mock_hal_set_register(uint32_t addr, uint32_t value) {
    mock_registers[addr] = value;
}

void mock_hal_set_time(uint32_t ms) {
    mock_time_ms = ms;
}

void mock_hal_reset(void) {
    memset(mock_registers, 0, sizeof(mock_registers));
    mock_time_ms = 0;
}
```

### Using in Tests
```c
void setup(void) {
    mock_hal_reset();
    module_init(&test_module, &hal_mock);
}

void test_module_readsstatusregister(void) {
    mock_hal_set_register(STATUS_REG, 0x42);

    uint32_t status = module_get_status(&test_module);

    TEST_ASSERT_EQUAL_HEX32(0x42, status);
}
```

### Single Function Injection
For simpler cases, inject individual functions:

```c
// timer.h
typedef uint32_t (*get_ticks_fn)(void);

void timer_init(timer_t* t, get_ticks_fn get_ticks);
bool timer_expired(timer_t* t);

// Production
void app_init(void) {
    timer_init(&app_timer, systick_get_ticks);
}

// Test
static uint32_t test_ticks = 0;
uint32_t mock_get_ticks(void) { return test_ticks; }

void test_Timer_Expiration(void) {
    timer_t timer;
    timer_init(&timer, mock_get_ticks);
    timer_start(&timer, 100);

    test_ticks = 99;
    TEST_ASSERT_FALSE(timer_expired(&timer));

    test_ticks = 100;
    TEST_ASSERT_TRUE(timer_expired(&timer));
}
```

## Register Mocking

### Direct Memory-Mapped Registers
Replace direct register access with accessor functions:

```c
// Bad: Direct access (not testable)
void driver_enable(void) {
    PERIPHERAL->CTRL |= CTRL_ENABLE_BIT;
}

// Good: Abstracted access (testable)
// register_access.h
uint32_t reg_read(volatile uint32_t* addr);
void reg_write(volatile uint32_t* addr, uint32_t value);
void reg_set_bits(volatile uint32_t* addr, uint32_t mask);
void reg_clear_bits(volatile uint32_t* addr, uint32_t mask);

// driver.c
void driver_enable(void) {
    reg_set_bits(&PERIPHERAL->CTRL, CTRL_ENABLE_BIT);
}
```

### Mock Register File
```c
// mock_registers.h
#define MAX_MOCK_REGISTERS 64

typedef struct {
    uintptr_t address;
    uint32_t value;
    uint32_t write_count;
    uint32_t read_count;
} mock_register_t;

void mock_reg_init(void);
void mock_reg_set(uintptr_t addr, uint32_t value);
uint32_t mock_reg_get(uintptr_t addr);
uint32_t mock_reg_get_write_count(uintptr_t addr);

// mock_registers.c
static mock_register_t registers[MAX_MOCK_REGISTERS];
static size_t register_count = 0;

uint32_t reg_read(volatile uint32_t* addr) {
    for (size_t i = 0; i < register_count; i++) {
        if (registers[i].address == (uintptr_t)addr) {
            registers[i].read_count++;
            return registers[i].value;
        }
    }
    return 0;
}

void reg_write(volatile uint32_t* addr, uint32_t value) {
    for (size_t i = 0; i < register_count; i++) {
        if (registers[i].address == (uintptr_t)addr) {
            registers[i].value = value;
            registers[i].write_count++;
            return;
        }
    }
    // Add new register
    registers[register_count++] = (mock_register_t){
        .address = (uintptr_t)addr,
        .value = value,
        .write_count = 1
    };
}
```

## Peripheral Mocking

### GPIO Mock
```c
// gpio_mock.h
#define GPIO_MAX_PINS 32

typedef struct {
    uint8_t state[GPIO_MAX_PINS];
    uint8_t direction[GPIO_MAX_PINS];  // 0=input, 1=output
    uint32_t toggle_count[GPIO_MAX_PINS];
} gpio_mock_t;

void gpio_mock_init(void);
void gpio_mock_set_input(uint8_t pin, uint8_t value);
uint8_t gpio_mock_get_output(uint8_t pin);

// gpio.c (mock implementation)
static gpio_mock_t mock_gpio;

void gpio_set_direction(uint8_t pin, gpio_dir_t dir) {
    mock_gpio.direction[pin] = (dir == GPIO_OUTPUT) ? 1 : 0;
}

void gpio_write(uint8_t pin, uint8_t value) {
    mock_gpio.state[pin] = value;
}

uint8_t gpio_read(uint8_t pin) {
    return mock_gpio.state[pin];
}

void gpio_toggle(uint8_t pin) {
    mock_gpio.state[pin] ^= 1;
    mock_gpio.toggle_count[pin]++;
}
```

### SPI Mock with Transaction Recording
```c
// spi_mock.h
typedef struct {
    uint8_t tx_data[256];
    uint8_t rx_data[256];
    size_t tx_len;
    size_t rx_len;
} spi_transaction_t;

typedef struct {
    spi_transaction_t transactions[16];
    size_t transaction_count;
    uint8_t* next_rx_data;
    size_t next_rx_len;
} spi_mock_t;

void spi_mock_init(void);
void spi_mock_set_rx_data(const uint8_t* data, size_t len);
const spi_transaction_t* spi_mock_get_transaction(size_t index);

// spi.c (mock implementation)
static spi_mock_t mock_spi;

spi_status_t spi_transfer(const uint8_t* tx, uint8_t* rx, size_t len) {
    spi_transaction_t* t = &mock_spi.transactions[mock_spi.transaction_count++];

    memcpy(t->tx_data, tx, len);
    t->tx_len = len;

    if (mock_spi.next_rx_data && rx) {
        size_t copy_len = (len < mock_spi.next_rx_len) ? len : mock_spi.next_rx_len;
        memcpy(rx, mock_spi.next_rx_data, copy_len);
        t->rx_len = copy_len;
    }

    return SPI_OK;
}
```

## Timer and Clock Mocking

### Controllable Time Source
```c
// time_mock.h
void time_mock_init(void);
void time_mock_set(uint32_t ms);
void time_mock_advance(uint32_t ms);
uint32_t time_mock_get(void);

// time_mock.c
static uint32_t mock_time_ms = 0;

void time_mock_set(uint32_t ms) {
    mock_time_ms = ms;
}

void time_mock_advance(uint32_t ms) {
    mock_time_ms += ms;
}

// Replaces real implementation
uint32_t get_system_time_ms(void) {
    return mock_time_ms;
}
```

### Testing Timeouts and Delays
```c
void test_watchdog_expiresaftertimeout(void) {
    time_mock_set(0);
    watchdog_init(1000);  // 1 second timeout
    watchdog_start();

    time_mock_advance(999);
    TEST_ASSERT_FALSE(watchdog_expired());

    time_mock_advance(1);
    TEST_ASSERT_TRUE(watchdog_expired());
}

void test_debounce_requiresstableinput(void) {
    time_mock_set(0);
    debounce_t db;
    debounce_init(&db, 50);  // 50ms debounce

    // Input goes high
    debounce_update(&db, 1);
    TEST_ASSERT_EQUAL(0, debounce_get_state(&db));

    // Wait partial debounce time
    time_mock_advance(30);
    debounce_update(&db, 1);
    TEST_ASSERT_EQUAL(0, debounce_get_state(&db));

    // Complete debounce time
    time_mock_advance(20);
    debounce_update(&db, 1);
    TEST_ASSERT_EQUAL(1, debounce_get_state(&db));
}
```

## Interrupt Mocking

### Interrupt Flag Simulation
```c
// interrupt_mock.h
void interrupt_mock_init(void);
void interrupt_mock_trigger(irq_t irq);
bool interrupt_mock_is_pending(irq_t irq);
void interrupt_mock_clear(irq_t irq);

// interrupt_mock.c
static uint32_t pending_interrupts = 0;

void interrupt_mock_trigger(irq_t irq) {
    pending_interrupts |= (1 << irq);
}

bool interrupt_mock_is_pending(irq_t irq) {
    return (pending_interrupts & (1 << irq)) != 0;
}

// Test
void test_uart_rxinterrupt_buffersdata(void) {
    uart_mock_set_rx_byte(0x42);
    interrupt_mock_trigger(UART_RX_IRQ);

    // Simulate interrupt handler call
    uart_rx_handler();

    TEST_ASSERT_EQUAL(1, uart_rx_available());
    TEST_ASSERT_EQUAL(0x42, uart_rx_read());
}
```

### Callback Testing
```c
static int callback_invocations = 0;
static void* callback_context = NULL;

void mock_callback(void* ctx) {
    callback_invocations++;
    callback_context = ctx;
}

void setUp(void) {
    callback_invocations = 0;
    callback_context = NULL;
}

void test_timer_invokescallbackonexpiry(void) {
    int user_data = 42;
    timer_set_callback(mock_callback, &user_data);
    timer_start(100);

    time_mock_advance(100);
    timer_poll();  // Or simulate interrupt

    TEST_ASSERT_EQUAL(1, callback_invocations);
    TEST_ASSERT_EQUAL_PTR(&user_data, callback_context);
}
```

## Link-Time Substitution

### Separate Mock Implementation Files
```
src/
  hal_gpio.c          # Production HAL
  driver.c            # Module under test
test/
  mock_hal_gpio.c     # Mock HAL for testing
  test_driver.c       # Tests
```

### CMake Configuration
```cmake
# Production build
add_library(hal src/hal_gpio.c src/hal_spi.c src/hal_timer.c)
add_executable(firmware src/main.c src/driver.c)
target_link_libraries(firmware hal)

# Test build
add_library(mock_hal test/mock_hal_gpio.c test/mock_hal_spi.c test/mock_hal_timer.c)
add_executable(tests test/test_driver.c src/driver.c)
target_link_libraries(tests mock_hal unity)
```

### Weak Symbol Pattern
```c
// hal.c
__attribute__((weak)) uint32_t hal_get_time_ms(void) {
    return SYSTICK->VAL;  // Real implementation
}

// mock_hal.c (in test build)
static uint32_t mock_time = 0;

uint32_t hal_get_time_ms(void) {
    return mock_time;  // Overrides weak symbol
}
```