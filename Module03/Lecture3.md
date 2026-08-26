# Zephyr RTOS — Build Process

Zephyr build process, including the configuration phase using Kconfig and Devicetree and the normal compilation pipeline from source files to `zephyr.elf`.

The complete flow is:

```text
west → CMake → Kconfig + Devicetree → Generated Headers
                                      ↓
                              Preprocessor
                                      ↓
                                  Compiler
                                      ↓
                                  Assembler
                                      ↓
                              Object Files
                                      ↓
                                   Linker
                                      ↓
                                 zephyr.elf
```

The main idea is that **Zephyr first configures the software and hardware, then performs the normal compilation, assembly, and linking process.**

## Key Components

| Component      | Responsibility                                    |
| -------------- | ------------------------------------------------- |
| West           | Manages the Zephyr workspace and starts the build |
| CMake          | Configures and orchestrates the build             |
| Kconfig        | Configures Zephyr software features               |
| `prj.conf`     | Contains application-specific Kconfig settings    |
| Devicetree     | Describes the target hardware                     |
| Board DTS      | Defines the board's default hardware              |
| Overlay        | Modifies or extends the hardware description      |
| `autoconf.h`   | Generated Kconfig configuration                   |
| `devicetree.h` | Generated Devicetree definitions                  |
| Preprocessor   | Expands headers and macros                        |
| Compiler       | Converts C/C++ into assembly                      |
| Assembler      | Converts assembly into object files               |
| Linker         | Combines objects and produces the executable      |
| `zephyr.elf`   | Final ELF executable/debug image                  |

## Configuration Phase

The configuration phase determines which Zephyr features are enabled and how the target hardware is configured.

It mainly consists of:

```text
             CMake
               │
       ┌───────┴───────┐
       ▼               ▼
    Kconfig        Devicetree
       │               │
   prj.conf       board.dts
                       +
                   overlays
       │               │
       └───────┬───────┘
               ▼
      Generated Headers
       ├── autoconf.h
       └── devicetree.h
```

### Kconfig

Kconfig controls software configuration such as:

- Kernel features
- Drivers
- Logging
- Networking
- Bluetooth
- File systems
- Memory configuration

Example:

```ini
CONFIG_GPIO=y
CONFIG_LOG=y
CONFIG_PRINTK=y
```

### Devicetree

Devicetree describes the hardware:

- GPIOs
- LEDs
- Buttons
- UART
- SPI
- I2C
- Sensors
- Interrupts
- Memory

Board-specific hardware is commonly described by `board.dts`, while application-specific modifications can be provided through `.overlay` files.

## Normal Build Process

After configuration, the regular toolchain takes over:

```text
Source Files
     ↓
Preprocessor
     ↓
Preprocessed Source
     ↓
Compiler
     ↓
Assembly (.s)
     ↓
Assembler
     ↓
Object Files (.o)
     ↓
Linker
     ↓
zephyr.elf
```

### Preprocessor

Processes:

- `#include`
- `#define`
- Conditional compilation
- Configuration macros

### Compiler

Converts the preprocessed C/C++ source into assembly code and performs optimization and code generation.

### Assembler

Converts assembly instructions into machine-code object files.

### Linker

Combines application code, Zephyr kernel, drivers, libraries, and other object files according to the linker script.

The result is:

```text
zephyr.elf
```

## Example

A typical Zephyr application may look like:

```text
blink/
├── CMakeLists.txt
├── prj.conf
├── src/
│   └── main.cpp
└── boards/
    └── <board>.overlay
```

Build it using:

```bash
west build -b <board>
```

The build then follows:

```text
west
 ↓
CMake
 ↓
Kconfig + Devicetree
 ↓
Generated configuration
 ↓
Preprocessor
 ↓
Compiler
 ↓
Assembler
 ↓
Object files
 ↓
Linker
 ↓
zephyr.elf
```

## Final Takeaway

A traditional build is usually:

```text
Source → Preprocessor → Compiler → Assembler → Linker → Executable
```

Zephyr adds a powerful configuration stage:

```text
Kconfig + Devicetree
          ↓
Generated Configuration
          ↓
Normal Build
          ↓
zephyr.elf
```

This architecture allows the same Zephyr application to be configured for different boards, MCUs, peripherals, drivers, and system features.

**In short:** West starts the build, CMake orchestrates it, Kconfig configures software, Devicetree describes hardware, and the compiler toolchain produces the final `zephyr.elf`.
:::

# Zephyr Kconfig — Build-Time Configuration System

## Overview

**Kconfig** is Zephyr's build-time configuration system. It originated in the Linux kernel and is used to configure which software features are enabled or disabled **before compilation**.

Kconfig allows Zephyr to create a customized firmware image based on:

- Application requirements
- Board capabilities
- MCU architecture
- Enabled drivers
- Kernel features
- Subsystems
- Memory and system configuration

The main idea is:

```text
Configuration
     ↓
Kconfig
     ↓
Generated configuration
     ↓
Compilation
     ↓
Firmware
```

---

# 1. What is Kconfig?

Kconfig is a configuration language and system originally developed for the **Linux kernel** and widely used to manage configurable software components.

In Zephyr, Kconfig determines which features are included in the final firmware.

For example, a project may enable:

```ini
CONFIG_GPIO=y
CONFIG_LOG=y
CONFIG_PRINTK=y
```

or disable a feature:

```ini
CONFIG_BLUETOOTH=n
```

The selected configuration affects what code is compiled into the final application.

---

# 2. What Does Kconfig Control?

Kconfig can control many parts of a Zephyr system.

### Drivers

Examples:

```ini
CONFIG_GPIO=y
CONFIG_I2C=y
CONFIG_SPI=y
CONFIG_UART_INTERRUPT_DRIVEN=y
```

### Kernel Features

Examples include:

- Threads
- Semaphores
- Mutexes
- Timers
- Work queues
- Scheduling options

### Subsystems

Kconfig can configure subsystems such as:

- Networking
- Bluetooth
- USB
- File systems
- Logging
- Shell
- Sensor subsystem

### Memory Configuration

Kconfig can also control memory-related options such as:

- Heap size
- Stack sizes
- Memory pools
- Kernel memory features

---

# 3. Why Is Kconfig Needed?

Embedded systems have limited resources.

An application may not need every Zephyr feature, driver, or subsystem.

For example, a simple LED application may only need:

```ini
CONFIG_GPIO=y
```

There is no reason to include Bluetooth, networking, USB, or other unused subsystems.

Kconfig allows Zephyr to build a firmware image containing only the required features.

This helps with:

- Flash usage
- RAM usage
- Build customization
- Hardware support
- Feature management
- Dependency handling

---

# 4. Kconfig and Build-Time Configuration

Kconfig works **before the actual compilation of the application**.

The process can be simplified as:

```text
Application Configuration
          │
          ▼
       Kconfig
          │
          ▼
 Final Configuration
          │
          ▼
 Generated Headers
          │
          ▼
      Compilation
```

This means that Kconfig is fundamentally a **build-time configuration system**, not a runtime configuration system.

For example:

```ini
CONFIG_LOG=y
```

decides during the build whether logging support is included.

---

# 5. `prj.conf` — Your Project Configuration

For most Zephyr applications, the developer configures Kconfig through:

```text
prj.conf
```

Example:

```ini
CONFIG_GPIO=y
CONFIG_LOG=y
CONFIG_PRINTK=y
CONFIG_MAIN_STACK_SIZE=2048
```

This file contains the configuration choices required by the application.

### Important

You normally **do not modify Zephyr's internal Kconfig files** just to configure your application.

Instead, add the required `CONFIG_*` options to your project's:

```text
prj.conf
```



---

# 6. What Happens During the Build?

Suppose your project contains:

```text
my_app/
├── CMakeLists.txt
├── prj.conf
└── src/
    └── main.c
```

When you run:

```bash
west build -b <board>
```

Zephyr processes your configuration.

A simplified workflow is:

```text
                 prj.conf
                    │
                    ▼
              Kconfig System
                    │
                    ▼
        Dependency Resolution
                    │
                    ▼
        Final/Merged Configuration
                    │
                    ▼
          build/zephyr/.config
                    │
                    ▼
              autoconf.h
                    │
                    ▼
             CONFIG_* macros
                    │
                    ▼
              Source Code
                    │
                    ▼
               Compilation
```

---

# 7. `build/zephyr/.config`

After Kconfig processes the configuration, Zephyr generates:

```text
build/zephyr/.config
```

This file represents the **final configuration selected for the build**.

It can contain configuration values coming from:

- `prj.conf`
- Board configuration
- Architecture configuration
- Zephyr defaults
- Dependencies
- Other Kconfig fragments

For example:

```text
CONFIG_GPIO=y
CONFIG_LOG=y
CONFIG_PRINTK=y
CONFIG_MAIN_STACK_SIZE=2048
```

### Important Difference

`prj.conf` is what **you request**.

```text
prj.conf
    ↓
"What does my application want?"
```

`.config` is what **Kconfig finally resolves**.

```text
build/zephyr/.config
    ↓
"What configuration will actually be used?"
```

---

# 8. `autoconf.h`

Kconfig configuration must eventually be accessible to C and C++ source code.

Zephyr generates:

```text
build/zephyr/include/generated/zephyr/autoconf.h
```

This file converts Kconfig options into C preprocessor macros.

For example, if:

```ini
CONFIG_GPIO=y
```

is enabled, the generated header contains a corresponding configuration macro.

Conceptually:

```c
#define CONFIG_GPIO 1
```

Then application code can use:

```c
#ifdef CONFIG_GPIO
    /* GPIO-related code */
#endif
```

Another example:

```c
#if CONFIG_LOG
    /* Logging code */
#endif
```

---

# 9. The `CONFIG_*` Naming Convention

Zephyr Kconfig options normally use the:

```text
CONFIG_
```

prefix.

Examples:

```text
CONFIG_GPIO
CONFIG_LOG
CONFIG_I2C
CONFIG_SPI
CONFIG_BT
CONFIG_PRINTK
CONFIG_MAIN_STACK_SIZE
```

These options can be used in C/C++ code to conditionally compile features.

Example:

```c
#ifdef CONFIG_LOG
#include <zephyr/logging/log.h>
#endif
```

---

# 10. Basic Workflow

The most important workflow to remember is:

```text
Edit prj.conf
      ↓
Run Build
      ↓
Kconfig processes configuration
      ↓
Generate .config
      ↓
Generate autoconf.h
      ↓
CONFIG_* available in code
      ↓
Compile
```

In practice:

### Step 1 — Edit `prj.conf`

```ini
CONFIG_GPIO=y
CONFIG_LOG=y
```

### Step 2 — Build

```bash
west build -b <board>
```

### Step 3 — Kconfig generates the final configuration

```text
build/zephyr/.config
```

### Step 4 — Zephyr generates C configuration macros

```text
build/zephyr/include/generated/zephyr/autoconf.h
```

### Step 5 — Use configuration in code

```c
#ifdef CONFIG_GPIO
    /* GPIO implementation */
#endif
```

---

# 11. Kconfig Menu Interface — `menuconfig`

Kconfig also provides a user-friendly interactive configuration interface called:

```text
menuconfig
```

It allows developers to browse and change available configuration options through a menu.

Run:

```bash
west build -t menuconfig
```

The interface lets you:

- Browse configuration categories
- Enable features
- Disable features
- Change configuration values
- See dependencies
- Inspect available options

A simplified structure may look like:

```text
Zephyr Kernel
├── General Kernel Options
├── Device Drivers
├── Networking
├── Bluetooth
├── USB
├── File Systems
└── Logging
```

After changing options, the resulting configuration can be saved and used by the build system.

---

# 12. Where Is Kconfig Stored?

Kconfig is **not stored in one single file**.

The Kconfig system is distributed throughout the Zephyr source tree.

Important locations include:

```text
zephyr/
├── Kconfig
├── Kconfig.zephyr
│
├── kernel/
│   └── Kconfig
│
├── drivers/
│   └── Kconfig
│
├── subsys/
│   └── Kconfig
│
└── arch/
    └── Kconfig
```

The exact organization can vary between Zephyr versions and modules.

### Root Kconfig

The Zephyr source tree contains the main Kconfig entry points:

```text
zephyr/Kconfig
zephyr/Kconfig.zephyr
```

These bring together configuration options from different parts of Zephyr.

### Kernel Kconfig

Kernel-specific options are defined under:

```text
zephyr/kernel/
```

### Driver Kconfig

Hardware driver configuration is defined under:

```text
zephyr/drivers/
```

### Subsystem Kconfig

Subsystem-specific configuration is found under:

```text
zephyr/subsys/
```

### Architecture Kconfig

Architecture-specific configuration is organized under:

```text
zephyr/arch/
```

---

# 13. Kconfig Files vs `prj.conf`

It is important not to confuse the Kconfig definition files with your project's configuration.

### Kconfig files

They define **what options exist** and how they behave.

Conceptually:

```text
Kconfig
    │
    ├── Define CONFIG_GPIO
    ├── Define CONFIG_LOG
    ├── Define CONFIG_I2C
    └── Define dependencies
```

### `prj.conf`

It specifies **which options your application wants**.

```ini
CONFIG_GPIO=y
CONFIG_LOG=y
CONFIG_I2C=y
```

So:

```text
Kconfig files
     ↓
"What options are available?"

prj.conf
     ↓
"What does my application select?"
```

---

# 14. Dependency Handling

One of the major advantages of Kconfig is dependency management.

For example, a feature may depend on another feature:

```text
Feature A
   │
   └── depends on Feature B
```

Kconfig can determine whether the required dependencies are satisfied.

This prevents developers from manually managing every low-level configuration option.

The final `.config` represents the configuration after Kconfig has resolved these relationships.

---

# 15. Example: GPIO Configuration

Suppose the application contains:

```c
#include <zephyr/drivers/gpio.h>

int main(void)
{
    /* GPIO application code */
}
```

The project can enable GPIO support through:

```ini
CONFIG_GPIO=y
```

The workflow becomes:

```text
prj.conf
   │
   │ CONFIG_GPIO=y
   ▼
Kconfig
   │
   ▼
build/zephyr/.config
   │
   ▼
autoconf.h
   │
   ▼
CONFIG_GPIO
   │
   ▼
C/C++ Compilation
```

This allows Zephyr's build system to include the required GPIO functionality.

---

# 16. Kconfig vs Devicetree

Kconfig and Devicetree are both important parts of Zephyr's configuration system, but they solve different problems.

| Kconfig                         | Devicetree                              |
| ------------------------------- | --------------------------------------- |
| Configures software features    | Describes hardware                      |
| Enables/disables drivers        | Describes peripherals                   |
| Controls kernel features        | Defines hardware relationships          |
| Controls memory-related options | Defines GPIO/pin/peripheral connections |
| Uses `CONFIG_*`                 | Uses Devicetree nodes/macros            |
| Main project file: `prj.conf`   | Often customized with `.overlay`        |

A simple way to remember:

```text
Kconfig
"What software should be enabled?"

Devicetree
"What hardware exists and how is it connected?"
```

---

# 17. Complete Kconfig Flow

The complete flow can be represented as:

```text
                         Zephyr Build
                              │
                              ▼
                            CMake
                              │
                              ▼
                         Kconfig System
                              │
              ┌───────────────┴────────────────┐
              │                                │
              ▼                                ▼
          Kconfig Files                    prj.conf
              │                                │
              └───────────────┬────────────────┘
                              ▼
                    Dependency Resolution
                              │
                              ▼
                    build/zephyr/.config
                              │
                              ▼
              build/zephyr/include/generated/
                         zephyr/autoconf.h
                              │
                              ▼
                       CONFIG_* macros
                              │
                              ▼
                       C/C++ Source Code
                              │
                              ▼
                         Compilation
```

---

# 18. Important Files to Remember

| File                                               | Meaning                                 |
| -------------------------------------------------- | --------------------------------------- |
| `zephyr/Kconfig`                                   | Root Kconfig entry point                |
| `zephyr/Kconfig.zephyr`                            | Main Zephyr configuration               |
| `prj.conf`                                         | Application/project configuration       |
| `build/zephyr/.config`                             | Final merged/resolved configuration     |
| `build/zephyr/include/generated/zephyr/autoconf.h` | Generated C macros                      |
| `CONFIG_*`                                         | Configuration macros used by C/C++ code |

---

# 19. Quick Reference

### Configure the project

```bash
west build -b <board>
```

### Open menuconfig

```bash
west build -t menuconfig
```

### Project configuration

```text
prj.conf
```

### Final configuration

```text
build/zephyr/.config
```

### Generated configuration header

```text
build/zephyr/include/generated/zephyr/autoconf.h
```

### Use a configuration option in C

```c
#ifdef CONFIG_FEATURE
    /* Feature-specific code */
#endif
```

---

# Zephyr Kconfig — Configurable LED Blink Interval

## Overview

Zephyr uses **Kconfig** to configure application and system features at build time.

This example defines a configurable LED blink interval using:

```kconfig
config BLINK_SLEEP_TIME_MS
    int "LED blink interval (ms)"
    default 1000
    range 100 5000
    help
      Time between LED toggles in milliseconds.
```

This configuration allows the LED blink delay to be changed without modifying the application source code.

------

## Kconfig Syntax

| Keyword                      | Description                      |
| ---------------------------- | -------------------------------- |
| `config BLINK_SLEEP_TIME_MS` | Defines the configuration symbol |
| `int`                        | Configuration type               |
| `"LED blink interval (ms)"`  | Prompt displayed in `menuconfig` |
| `default 1000`               | Default value                    |
| `range 100 5000`             | Allowed values                   |
| `help`                       | Description shown to the user    |

### Configuration Types

Kconfig supports several commonly used types:

| Type     | Example                           | Usage                   |
| -------- | --------------------------------- | ----------------------- |
| `bool`   | `CONFIG_GPIO=y`                   | Enable/disable features |
| `int`    | `CONFIG_BLINK_SLEEP_TIME_MS=2048` | Numeric values          |
| `string` | `CONFIG_BT_DEVICE_NAME="Zephyr"`  | Text values             |
| `hex`    | `CONFIG_SRAM_OFFSET=0x1000`       | Hexadecimal values      |

------

## Using Kconfig in C/C++

The `CONFIG_` prefix is automatically added to Kconfig symbols when they are generated for the application.

For example:

```kconfig
config BLINK_SLEEP_TIME_MS
    int "LED blink interval (ms)"
    default 1000
    range 100 5000
```

is accessed in C/C++ as:

```cpp
CONFIG_BLINK_SLEEP_TIME_MS
```

### Boolean Configuration

A boolean option can be checked in several ways:

```cpp
#if CONFIG_GPIO
```

or:

```cpp
#ifdef CONFIG_GPIO
```

or, preferably in many Zephyr applications:

```cpp
if (IS_ENABLED(CONFIG_GPIO)) {
    // GPIO feature enabled
}
```

`IS_ENABLED()` expands to a compile-time expression such as:

```cpp
if (0)
```

or:

```cpp
if (1)
```

### Other Configuration Types

For an integer configuration:

```cpp
int delay = CONFIG_BLINK_SLEEP_TIME_MS;
```

The value is available directly as a compile-time configuration constant.

------

## Setting the Value in `prj.conf`

The application configuration can be changed in:

```text
prj.conf
```

For example:

```conf
CONFIG_BLINK_SLEEP_TIME_MS=250
```

This sets the LED blink interval to **250 ms**.

The value must respect the range defined in Kconfig:

```text
100 ≤ CONFIG_BLINK_SLEEP_TIME_MS ≤ 5000
```

For example:

```conf
CONFIG_BLINK_SLEEP_TIME_MS=100
```

is valid, while:

```conf
CONFIG_BLINK_SLEEP_TIME_MS=50
```

is outside the allowed range.

------

## Using `menuconfig`

Zephyr provides an interactive configuration interface:

```bash
west build -t menuconfig
```

Inside `menuconfig`:

- Press `/` to search for a configuration symbol.
- Search for `BLINK_SLEEP_TIME_MS`.
- Change the value.
- Press `S` to save.
- Press `Q` to quit.

The resulting configuration is used during the next build.

------

## Command-Line Override

A configuration value can also be overridden directly from the command line:

```bash
west build -- -DCONFIG_BLINK_SLEEP_TIME_MS=2000
```

This sets the blink interval to:

```text
2000 ms = 2 seconds
```

This is useful for quickly testing different configuration values without permanently modifying `prj.conf`.

------

## Configuration Precedence

When the same configuration option receives values from multiple sources, the effective value follows this precedence:

```text
CLI
  ↓
menuconfig
  ↓
prj.conf
  ↓
Kconfig default
```

In other words:

**CLI > menuconfig > prj.conf > default**

For example, if Kconfig defines:

```kconfig
default 1000
```

and `prj.conf` contains:

```conf
CONFIG_BLINK_SLEEP_TIME_MS=250
```

the value becomes:

```text
250
```

If the build command specifies:

```bash
west build -- -DCONFIG_BLINK_SLEEP_TIME_MS=2000
```

the command-line value takes precedence.

------

## Example Project Structure

A typical Zephyr application using this configuration can have:

```text
blink/
├── CMakeLists.txt
├── prj.conf
├── Kconfig
└── src/
    └── main.cpp
```

### `Kconfig`

```kconfig
config BLINK_SLEEP_TIME_MS
    int "LED blink interval (ms)"
    default 1000
    range 100 5000
    help
      Time between LED toggles in milliseconds.
```

### `prj.conf`

```conf
CONFIG_GPIO=y
CONFIG_BLINK_SLEEP_TIME_MS=1000
```

### `main.cpp`

```cpp
#include <zephyr/kernel.h>

int main(void)
{
    int delay = CONFIG_BLINK_SLEEP_TIME_MS;

    while (1) {
        // Toggle LED

        k_msleep(delay);
    }

    return 0;
}
```

------

## Why Use Kconfig?

Using Kconfig instead of hard-coding values provides several advantages:

- **Configurable:** Change behavior without modifying source code.
- **Validated:** `range` prevents invalid values.
- **Reusable:** The same application can use different configurations.
- **Build-time configuration:** Values are selected during the build process.
- **menuconfig support:** Users can configure options interactively.
- **Command-line support:** Useful for automated builds and testing.
- **Clear separation:** Application configuration is separated from implementation.

------

## Quick Reference

```text
Kconfig:
    Define configuration options

prj.conf:
    Set project configuration

menuconfig:
    Configure interactively

CLI:
    Temporarily override configuration

C/C++:
    Access values using CONFIG_<SYMBOL>
```

For this example:

```text
Kconfig symbol:
    BLINK_SLEEP_TIME_MS

C/C++ symbol:
    CONFIG_BLINK_SLEEP_TIME_MS

Default:
    1000 ms

Minimum:
    100 ms

Maximum:
    5000 ms
```

### Useful Commands

```bash
# Configure interactively
west build -t menuconfig

# Build with a command-line override
west build -- -DCONFIG_BLINK_SLEEP_TIME_MS=2000

# Build the application
west build
```

---

# Zephyr Kconfig — `choice` and Derived Values

Kconfig's `choice` construct is used when the user should select **exactly one option from multiple alternatives**.

For example, an LED application may provide three blink speeds:

- Fast → 250 ms
- Medium → 1000 ms
- Slow → 2000 ms

Instead of asking the user to enter a numeric value, `choice` provides a clear set of predefined options.

------

## What Is `choice`?

The basic syntax is:

```kconfig
choice
    prompt "LED blink speed"

    # ... options

endchoice
```

The `choice` block groups multiple configuration options together so that the user selects one alternative.

### Example

```kconfig
choice
    prompt "LED blink speed"
    default BLINK_SLEEP_1000MS

config BLINK_SLEEP_250MS
    bool "Fast (250ms)"

config BLINK_SLEEP_1000MS
    bool "Medium (1s)"

config BLINK_SLEEP_2000MS
    bool "Slow (2s)"

endchoice
```

The `prompt` is the text displayed to the user in `menuconfig`.

The `default` specifies which option is selected when the user does not choose another option.

------

## Available Options

The resulting menu can conceptually look like:

```text
( ) Fast (250ms)
(*) Medium (1s)
( ) Slow (2s)
```

Only **one** option can be selected at a time.

Here:

```kconfig
default BLINK_SLEEP_1000MS
```

means that **Medium (1s)** is selected by default.

------

# Hidden Symbols — Derived Values

A `choice` is useful for selecting the user's preference, but the application may need an actual numeric value.

For example, the application wants:

```cpp
k_msleep(CONFIG_BLINK_SLEEP_TIME_MS);
```

We can create a hidden `int` configuration symbol that converts the selected choice into a numeric value.

```kconfig
config BLINK_SLEEP_TIME_MS
    int
    default 250  if BLINK_SLEEP_250MS
    default 1000 if BLINK_SLEEP_1000MS
    default 2000 if BLINK_SLEEP_2000MS
    default 0
```

Notice that `BLINK_SLEEP_TIME_MS` has **no prompt**.

Therefore, it is a **hidden configuration symbol**.

------

## Why Is the Symbol Hidden?

A configuration symbol with a prompt is visible to the user:

```kconfig
config BLINK_SLEEP_250MS
    bool "Fast (250ms)"
```

But this symbol has no prompt:

```kconfig
config BLINK_SLEEP_TIME_MS
    int
```

Therefore, `BLINK_SLEEP_TIME_MS` does not appear as a separate setting in `menuconfig`.

It is used internally to **derive a value from the selected choice**.

------

## Complete Example

A complete Kconfig implementation can look like this:

```kconfig
choice
    prompt "LED blink speed"
    default BLINK_SLEEP_1000MS

config BLINK_SLEEP_250MS
    bool "Fast (250ms)"

config BLINK_SLEEP_1000MS
    bool "Medium (1s)"

config BLINK_SLEEP_2000MS
    bool "Slow (2s)"

endchoice


config BLINK_SLEEP_TIME_MS
    int
    default 250  if BLINK_SLEEP_250MS
    default 1000 if BLINK_SLEEP_1000MS
    default 2000 if BLINK_SLEEP_2000MS
    default 0
```

------

## How the Derived Value Works

The selected choice determines which condition is true.

### Fast

If the user selects:

```text
Fast (250ms)
```

then:

```text
BLINK_SLEEP_250MS = y
```

and:

```text
CONFIG_BLINK_SLEEP_TIME_MS = 250
```

### Medium

If the user selects:

```text
Medium (1s)
```

then:

```text
BLINK_SLEEP_1000MS = y
```

and:

```text
CONFIG_BLINK_SLEEP_TIME_MS = 1000
```

### Slow

If the user selects:

```text
Slow (2s)
```

then:

```text
BLINK_SLEEP_2000MS = y
```

and:

```text
CONFIG_BLINK_SLEEP_TIME_MS = 2000
```

The application therefore only needs to work with:

```cpp
CONFIG_BLINK_SLEEP_TIME_MS
```

------

## Using the Value in C/C++

The derived configuration value can be used directly:

```cpp
#include <zephyr/kernel.h>

int main(void)
{
    while (1) {
        // Toggle LED

        k_msleep(CONFIG_BLINK_SLEEP_TIME_MS);
    }

    return 0;
}
```

The application does not need to know which choice was selected.

It simply uses:

```cpp
CONFIG_BLINK_SLEEP_TIME_MS
```

------

## Configuration Flow

The overall flow is:

```text
             Kconfig choice
                   │
                   ▼
        ┌─────────────────────┐
        │   LED blink speed   │
        └─────────────────────┘
             │     │     │
             ▼     ▼     ▼
           Fast  Medium  Slow
          250ms   1s     2s
             │     │     │
             └─────┼─────┘
                   ▼
       BLINK_SLEEP_TIME_MS
                   │
                   ▼
       CONFIG_BLINK_SLEEP_TIME_MS
                   │
                   ▼
             C/C++ code
                   │
                   ▼
             k_msleep(...)
```

------

## `choice` vs. Hidden Derived Symbol

These two parts have different responsibilities:

| Component                    | Purpose                                     |
| ---------------------------- | ------------------------------------------- |
| `choice`                     | Lets the user select one option             |
| `BLINK_SLEEP_250MS`          | Represents the Fast selection               |
| `BLINK_SLEEP_1000MS`         | Represents the Medium selection             |
| `BLINK_SLEEP_2000MS`         | Represents the Slow selection               |
| `BLINK_SLEEP_TIME_MS`        | Converts the selection into a numeric value |
| `CONFIG_BLINK_SLEEP_TIME_MS` | Value used by C/C++                         |

This pattern is useful when you want a **friendly configuration interface** while keeping the application code simple.

------

## Key Takeaways

### `choice`

Use `choice` when the user should select **one option from multiple alternatives**.

```kconfig
choice
    prompt "LED blink speed"
    ...
endchoice
```

### Hidden Symbol

A configuration symbol without a prompt is hidden:

```kconfig
config BLINK_SLEEP_TIME_MS
    int
```

It can be used internally to calculate or derive a configuration value.

### Conditional Defaults

The value can depend on the selected choice:

```kconfig
default 250  if BLINK_SLEEP_250MS
default 1000 if BLINK_SLEEP_1000MS
default 2000 if BLINK_SLEEP_2000MS
```

### Application Code

The C/C++ code only needs:

```cpp
CONFIG_BLINK_SLEEP_TIME_MS
```

This gives the application a clean interface while Kconfig handles the user's configuration choice.

---



# Zephyr Kconfig — `range` and `depends on`

Kconfig provides mechanisms for **validating configuration values** and controlling **when configuration options are available**.

Two important features are:

- `range` → validates numeric configuration values.
- `depends on` → controls whether a configuration symbol is available based on another configuration option.

------

# `range` — Value Validation

The `range` keyword limits the values that can be assigned to a configuration symbol.

For example:

```kconfig
config LED_BRIGHTNESS
    int "LED brightness (0-100)"
    range 0 100
    default 100

config LED_FADE_DURATION_MS
    int "Fade duration (ms)"
    range 0 5000
    default 500
```

Here:

```text
LED_BRIGHTNESS
    Minimum = 0
    Maximum = 100
    Default = 100
```

and:

```text
LED_FADE_DURATION_MS
    Minimum = 0
    Maximum = 5000
    Default = 500
```

------

## Why Use `range`?

`range` provides **configuration-time validation**.

For example:

```conf
CONFIG_LED_BRIGHTNESS=75
```

is valid because:

```text
0 ≤ 75 ≤ 100
```

But:

```conf
CONFIG_LED_BRIGHTNESS=150
```

is outside the defined range.

Likewise:

```conf
CONFIG_LED_FADE_DURATION_MS=6000
```

is invalid because the maximum allowed value is `5000`.

This helps prevent invalid configuration values from reaching the application.

------

## `range` with `int` and `hex`

`range` is commonly used with numeric Kconfig types such as:

```kconfig
int
```

and:

```kconfig
hex
```

For example:

```kconfig
config BUFFER_SIZE
    int "Buffer size"
    range 1 1024
    default 256
```

The same concept can be applied to hexadecimal configuration values.

------

# `depends on` — Conditional Availability

The `depends on` keyword controls whether a configuration symbol is available based on another configuration symbol.

For example:

```kconfig
config LED_BRIGHTNESS
    int "LED brightness (0-100)"
    depends on LED_SUBSYSTEM
    default 100
```

This means:

> `LED_BRIGHTNESS` is only available when `LED_SUBSYSTEM` is enabled.

If:

```text
LED_SUBSYSTEM = n
```

then `LED_BRIGHTNESS` cannot be enabled independently.

The user must first enable:

```text
LED_SUBSYSTEM = y
```

------

# Using `if` for Multiple Symbols

When several configuration symbols share the same dependency, an `if` block can be cleaner.

Instead of repeating:

```kconfig
config LED_BRIGHTNESS
    int "LED brightness (0-100)"
    depends on LED_SUBSYSTEM
    default 100

config LED_FADE_DURATION_MS
    int "Fade duration (ms)"
    depends on LED_SUBSYSTEM
    default 500
```

you can write:

```kconfig
if LED_SUBSYSTEM

config LED_BRIGHTNESS
    int "LED brightness (0-100)"
    default 100

config LED_FADE_DURATION_MS
    int "Fade duration (ms)"
    default 500

endif
```

All configuration symbols inside the block automatically inherit:

```text
depends on LED_SUBSYSTEM
```

This is particularly useful when many symbols share the same dependency.

------

# `if` Block — Bulk Dependency

An `if` block groups multiple configuration symbols under the same condition.

```kconfig
if LED_SUBSYSTEM

config LED_BRIGHTNESS
    int "LED brightness (0-100)"
    range 0 100
    default 100

config LED_FADE_DURATION_MS
    int "Fade duration (ms)"
    range 0 5000
    default 500

endif
```

Conceptually, Kconfig treats these symbols as if they had:

```kconfig
depends on LED_SUBSYSTEM
```

This keeps the Kconfig file cleaner and avoids repeating the same dependency.

------

# What Happens When the Dependency Is Disabled?

Suppose:

```text
LED_SUBSYSTEM = n
```

Then:

```text
LED_BRIGHTNESS
LED_FADE_DURATION_MS
```

will not be available to the user in the configuration menu.

They cannot be enabled manually while their dependency is unmet.

The user must first enable:

```text
LED_SUBSYSTEM
```

After that, the dependent options become available.

------

# `depends on` Example

Consider:

```kconfig
config LED_SUBSYSTEM
    bool "Enable LED subsystem"

if LED_SUBSYSTEM

config LED_BRIGHTNESS
    int "LED brightness (0-100)"
    range 0 100
    default 100

config LED_FADE_DURATION_MS
    int "Fade duration (ms)"
    range 0 5000
    default 500

endif
```

The configuration flow becomes:

```text
LED_SUBSYSTEM
     │
     ├── n ──► LED configuration unavailable
     │
     └── y ──► LED_BRIGHTNESS available
               LED_FADE_DURATION_MS available
```

------

# `range` vs. `depends on`

These keywords solve different problems:

| Keyword      | Purpose                                        |
| ------------ | ---------------------------------------------- |
| `range`      | Restricts the valid numeric value              |
| `depends on` | Controls whether a symbol is available         |
| `if`         | Applies a common condition to multiple symbols |

For example:

```kconfig
config LED_BRIGHTNESS
    int "LED brightness (0-100)"
    range 0 100
    depends on LED_SUBSYSTEM
    default 100
```

means:

1. `LED_BRIGHTNESS` requires `LED_SUBSYSTEM`.
2. Its value must be between `0` and `100`.
3. Its default value is `100`.

------

# Complete Example

A practical configuration can combine all three features:

```kconfig
config LED_SUBSYSTEM
    bool "Enable LED subsystem"
    default y

if LED_SUBSYSTEM

config LED_BRIGHTNESS
    int "LED brightness (0-100)"
    range 0 100
    default 100

config LED_FADE_DURATION_MS
    int "Fade duration (ms)"
    range 0 5000
    default 500

endif
```

Then the C/C++ application can use:

```cpp
int brightness = CONFIG_LED_BRIGHTNESS;
int fade_duration = CONFIG_LED_FADE_DURATION_MS;
```

The application receives validated configuration values while Kconfig controls their availability.

------

## Key Takeaways

### `range`

Use `range` for **numeric validation**:

```kconfig
range 0 100
```

### `depends on`

Use `depends on` when a symbol should only be available if another feature is enabled:

```kconfig
depends on LED_SUBSYSTEM
```

### `if`

Use `if` when **multiple symbols share the same dependency**:

```kconfig
if LED_SUBSYSTEM

config LED_BRIGHTNESS
    ...

config LED_FADE_DURATION_MS
    ...

endif
```

### Simple Rule

```text
range
  → "What values are allowed?"

depends on
  → "When is this option available?"

if
  → "Apply this dependency to a group of options."
```

---

# Zephyr Kconfig — `menu`, `visible if`, `if`, and `menuconfig`

Kconfig provides several ways to organize configuration options in `menuconfig`.

The most important constructs are:

- `menu` → groups related options into a submenu.
- `visible if` → controls whether a menu is displayed.
- `if` → conditionally enables/disables configuration symbols.
- `menuconfig` → creates a Boolean symbol together with a submenu.

These constructs may look similar in `menuconfig`, but they have **different behavior**.

------

# `menu` — Group Options Together

The `menu` keyword creates a visual container for related configuration options.

```kconfig
menu "LED settings"

config LED_BRIGHTNESS
    int "LED brightness (0-100)"

config LED_FADE_DURATION_MS
    int "Fade duration (ms)"

endmenu
```

In `menuconfig`, this appears approximately as:

```text
LED settings  --->
```

Selecting it opens:

```text
    LED brightness (0-100)
    Fade duration (ms)
```

### Important

`menu` itself does **not create a configuration symbol**.

It is purely a way to organize existing symbols visually.

Think of it as:

```text
menu = folder
```

------

# `visible if` — Hide a Menu

`visible if` controls whether a menu is displayed.

For example:

```kconfig
menu "Expert settings"

visible if LED_ADVANCED

config LED_DEBUG
    bool "Enable LED debugging"

endmenu
```

If:

```text
LED_ADVANCED = y
```

the menu is visible:

```text
Expert settings  --->
```

If:

```text
LED_ADVANCED = n
```

the menu is hidden.

------

## Important Difference: `visible if` Does Not Add a Dependency

Consider:

```kconfig
menu "Expert settings"

visible if LED_ADVANCED

config LED_DEBUG
    bool "Enable LED debugging"

endmenu
```

The `visible if` condition controls the **visibility of the menu**.

It does **not** mean that:

```text
LED_DEBUG depends on LED_ADVANCED
```

The symbol itself does not automatically receive the same dependency.

Therefore:

```text
visible if
    → controls visibility
```

rather than:

```text
visible if
    → controls whether the symbol can work
```

------

# `if` — Conditional Configuration

An `if` block applies a dependency to the symbols inside it.

```kconfig
if LED_SUBSYSTEM

config LED_BRIGHTNESS
    int "LED brightness (0-100)"
    default 100

config LED_FADE_DURATION_MS
    int "Fade duration (ms)"
    default 500

endif
```

The symbols effectively inherit:

```kconfig
depends on LED_SUBSYSTEM
```

Therefore, when:

```text
LED_SUBSYSTEM = n
```

the dependent symbols are unavailable.

When:

```text
LED_SUBSYSTEM = y
```

they become available.

------

# `if` vs. `visible if`

This distinction is extremely important.

## `if`

```kconfig
if FEAT

config FEATURE_OPTION
    bool "Feature option"

endif
```

The symbol is actually **dependent on `FEAT`**.

Conceptually:

```kconfig
config FEATURE_OPTION
    bool "Feature option"
    depends on FEAT
```

So:

```text
FEAT = n
    ↓
FEATURE_OPTION cannot be enabled
```

------

## `visible if`

```kconfig
menu "Feature options"
    visible if FEAT

config FEATURE_OPTION
    bool "Feature option"

endmenu
```

Here, `FEAT` controls the **visibility of the menu**, not the symbol's dependency.

So:

```text
FEAT = n
    ↓
Menu is hidden
    ↓
But the symbol does not automatically depend on FEAT
```

------

# Rule of Thumb

A useful way to remember the difference is:

```text
if
    → The symbols shouldn't work without the dependency.

visible if
    → The symbols can work; just hide the menu for clarity.
```

Use `if` when there is a real configuration dependency.

Use `visible if` when you only want to simplify the user interface.

------

# `menuconfig` — Checkbox + Submenu

`menuconfig` combines two things:

1. A Boolean configuration symbol.
2. A submenu containing related configuration options.

Example:

```kconfig
menuconfig LED_SUBSYSTEM
    bool "LED Subsystem"
    default y

if LED_SUBSYSTEM

config LED_BRIGHTNESS
    int "LED brightness (0-100)"
    default 100

config LED_FADE_DURATION_MS
    int "Fade duration (ms)"
    default 500

endif
```

`menuconfig` creates the symbol:

```text
CONFIG_LED_SUBSYSTEM
```

and also provides a submenu.

------

## What Does `menuconfig` Look Like?

In `menuconfig`, you can get:

```text
[*] LED Subsystem  --->
```

The:

```text
[*]
```

is the Boolean enable/disable state.

The:

```text
--->
```

indicates that the option contains a submenu.

If the user disables it:

```text
[ ] LED Subsystem  --->
```

the dependent child options are hidden and disabled.

------

# `menu` vs. `menuconfig`

Consider:

```kconfig
menu "LED settings"

config LED_BRIGHTNESS
    int "LED brightness (0-100)"

endmenu
```

The menu is simply a folder:

```text
LED settings  --->
```

There is no enable/disable checkbox for the menu itself.

Compare this with:

```kconfig
menuconfig LED_SUBSYSTEM
    bool "LED Subsystem"
```

which creates:

```text
[*] LED Subsystem  --->
```

The user can turn the subsystem on or off.

### Simple Comparison

| Construct    | Creates Symbol? | Submenu?          | Enable/Disable? | Main Purpose              |
| ------------ | --------------- | ----------------- | --------------- | ------------------------- |
| `menu`       | No              | Yes               | No              | Organize options          |
| `visible if` | No              | Usually with menu | No              | Hide/show UI              |
| `if`         | No              | No                | No              | Add dependency to symbols |
| `menuconfig` | Yes (`bool`)    | Yes               | Yes             | Feature switch + settings |

------

# Combining `menuconfig` and `menu`

You can also use a feature switch with separate visual groups:

```kconfig
menuconfig LED_SUBSYSTEM
    bool "LED Subsystem"
    default y

if LED_SUBSYSTEM

menu "LED settings"

config LED_BRIGHTNESS
    int "LED brightness (0-100)"
    range 0 100
    default 100

config LED_FADE_DURATION_MS
    int "Fade duration (ms)"
    range 0 5000
    default 500

endmenu

endif
```

The hierarchy becomes:

```text
[*] LED Subsystem
    LED settings  --->
        LED brightness (0-100)
        Fade duration (ms)
```

This is useful for larger projects where you want both:

- a master feature switch
- organized groups of settings

------

# Mental Model

The easiest way to remember these constructs is:

```text
menu
    = Folder
    = Organizes options
    = No configuration symbol

visible if
    = Hide/show folder
    = UI visibility only
    = Does not automatically create a dependency

if
    = Dependency boundary
    = Symbols inside inherit the condition

menuconfig
    = Checkbox + Folder
    = Creates a bool configuration symbol
    = Can control whether child settings are available
```

------

# Key Takeaways

### Use `menu` when:

You only need to group related settings:

```kconfig
menu "LED settings"
    ...
endmenu
```

### Use `visible if` when:

The options are valid but you want to hide them unless a condition is true:

```kconfig
menu "Expert settings"
    visible if LED_ADVANCED
    ...
endmenu
```

### Use `if` when:

The symbols genuinely depend on a feature:

```kconfig
if LED_SUBSYSTEM
    ...
endif
```

### Use `menuconfig` when:

You want a **feature enable/disable switch plus a submenu**:

```kconfig
menuconfig LED_SUBSYSTEM
    bool "LED Subsystem"
```

## Final Rule

```text
menu
    → Organize

visible if
    → Hide

if
    → Depend

menuconfig
    → Enable/disable + organize
```

---

