# Coding convention

This repository contains embedded firmware. All generated or modified code, including code produced by autonomous coding agents, must follow the rules in this document.

The primary goals are:

1. Correctness
2. Deterministic behavior
3. Memory safety
4. Hardware safety
5. Maintainability
6. Static analyzability
7. Testability

Do not optimize for code brevity at the expense of these properties.

---

# 1. Rule Hierarchy

Apply the following rules in descending order of priority:

1. Project requirements and hardware constraints
2. This `CONTRIBUTION.md`
3. MISRA C:2023
4. Adapted NASA/JPL Power of Ten rules
5. Compiler and static-analysis diagnostics
6. Existing project conventions

If two rules conflict, follow the rule with the higher priority.

Do not modify third-party or vendor code merely to make it conform to these rules unless explicitly requested.

Typical vendor code includes:

* ESP-IDF
* FreeRTOS
* HAL libraries
* device vendor SDKs
* generated register definitions
* external components

Apply strict rules primarily to code owned by this repository.

---

# 2. Language

Unless otherwise specified:

* Use C.
* Prefer C11-compatible constructs supported by the project's toolchain.
* Do not use compiler extensions unless required for hardware, ABI, linker, ISR, DMA, or SDK integration.
* When an extension is required, isolate it and document why it is necessary.

Examples of acceptable implementation-specific constructs include:

```c
IRAM_ATTR
__attribute__((packed))
__attribute__((aligned(...)))
__attribute__((section(...)))
```

when required by the target platform.

---

# 3. MISRA C

MISRA C:2023 is the primary C-language safety baseline.

## 3.1 Mandatory rules

MISRA Mandatory rules shall not be violated by project-owned code.

## 3.2 Required rules

MISRA Required rules shall be followed unless there is a concrete technical reason for deviation.

A deviation must be:

* local
* intentional
* documented
* reviewable
* as narrow as possible

Do not globally disable a MISRA rule to work around a single violation.

## 3.3 Advisory rules

Apply Advisory rules when they improve:

* safety
* readability
* analyzability
* portability
* maintainability

Do not introduce unnecessary complexity solely to satisfy an Advisory rule.

---

# 4. MISRA Deviations

When a MISRA deviation is necessary, document it adjacent to the affected code whenever practical.

Example:

```c
/*
 * MISRA deviation:
 * Integer-to-pointer conversion is required for memory-mapped I/O.
 * Address is defined by the target MCU reference manual.
 */
volatile uint32_t * const reg =
    (volatile uint32_t *)PERIPHERAL_BASE;
```

A valid deviation must explain:

* which behavior requires the deviation
* why the compliant alternative is unsuitable
* why the resulting code remains safe

The following is not sufficient:

```c
/* MISRA exception */
```

Do not hide violations using casts unless the cast itself is semantically correct.

---

# 5. Adapted Power of Ten Rules

The following rules derived from NASA/JPL Power of Ten are mandatory for project-owned firmware unless explicitly waived.

---

## 5.1 No recursion

Recursive function calls are prohibited.

Do not use:

```c
void walk(node_t *node)
{
    if (node != NULL) {
        walk(node->next);
    }
}
```

Use an explicit bounded stack, queue, or iterative algorithm instead.

This applies to both direct and indirect recursion.

---

## 5.2 Dynamic allocation after initialization

Avoid dynamic memory allocation after system initialization.

The following functions or equivalents should not normally be called during steady-state operation:

```c
malloc()
calloc()
realloc()
free()
```

Platform-specific heap APIs are subject to the same rule.

Allocation during initialization may be used when necessary.

Prefer:

* static allocation
* compile-time-sized objects
* fixed-size pools
* caller-provided buffers
* bounded object pools

Exceptions may be made for components whose architecture inherently depends on dynamic allocation, such as selected network stacks or external SDK components.

Do not add new runtime heap dependencies without justification.

---

## 5.3 Bounded loops

Every loop intended to terminate must have a demonstrable bound.

Preferred:

```c
for (size_t i = 0U; i < ARRAY_SIZE(items); ++i) {
    process_item(&items[i]);
}
```

Avoid loops whose termination depends on uncontrolled external state.

Bad:

```c
while (!device_ready()) {
}
```

Use a timeout or maximum iteration count:

```c
for (uint32_t retry = 0U; retry < DEVICE_READY_RETRIES; ++retry) {
    if (device_ready()) {
        ready = true;
        break;
    }

    delay_us(DEVICE_READY_POLL_US);
}
```

Intentional infinite loops are permitted for:

* scheduler loops
* RTOS tasks
* event processors
* idle loops
* top-level firmware main loops

Their purpose must be obvious from the surrounding code.

Example:

```c
for (;;) {
    process_events();
}
```

An intentional infinite loop does not require an artificial iteration limit.

---

## 5.4 Check return values

Return values indicating success, failure, status, bytes transferred, or resource acquisition must be checked.

Bad:

```c
spi_write(dev, data, length);
```

Preferred:

```c
err_t err = spi_write(dev, data, length);

if (err != ERR_OK) {
    return err;
}
```

If a return value is intentionally ignored, make that explicit:

```c
(void)log_flush();
```

Do not silently discard error information.

---

## 5.5 Minimize scope

Declare objects in the narrowest practical scope.

Preferred:

```c
for (uint32_t i = 0U; i < count; ++i) {
    ...
}
```

Avoid unnecessary file-scope state.

Global mutable state should be treated as an exception.

When file-scope state is necessary:

```c
static device_state_t s_device_state;
```

is preferred over externally visible mutable objects.

---

## 5.6 Static analysis and warnings

Project-owned code must compile without unexplained warnings.

Enable strict compiler diagnostics where supported.

For GCC/Clang-based projects, consider:

```text
-Wall
-Wextra
-Werror
-Wshadow
-Wundef
-Wformat=2
-Wdouble-promotion
-Wconversion
-Wsign-conversion
```

Do not blindly enable warnings that produce invalid diagnostics for the target SDK. Configure warning policies separately for project code and third-party code where necessary.

Never fix a warning by adding an arbitrary cast without understanding the underlying type semantics.

---

# 6. Integer Arithmetic

Integer conversions are a major source of embedded C defects.

Avoid implicit conversion between:

* signed and unsigned values
* different integer widths
* enums and arbitrary integers
* floating-point and integer values

Use fixed-width integer types for hardware-facing data:

```c
uint8_t
uint16_t
uint32_t
uint64_t
int8_t
int16_t
int32_t
int64_t
```

Use `size_t` for object sizes and array indices when appropriate.

Use explicit suffixes where relevant:

```c
uint32_t mask = 1UL << bit;
uint32_t timeout_ms = 1000U;
```

Before narrowing a value, ensure it fits the destination type.

Preferred:

```c
if (value <= UINT16_MAX) {
    output = (uint16_t)value;
}
```

Do not rely on integer overflow.

Unsigned wraparound may only be used intentionally and must be obvious.

Signed integer overflow must never be relied upon.

---

# 7. Bit Operations

Bit shifts must be type-safe and within the width of the operand.

Preferred:

```c
#define BIT_U32(n) (UINT32_C(1) << (n))
```

Validate externally supplied shift counts before use.

Do not perform:

```c
1 << bit
```

when the intended type is unsigned 32-bit.

Avoid mixing signed values with masks and register fields.

---

# 8. Pointer Rules

Pointer use must remain simple and analyzable.

Avoid:

* unnecessary pointer arithmetic
* integer/pointer round trips
* incompatible pointer casts
* aliasing tricks
* deeply nested pointer indirection

Prefer no more than two levels of pointer indirection in ordinary application code.

Example:

```c
device_t *dev;
const uint8_t *data;
```

Use higher levels only when required by APIs or data structures.

Never dereference a pointer unless its validity is established by:

* API contract
* object lifetime
* explicit validation
* static construction

---

# 9. Memory-Mapped I/O

Hardware register access is an intentional exception to ordinary portable-C rules.

Use the project's HAL or register definitions where available.

Do not duplicate raw hardware addresses throughout the codebase.

Preferred:

```c
PERIPHERAL->CTRL |= CTRL_ENABLE;
```

over:

```c
*(volatile uint32_t *)0x50000010 = 1U;
```

If direct addressing is unavoidable, isolate it in the hardware abstraction layer.

All MMIO objects must use appropriate `volatile` semantics.

`volatile` is not a concurrency primitive.

Do not use `volatile` as a replacement for:

* atomics
* locks
* critical sections
* memory barriers

---

# 10. Function Design

Functions should have one clearly identifiable responsibility.

Prefer:

* short functions
* shallow nesting
* simple control flow
* explicit failure handling

Guidelines:

* cyclomatic complexity should normally be <= 10
* nesting depth should normally be <= 4
* functions should normally remain below approximately 100 logical lines

These are guidelines rather than arbitrary hard limits.

A large but simple lookup or `switch` function may be preferable to artificially fragmenting the implementation.

Do not split functions merely to satisfy a line-count metric.

---

# 11. Control Flow

Prefer straightforward control flow.

Avoid:

```c
goto
setjmp
longjmp
```

`goto` may be used for centralized cleanup in a function when it substantially simplifies safe resource unwinding.

Example:

```c
err = acquire_a(&a);
if (err != ERR_OK) {
    goto cleanup;
}

err = acquire_b(&b);
if (err != ERR_OK) {
    goto cleanup;
}

...

cleanup:
release_b(b);
release_a(a);

return err;
```

Do not use `goto` to implement ordinary control flow.

---

# 12. Assertions

Assertions are intended to detect programmer errors and violated invariants.

Use assertions for conditions that should be impossible in a correct program.

Example:

```c
assert(ctx != NULL);
assert(ctx->state != STATE_UNINITIALIZED);
```

Do not use assertions for recoverable runtime errors.

Bad:

```c
assert(i2c_read(...) == ESP_OK);
```

Preferred:

```c
esp_err_t err = i2c_read(...);

if (err != ESP_OK) {
    return err;
}
```

Do not add meaningless assertions merely to increase assertion count.

---

# 13. Error Handling

Every operation that can fail must have a defined failure policy.

Possible policies include:

* return the error
* retry with a bounded retry count
* transition the subsystem into a safe state
* restart a peripheral
* restart a task
* trigger a controlled system reset

Never create an unbounded retry loop.

Bad:

```c
while (spi_write(...) != ESP_OK) {
}
```

Preferred:

```c
for (uint32_t attempt = 0U; attempt < SPI_MAX_RETRIES; ++attempt) {
    if (spi_write(...) == ESP_OK) {
        return ESP_OK;
    }
}

return ESP_ERR_TIMEOUT;
```

Do not discard the original failure reason unless intentionally converting it into a defined abstraction-layer error.

---

# 14. Timeouts

Blocking operations involving external state must normally have a timeout.

This includes:

* peripheral waits
* network operations
* queue operations
* semaphore acquisition
* device startup
* hardware-ready polling

Avoid:

```c
wait_forever();
```

unless infinite waiting is explicitly part of the subsystem design.

Use named timeout constants.

Preferred:

```c
#define RADIO_START_TIMEOUT_MS 100U
```

over:

```c
wait_ready(100);
```

---

# 15. Interrupt Service Routines

ISRs must be short and bounded.

An ISR should normally:

1. capture status
2. acknowledge/clear the interrupt
3. store minimal data
4. notify deferred processing
5. return

Do not perform the following in an ISR unless explicitly supported and justified:

* blocking operations
* heap allocation
* long computations
* unbounded loops
* filesystem access
* network operations
* ordinary mutex acquisition
* non-ISR-safe RTOS calls

Use ISR-safe APIs where required by the RTOS.

Shared ISR/task state must have an explicit synchronization policy.

---

# 16. RTOS Rules

For RTOS-based firmware:

* every task must have a defined responsibility
* task stack size must be justified
* task priority must be intentional
* resource ownership must be clear
* blocking waits should normally have timeouts
* mutex acquisition order must remain consistent
* task deletion must not leave owned resources behind

Avoid hidden synchronization through timing assumptions.

Bad:

```c
delay_ms(10);
read_result();
```

Preferred:

```c
wait_for_completion(timeout);
```

Do not assume another task has executed merely because a delay has elapsed.

---

# 17. Locking

Keep critical sections short.

Do not:

* perform blocking I/O while holding a spinlock
* sleep while holding a spinlock
* call unknown callbacks while holding internal locks
* acquire locks in inconsistent order

Document non-trivial lock ordering.

Example:

```text
Lock order:
    system_lock
        -> bus_lock
            -> device_lock
```

Do not reverse this ordering elsewhere.

---

# 18. DMA

DMA buffers must have clearly defined:

* ownership
* lifetime
* alignment
* size
* memory capability
* cache-coherency requirements

Do not free or reuse a buffer while DMA may still access it.

When the platform has a data cache, use the required cache maintenance operations.

Do not assume ordinary memory is DMA-accessible.

Prefer platform-defined DMA allocation APIs when required.

---

# 19. Hardware Safety

Firmware controlling physical hardware must fail safely.

Before modifying code affecting:

* power stages
* RF transmit enable
* motors
* relays
* high-current outputs
* power converters
* battery systems
* clocks
* reset signals

identify the safe hardware state.

On initialization, error, timeout, watchdog recovery, and shutdown, safety-critical outputs must transition to their defined safe state whenever technically possible.

Do not enable an output before all required configuration and validation has completed.

---

# 20. Preprocessor

Use the preprocessor conservatively.

Acceptable uses include:

* header guards
* conditional platform configuration
* compile-time constants
* hardware definitions
* simple function-like macros when necessary

Prefer:

```c
static inline uint32_t make_mask(uint32_t bit)
{
    return UINT32_C(1) << bit;
}
```

over a complex macro when both are viable.

Macros must parenthesize parameters and expressions correctly.

Avoid macros that:

* evaluate arguments multiple times
* hide control flow
* declare unexpected variables
* contain large algorithmic implementations
* depend on undocumented external state

Avoid:

```c
#define MAX(a, b) ((a) > (b) ? (a) : (b))
```

when arguments may have side effects.

---

# 21. Global State

Avoid externally visible mutable global variables.

Prefer:

```c
static module_state_t s_state;
```

with accessor functions over:

```c
module_state_t g_state;
```

When state is shared between execution contexts, document how it is synchronized.

Do not introduce singleton/global state merely because it is convenient.

---

# 22. Initialization

Initialize every object before use.

Prefer explicit initialization:

```c
device_config_t config = {
    .frequency_hz = 1000000U,
    .mode = DEVICE_MODE_0,
};
```

Do not assume stack objects are zero-initialized.

For structures whose future fields may be added, designated initializers are preferred.

---

# 23. Constants

Avoid unexplained numeric literals.

Bad:

```c
if (retry > 7U) {
```

Preferred:

```c
#define MAX_START_RETRIES 7U

if (retry > MAX_START_RETRIES) {
```

Use units in names:

```c
timeout_ms
frequency_hz
voltage_mv
current_ma
period_us
```

Do not encode unit information only in comments.

---

# 24. API Design

Public APIs should:

* validate externally supplied arguments
* document ownership
* document lifetime requirements
* document valid ranges
* return meaningful errors
* avoid exposing internal implementation details

Prefer explicit context objects:

```c
esp_err_t radio_start(radio_handle_t *radio);
```

over hidden global state:

```c
esp_err_t radio_start(void);
```

when multiple instances may reasonably exist.

---

# 25. Buffer APIs

Functions receiving buffers should normally receive their size explicitly.

Preferred:

```c
err_t packet_encode(
    uint8_t *dst,
    size_t dst_size,
    const packet_t *packet);
```

Avoid APIs that rely on undocumented fixed buffer sizes.

Before every write, the implementation must be able to prove sufficient capacity exists.

---

# 26. Strings

Do not use unbounded string functions.

Avoid:

```c
strcpy()
strcat()
sprintf()
gets()
```

Use bounded alternatives and verify truncation behavior.

When possible, avoid manipulating strings entirely in low-level firmware paths.

---

# 27. Resource Ownership

For every resource, ownership must be clear.

Resources include:

* allocated memory
* DMA buffers
* peripheral handles
* mutexes
* semaphores
* file descriptors
* sockets
* timers
* event objects

The code must make it possible to determine:

* who creates the resource
* who owns it
* who may access it
* who destroys it
* when destruction is safe

---

# 28. Cleanup

Functions acquiring multiple resources must correctly release already-acquired resources when later operations fail.

Every failure path must be reviewed for:

* memory leaks
* lock leaks
* peripheral state leaks
* dangling callbacks
* DMA ownership violations

Do not implement only the success path.

---

# 29. Logging

Logging must not change correctness.

Do not depend on logging timing.

Avoid excessive logging in:

* ISR context
* timing-critical paths
* high-frequency loops
* RF/audio sample paths

Do not log secrets or sensitive binary data.

Use appropriate log levels.

---

# 30. Comments

Comments should explain:

* why something is necessary
* hardware constraints
* unusual invariants
* concurrency assumptions
* MISRA deviations
* non-obvious timing constraints

Do not write comments that merely restate the code.

Bad:

```c
/* increment i */
++i;
```

Useful:

```c
/*
 * Hardware requires at least 10 us between reset release
 * and the first SPI transaction.
 */
delay_us(10U);
```

---

# 31. Naming

Names should communicate purpose and units.

Prefer:

```c
sample_rate_hz
timeout_ms
rx_buffer_size
is_initialized
```

Avoid:

```c
x
tmp2
val
data1
```

except for very narrow local contexts where the meaning is obvious.

Boolean variables should preferably read as predicates:

```c
is_ready
has_error
should_retry
```

---

# 32. Tests

Code changes should include tests whenever the behavior can reasonably be tested.

At minimum test:

* normal behavior
* boundary conditions
* invalid arguments
* failure paths
* timeout behavior
* state transitions

For hardware-independent logic, prefer host-side unit tests.

Hardware-dependent features may require:

* mocks
* fakes
* HAL abstraction
* hardware-in-the-loop tests

Do not distort production architecture solely to satisfy a test framework.

---

# 33. Boundary Values

Explicitly test boundary values around:

* zero
* minimum
* maximum
* buffer boundaries
* timeout limits
* counter wraparound
* enum boundaries
* frequency/rate ranges
* hardware register field limits

For a permitted interval:

```text
MIN <= x <= MAX
```

tests should normally include:

```text
MIN - 1
MIN
MIN + 1
MAX - 1
MAX
MAX + 1
```

when representable.

---

# 34. Static Analysis

Run available static-analysis tools on modified project-owned code.

Fix genuine defects rather than suppressing the warning.

A suppression is acceptable only when:

1. the diagnostic is understood
2. the behavior is intentional
3. the suppression is as narrow as possible
4. the reason is documented

Do not globally suppress categories simply to obtain a clean report.

---

# 35. Generated Code

Do not manually modify generated code unless explicitly required.

Modify the generator, source description, or configuration instead whenever possible.

Generated files should be clearly identifiable.

Generated code may be excluded from project style rules when appropriate.

---

# 36. Third-Party Code

Do not refactor third-party code merely for style consistency.

Keep vendor modifications minimal because they increase:

* merge cost
* upgrade cost
* regression risk

If third-party code contains a defect that must be fixed, isolate the patch and document it.

---

# 37. Agent Workflow

An autonomous coding agent must follow this sequence for non-trivial changes.

## Step 1 — Understand

Before editing:

* identify the requested behavior
* locate relevant implementation
* identify interfaces and call sites
* identify hardware or RTOS constraints
* inspect existing tests
* inspect existing project conventions

Do not implement based solely on filenames or assumptions.

## Step 2 — Plan

For non-trivial modifications, establish internally:

* affected modules
* API changes
* state changes
* error paths
* concurrency implications
* test strategy

Prefer the smallest correct change.

## Step 3 — Implement

During implementation:

* preserve existing architecture unless a change is necessary
* avoid unrelated refactoring
* keep diffs focused
* follow this document
* preserve backward compatibility unless requirements explicitly change it

## Step 4 — Verify

After editing:

1. build the relevant target
2. run relevant unit tests
3. run static analysis if available
4. inspect compiler warnings
5. inspect the final diff

Do not report success unless the performed verification actually succeeded.

## Step 5 — Report

The final report should state:

* what changed
* why it changed
* files materially affected
* tests/builds performed
* static analysis performed
* known limitations
* anything that could not be verified

Do not claim hardware validation unless the change was actually exercised on hardware.

---

# 38. Agent Editing Rules

Agents must not:

* rewrite unrelated code
* reformat entire files without need
* rename public APIs unnecessarily
* remove error handling to simplify implementation
* silence compiler warnings without understanding them
* change hardware constants based on guesses
* invent undocumented register values
* invent SDK APIs
* assume an API exists without checking the installed version
* silently change behavior outside the requested scope

When uncertain about an SDK or hardware API, inspect the project's actual headers, source, documentation, or dependency version before implementing.

Existing code is evidence, not automatically truth.

---

# 39. Do Not Guess Hardware Behavior

Never infer hardware register semantics solely from names.

Before changing register-level code, verify relevant information from one or more of:

* MCU technical reference manual
* datasheet
* SDK headers
* SDK implementation
* vendor documentation
* board schematic

Pay particular attention to:

* write-one-to-clear bits
* read side effects
* reserved bits
* reset values
* clock dependencies
* peripheral synchronization
* interrupt ordering
* cache requirements

---

# 40. Concurrency Review

Any code accessed from more than one execution context must be reviewed for concurrency.

Execution contexts include:

* multiple RTOS tasks
* ISR and task
* multiple CPU cores
* DMA and CPU
* callbacks and owner task

For shared state, determine whether synchronization requires:

* immutable data
* atomic access
* critical section
* spinlock
* mutex
* queue
* event
* ownership transfer

`volatile` alone is not sufficient synchronization.

---

# 41. Timing-Critical Code

Do not add unpredictable execution time to timing-critical code.

Avoid in such paths:

* dynamic allocation
* unbounded iteration
* filesystem access
* network operations
* verbose logging
* blocking mutex waits

When performance matters, optimize only after correctness is established.

Document hard real-time or latency requirements when known.

---

# 42. State Machines

Complex asynchronous firmware should prefer explicit state machines over combinations of loosely related boolean flags.

Preferred:

```c
typedef enum {
    RADIO_STATE_OFF,
    RADIO_STATE_STARTING,
    RADIO_STATE_READY,
    RADIO_STATE_ERROR,
} radio_state_t;
```

over:

```c
bool enabled;
bool initialized;
bool starting;
bool failed;
```

when mutually exclusive states are intended.

State transitions must be explicit and valid.

---

# 43. Error-State Safety

A failure should not leave hardware in an ambiguous state.

For hardware control functions, consider failure at every intermediate step.

Example:

```text
enable clock
configure peripheral
enable amplifier
start transfer
```

If configuration fails after the clock is enabled, determine whether the clock must be disabled again.

If transmission setup fails, TX-related outputs must remain or return disabled.

---

# 44. Watchdog

The watchdog is a last-resort fault recovery mechanism, not normal control flow.

Do not intentionally allow watchdog reset instead of handling a known recoverable error.

Tasks responsible for feeding a watchdog must do so only after confirming required work is progressing correctly.

Avoid feeding a watchdog from an unrelated timer solely to prevent reset.

---

# 45. Commit Scope

Each change should represent one coherent purpose.

Do not combine:

* feature implementation
* unrelated refactoring
* broad formatting
* dependency upgrades

unless they are inseparable.

A reviewer must be able to determine why each changed line is necessary.

---

# 46. Definition of Done

A firmware change is complete only when all applicable conditions are satisfied:

* [ ] Requirements are implemented.
* [ ] Project code complies with this document.
* [ ] Relevant MISRA rules were considered.
* [ ] No unintended runtime heap dependency was introduced.
* [ ] No recursion was introduced.
* [ ] Terminating loops are bounded.
* [ ] Error-return values are handled.
* [ ] External waits have appropriate timeouts.
* [ ] ISR behavior remains bounded.
* [ ] Concurrency implications were reviewed.
* [ ] DMA ownership/cache implications were reviewed where applicable.
* [ ] Failure paths leave hardware in a valid state.
* [ ] Compiler warnings were reviewed.
* [ ] Relevant tests pass.
* [ ] Relevant build succeeds.
* [ ] Static analysis was run when available.
* [ ] Final diff contains no unrelated changes.
* [ ] Remaining limitations are explicitly reported.

---

# 47. Core Principle

When multiple implementations satisfy the requirement, prefer the implementation that is:

1. easier to reason about
2. easier to statically analyze
3. bounded in memory and execution behavior
4. explicit about ownership and failure
5. simpler to test
6. less dependent on hidden state

Firmware should behave predictably not only when everything works, but also when hardware, timing, memory, or external inputs fail.
