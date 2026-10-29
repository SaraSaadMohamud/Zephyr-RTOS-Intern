# Testing with Zephyr — Unit Testing

## 1. Introduction

Testing is an essential part of embedded software development. It helps detect bugs early, verify component behavior, and prevent regressions when the code changes.

Zephyr RTOS provides **Ztest**, a unit testing framework, and **Twister**, a test runner that discovers, builds, and executes tests across supported platforms.

### Testing Levels

| Testing Type                | Scope                           | Purpose                                                      |
| --------------------------- | ------------------------------- | ------------------------------------------------------------ |
| Unit Testing                | One module in isolation         | Verify individual functions and components.                  |
| Integration Testing         | Two or more components together | Verify that components work correctly together.              |
| System / End-to-End Testing | The whole device or application | Verify complete system behavior, including real hardware and I/O when required. |

### Why Unit Testing?

- **Catch bugs early:** Fixing defects at the unit level is usually cheaper than fixing them later.
- **Prevent regressions:** Verify that existing functionality continues to work after code changes.
- **Automate verification:** Run tests on every commit and integrate them into CI pipelines.
- **Reduce hardware dependency:** Run suitable unit tests on emulated or native platforms without requiring a physical device.

> A passing test suite provides evidence that the tested behaviors work as expected. It does not guarantee that the entire system is free of defects.

## 2. The Module Under Test: Calculator

This module implements a simple calculator with a last-result cache.

The module exposes public APIs for arithmetic operations and retrieving or resetting the cached result.

### Header File

**File:** `app/modules/calculator/include/calculator.h`

```
void calculator_reset(void);

int calculator_get_last_result(int *result);

int calculator_add(int a, int b, int *result);

int calculator_sub(int a, int b, int *result);

int calculator_mul(int a, int b, int *result);

int calculator_div(int a, int b, int *result);
```

### API Design

The calculator follows a consistent return-value convention:

- `0`: Operation completed successfully.
- Negative errno: Operation failed.
- `result`: Pointer used to return the calculated value.
- Last-result cache: Stores the most recent successful result, according to the module's implementation.

For example:

```
int result;

int status = calculator_add(2, 3, &result);

if (status == 0) {
    /* result contains 5 */
}
```

**Important:** The test suite should verify both successful operations and failure behavior, including whether output parameters and cached state remain unchanged after errors when that is the intended API contract.

## 3. Test Environment: Required Files

A typical test directory for this module looks like this:

```
tests/
└── calculator/
    ├── CMakeLists.txt
    ├── prj.conf
    ├── testcase.yaml
    └── src/
        └── test_basic.c
```

### File Responsibilities

| File               | Responsibility                                               |
| ------------------ | ------------------------------------------------------------ |
| `CMakeLists.txt`   | Configures the test build and links the test code with the calculator implementation. |
| `prj.conf`         | Enables Ztest and the required configuration options.        |
| `testcase.yaml`    | Defines test scenarios and metadata used by Twister.         |
| `src/test_basic.c` | Contains test suites, test cases, and assertions.            |

The test build must compile both the test source and the module being tested.

## 4. CMake Configuration

**File:** `tests/calculator/CMakeLists.txt`

```
cmake_minimum_required(VERSION 3.20.0)

find_package(Zephyr REQUIRED HINTS $ENV{ZEPHYR_BASE})

project(calculator_unit_test)

target_sources(app PRIVATE
    src/test_basic.c
    ${CMAKE_CURRENT_SOURCE_DIR}/../../app/modules/calculator/src/calculator.c
)

target_include_directories(app PRIVATE
    ${CMAKE_CURRENT_SOURCE_DIR}/../../app/modules/calculator/include
)
```

### Explanation

- `find_package(Zephyr ...)` loads the Zephyr build system.
- `project()` defines the test application's project name.
- `target_sources()` adds the test source and calculator implementation.
- `target_include_directories()` makes `calculator.h` available to the test code.

**Note:** The relative paths assume the directory structure shown in this README. Adjust them if your repository uses a different layout.

## 5. Ztest Configuration

**File:** `tests/calculator/prj.conf`

```
CONFIG_ZTEST=y
CONFIG_ZTEST_SHUFFLE=y
```

### Configuration Options

| Option                   | Purpose                                                      |
| ------------------------ | ------------------------------------------------------------ |
| `CONFIG_ZTEST=y`         | Enables Zephyr's Ztest framework.                            |
| `CONFIG_ZTEST_SHUFFLE=y` | Enables randomized test ordering where supported by the configured Ztest environment. |

Test shuffling helps reveal hidden dependencies between test cases.

For example, a test may pass when executed after `calculator_reset()` but fail when another test modifies the cached result first.

Each test should establish its own required initial state rather than depending on another test's execution order.

## 6. Ztest Test Suites

Ztest organizes tests into suites and individual test cases.

A suite groups tests that exercise related functionality, such as addition, division, or error handling.

### Registering a Test Suite

The general syntax is:

```
ZTEST_SUITE(
    suite_name,
    predicate,
    setup,
    before,
    after,
    teardown
);
```

### The Six Arguments

| Argument     | Purpose                                          | When It Runs                                      |
| ------------ | ------------------------------------------------ | ------------------------------------------------- |
| `suite_name` | Identifies the test suite.                       | Used to register the suite.                       |
| `predicate`  | Determines whether the suite should run.         | Evaluated when deciding whether to run the suite. |
| `setup`      | Creates or initializes shared test-suite state.  | Once before the suite's tests, when applicable.   |
| `before`     | Prepares the environment for an individual test. | Before each test.                                 |
| `after`      | Performs per-test cleanup.                       | After each test.                                  |
| `teardown`   | Releases suite-level resources.                  | After the suite finishes, when applicable.        |

The setup, before, after, and teardown callbacks receive a fixture pointer. The fixture can be used to share test data between callbacks and test cases.

For suites that do not require fixtures or callbacks, pass `NULL`.

## 7. A Minimal Test Suite

**File:** `tests/calculator/src/test_basic.c`

```
#include <zephyr/ztest.h>

#include "calculator.h"

ZTEST_SUITE(calculator_add, NULL, NULL, NULL, NULL, NULL);

ZTEST(calculator_add, test_positive_numbers)
{
    int result;

    zassert_ok(
        calculator_add(2, 3, &result),
        "Addition should succeed"
    );

    zassert_equal(
        result,
        5,
        "2 + 3 should equal 5"
    );
}
```

### How It Works

1. Include `zephyr/ztest.h` to access Ztest macros.
2. Include `calculator.h` to access the module's public APIs.
3. Register the `calculator_add` suite using `ZTEST_SUITE()`.
4. Register `test_positive_numbers` using `ZTEST()`.
5. Use assertions to verify the return status and calculated value.

### Registering a Test Case

```
ZTEST(suite_name, test_name)
{
    /* Arrange, act, and assert */
}
```

Each test case should have a descriptive name that explains the behavior being verified.

## 8. The `before()` Hook

The calculator maintains a last-result cache. This introduces shared state that can affect multiple tests.

To make tests independent, reset the calculator before each test.

```
static void before(void *f)
{
    ARG_UNUSED(f);

    calculator_reset();
}
```

Register the callback with the suite:

```
ZTEST_SUITE(
    calculator_add,
    NULL,
    NULL,
    before,
    NULL,
    NULL
);

ZTEST_SUITE(
    calculator_last_result,
    NULL,
    NULL,
    before,
    NULL,
    NULL
);

ZTEST_SUITE(
    calculator_edge_cases,
    NULL,
    NULL,
    before,
    NULL,
    NULL
);
```

### Why Use `before()`?

- It runs before each test in the suite.
- It resets shared state to a known starting condition.
- It prevents one test from affecting another.
- It makes tests more reliable when their execution order changes.

`ARG_UNUSED(f)` tells the compiler that the callback does not use its fixture argument.

> Use `before()` for per-test initialization. Use `setup()` and `teardown()` for resources whose lifetime spans the suite.

## 9. Organizing Multiple Suites

A practical approach is to create one suite for each public function or related group of behaviors.

```
ZTEST_SUITE(calculator_add, NULL, NULL, before, NULL, NULL);

ZTEST(calculator_add, test_positive_numbers)
{
    /* Verify positive operands. */
}

ZTEST(calculator_add, test_negative_numbers)
{
    /* Verify negative operands. */
}

ZTEST_SUITE(calculator_div, NULL, NULL, before, NULL, NULL);

ZTEST(calculator_div, test_exact_division)
{
    /* Verify exact division. */
}

ZTEST(calculator_div, test_divide_by_zero)
{
    /* Verify division-by-zero handling. */
}

ZTEST(calculator_div, test_negative_dividend_truncation)
{
    /* Verify the documented integer-division behavior. */
}
```

These examples illustrate suite organization; each test body must contain its actual assertions before it can verify the behavior.

### Suggested Test Suites

| Suite                    | Behaviors to Test                                            |
| ------------------------ | ------------------------------------------------------------ |
| `calculator_add`         | Positive, negative, and zero operands; overflow handling if specified. |
| `calculator_sub`         | Positive and negative results; zero operands.                |
| `calculator_mul`         | Positive and negative operands; multiplication by zero.      |
| `calculator_div`         | Exact division, negative operands, division by zero.         |
| `calculator_last_result` | Cache updates, retrieval, reset, and invalid pointers.       |
| `calculator_edge_cases`  | Boundary values, invalid arguments, and chained operations.  |

Multiple suites can exist in one source file. They can also be distributed across multiple source files, provided those files are linked into the same test binary.

## 10. Three Families of Ztest Assertions

Ztest provides three important assertion families:

- `zassert_*`
- `zexpect_*`
- `zassume_*`

They share many assertion suffixes but have different behaviors when a condition fails.

### Comparison

| Family      | Behavior on Failure                                          | Best Use                                                     |
| ----------- | ------------------------------------------------------------ | ------------------------------------------------------------ |
| `zassert_*` | Fails the current test immediately.                          | When subsequent steps depend on the failed condition.        |
| `zexpect_*` | Records the failure and continues executing the test.        | When several independent checks should be reported together. |
| `zassume_*` | Aborts the current test as skipped when the assumption fails. | When a required precondition is not satisfied.               |

### Choosing the Right Assertion

**Use** `**zassert_\***` **when:**

A failed condition makes subsequent operations meaningless or unsafe.

**Use** `**zexpect_\***` **when:**

You want to collect multiple failures in one test instead of stopping at the first one.

**Use** `**zassume_\***` **when:**

The test cannot meaningfully proceed because a required precondition is missing.

An assumption should not replace an assertion for the actual behavior being tested. Otherwise, a genuine defect could be hidden by skipping the test.

## 11. Assertion Suffix Reference

The following table covers commonly used assertion suffixes.

| Suffix             | Expected Condition |
| ------------------ | ------------------ |
| `_ok(expr)`        | `expr == 0`        |
| `_not_ok(expr)`    | `expr != 0`        |
| `_equal(a, b)`     | `a == b`           |
| `_not_equal(a, b)` | `a != b`           |
| `_true(cond)`      | `cond` is true.    |
| `_false(cond)`     | `cond` is false.   |
| `_not_null(ptr)`   | `ptr != NULL`      |
| `_is_null(ptr)`    | `ptr == NULL`      |

The assertion family determines how a failure affects execution. The suffix determines the condition being checked.

For example:

```
zassert_ok(status, "Operation should succeed");

zexpect_equal(result, 5, "Unexpected result");

zassume_not_null(ptr, "A valid pointer is required");
```

Consult the official [Zephyr documentation](https://docs.zephyrproject.org/latest/develop/test/ztest.html) for the complete assertion API and the exact behavior supported by your Zephyr version.

## 12. Using `zassert_*` in Practice

### Example 1: Successful Addition

```
ZTEST(calculator_add, test_positive_numbers)
{
    int result;

    zassert_ok(
        calculator_add(2, 3, &result),
        "Addition should succeed"
    );

    zassert_equal(
        result,
        5,
        "2 + 3 should equal 5"
    );
}
```

The first assertion checks the API return status.

The second assertion checks the calculated result.

If the API call fails, the test stops before checking the output value.

### Example 2: Division by Zero

```
ZTEST(calculator_div, test_divide_by_zero)
{
    int result = 999;

    zassert_equal(
        calculator_div(10, 0, &result),
        -EINVAL,
        "Division by zero should return -EINVAL"
    );

    zassert_equal(
        result,
        999,
        "Result should not be modified on error"
    );
}
```

This test checks two behaviors:

1. Division by zero returns `-EINVAL`.
2. The output parameter remains unchanged after the failed operation.

The second check is appropriate only if preserving the output parameter on error is part of the calculator API contract.

The `EINVAL` error constant is provided by the standard errno definitions used by Zephyr. Include the appropriate header if your source file requires it explicitly.

**Important:** The test assumes that the calculator implementation actually returns `-EINVAL` and preserves the output value on failure. Confirm those behaviors in the implementation or its API contract.

## 13. Combining `zexpect_*` and `zassume_*`

Consider a chained-operation test that performs addition, subtraction, multiplication, and division.

```
ZTEST(calculator_edge_cases, test_chained_operations)
{
    int result;

    zassume_ok(
        calculator_add(10, 20, &result),
        "Precondition: addition must succeed"
    );

    zexpect_equal(
        result,
        30,
        "Expected 10 + 20 = 30, got %d",
        result
    );

    zexpect_ok(
        calculator_sub(result, 5, &result),
        "Subtraction should succeed"
    );

    zexpect_equal(
        result,
        25,
        "Expected 30 - 5 = 25, got %d",
        result
    );

    zexpect_ok(
        calculator_mul(result, 2, &result),
        "Multiplication should succeed"
    );

    zexpect_equal(
        result,
        50,
        "Expected 25 * 2 = 50, got %d",
        result
    );

    zexpect_ok(
        calculator_div(result, 10, &result),
        "Division should succeed"
    );

    zexpect_equal(
        result,
        5,
        "Expected 50 / 10 = 5, got %d",
        result
    );
}
```

### Expected Calculation

| Step | Operation | Expected Result |
| ---- | --------- | --------------- |
| 1    | `10 + 20` | 30              |
| 2    | `30 - 5`  | 25              |
| 3    | `25 * 2`  | 50              |
| 4    | `50 / 10` | 5               |

### Why Use These Assertions?

- `zassume_ok()` checks that the initial operation succeeds. If it fails, the test is skipped because the required precondition was not established.
- `zexpect_ok()` records operation failures without immediately stopping the test.
- `zexpect_equal()` records incorrect intermediate or final results.

**Testing consideration:** Chained operations depend on the previous operation's output. If an operation fails, continuing the chain may produce misleading results. For a stricter dependency-aware test, use `zassert_ok()` before checking the corresponding output, or structure independent checks as separate test cases.

Also, `zexpect_ok()` does not guarantee that the output is valid. The test should not blindly use an output parameter after a failed operation.

## 14. Test Discovery with Twister

Twister is Zephyr's test runner. It discovers test scenarios, configures builds, compiles applications for selected platforms, executes tests where supported, and reports results.

The `testcase.yaml` file provides metadata that allows Twister to discover and select the test.

A minimal example is:

```
tests:
  example.unit.calculator.ordered:
    tags:
      - unit
      - calculator
    integration_platforms:
      - native_sim
```

### Configuration Notes

- `example.unit.calculator.ordered` is the test scenario identifier.
- `tags` categorizes the test.
- `integration_platforms` specifies platforms on which the scenario should be built and executed as an integration test.

Use metadata appropriate to your repository and Zephyr version. The platform must be supported by the selected test configuration.

### Running the Test

From the Zephyr workspace or repository directory:

```
west twister -T tests/calculator -p native_sim -v
```

This command:

- `-T tests/calculator`: Searches for tests in the specified directory.
- `-p native_sim`: Selects the native simulation platform.
- `-v`: Enables verbose output.

To select one scenario by name:

```
west twister \
    -T tests/calculator \
    -p native_sim \
    -s example.unit.calculator.ordered \
    -v
```

**Note:** The scenario name passed to `-s` must match the name declared in `testcase.yaml`.

If your test directory is `tests/calculator`, use that path with `-T`. If your repository uses another directory structure, update the command accordingly.

For additional options, see the official [Zephyr Twister documentation](https://docs.zephyrproject.org/latest/develop/test/twister.html).

## 15. Recommended Testing Workflow

A reliable workflow for unit testing with Zephyr is:

1. Identify a module and its public APIs.
2. Define the expected behavior of each API.
3. Create a dedicated test directory.
4. Configure the test build in `CMakeLists.txt`.
5. Enable Ztest and required features in `prj.conf`.
6. Define the test scenario in `testcase.yaml`.
7. Write test suites and test cases.
8. Reset shared state between tests.
9. Verify both successful and unsuccessful API calls.
10. Run tests with Twister.
11. Fix failures and rerun the suite.
12. Integrate the tests into CI so they execute regularly.

### Example Test Checklist

- Positive arithmetic operations.
- Negative operands and results.
- Zero operands.
- Division by zero.
- Invalid pointers, if supported by the API contract.
- Integer boundary and overflow behavior, where specified.
- Last-result cache updates.
- Last-result cache reset.
- Output parameter behavior on errors.
- Independence between test cases.
- Successful execution under randomized test ordering, where enabled.

## 16. Key Takeaways

- **Unit testing** verifies one module in isolation.
- **Ztest** provides test suites, test cases, hooks, and assertion macros.
- **Twister** discovers, builds, and executes Zephyr test scenarios.
- `CMakeLists.txt` connects the test code with the module implementation.
- `prj.conf` enables the required Zephyr features.
- `testcase.yaml` describes test scenarios for Twister.
- `before()` is useful for resetting shared state before each test.
- `zassert_*` stops a test on failure.
- `zexpect_*` records failures and continues.
- `zassume_*` skips a test when a required precondition is not met.
- Tests should be deterministic, independent, and automated whenever practical.

The goal is not simply to make tests pass. It is to build confidence that the module behaves according to its documented contract and continues to do so as development progresses.

---

# Part2: Testing with Zephyr — Twister Test Runner

## 1. Introduction

**Twister** is Zephyr RTOS's central test runner and test orchestration tool. It automates the process of discovering, building, executing, and evaluating tests across different platforms.

Twister helps developers validate embedded software consistently on their local machines and in Continuous Integration (CI) pipelines.

### What Is Twister?

Twister provides the following capabilities:

- **Test discovery:** Recursively discovers test scenarios described in `testcase.yaml` and supported sample metadata.
- **Platform selection:** Selects suitable platforms for each test scenario.
- **Automated building:** Builds test scenarios for their matching platforms, with parallel execution where possible.
- **Test execution:** Runs tests on native hosts, simulators, emulators, or real hardware.
- **Result evaluation:** Uses configured test harnesses to interpret output and determine whether tests pass or fail.
- **Reporting:** Produces machine-readable JSON, JUnit XML, logs, and optional code coverage reports.

The same Twister command-line interface can be used locally and in CI, making automated testing easier to integrate into development workflows.

## 2. The Twister Pipeline

Twister follows a sequence of stages to execute a test suite.

```
        Test Metadata
   testcase.yaml / sample.yaml
               |
               v
       +----------------+
       |   Discovery    |
       +----------------+
               |
               v
       +----------------+
       |    Filtering   |
       +----------------+
               |
               v
       +----------------+
       |    Building    |
       +----------------+
               |
               v
       +----------------+
       |   Execution    |
       +----------------+
               |
               v
       +----------------+
       |   Evaluation   |
       +----------------+
               |
               v
       +----------------+
       |   Reporting    |
       +----------------+
               |
               v
       JSON / JUnit XML / Logs
         / Coverage Data
```

### Pipeline Stages

| Stage      | Description                                                  |
| ---------- | ------------------------------------------------------------ |
| Discovery  | Finds test scenarios by scanning supported metadata files.   |
| Filtering  | Applies platform, architecture, tag, and Kconfig constraints, along with other selection criteria. |
| Building   | Compiles each selected scenario for its matching platform. Builds may run in parallel. |
| Execution  | Runs the compiled test on a native host, simulator, emulator, or physical device. |
| Evaluation | Uses the configured harness to interpret the test output and determine the result. |
| Reporting  | Collects results and generates machine-readable reports and logs. |

### Why This Pipeline Matters

Automating these stages reduces repetitive manual work and makes test results more consistent.

For example, a unit test can run on `native_sim` during development without flashing a physical board. Hardware-specific tests can then run on a development kit during later validation.

## 3. Test Platforms

Twister supports different types of test platforms, each suited to particular testing requirements.

| Platform       | What You Get                                                 | When to Use It                                               |
| -------------- | ------------------------------------------------------------ | ------------------------------------------------------------ |
| `native_sim`   | Zephyr executes as a native Linux application using its supported simulation environment. | A practical default for many unit tests.                     |
| `unit_testing` | A specialized environment for testing code with minimal operating-system dependencies. | Pure utilities and components that do not require a complete Zephyr kernel environment. |
| QEMU           | Emulated hardware for supported architectures and boards.    | Testing architecture-specific behavior without physical hardware. |
| Renode         | Virtualized embedded platforms and peripherals for supported targets. | Integration testing and simulated hardware interactions.     |
| Real hardware  | An actual development board and its peripherals.             | Final validation, hardware abstraction layer (HAL) coverage, and physical I/O testing. |

### Choosing a Platform

**Use** `**native_sim**` **when:**

- Testing application logic or individual modules.
- You want to avoid flashing physical hardware.
- You need a fast feedback loop during development.

**Use** `**unit_testing**` **when:**

- The component can be tested in a minimal environment.
- You want to isolate utilities from full operating-system behavior.

**Use QEMU or Renode when:**

- You need to exercise an emulated target architecture.
- The test depends on supported simulated hardware or peripherals.

**Use real hardware when:**

- Testing physical peripherals or electrical behavior.
- Validating hardware-specific drivers.
- Verifying behavior that cannot be adequately represented by simulation.

> A useful strategy is to run fast unit tests frequently, then run hardware-dependent tests when appropriate.

## 4. `testcase.yaml` Anatomy

The `testcase.yaml` file describes test scenarios and the conditions under which they should be built and executed.

Consider this example:

```
common:
  tags:
    - unit
    - calculator
    - ztest

tests:
  example.unit.calculator:
    platform_allow:
      - native_sim
    timeout: 60
```

### Understanding the Structure

#### `common`

Defines configuration shared by scenarios in the same file.

```
common:
  tags:
    - unit
    - calculator
    - ztest
```

These tags help categorize tests and make it easier to select them using Twister's filtering options.

#### `tests`

Contains the individual test scenario definitions.

```
tests:
  example.unit.calculator:
    platform_allow:
      - native_sim
    timeout: 60
```

The scenario name is:

```
example.unit.calculator
```

The configuration specifies:

- `platform_allow`: Allows the scenario to run on `native_sim`.
- `timeout`: Sets the test execution timeout to 60 seconds.

The exact supported fields and their behavior depend on the Zephyr version and test configuration.

## 5. Multiple Test Scenarios

A single `testcase.yaml` file can describe multiple scenarios.

For example:

```
common:
  tags:
    - unit
    - calculator
    - ztest

tests:
  example.unit.calculator:
    platform_allow:
      - native_sim
    timeout: 60

  example.unit.calculator.ordered:
    platform_allow:
      - native_sim
    timeout: 60
    extra_args:
      - CONFIG_ZTEST_SHUFFLE=n

  example.unit.calculator.hw:
    platform_allow:
      - nrf54l15dk/nrf54l15/cpuapp
    timeout: 120
```

### Scenario Breakdown

| Scenario                          | Platform                     | Purpose                                             |
| --------------------------------- | ---------------------------- | --------------------------------------------------- |
| `example.unit.calculator`         | `native_sim`                 | General calculator unit testing.                    |
| `example.unit.calculator.ordered` | `native_sim`                 | Demonstrates testing with Ztest shuffling disabled. |
| `example.unit.calculator.hw`      | `nrf54l15dk/nrf54l15/cpuapp` | Hardware-specific test scenario.                    |

### Important Notes

**1. Test scenario names**

Each key under `tests:` defines a scenario that can be selected individually.

**2.** `**extra_args**`

This property passes additional arguments to the build configuration for the scenario.

In the example, disabling `CONFIG_ZTEST_SHUFFLE` allows a demonstration of ordered test execution.

**3. Hardware scenarios**

A hardware-specific scenario should be configured with the correct board target and any additional requirements needed by its test application.

**4. Shared settings**

The `common` section is useful when multiple scenarios share tags or other supported settings.

## 6. Running Twister

Twister is invoked through the `west` command-line tool.

Run these commands from your Zephyr workspace or repository, using paths that match your project structure.

### Run All Scenarios in a Directory

```
west twister -T tests/calculator
```

`-T` specifies the directory Twister should search for test scenarios.

### Restrict Testing to One Platform

```
west twister -T tests/calculator -p native_sim
```

The `-p` option selects the platform to test.

### Run One Specific Scenario

```
west twister \
    -T tests/calculator \
    -s example.unit.calculator.ordered
```

The `-s` option selects a specific test scenario by its registered name.

### Enable Verbose Output

```
west twister -T tests/calculator -v
```

For more detailed diagnostic output, use:

```
west twister -T tests/calculator -vvv
```

Verbosity helps when investigating build failures, test execution problems, or unexpected results.

### Command Summary

| Command                                                      | Purpose                                                      |
| ------------------------------------------------------------ | ------------------------------------------------------------ |
| `west twister -T tests/calculator`                           | Run all discovered scenarios under the directory, subject to the applicable configuration and filters. |
| `west twister -T tests/calculator -p native_sim`             | Select `native_sim` as the test platform.                    |
| `west twister -T tests/calculator -s example.unit.calculator.ordered` | Select one scenario.                                         |
| `west twister -T tests/calculator -v`                        | Enable verbose output.                                       |
| `west twister -T tests/calculator -vvv`                      | Request the highest verbosity level shown in this example.   |

For complete command-line options, see the official [Zephyr Twister documentation](https://docs.zephyrproject.org/latest/develop/test/twister.html).

## 7. Hardware Testing

Twister can run tests on physical development boards when the required device-testing configuration is available.

For example, the following command targets an nRF52840 development kit:

```
west twister \
    -T zephyr/samples/hello_world \
    --device-testing \
    --device-serial /dev/ttyACM0 \
    --device-serial-baud 115200 \
    -p nrf52840dk/nrf52840
```

### Command Options

| Option                 | Purpose                                                      |
| ---------------------- | ------------------------------------------------------------ |
| `-T`                   | Specifies the test directory.                                |
| `--device-testing`     | Enables device-testing mode.                                 |
| `--device-serial`      | Specifies the serial port used for device communication and output capture. |
| `--device-serial-baud` | Sets the serial baud rate.                                   |
| `-p`                   | Selects the target board.                                    |

### What Happens During Hardware Testing?

Depending on the board and test configuration, Twister can:

1. Build the test application for the selected board.
2. Flash the device using the configured runner.
3. Execute the application on the physical hardware.
4. Capture test output through the configured serial connection.
5. Evaluate the output using the test harness.
6. Record the test result in the generated reports.

### Hardware Testing Requirements

Before running the command, verify that:

- The board is connected and powered.
- The selected board target is correct.
- The serial device path exists.
- Your user has permission to access the serial device.
- The required flashing tools and runners are installed.
- The configured harness can recognize the test output.

**Note:** `--device-serial` specifies the serial connection used by Twister. The exact flashing and execution workflow depends on the board's runner and the test configuration.

## 8. What Gets Produced?

Twister writes test artifacts into an output directory, typically named `twister-out/`.

A simplified directory structure looks like this:

```
twister-out/
├── twister.json
├── twister.xml
├── twister.log
├── coverage/
│   ├── index.html
│   └── coverage.json
└── native_sim/
    └── <scenario>/
        ├── build.log
        ├── handler.log
        └── build/
            └── zephyr/
                └── zephyr.exe
```

The exact structure depends on the Zephyr version, platform, build configuration, and enabled reporting options.

### Important Artifacts

| Artifact       | Purpose                                                      |
| -------------- | ------------------------------------------------------------ |
| `twister.json` | Machine-readable summary of test results.                    |
| `twister.xml`  | JUnit XML report suitable for compatible CI systems.         |
| `twister.log`  | Top-level Twister execution log.                             |
| `build.log`    | Build output for an individual scenario.                     |
| `handler.log`  | Captured output from test execution, when produced by the configured handler. |
| `coverage/`    | Coverage reports, when coverage collection is enabled and supported. |
| `zephyr.exe`   | Native executable produced by a native build, when applicable. |

### Why These Artifacts Matter

- **JSON:** Enables automated processing and custom reporting.
- **JUnit XML:** Allows compatible CI systems to display test results.
- **Logs:** Help diagnose configuration, compilation, and runtime failures.
- **Coverage reports:** Show which parts of the source code were exercised.

The generated executable can also be useful for local debugging, provided the build target and runtime requirements are understood.

## 9. Code Coverage

Code coverage measures which portions of the source code were reached during test execution.

Twister can collect coverage information for supported targets and toolchains.

### Running with Coverage

For the calculator test, an example command is:

```
west twister \
    -T tests/calculator \
    -p native_sim \
    --coverage \
    --coverage-tool gcovr \
    --coverage-basedir app/modules/calculator
```

### Command Options

| Option                  | Purpose                                                      |
| ----------------------- | ------------------------------------------------------------ |
| `--coverage`            | Enables coverage collection where supported.                 |
| `--coverage-tool gcovr` | Selects `gcovr` as the coverage reporting tool.              |
| `--coverage-basedir`    | Sets the source base directory used to focus coverage reporting. |

Depending on the Zephyr version and configuration, coverage output may include files such as:

```
twister-out/
├── coverage/
│   └── index.html
└── coverage.json
```

Check the generated output directory for the actual artifact locations.

### What Coverage Tells You

Coverage can help answer questions such as:

- Which source lines were executed?
- Which branches were reached?
- Which parts of a module have not been exercised?
- Where might additional tests be needed?

### What Coverage Does Not Tell You

Coverage does not prove that the code behaves correctly.

A test can execute a line without verifying its result. A test suite can also achieve high line coverage while missing important edge cases or incorrect behavior.

For example, calling a division function only with valid operands does not test its behavior when the divisor is zero.

**Best practice:** Use coverage to identify testing gaps, then add assertions that verify the expected behavior of those paths.

## 10. Integrating Twister into CI

Automated tests are most valuable when they run consistently as the code changes.

Twister can be integrated into a CI pipeline to build and execute selected tests automatically.

### Recommended Workflow

1. A developer changes the source code.
2. The changes are committed or submitted for review.
3. The CI environment checks out the repository.
4. The required Zephyr environment and dependencies are configured.
5. Twister discovers and builds the selected test scenarios.
6. Supported tests execute on the configured platforms.
7. CI collects the results and logs.
8. JUnit XML and other reports are published when configured.
9. A failing required test can cause the pipeline to fail.

### Example CI Command

```
west twister \
    -T tests/calculator \
    -p native_sim \
    -v
```

This provides a starting point for a CI job that runs calculator unit tests on a native simulation platform.

A hardware-testing stage can be added separately when a connected development board and the required test infrastructure are available.

### Recommended Strategy

- Run fast unit tests frequently.
- Use native platforms for tests that do not require physical hardware.
- Add emulated or hardware-specific testing for integration and system validation.
- Preserve test logs and machine-readable reports.
- Investigate flaky tests rather than relying on repeated retries.
- Use code coverage to guide additional testing.

## 11. Troubleshooting

### Twister Does Not Discover the Test

Check that:

- The directory passed to `-T` is correct.
- The test metadata file exists.
- The YAML indentation is valid.
- The scenario is declared under `tests:`.
- The scenario satisfies the active filters.

### No Matching Platform Is Found

Check that:

- The selected platform matches the test's `platform_allow` configuration.
- The board target is valid for the installed Zephyr version.
- The required architecture and Kconfig constraints are satisfied.
- The platform is supported by the test application.

### The Build Fails

Inspect the scenario's build log and verify:

- Required Kconfig options are enabled.
- Source files and include directories are configured correctly.
- The selected platform supports the required features.
- The correct Zephyr environment is active.

### A Test Fails During Execution

Inspect the captured test output and determine whether the failure is caused by:

- An incorrect assertion.
- A defect in the module.
- Invalid test initialization.
- Hidden shared-state dependencies.
- A timeout or runtime issue.
- An incorrect harness configuration.

### Hardware Testing Does Not Work

Check the board connection, serial port, device permissions, flashing runner, and configured baud rate.

Also verify that the selected board target and test scenario support the intended device-testing workflow.

## 12. Key Takeaways

- **Twister** is Zephyr's central test runner and orchestration tool.
- It discovers scenarios from supported test metadata files.
- It filters tests according to platforms and configuration constraints.
- It builds scenarios for matching platforms, with parallelism where possible.
- It runs tests on native environments, emulators, simulators, and real hardware.
- It evaluates execution output through configured test harnesses.
- It produces JSON, JUnit XML, logs, and optional coverage reports.
- It supports both local development and CI automation.
- Code coverage helps identify untested code but does not prove correctness.
- Combining native unit tests with selected hardware tests provides a more complete validation strategy.

**The main goal of Twister is to make testing repeatable, scalable, and automatable across the Zephyr development workflow.**

## References

- [Zephyr Twister Documentation](https://docs.zephyrproject.org/latest/develop/test/twister.html)
- [Zephyr Ztest Documentation](https://docs.zephyrproject.org/latest/develop/test/ztest.html)
- [Zephyr Testing Overview](https://docs.zephyrproject.org/latest/develop/test/index.html)
- [Zephyr Project Documentation](https://docs.zephyrproject.org/latest/)