# Zephyr RTOS — Module 06: Device Drivers and Modules

## Table of Contents

- [1. What Is a Zephyr Driver?](#1-what-is-a-zephyr-driver)
- [2. Module vs. Driver](#2-module-vs-driver)
- [3. Zephyr Driver Lifecycle](#3-zephyr-driver-lifecycle)
- [4. Key Takeaways](#4-key-takeaways)

## 1. What Is a Zephyr Driver?

A **device driver** is a software component that provides an abstraction layer between an application and a hardware peripheral.

It implements or exposes the appropriate Zephyr API, allowing applications to interact with hardware without handling all the hardware-specific implementation details.

### 1.1 Purpose of a Driver

Device drivers help applications:

- Access hardware through standardized APIs.
- Separate application logic from hardware-specific implementation details.
- Reuse application code across supported boards and devices.
- Reduce direct dependencies on peripheral registers and low-level communication details.

### 1.2 Examples of Zephyr Drivers

| Driver | Purpose                                                      |
| ------ | ------------------------------------------------------------ |
| GPIO   | Controls digital input and output pins.                      |
| UART   | Provides asynchronous or polling-based serial communication interfaces, depending on the API used. |
| SPI    | Supports communication with SPI peripherals.                 |
| I2C    | Supports communication with I2C peripherals.                 |
| Sensor | Provides standardized interfaces for retrieving sensor measurements. |

### 1.3 Hardware Abstraction

A typical driver-based architecture consists of three layers:

```
+--------------------------------+
|          Application           |
+--------------------------------+
                |
                v
+--------------------------------+
|       Standard Zephyr API      |
+--------------------------------+
                |
                v
+--------------------------------+
|          Device Driver         |
+--------------------------------+
                |
                v
+--------------------------------+
|        Hardware Peripheral     |
+--------------------------------+
```

The application uses an API, while the driver implements the operations required to interact with the underlying hardware.

## 2. Module vs. Driver

A **module** is a packaging and integration concept, while a **driver** is a software component that implements hardware-related functionality.

Zephyr uses West to manage workspaces and integrate repositories, including external modules.

### 2.1 Comparison

| Feature          | Module                                                    | Driver                                                       |
| ---------------- | --------------------------------------------------------- | ------------------------------------------------------------ |
| Definition       | A packaged and integrated software component              | An implementation that provides access to hardware functionality |
| Purpose          | Organizes and integrates reusable software                | Abstracts a device or peripheral behind an API               |
| Typical contents | Drivers, libraries, HALs, utilities, and other software   | Hardware-specific implementation and related support code    |
| Examples         | An external repository containing sensor drivers or a HAL | A sensor driver, UART driver, or SPI driver                  |

### 2.2 Relationship Between Modules and Drivers

A module may contain one or more drivers, but it does not have to contain any drivers.

For example, an external repository may contain:

```
external_module/
├── CMakeLists.txt
├── Kconfig
├── zephyr/
│   └── module.yml
├── drivers/
│   └── sensor/
│       └── sensor_driver.c
├── lib/
│   └── utility.c
└── README.md
```

This is an illustrative directory structure. Actual module layouts vary.

In this example:

- The repository represents the module.
- `sensor_driver.c` implements a driver.
- `utility.c` represents supporting library functionality.
- The module metadata and build files support integration with Zephyr.

**Key distinction:** A module is a packaging concept; a driver is a software implementation pattern.

## 3. Zephyr Driver Lifecycle

A Zephyr driver typically passes through several stages before an application can use it.

### 3.1 Device Tree Declaration

The Device Tree describes the hardware and its configuration.

Example:

```
my_sensor: sensor@48 {
    compatible = "our,driver";
    reg = <0x48>;
    status = "okay";
};
```

The `compatible` property identifies the device type, while other properties describe its resources and configuration.

This example assumes a device on a bus where `reg` represents an address. A real node must follow the appropriate bus and binding conventions.

### 3.2 Compatible String

The `compatible` string connects a Device Tree node to the corresponding driver support.

For example:

```
compatible = "our,driver";
```

Driver code can use Device Tree macros to identify compatible nodes and obtain their configuration.

### 3.3 Device Tree Binding

A Device Tree binding defines the supported properties for a compatible device.

Bindings are commonly written in YAML and describe:

- The compatible string.
- Required properties.
- Optional properties.
- Property types and constraints.

Bindings describe the hardware configuration schema; they do not implement the driver.

### 3.4 Kconfig Configuration

Kconfig controls whether the driver and its features are enabled.

Simplified example:

```
config MY_SENSOR
    bool "Our sensor driver"
    depends on I2C
```

The configuration symbol `CONFIG_MY_SENSOR` can be used by the build system to determine whether the driver should be compiled.

### 3.5 CMake Build Integration

CMake specifies which source files are compiled into the application.

Example:

```
zephyr_library()

zephyr_library_sources_ifdef(CONFIG_MY_SENSOR sensor_driver.c)
```

This conditionally adds the driver source file when `CONFIG_MY_SENSOR` is enabled.

The exact CMake configuration depends on the driver's location and module structure.

### 3.6 Driver Instance Creation

The driver uses Zephyr's device model to associate its implementation with the hardware described in the Device Tree.

Device-definition mechanisms include macros such as:

- `DEVICE_DT_DEFINE()`
- `DEVICE_DT_INST_DEFINE()`

These mechanisms define device objects and their initialization behavior.

The driver must be correctly configured and compiled, and its initialization must succeed before the device can be used as intended.

### 3.7 Application Access

The application obtains a device handle and uses the appropriate Zephyr API.

For example, `DEVICE_DT_GET()` can be used to obtain a device reference associated with a Device Tree node.

Drivers that support Zephyr's device model generally expose readiness information that applications can check before accessing the device.

The actual API depends on the device type.

### 3.8 Lifecycle Summary

```
Device Tree Node
       |
       v
Compatible String
       |
       v
Device Tree Binding
       |
       v
Kconfig Configuration
       |
       v
CMake Build Integration
       |
       v
Driver Compilation and Device Definition
       |
       v
Device Initialization
       |
       v
Application Access Through the API
```

The sequence is a conceptual overview. Some build-time operations occur together, and the exact process varies by driver.

## 4. Key Takeaways

- A driver provides an interface between an application and a hardware peripheral.
- Zephyr offers standardized APIs for peripherals such as GPIO, UART, SPI, I2C, and sensors.
- A module is a packaging and integration concept; a driver implements hardware-related functionality.
- A module can contain multiple drivers, libraries, HALs, and utilities.
- Device Tree describes hardware and its configuration.
- Compatible strings connect Device Tree nodes to supported device implementations.
- Device Tree bindings define valid properties and configuration constraints.
- Kconfig controls feature and driver configuration.
- CMake integrates driver source files into the build.
- Zephyr's device model provides device objects that applications can access through the appropriate APIs.

**Core principle:** Zephyr separates hardware description, build-time configuration, driver implementation, and application-level access to make embedded software more modular and reusable.

---

# Part2: Zephyr RTOS — Module 06: Custom Device Drivers 

## Table of Contents

- [1. Zephyr Driver Architecture](#1-zephyr-driver-architecture)
- [2. Module vs. Driver](#2-module-vs-driver)
- [3. Custom Driver Module Structure](#3-custom-driver-module-structure)
- [4. Device Tree Node and Binding](#4-device-tree-node-and-binding)
- [5. Connecting Device Tree to C Code](#5-connecting-device-tree-to-c-code)
- [6. Kconfig and CMake Integration](#6-kconfig-and-cmake-integration)
- [7. Implementing a Custom Sensor Driver](#7-implementing-a-custom-sensor-driver)
- [8. Driver Initialization](#8-driver-initialization)
- [9. Multiple Device Instances](#9-multiple-device-instances)
- [10. Key Takeaways](#10-key-takeaways)

---

## 1. Zephyr Driver Architecture

A **device driver** provides an abstraction between an application and a hardware peripheral. It exposes an API while handling hardware-specific operations.

A driver architecture can be viewed as three connected parts:

| Component                        | Responsibility                                      |
| -------------------------------- | --------------------------------------------------- |
| **Device Driver API**            | Defines operations available to applications.       |
| **Device Driver Instance**       | Represents one configured device on the board.      |
| **Device Driver Implementation** | Implements the operations that access the hardware. |

For example, two I2C sensors can share one driver implementation and API while maintaining separate configurations and runtime data.

```text
                  Application
                       |
                       v
              Standard Sensor API
                       |
              +--------+--------+
              |                 |
              v                 v
        Sensor Instance 1  Sensor Instance 2
              |                 |
              +--------+--------+
                       |
                       v
              Shared Driver Logic
                       |
                       v
                   I2C Bus
```

### 1.1 Device Objects

Zephyr represents devices using its device model. Conceptually, a device instance is associated with:

- **Configuration:** Usually fixed hardware settings.
- **API:** The operations supported by the device.
- **Runtime data:** Mutable state maintained while the device operates.

The slide's `struct device` illustration is conceptual. Do not redeclare Zephyr's internal device structure in an application; use Zephyr's device macros and documented APIs.

---

## 2. Module vs. Driver

A **module** is a packaging and integration concept. A **driver** is a software implementation that provides hardware functionality.

| Feature          | Module                                                       | Driver                                               |
| ---------------- | ------------------------------------------------------------ | ---------------------------------------------------- |
| Meaning          | A packaged and integrated software component                 | Implements access to a device or peripheral          |
| Typical contents | Drivers, libraries, HALs, utilities                          | Hardware-specific code and related support           |
| Integration      | Often discovered and managed through West/Zephyr module integration | Integrated through the device model and build system |
| Example          | External repository containing sensor drivers and utilities  | A particular I2C sensor driver                       |

A module may contain several drivers, libraries, HALs, or utilities. **Not every module is a driver.**

---

## 3. Custom Driver Module Structure

An external driver module may use a structure such as:

```text
our-driver-module/
├── include/
│   └── our_drivers/
│       └── our_driver.h
├── drivers/
│   └── our_driver/
│       ├── our_driver.c
│       ├── CMakeLists.txt
│       └── Kconfig
├── dts/
│   └── bindings/
│       └── sensor/
│           └── our,driver.yaml
├── zephyr/
│   └── module.yml
├── CMakeLists.txt
└── Kconfig
```

The exact layout depends on the module and Zephyr version.

| File                               | Purpose                                                  |
| ---------------------------------- | -------------------------------------------------------- |
| `include/our_drivers/our_driver.h` | Public declarations for applications or other components |
| `drivers/our_driver/our_driver.c`  | Driver implementation                                    |
| Driver `CMakeLists.txt`            | Adds driver source files to the build                    |
| Driver `Kconfig`                   | Defines driver configuration options                     |
| `our,driver.yaml`                  | Defines the Device Tree binding                          |
| `zephyr/module.yml`                | Describes module integration points                      |
| Root `CMakeLists.txt`              | Module-level build integration                           |
| Root `Kconfig`                     | Module-level configuration integration                   |

### 3.1 Module Metadata

Example `zephyr/module.yml`:

```yaml
name: our_custom_drivers

build:
  cmake: .
  kconfig: Kconfig
  settings:
    dts_root: .
```

This tells Zephyr where to find the module's CMake, Kconfig, and Device Tree binding roots. The module must also be discoverable through the workspace or the applicable Zephyr module integration mechanism.

---

## 4. Device Tree Node and Binding

Device Tree describes a hardware instance. A binding defines which properties are valid for a particular `compatible` string.

### 4.1 Example Device Tree Node

For an I2C sensor, the node normally appears beneath its I2C controller:

```dts
&i2c1 {
    status = "okay";

    sensor: sensor@76 {
        compatible = "our,driver";
        reg = <0x76>;
        calibration = <100>;
        status = "okay";
    };
};
```

This example assumes that `i2c1` exists on the selected board and that address `0x76` is valid for the sensor.

- `compatible`: Identifies the supported device type.
- `reg`: Specifies the address or resource according to the parent bus.
- `calibration`: An example driver-specific integer property.
- `status`: Enables or disables the node.

### 4.2 Example Binding

Example `dts/bindings/sensor/our,driver.yaml`:

```yaml
description: Our custom I2C sensor driver

compatible: "our,driver"

include: i2c-device.yaml

properties:
  calibration:
    type: int
    required: true
```

The binding defines the supported compatible string and its properties. The `i2c-device.yaml` include supplies common I2C-device binding properties.

**Naming consistency matters:** the property name used in the DTS and C code must match the binding. This example uses `calibration` (correct spelling).

### 4.3 Device Tree and Kconfig

Zephyr can generate a hardware-presence Kconfig symbol for an enabled compatible node. For this example, the symbol is typically:

```text
DT_HAS_OUR_DRIVER_ENABLED
```

This indicates that matching hardware is enabled in the processed Device Tree. A separate driver option controls whether the driver itself is enabled.

---

## 5. Connecting Device Tree to C Code

### 5.1 `DT_DRV_COMPAT`

The DTS compatible string is:

```dts
compatible = "our,driver";
```

The corresponding C source typically defines:

```c
#define DT_DRV_COMPAT our_driver
```

Zephyr's driver-instance macros use this convention to find matching Device Tree nodes.

### 5.2 Reading Device Tree Properties

Given this DTS property:

```dts
calibration = <100>;
```

the driver can retrieve its compile-time value with:

```c
DT_INST_PROP(inst, calibration)
```

Example configuration structure and initializer:

```c
#define DT_DRV_COMPAT our_driver

struct our_driver_config {
    struct i2c_dt_spec i2c;
    int calibration;
};

#define OUR_DRIVER_CONFIG(inst)                         \
    static const struct our_driver_config config_##inst = { \
        .i2c = I2C_DT_SPEC_INST_GET(inst),               \
        .calibration = DT_INST_PROP(inst, calibration), \
    };
```

This snippet illustrates property extraction; by itself, it does not define a complete device instance. The property is resolved during compilation, so no runtime Device Tree parsing is required.

---

## 6. Kconfig and CMake Integration

### 6.1 Driver Kconfig

A simplified driver option might look like:

```kconfig
menuconfig OUR_DRIVER
    bool "Our Custom Driver"
    default y
    depends on DT_HAS_OUR_DRIVER_ENABLED
    select I2C

    help
      Enable the custom I2C sensor driver.
```

- `OUR_DRIVER` creates the configuration option `CONFIG_OUR_DRIVER`.
- `default y` enables it by default when its dependencies are satisfied.
- `depends on DT_HAS_OUR_DRIVER_ENABLED` restricts it to configurations with matching enabled hardware.
- `select I2C` selects the I2C configuration symbol.

**Caution:** `select` should be used carefully because it can enable a symbol without satisfying all of that symbol's normal dependencies. Follow the conventions used by the target Zephyr version and existing drivers.

### 6.2 Driver CMake

Example `drivers/our_driver/CMakeLists.txt`:

```cmake
if(CONFIG_OUR_DRIVER)
    zephyr_library()
    zephyr_library_sources(our_driver.c)
endif()
```

This adds the source file to the build when `CONFIG_OUR_DRIVER` is enabled. The exact arrangement depends on the module's CMake integration.

### 6.3 Integration Flow

1. Device Tree declares an enabled node with `compatible = "our,driver"`.
2. Zephyr processes the node and makes the matching hardware-presence symbol available.
3. Kconfig determines whether `CONFIG_OUR_DRIVER` is enabled.
4. CMake includes `our_driver.c` when the option is enabled.
5. Driver code defines device instances for the matching nodes.
6. Zephyr initializes the devices according to their initialization settings.

Compilation alone does not guarantee that an instance exists or initializes successfully.

---

## 7. Implementing a Custom Sensor Driver

When a device fits an existing Zephyr subsystem, implementing that subsystem's API lets applications use standardized operations.

Common API headers include:

| Subsystem | Header                    |
| --------- | ------------------------- |
| Sensor    | `zephyr/drivers/sensor.h` |
| GPIO      | `zephyr/drivers/gpio.h`   |
| PWM       | `zephyr/drivers/pwm.h`    |
| I2C       | `zephyr/drivers/i2c.h`    |
| SPI       | `zephyr/drivers/spi.h`    |
| UART      | `zephyr/drivers/uart.h`   |

### 7.1 C File Skeleton

```c
#define DT_DRV_COMPAT our_driver

#include <zephyr/drivers/i2c.h>
#include <zephyr/drivers/sensor.h>
#include <zephyr/kernel.h>
#include <zephyr/logging/log.h>

LOG_MODULE_REGISTER(our_driver, CONFIG_SENSOR_LOG_LEVEL);
```

The compatible macro links instance macros to the appropriate Device Tree nodes. The logging registration enables Zephyr logging for the driver.

### 7.2 Configuration and Runtime Data

```c
struct our_driver_config {
    struct i2c_dt_spec i2c;
    int calibration;
};

struct our_driver_data {
    int32_t last_temp;
    int32_t last_press;
};
```

- **Configuration** stores hardware settings that are generally fixed after compilation.
- **Runtime data** stores mutable state, such as the latest measurements.

Each device instance can have its own configuration and runtime data.

### 7.3 Private I2C Helpers

```c
static inline int
our_driver_internal(const struct device *dev,
                    uint8_t reg,
                    void *buf,
                    size_t size)
{
    const struct our_driver_config *cfg = dev->config;

    return i2c_burst_read_dt(&cfg->i2c, reg, buf, size);
}
```

A register-write helper can be implemented as follows:

```c
static inline int
our_driver_reg_write(const struct device *dev,
                     uint8_t reg,
                     uint8_t value)
{
    const struct our_driver_config *cfg = dev->config;

    return i2c_reg_write_byte_dt(&cfg->i2c, reg, value);
}
```

These helpers reuse the configured I2C specification and return status codes so the caller can handle communication errors. The actual register operations depend on the sensor's datasheet.

### 7.4 Sensor API Callbacks

```c
static int
our_driver_sample_fetch(const struct device *dev,
                        enum sensor_channel chan);

static int
our_driver_channel_get(const struct device *dev,
                       enum sensor_channel chan,
                       struct sensor_value *val);
```

Their responsibilities differ:

- `sample_fetch()` communicates with the hardware and updates the driver's runtime data.
- `channel_get()` returns a stored measurement in the standard `sensor_value` representation.

Implementations should handle unsupported channels and hardware errors appropriately.

### 7.5 Registering the Sensor API

```c
static DEVICE_API(sensor, our_driver_api) = {
    .sample_fetch = our_driver_sample_fetch,
    .channel_get  = our_driver_channel_get,
};
```

This associates the standard Sensor API operations with the driver's callback functions.

An application can then use generic calls:

```c
sensor_sample_fetch(dev);

sensor_channel_get(dev,
                   SENSOR_CHAN_AMBIENT_TEMP,
                   &val);
```

Production code should check the return value of each call and ensure the device handle is valid and ready. The exact API structure should be verified against the Zephyr version in use.

---

## 8. Driver Initialization

A driver's initialization function prepares the hardware for use. For an I2C sensor, typical tasks include:

1. Checking whether the I2C controller is ready.
2. Resetting the sensor when required.
3. Verifying the device ID.
4. Applying configuration and calibration.
5. Returning an appropriate status code.

Example:

```c
static int our_driver_init(const struct device *dev)
{
    const struct our_driver_config *cfg = dev->config;

    if (!i2c_is_ready_dt(&cfg->i2c)) {
        return -ENODEV;
    }

    /* Reset the sensor if required. */
    /* Verify the device ID. */
    /* Apply calibration and configuration. */

    return 0;
}
```

This is a skeleton, not a complete hardware implementation. A real driver must perform the required device-specific operations and report initialization failures.

### 8.1 Initialization Levels

| Level          | Typical purpose                                              |
| -------------- | ------------------------------------------------------------ |
| `EARLY`        | Very early initialization before normal kernel services are available |
| `PRE_KERNEL_1` | Early hardware initialization                                |
| `PRE_KERNEL_2` | Initialization following the first pre-kernel stage          |
| `POST_KERNEL`  | Initialization after the kernel is running                   |
| `APPLICATION`  | Application-level initialization                             |
| `SMP`          | Secondary-CPU initialization in supported SMP configurations |

The device's dependencies determine the appropriate initialization level. For example, an I2C sensor must not initialize before its I2C controller is ready.

### 8.2 Initialization Priority

```c
DEVICE_DT_INST_DEFINE(
    inst,
    our_driver_init,
    NULL,
    &data_##inst,
    &cfg_##inst,
    POST_KERNEL,
    CONFIG_SENSOR_INIT_PRIORITY,
    &our_driver_api
);
```

The parameters specify the instance, initialization function, optional power-management control, runtime data, configuration, initialization level, priority, and API.

Within the same initialization level, a lower numerical priority runs earlier than a higher one. Choose the level and priority according to the dependencies of the actual device.

---

## 9. Multiple Device Instances

One driver implementation can support multiple matching Device Tree nodes. Each instance receives its own configuration, runtime data, and device object.

Example:

```dts
&i2c1 {
    status = "okay";

    sensor0: sensor@48 {
        compatible = "our,driver";
        reg = <0x48>;
        calibration = <100>;
        status = "okay";
    };

    sensor1: sensor@49 {
        compatible = "our,driver";
        reg = <0x49>;
        calibration = <105>;
        status = "okay";
    };
};
```

Both sensors use the same driver code but have different addresses and calibration values.

### 9.1 Instance Definition Macro

```c
#define OUR_DRIVER_DEFINE(inst)                          \
    static struct our_driver_data data_##inst;           \
    static const struct our_driver_config cfg_##inst = { \
        .i2c = I2C_DT_SPEC_INST_GET(inst),               \
        .calibration = DT_INST_PROP(inst, calibration),  \
    };                                                   \
    DEVICE_DT_INST_DEFINE(                               \
        inst,                                            \
        our_driver_init,                                 \
        NULL,                                            \
        &data_##inst,                                    \
        &cfg_##inst,                                     \
        POST_KERNEL,                                     \
        CONFIG_SENSOR_INIT_PRIORITY,                     \
        &our_driver_api                                  \
    );
```

Instantiate the driver for every enabled matching Device Tree node:

```c
DT_INST_FOREACH_STATUS_OKAY(OUR_DRIVER_DEFINE)
```

This pattern avoids manually duplicating the driver implementation for each physical device.

> **Note:** This is an instructional example. A complete driver must include all required declarations, subsystem APIs, initialization logic, error handling, and correct build integration.

---

## 10. Key Takeaways

- A driver abstracts hardware behind a software API.
- Device Tree describes hardware instances and their configuration.
- Bindings define valid Device Tree properties.
- `DT_DRV_COMPAT` associates a driver with compatible Device Tree nodes.
- `DT_INST_PROP()` retrieves Device Tree properties at compile time.
- Kconfig controls driver configuration and dependencies.
- CMake integrates driver source files into the build.
- Device configuration, API callbacks, and runtime data have distinct roles.
- Standard subsystem APIs help applications remain hardware-agnostic.
- Initialization levels and priorities determine when devices are initialized.
- `DT_INST_FOREACH_STATUS_OKAY()` supports creating instances for enabled matching nodes.
- A single driver implementation can support multiple devices with independent configuration and runtime state.

**Core principle:** Zephyr's device model combines reusable driver implementations with instance-specific configuration so applications can access hardware through standardized APIs.

---

