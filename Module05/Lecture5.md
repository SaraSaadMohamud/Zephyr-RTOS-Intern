# Zephyr RTOS — Module 5: Boards & Custom Board Creation

## Overview

A **board** in Zephyr RTOS represents a complete hardware platform.

It is more than a physical PCB description. A Zephyr board definition provides the **hardware abstraction and configuration** required to build and run Zephyr applications on a specific hardware platform.

A board definition describes and configures:

- The **SoC (System-on-Chip)** or microcontroller
- CPU cores and processing clusters
- On-board peripherals
- GPIO assignments and pin mappings
- Hardware-specific configuration
- Default Kconfig settings
- Device Tree configuration
- Optional build and flashing configuration

Zephyr uses primarily:

- **Device Tree Source (DTS)** files to describe hardware
- **Kconfig** files to configure software and board-specific options
- **board.yml** to describe board metadata and supported SoCs/variants

------

# 1. What Is a Board?

In Zephyr, a board can be represented conceptually using the following naming structure:

```text
board_name@revision/soc/cpu_cluster/variant
```

### Example

```text
nrf5340dk/nrf5340/cpuapp
```

This identifies the **nRF5340 DK** board using:

| Component   | Meaning                        |
| ----------- | ------------------------------ |
| `nrf5340dk` | Board name                     |
| `nrf5340`   | SoC                            |
| `cpuapp`    | CPU cluster / application core |

Another example of a board revision is:

```text
board_name@1.2.0
```

Here, `1.2.0` represents a specific hardware revision.

------

## 1.1 Board Naming Components

### Board Name

The board name is the unique identifier used by Zephyr to select the hardware platform.

Example:

```text
nrf5340dk
```

### Board Revision

The revision identifies a specific hardware version.

Example:

```text
@1.2.0
```

### SoC

The **System-on-Chip (SoC)** contains the processor core and hardware peripherals.

Example:

```text
nrf5340
```

### CPU Cluster

Some SoCs contain multiple CPU cores or processing domains.

Example:

```text
cpuapp
```

represents the application core of the nRF5340.

### Variant

A variant identifies a specific build configuration of the board.

For example:

```text
ns
```

can represent a **non-secure** build variant.

------

# 2. How Zephyr Defines a Board

A board is primarily described using three types of files:

```text
DTS / Device Tree
        │
        ├── Hardware description
        ├── Peripherals
        ├── GPIOs
        └── Pin assignments
       
Kconfig
        │
        ├── Board configuration
        ├── Default options
        └── SoC selection

board.yml
        │
        ├── Board metadata
        ├── Supported SoCs
        ├── Revisions
        └── Variants
```

Together, these files allow Zephyr's build system to understand how the board is structured and how the application should be built for it.

------

# 3. Creating a Custom Board

Zephyr provides two common approaches for creating a custom board.

## Approach 1 — Copy and Rename

Start from an existing board that uses the same or a closely related MCU/SoC.

Then:

1. Copy the existing board definition.
2. Rename the board directory.
3. Rename board-specific files.
4. Modify the hardware description.
5. Update Kconfig configuration.
6. Build the application for the new board.

This approach is generally useful when an existing board already provides most of the required configuration.

------

## Approach 2 — From Scratch

Create a complete board definition manually.

This requires defining the required board files and configuring the:

- Board metadata
- SoC
- Device Tree
- Kconfig
- Default configuration
- Optional pin control
- Optional build and flashing configuration

This approach provides more control but requires a deeper understanding of Zephyr's board and build-system architecture.

------

# 4. Board Directory Structure

A minimal custom board can have a structure similar to:

```text
boards/
└── our_board/
    ├── board.yml
    ├── our_board.dts
    └── Kconfig.our_board
```

Additional files can be added when required:

```text
boards/
└── our_board/
    ├── board.yml
    ├── board.cmake
    ├── Kconfig.our_board
    ├── Kconfig.defconfig
    ├── our_board.dts
    ├── our_board_defconfig
    └── our_board_pinctrl.dtsi
```

### File Responsibilities

| File                     | Purpose                                  |
| ------------------------ | ---------------------------------------- |
| `board.yml`              | Board metadata and supported hardware    |
| `our_board.dts`          | Hardware description using Device Tree   |
| `Kconfig.our_board`      | Board Kconfig definition                 |
| `Kconfig.defconfig`      | Board-specific default Kconfig values    |
| `our_board_defconfig`    | Default configuration for the board      |
| `board.cmake`            | Board-specific build/flash configuration |
| `our_board_pinctrl.dtsi` | Pin-control configuration                |

Not every board requires all of these files.

------

# 5. Choosing an Existing Board

When using the **Copy and Rename** method, select an existing board that uses the same MCU or SoC whenever possible.

For example, if the custom board uses an:

```text
STM32F429
```

a suitable starting point could be:

```text
nucleo_f429zi
```

The existing board provides a working foundation for:

- SoC configuration
- Device Tree structure
- Kconfig configuration
- Pin configuration
- Build configuration

------

# 6. Copy and Rename Method

Assume the original board is:

```text
nucleo_f429zi
```

Copy its board directory into:

```text
boards/our_board/
```

The resulting structure may initially contain files such as:

```text
nucleo_f429zi.yaml
Kconfig.nucleo_f429zi
nucleo_f429zi_defconfig
nucleo_f429zi.dts
```

Rename the board-specific files:

```text
nucleo_f429zi.yaml
        ↓
our_board.yaml

Kconfig.nucleo_f429zi
        ↓
Kconfig.our_board

nucleo_f429zi_defconfig
        ↓
our_board_defconfig

nucleo_f429zi.dts
        ↓
our_board.dts
```

The final structure becomes:

```text
boards/
└── our_board/
    ├── board.cmake
    ├── board.yml
    ├── Kconfig.defconfig
    ├── Kconfig.our_board
    ├── our_board_defconfig
    ├── our_board.dts
    └── our_board.yaml
```

> **Note:** The exact board file structure can vary between Zephyr versions. Always follow the board format used by the Zephyr version being used.

------

# 7. `board.yml`

The `board.yml` file describes the board to Zephyr's build system and West.

A simplified structure looks like:

```yaml
board:
  name: board-name
  vendor: board-vendor

  revision:
    format: major.minor.patch
    default: 1.0.0
    exact: true

    revisions:
      - name: revA
      - name: revB

  socs:
    - name: soc-1
      variants:
        - name: variant-1
        - name: variant-2

    - name: soc-2
```

------

## 7.1 Board Metadata

For example:

```yaml
board:
  name: our_board
  vendor: our_vendor
```

This identifies the board.

------

## 7.2 Board Revisions

A board may have multiple hardware revisions.

For example:

```yaml
revision:
  format: major.minor.patch
  default: 1.0.0
  exact: true
```

Revisions can also be explicitly listed:

```yaml
revisions:
  - name: revA
  - name: revB
```

------

## 7.3 Supported SoCs

The board definition specifies which SoCs it supports:

```yaml
socs:
  - name: soc-1
```

Variants can also be defined:

```yaml
socs:
  - name: soc-1
    variants:
      - name: variant-1
      - name: variant-2
```

This allows one board definition to support multiple hardware configurations.

------

# 8. Kconfig

Zephyr uses **Kconfig** to configure software features and hardware-specific options.

A custom board typically contains:

```text
Kconfig.our_board
```

For example:

```kconfig
config BOARD_OUR_BOARD
    select SOC_STM32F429XX
```

This tells Zephyr that:

- The board is `OUR_BOARD`
- The board uses the STM32F429XX SoC

------

## 8.1 `Kconfig.defconfig`

`Kconfig.defconfig` provides board-specific default values.

Example:

```kconfig
if BOARD_OUR_BOARD

if NETWORKING

config NET_L2_ETHERNET
    default y

endif # NETWORKING

endif # BOARD_OUR_BOARD
```

This means that when:

```text
BOARD_OUR_BOARD
```

is selected, Ethernet networking can be enabled by default when networking support is enabled.

------

# 9. Device Tree

The **Device Tree** describes the hardware configuration of the board.

The board's DTS file can define:

- LEDs
- Buttons
- GPIOs
- UARTs
- SPI
- I2C
- ADC
- PWM
- Sensors
- Ethernet
- Pin assignments
- Peripheral status

The board-specific Device Tree file is typically:

```text
our_board.dts
```

------

# 10. Modifying Hardware Configuration

After copying an existing board, we can modify its Device Tree to match the actual custom hardware.

For example, suppose the original board defines an LED on one GPIO, but the custom board connects the LED to:

```text
GPIOB pin 1
```

The Device Tree can be modified:

```dts
leds {
    compatible = "gpio-leds";

    green_led_1: led_1 {
        gpios = <&gpiob 1 GPIO_ACTIVE_HIGH>;
        label = "User LD1";
    };
};
```

The important change is:

```dts
gpios = <&gpiob 1 GPIO_ACTIVE_HIGH>;
```

which assigns the LED to:

```text
GPIOB.1
```

instead of the GPIO used by the original board.

------

# 11. Device Tree and Hardware Abstraction

One of the major benefits of Device Tree is that application code does not need to hard-code hardware addresses and GPIO numbers.

For example, application code can access an LED through a logical alias:

```dts
aliases {
    led0 = &green_led_1;
};
```

The application can then use Zephyr's GPIO/LED APIs without directly depending on the physical pin number.

This provides a cleaner separation between:

```text
Application
     │
     ▼
Zephyr Driver/API
     │
     ▼
Device Tree
     │
     ▼
Physical Hardware
```

------

# 12. Build the Application for the Custom Board

Once the custom board definition is ready, the application can be built using:

```bash
west build -b our_board --board-dir boards/our_board
```

### Explanation

```text
west build
```

Starts a Zephyr build.

```text
-b our_board
```

Selects the custom board.

```text
--board-dir boards/our_board
```

Tells West where the custom board definition is located.

------

# 13. Adding the Board to the Project

Instead of specifying:

```bash
--board-dir boards/our_board
```

every time, the project can add its board directory to `BOARD_ROOT`.

In the project's `CMakeLists.txt`, place the following before `find_package()`:

```cmake
list(APPEND BOARD_ROOT ${CMAKE_CURRENT_SOURCE_DIR})

find_package(Zephyr REQUIRED HINTS $ENV{ZEPHYR_BASE})
```

Now Zephyr knows that the project itself contains board definitions.

The build command can then be simplified to:

```bash
west build -b our_board
```

------

# 14. Complete Custom Board Workflow

The complete workflow can be summarized as:

```text
                Select an Existing Board
                         │
                         ▼
             Same MCU / SoC if possible
                         │
                         ▼
                  Copy Board Files
                         │
                         ▼
                Rename Board Files
                         │
                         ▼
                 Update board.yml
                         │
                         ▼
              Update Kconfig Files
                         │
                         ▼
               Update Device Tree
                         │
                         ▼
          Modify GPIO / Peripheral Mapping
                         │
                         ▼
                  Add Defconfig
                         │
                         ▼
                 Configure BOARD_ROOT
                         │
                         ▼
                  west build -b
                         │
                         ▼
                  Test on Hardware
```

---

# 15. Creating a Custom Board From Scratch

The second approach for creating a custom Zephyr board is to build the board definition **from scratch**.

Instead of copying an existing board, we create the minimum required files ourselves and explicitly describe:

- Board metadata
- Board registration
- Software configuration
- Hardware configuration
- Device Tree
- Pin assignments

A minimal custom board repository can look like:

```text
.
├── boards/
│   └── our_board/
│       ├── board.yml
│       ├── our_board.dts
│       └── Kconfig.our_board
```

These three files provide the basic information needed to introduce the board to Zephyr.

------

# 16. Minimal Board Files

## `board.yml`

```text
board.yml
```

**Purpose:** Board metadata.

It describes information such as:

- Board name
- Vendor
- Revision
- Supported SoCs
- Variants

------

## `our_board.dts`

```text
our_board.dts
```

**Purpose:** Hardware definition.

It describes:

- SoC
- Peripherals
- Memory
- GPIOs
- Pin assignments
- Hardware relationships
- Zephyr-specific chosen nodes

------

## `Kconfig.our_board`

```text
Kconfig.our_board
```

**Purpose:** Registers the board with Zephyr's Kconfig/build system and associates it with the appropriate SoC.

------

# 17. Board Files Architecture

A complete board definition can be viewed as several layers:

```text
                         Custom Board
                              │
                              ▼
                    ┌──────────────────┐
                    │    board.yml     │
                    │    Metadata      │
                    └────────┬─────────┘
                             │
                             ▼
                    ┌──────────────────┐
                    │ Kconfig.our_board│
                    │ Board Registration│
                    └────────┬─────────┘
                             │
                             ▼
                 ┌────────────────────────┐
                 │ Board Configuration    │
                 │                        │
                 │ Kconfig.defconfig      │
                 │ our_board_defconfig    │
                 └───────────┬────────────┘
                             │
                             ▼
                 ┌────────────────────────┐
                 │   Hardware Mapping     │
                 │                        │
                 │   our_board.dts        │
                 └───────────┬────────────┘
                             │
                             ▼
                 ┌────────────────────────┐
                 │ Device Tree Includes   │
                 │                        │
                 │ SoC DTSI               │
                 │ Common DTSI             │
                 │ Pin-control DTSI        │
                 └────────────────────────┘
```

The important idea is that **different files have different responsibilities**.

------

# 18. Build System Identification

Before Zephyr can build an application for a custom board, it must first know that the board exists.

This is handled through:

```text
board.yml
Kconfig.our_board
```

Conceptually:

```text
board.yml
    │
    └── Identifies the board
            │
            ▼
Kconfig.our_board
    │
    └── Registers/configures the board
            │
            ▼
       Zephyr Build System
```

After the board is recognized, additional configuration files define its software defaults and hardware.

------

# 19. Board Configuration

Board configuration is mainly handled through:

```text
Kconfig.defconfig
our_board_defconfig
```

These files are used for **software features and default settings**.

### `Kconfig.defconfig`

Provides conditional defaults for Kconfig options.

### `our_board_defconfig`

Contains board-specific configuration that should be enabled by default.

For example:

```text
our_board_defconfig
```

may enable:

```text
MPU
Hardware stack protection
Serial
Console
UART console
GPIO
```

------

# 20. Hardware Mapping

The hardware mapping is primarily described by:

```text
our_board.dts
```

The board Device Tree usually includes additional Device Tree files.

For example:

```text
our_board.dts
      │
      ├── SoC DTSI
      │
      ├── Common board DTSI
      │
      └── Pin-control DTSI
```

A possible structure is:

```text
our_board.dts
    │
    ├── stm32f429Xi.dtsi
    │
    ├── our_board_common.dtsi
    │
    └── our_board-pinctrl.dtsi
```

The included files provide reusable hardware definitions instead of placing everything directly inside the board DTS.

------

# 21. Entry Point: `our_board.dts`

The main Device Tree file is:

```text
our_board.dts
```

A simplified example:

```dts
/dts-v1/;

#include <st/f4/stm32f429Xi.dtsi>
#include <st/f4/stm32f429zitx-pinctrl.dtsi>
#include <zephyr/dt-bindings/input/input-event-codes.h>

/ {
    model = "Our Board";
    compatible = "iomico,our-board";

    chosen {
        zephyr,flash = &flash0;
        zephyr,sram = &sram0;
        zephyr,code-partition = &slot0_partition;
    };
};
```

------

# 22. Device Tree Version

The first line:

```dts
/dts-v1/;
```

specifies the Device Tree source version.

It is normally placed at the beginning of the DTS file.

------

# 23. Including the SoC Definition

The board DTS includes the appropriate SoC Device Tree description:

```dts
#include <st/f4/stm32f429Xi.dtsi>
```

The SoC DTSI provides definitions for hardware that belongs to the MCU itself, such as:

- CPU
- Flash
- SRAM
- Timers
- UARTs
- SPI
- I2C
- ADC
- GPIO controllers
- Other SoC peripherals

This allows the board DTS to build upon the existing SoC description instead of redefining the MCU from scratch.

------

# 24. Including Pin-Control Definitions

A board can also include a pin-control Device Tree file:

```dts
#include <st/f4/stm32f429zitx-pinctrl.dtsi>
```

Pin control describes how physical MCU pins are configured for different peripheral functions.

For example, a pin may be configured as:

```text
GPIO
UART TX
UART RX
SPI
I2C
ADC
PWM
```

------

# 25. Board Identity

The root node identifies the board:

```dts
/ {
    model = "Our Board";
    compatible = "iomico,our-board";
};
```

### `model`

Human-readable board name:

```dts
model = "Our Board";
```

### `compatible`

Machine-readable compatibility string:

```dts
compatible = "iomico,our-board";
```

The compatible string is used by Device Tree and drivers to determine hardware compatibility.

------

# 26. The `chosen` Node

The `chosen` node tells Zephyr which hardware resources should be used for specific system purposes.

Example:

```dts
chosen {
    zephyr,flash = &flash0;
    zephyr,sram = &sram0;
    zephyr,code-partition = &slot0_partition;
};
```

### Flash

```dts
zephyr,flash = &flash0;
```

Selects the Flash memory device.

### SRAM

```dts
zephyr,sram = &sram0;
```

Selects the regular RAM used by Zephyr.

### Code Partition

```dts
zephyr,code-partition = &slot0_partition;
```

Selects the partition used for the application code.

Conceptually:

```text
chosen
  │
  ├── Flash ───────────────► Program storage
  │
  ├── SRAM ────────────────► Runtime memory
  │
  └── Code partition ──────► Application image
```

------

# 27. Device Tree Include Files

Complex board definitions are usually divided into multiple files.

For example:

```text
our_board.dts
│
├── SoC DTSI
│     └── STM32F429 hardware
│
├── Common board DTSI
│     └── Shared board configuration
│
└── Pin-control DTSI
      └── Pin assignments
```

This improves:

- Reusability
- Maintainability
- Organization
- Separation of hardware responsibilities

------

# 28. Pin-Control Configuration

A pin-control DTSI can contain definitions such as:

```dts
#include <zephyr/dt-bindings/pinctrl/stm32-pinctrl.h>

/ {
    soc {
        pinctrl: pin-controller@40020000 {

            /omit-if-no-ref/
            adc1_in0_pa0: adc1_in0_pa0 {
                pinmux = <STM32_PINMUX('A', 0, ANALOG)>;
            };

            /omit-if-no-ref/
            adc1_in1_pa1: adc1_in1_pa1 {
                pinmux = <STM32_PINMUX('A', 1, ANALOG)>;
            };
        };
    };
};
```

------

# 29. Understanding `STM32_PINMUX`

The following:

```dts
STM32_PINMUX('A', 0, ANALOG)
```

describes the pin configuration.

Conceptually:

```text
STM32_PINMUX(
    Port,
    Pin,
    Function
)
```

So:

```dts
STM32_PINMUX('A', 0, ANALOG)
```

means:

```text
Port A
Pin 0
Function = Analog
```

Similarly:

```dts
STM32_PINMUX('A', 1, ANALOG)
```

represents:

```text
PA1 → Analog
```

------

# 30. `/omit-if-no-ref/`

The following syntax:

```dts
/omit-if-no-ref/
```

allows the Device Tree compiler to omit a node if nothing references it.

For example:

```dts
/omit-if-no-ref/
adc1_in0_pa0: adc1_in0_pa0 {
    pinmux = <STM32_PINMUX('A', 0, ANALOG)>;
};
```

This helps keep the generated Device Tree optimized when a particular pin configuration is not actually used.

------

# 31. `Kconfig.our_board`

The board also needs to be registered through Kconfig.

A simplified example is:

```kconfig
config BOARD_OUR_BOARD
    select SOC_STM32F429XX
```

The important relationship is:

```text
BOARD_OUR_BOARD
       │
       ▼
SOC_STM32F429XX
```

This tells Zephyr that selecting the custom board also selects the appropriate SoC configuration.

------

# 32. `Kconfig.defconfig`

Board-specific default configuration can be placed in:

```text
Kconfig.defconfig
```

For example:

```kconfig
if BOARD_OUR_BOARD

if NETWORKING

config NET_L2_ETHERNET
    default y

endif # NETWORKING

endif # BOARD_OUR_BOARD
```

The configuration is conditional on the selected board:

```text
BOARD_OUR_BOARD
       │
       └── NETWORKING
              │
              └── NET_L2_ETHERNET = y
```

This allows the board to provide sensible defaults without forcing those options globally.

------

# 33. `our_board_defconfig`

The board's default configuration can enable the features required for the hardware.

Example:

```config
# Enable MPU
CONFIG_ARM_MPU=y

# Enable hardware stack protection
CONFIG_HW_STACK_PROTECTION=y

# Enable serial
CONFIG_SERIAL=y

# Enable console
CONFIG_CONSOLE=y

# Enable UART console
CONFIG_UART_CONSOLE=y

# Enable GPIO
CONFIG_GPIO=y
```

These options establish a useful default configuration for the board.

------

# 34. Optional `CMakeLists.txt`

Some boards require additional C source code.

In that case, the board directory can contain:

```text
CMakeLists.txt
board.c
```

Example:

```cmake
zephyr_library()

zephyr_library_sources(board.c)
```

This tells Zephyr to include `board.c` in the board-specific build.

------

# 35. Optional `board.c`

A board-specific C file can contain initialization code that must execute during the boot process.

Example:

```c
#include <zephyr/init.h>
#include <zephyr/kernel.h>

static int board_our_board_init(void)
{
    /* Board initialization before boot */

    return 0;
}
```

The exact initialization mechanism and initialization priority should be selected according to what the board actually needs.

> **Important:** A `board.c` file is optional. Do not add board-specific C code when Device Tree and existing Zephyr drivers can already describe and initialize the hardware.

------

# 36. Optional `board.cmake`

A board can also provide:

```text
board.cmake
```

This file defines board-specific flashing/debugging support.

For example:

```cmake
board_runner_args(stm32cubeprogrammer "--port=swd"
                                      "--reset-mode=hw")

board_runner_args(jlink "--device=STM32F429ZI"
                        "--speed=4000")

include(${ZEPHYR_BASE}/boards/common/stm32cubeprogrammer.board.cmake)
include(${ZEPHYR_BASE}/boards/common/openocd-stm32.board.cmake)
include(${ZEPHYR_BASE}/boards/common/jlink.board.cmake)
```

------

# 3. Understanding `board_runner_args`

For STM32CubeProgrammer:

```cmake
board_runner_args(stm32cubeprogrammer
                  "--port=swd"
                  "--reset-mode=hw")
```

This provides arguments to the STM32CubeProgrammer runner.

For J-Link:

```cmake
board_runner_args(jlink
                  "--device=STM32F429ZI"
                  "--speed=4000")
```

This specifies:

```text
Device → STM32F429ZI
Debug speed → 4000 kHz
```

The exact runner options depend on the selected runner and Zephyr version.

------

# 38. Including Common Board Runner Files

Instead of implementing flashing/debugging logic from scratch, the board can reuse Zephyr's common runner configurations:

```cmake
include(${ZEPHYR_BASE}/boards/common/stm32cubeprogrammer.board.cmake)
include(${ZEPHYR_BASE}/boards/common/openocd-stm32.board.cmake)
include(${ZEPHYR_BASE}/boards/common/jlink.board.cmake)
```

This allows the custom board to use existing Zephyr infrastructure.

------

# 39. Final From-Scratch Board Structure

After adding the optional components, the board can have a structure similar to:

```text
boards/
└── our_board/
    ├── board.yml
    ├── our_board.dts
    ├── Kconfig.our_board
    ├── Kconfig.defconfig
    ├── our_board_defconfig
    ├── our_board-pinctrl.dtsi
    ├── our_board_common.dtsi
    ├── CMakeLists.txt
    ├── board.c
    └── board.cmake
```

Not every board needs every file.

The required files depend on the hardware and the functionality that must be customized.

------

# 45. Build the Custom Board

Once the board definition has been created, build the application using:

```bash
west build -b our_board
```

The command tells West:

```text
west build
    │
    └── Build the current application
             │
             └── Target board = our_board
```

If the board directory has not been registered through `BOARD_ROOT`, specify it explicitly:

```bash
west build -b our_board --board-dir boards/our_board
```

------

# 46. Flash the Board

After a successful build, the application can be flashed using:

```bash
west flash
```

The selected runner from the board configuration is then used to program the target hardware.

Conceptually:

```text
west build
     │
     ▼
Application Image
     │
     ▼
west flash
     │
     ▼
Debug / Flash Runner
     │
     ▼
Custom Board
```

------

# 47. Complete From-Scratch Workflow

The complete process can be summarized as:

```text
Create Board Directory
        │
        ▼
Create board.yml
        │
        ▼
Create Kconfig.our_board
        │
        ▼
Create our_board.dts
        │
        ▼
Configure SoC
        │
        ▼
Configure Board Hardware
        │
        ▼
Configure Pin Control
        │
        ▼
Create Kconfig.defconfig
        │
        ▼
Create our_board_defconfig
        │
        ▼
Optional:
CMakeLists.txt / board.c / board.cmake
        │
        ▼
Register BOARD_ROOT
        │
        ▼
west build -b our_board
        │
        ▼
west flash
        │
        ▼
Run on Hardware
```

------

# 48. Complete Architecture Overview

The relationship between the major components can be visualized as:

```text
                         Zephyr Application
                                │
                                ▼
                         Zephyr Build System
                                │
              ┌─────────────────┴─────────────────┐
              │                                   │
              ▼                                   ▼
         Board Metadata                       Kconfig
          board.yml                      Kconfig.our_board
              │                                   │
              │                                   ▼
              │                            SoC Selection
              │                                   │
              ▼                                   ▼
        Board Identity                     Board Defaults
                                                │
                                  ┌─────────────┴─────────────┐
                                  ▼                           ▼
                         Kconfig.defconfig         our_board_defconfig
                                  │
                                  └─────────────┬─────────────┘
                                                │
                                                ▼
                                         Hardware Definition
                                                │
                                                ▼
                                        our_board.dts
                                                │
                           ┌────────────────────┼────────────────────┐
                           │                    │                    │
                           ▼                    ▼                    ▼
                         SoC DTSI          Common DTSI        Pin Control DTSI
                           │                    │                    │
                           └────────────────────┼────────────────────┘
                                                │
                                                ▼
                                           Hardware
```

------

# 49. Minimal vs. Complete Board

It is important to distinguish between the **minimal board definition** and a **complete board implementation**.

### Minimal

```text
our_board/
├── board.yml
├── our_board.dts
└── Kconfig.our_board
```

### Extended

```text
our_board/
├── board.yml
├── our_board.dts
├── Kconfig.our_board
├── Kconfig.defconfig
├── our_board_defconfig
├── our_board-pinctrl.dtsi
├── our_board_common.dtsi
├── CMakeLists.txt
├── board.c
└── board.cmake
```

The additional files are introduced only when the board requires their functionality.

------

# 50. Key Takeaways

The **From Scratch** method demonstrates how a Zephyr board is constructed from its fundamental components.

### Board Identification

```text
board.yml
```

Defines what the board is.

### Board Registration

```text
Kconfig.our_board
```

Makes the board available to Zephyr's configuration/build system and connects it to the appropriate SoC.

### Software Defaults

```text
Kconfig.defconfig
our_board_defconfig
```

Define board-specific software configuration and defaults.

### Hardware Definition

```text
our_board.dts
```

Describes the actual hardware and connects the board to the SoC, memory, peripherals, and pins.

### Pin Configuration

```text
our_board-pinctrl.dtsi
```

Defines the physical pin mappings and peripheral pin functions.

### Optional Board Code

```text
CMakeLists.txt
board.c
```

Can provide additional board-specific initialization when required.

### Flashing Configuration

```text
board.cmake
```

Provides board-specific runner configuration for flashing and debugging.

------

# 51. Final Commands

With the board correctly registered:

```bash
west build -b our_board
```

Flash the generated application:

```bash
west flash
```

If the board is not automatically discovered:

```bash
west build -b our_board --board-dir boards/our_board
```

------

## Module 5 — Final Mental Model

The most important concept from this module is:

```text
                    CUSTOM BOARD
                         │
       ┌─────────────────┼─────────────────┐
       │                 │                 │
       ▼                 ▼                 ▼
   board.yml          Kconfig          Device Tree
       │                 │                 │
       │                 │                 │
   "What is it?"    "How is it      "What hardware
                    configured?"       does it have?"
       │                 │                 │
       └─────────────────┼─────────────────┘
                         │
                         ▼
                 Zephyr Build System
                         │
                         ▼
                    Application
                         │
                         ▼
                     Hardware
```

A Zephyr board definition therefore acts as the **bridge between the application and the physical hardware**.

The application describes **what the system does**, while the board definition describes **where and how that functionality is implemented on the target hardware**.

---

