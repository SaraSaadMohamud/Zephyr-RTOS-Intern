# Zephyr Devicetree — Board Layout, Nodes & Properties

## 📌 Overview

**Devicetree (DTS)** is used in Zephyr RTOS to describe the **hardware structure of a board**.

It tells Zephyr:

- What peripherals exist on the board.
- Where those peripherals are connected.
- Which GPIO pins are used.
- What hardware drivers should be used.
- How hardware components are connected to the MCU.

The main idea is:

> **Devicetree describes the hardware, while application code describes what the application does with that hardware.**

------

# 1️⃣ Abstracts Board Layout and Peripherals

Devicetree provides an **hardware abstraction layer** between the application and the physical board.

For example, instead of writing application code that directly depends on:

```text
GPIO Port 0
Pin 0
Active Low
```

we can describe the LED in Devicetree:

```dts
blue_led: led_name {
    gpios = <&gpio0 0 GPIO_ACTIVE_LOW>;
    label = "Blue LED";
};
```

Then the application can reference the LED through Devicetree macros.

This makes the application more independent from the exact hardware wiring.

------

# 2️⃣ Independent of Application Logic

Devicetree should describe **what hardware exists and how it is connected**, not application behavior.

### Devicetree

```dts
blue_led: led_name {
    gpios = <&gpio0 0 GPIO_ACTIVE_LOW>;
    label = "Blue LED";
};
```

This tells Zephyr:

- There is an LED.
- The LED is connected to GPIO controller `gpio0`.
- It uses GPIO pin `0`.
- The LED is active-low.
- Its label is `"Blue LED"`.

### Application

The application decides what to do with it:

```cpp
gpio_pin_set_dt(&led, 1);
```

or:

```cpp
gpio_pin_toggle_dt(&led);
```

So the responsibilities are separated:

```text
┌──────────────────────────┐
│       Devicetree         │
│                          │
│ Hardware description     │
│ GPIO connections         │
│ Peripheral configuration │
└────────────┬─────────────┘
             │
             ▼
┌──────────────────────────┐
│      Application         │
│                          │
│ LED behavior             │
│ Timing                   │
│ Logic                    │
└──────────────────────────┘
```

------

# 3️⃣ Devicetree Structure

A simple LED definition can look like this:

```dts
/ {
    leds {
        compatible = "gpio-leds";

        blue_led: led_name {
            gpios = <&gpio0 0 GPIO_ACTIVE_LOW>;
            label = "Blue LED";
        };
    };
};
```

Let's break it down.

------

# 4️⃣ Nodes

A **node** represents a hardware component or a logical grouping of hardware.

Example:

```dts
leds {
    compatible = "gpio-leds";

    blue_led: led_name {
        gpios = <&gpio0 0 GPIO_ACTIVE_LOW>;
        label = "Blue LED";
    };
};
```

Here:

```text
leds
└── blue_led: led_name
```

`leds` is a node containing another node called `led_name`.

The Devicetree is therefore organized like a tree:

```text
/
├── leds
│   ├── blue_led
│   └── red_led
│
├── gpio0
├── uart0
└── spi1
```

------

# 5️⃣ Node Labels

A **node label** provides a convenient way to reference a node from another part of the Devicetree.

Example:

```dts
blue_led: led_name {
    ...
};
```

Here:

```text
blue_led
    │
    └── Node label

led_name
    │
    └── Node name
```

The label can later be used with `&`:

```dts
&blue_led {
    label = "Blue LED";
};
```

Node labels are especially useful when working with existing board definitions.

------

# 6️⃣ Node Names

The **node name** identifies the node in the Devicetree hierarchy.

Example:

```dts
blue_led: led_name {
```

The node name is:

```text
led_name
```

The node label is:

```text
blue_led
```

So:

```text
blue_led: led_name
   │          │
   │          └── Node name
   │
   └───────────── Node label
```

------

# 7️⃣ Properties

**Properties** contain configuration information about a node.

Example:

```dts
blue_led: led_name {
    gpios = <&gpio0 0 GPIO_ACTIVE_LOW>;
    label = "Blue LED";
};
```

This node has two properties:

```text
gpios
label
```

Properties are written as:

```dts
property = value;
```

For example:

```dts
label = "Blue LED";
```

------

# 8️⃣ `compatible` Property

The `compatible` property tells Zephyr which **driver or hardware type** the node represents.

Example:

```dts
compatible = "gpio-leds";
```

This tells Zephyr that the node represents LEDs controlled through GPIOs.

The `compatible` string is used by Zephyr's device model and drivers to determine how the hardware should be handled.

------

# 9️⃣ `gpios` Property

The `gpios` property describes the GPIO connection.

Example:

```dts
gpios = <&gpio0 0 GPIO_ACTIVE_LOW>;
```

It contains:

```text
&gpio0
   │
   └── GPIO controller

0
   │
   └── GPIO pin number

GPIO_ACTIVE_LOW
   │
   └── GPIO active state
```

So this means:

> The LED is connected to GPIO controller `gpio0`, pin `0`, and the LED is active-low.

------

# 🔟 Phandles

A **phandle** is a reference to another Devicetree node.

For example:

```dts
&gpio0
```

references the GPIO controller node.

A property can therefore connect one hardware node to another.

Example:

```dts
gpios = <&gpio0 0 GPIO_ACTIVE_LOW>;
```

Conceptually:

```text
LED
 │
 │ gpios
 ▼
GPIO0
 │
 └── Pin 0
```



------

# 1️⃣1️⃣ Property Value Types

Devicetree supports different types of property values.

## 🔹 String

A string is written inside quotation marks.

```dts
label = "Blue LED";
```

Example:

```text
Property: label
Value: "Blue LED"
Type: String
```

------

## 🔹 Integer

An integer is normally written inside angle brackets.

```dts
clock-frequency = <48000000>;
```

This represents:

```text
48,000,000 Hz
```

Example:

```dts
clock-frequency = <48000000>;
```

------

## 🔹 Array of Integers

Multiple integers can be placed inside angle brackets.

```dts
reg = <0x40000000 0x1000>;
```

Here:

```text
0x40000000
     │
     └── Base address

0x1000
     │
     └── Region size
```

So:

```text
reg = <address size>;
```

------

## 🔹 Phandle + Arguments

A property can reference another node and provide additional arguments.

Example:

```dts
gpios = <&gpio0 0 GPIO_ACTIVE_LOW>;
```

This contains:

```text
&gpio0
   │
   └── Phandle/reference

0
   │
   └── GPIO pin

GPIO_ACTIVE_LOW
   │
   └── GPIO configuration
```

This is a very common pattern in Zephyr Devicetree.

------

## 🔹 Boolean

A boolean property is considered **true when it exists**.

Example:

```dts
read-only;
```

There is no value after `=`.

If the property exists:

```text
read-only = true
```

If it does not exist:

```text
read-only = false
```

Example:

```dts
storage {
    read-only;
};
```

------

# 1️⃣2️⃣ Complete Example

Here is a complete example combining the concepts:

```dts
/ {
    leds {
        compatible = "gpio-leds";

        blue_led: led_name {
            gpios = <&gpio0 0 GPIO_ACTIVE_LOW>;
            label = "Blue LED";
        };
    };
};
```

### Breakdown

| Element           | Meaning                             |
| ----------------- | ----------------------------------- |
| `/`               | Root node                           |
| `leds`            | Node                                |
| `compatible`      | Identifies the hardware type/driver |
| `blue_led`        | Node label                          |
| `led_name`        | Node name                           |
| `gpios`           | GPIO configuration                  |
| `&gpio0`          | Reference to GPIO controller        |
| `0`               | GPIO pin number                     |
| `GPIO_ACTIVE_LOW` | Active-low configuration            |
| `label`           | Human-readable string               |

------

# 1️⃣3️⃣ Devicetree vs Application Code

A good Zephyr project separates **hardware description** from **application logic**.

### ❌ Avoid hard-coding hardware details

```cpp
gpio_pin_configure(gpio0, 0, GPIO_OUTPUT);
```

This makes the application dependent on a specific GPIO.

### ✅ Use Devicetree

Devicetree:

```dts
blue_led: led_name {
    gpios = <&gpio0 0 GPIO_ACTIVE_LOW>;
};
```

Application:

```cpp
#define LED_NODE DT_NODELABEL(blue_led)

static const struct gpio_dt_spec led =
    GPIO_DT_SPEC_GET(LED_NODE, gpios);
```

Now the application gets the GPIO configuration from Devicetree.

------

# 1️⃣4️⃣ Why Devicetree Is Important

Using Devicetree provides several advantages:

### 🔹 Hardware Abstraction

The application does not need to know the exact hardware wiring.

### 🔹 Portability

The same application can often be moved to another board by changing the Devicetree configuration.

### 🔹 Maintainability

Hardware configuration is kept separate from application logic.

### 🔹 Driver Integration

The `compatible` property allows Zephyr to associate hardware with the appropriate driver.

### 🔹 Configuration Flexibility

GPIOs, buses, interrupts, clocks, peripherals, and other hardware resources can be described without changing application logic.

------

# 🧠 Key Concepts to Remember

```text
Devicetree
│
├── Node
│   └── Represents hardware/component
│
├── Node Label
│   └── Shortcut/reference to a node
│
├── Node Name
│   └── Identifies the node in the tree
│
├── Property
│   └── Contains configuration information
│
├── compatible
│   └── Identifies the hardware type/driver
│
└── Phandle
    └── References another node
```

------

# 📚 Quick Reference

| Syntax                  | Meaning                       |
| ----------------------- | ----------------------------- |
| `/ { ... };`            | Root node                     |
| `node { ... };`         | Define a node                 |
| `label: node { ... };`  | Node with a label             |
| `property = "text";`    | String property               |
| `property = <123>;`     | Integer property              |
| `property = <1 2 3>;`   | Integer array                 |
| `property = <&node 0>;` | Phandle + arguments           |
| `property;`             | Boolean property              |
| `compatible = "...";`   | Hardware/driver compatibility |
| `&node`                 | Reference to a node           |

> **Think of Devicetree as the hardware map of your Zephyr application.**

---

# Zephyr Devicetree Overlays

A **Devicetree Overlay** is a patch file used to modify the base board Devicetree without changing the original board files.

Overlays allow you to:

- Add new hardware nodes.
- Override existing node properties.
- Change GPIO configurations.
- Add aliases.
- Customize hardware for a specific application.

The main idea is:

> **Board DTS describes the default hardware, while overlays customize that hardware for your application.**

------

# 1️⃣ What Are Overlays?

A Devicetree overlay is a file containing changes that are applied on top of the board's existing Devicetree.

For example, suppose the board has a default Devicetree, and we want to define an LED:

```dts
/* boards/frdm_rw612.overlay */

 / {
    leds {
        blue_led: led_name {
            gpios = <&hsgpio0 0 GPIO_ACTIVE_LOW>;
        };
    };
};
```

The overlay does **not** replace the entire board Devicetree.

Instead, Zephyr merges it with the existing board Devicetree.

Conceptually:

```text
Base Board DTS
       │
       │
       ▼
┌──────────────────┐
│ Board Hardware   │
│ Description      │
└────────┬─────────┘
         │
         │ + Overlay
         ▼
┌──────────────────┐
│ Final Devicetree │
│ Board + Changes  │
└──────────────────┘
```

------

# 2️⃣ Why Use Overlays?

Suppose you are using a development board and want to connect an external LED.

You don't need to modify the original board `.dts` file.

Instead, create:

```text
boards/<BOARD>.overlay
```

and describe the additional hardware there.

This has several advantages:

### 🔹 Keep Board Files Untouched

You don't modify Zephyr's original board definitions.

### 🔹 Application-Specific Hardware

Different applications can use different hardware configurations on the same board.

### 🔹 Easy Customization

You can override existing properties or add new nodes.

### 🔹 Portability

Your application can carry its hardware configuration in an overlay.

------

# 3️⃣ Adding a New Node

An overlay can add a completely new node.

Example:

```dts
/ {
    leds {
        blue_led: led_name {
            gpios = <&hsgpio0 0 GPIO_ACTIVE_LOW>;
        };
    };
};
```

Here we are adding:

```text
leds
└── led_name
```

with the label:

```text
blue_led
```

and configuring its GPIO connection.

------

# 4️⃣ Overriding Existing Properties

Overlays can also modify a node that already exists in the base Devicetree.

For example:

```dts
&uart0 {
    status = "okay";
};
```

This means:

> Find the existing `uart0` node and change its `status` property.

You can also modify other properties:

```dts
&uart0 {
    current-speed = <115200>;
};
```

The original board Devicetree remains unchanged.

------

# 5️⃣ Overlay File Locations

Zephyr supports automatic overlay discovery.

A common project structure is:

```text
app/
├── CMakeLists.txt
├── prj.conf
├── src/
│   └── main.cpp
└── boards/
    └── frdm_rw612.overlay
```

For a board named:

```text
frdm_rw612
```

the board-specific overlay is:

```text
boards/frdm_rw612.overlay
```

------

# 6️⃣ Overlay Auto-Discovery

When `DTC_OVERLAY_FILE` is **not** explicitly specified, Zephyr searches for overlays automatically.

The important search order is:

```text
1. boards/<BOARD>.overlay
2. app.overlay
```

For example, if the board is:

```text
frdm_rw612
```

Zephyr first checks:

```text
boards/frdm_rw612.overlay
```

If that file does not exist, Zephyr can use:

```text
app.overlay
```

------

# 7️⃣ First Match Behavior

Automatic discovery stops at the first matching overlay.

For example:

```text
boards/
└── frdm_rw612.overlay

app.overlay
```

If:

```text
boards/frdm_rw612.overlay
```

exists, Zephyr uses the board-specific overlay and does **not** use `app.overlay` as the fallback.

Conceptually:

```text
boards/frdm_rw612.overlay
        │
        ├── Found?
        │
        └── YES → Use it
                    │
                    └── Stop searching

app.overlay
        │
        └── Used only if the board overlay
            was not found
```

> **Important:** `app.overlay` is a fallback for automatic discovery; it is not automatically merged in addition to a matching `boards/<BOARD>.overlay`.

------

# 8️⃣ Forcing a Specific Overlay

You can explicitly tell Zephyr which overlay to use with:

```bash
west build -- -DDTC_OVERLAY_FILE="boards/frdm_rw612_red.overlay"
```

This is useful when you have multiple overlay configurations.

For example:

```text
boards/
├── frdm_rw612.overlay
├── frdm_rw612_red.overlay
└── frdm_rw612_blue.overlay
```

You can select a specific one:

```bash
west build -- -DDTC_OVERLAY_FILE="boards/frdm_rw612_red.overlay"
```

This overrides the normal automatic overlay discovery.

------

# 9️⃣ Adding an Extra Overlay

You can also add an additional overlay on top of the automatically discovered overlays.

Use:

```bash
west build -- -DEXTRA_DTC_OVERLAY_FILE="my_extra.overlay"
```

For example:

```text
boards/frdm_rw612.overlay
```

may be automatically discovered.

Then:

```bash
west build -- -DEXTRA_DTC_OVERLAY_FILE="my_extra.overlay"
```

adds another overlay.

Conceptually:

```text
Base Board DTS
      │
      ▼
Board Overlay
      │
      ▼
Extra Overlay
      │
      ▼
Final Devicetree
```

This is useful when you want to keep a common board overlay and add temporary or application-specific modifications.

------

# 🔟 Overlay Loading Options

There are three important mechanisms to remember.

| Method                   | Purpose                                         |
| ------------------------ | ----------------------------------------------- |
| `boards/<BOARD>.overlay` | Automatically discovered board-specific overlay |
| `app.overlay`            | Automatic fallback                              |
| `DTC_OVERLAY_FILE`       | Force a specific overlay                        |
| `EXTRA_DTC_OVERLAY_FILE` | Add extra overlay(s)                            |

### Automatic

```bash
west build -b frdm_rw612
```

Zephyr searches for the appropriate overlay automatically.

### Force an Overlay

```bash
west build -b frdm_rw612 -- \
    -DDTC_OVERLAY_FILE="boards/frdm_rw612_red.overlay"
```

### Add Extra Overlay

```bash
west build -b frdm_rw612 -- \
    -DEXTRA_DTC_OVERLAY_FILE="my_extra.overlay"
```

------

# 1️⃣1️⃣ Final Merged Devicetree

After the build, Zephyr generates the final merged Devicetree:

```text
build/zephyr/zephyr.dts
```

This file contains the result of merging:

```text
Board DTS
   +
Board Overlay
   +
Extra Overlay(s)
   ↓
Final Devicetree
```

So:

```text
build/zephyr/zephyr.dts
```

is extremely useful for debugging.

------

# 1️⃣2️⃣ Inspecting the Generated DTS

If your overlay is not behaving as expected, inspect:

```text
build/zephyr/zephyr.dts
```

You can search for your node:

```bash
grep -n "blue_led" build/zephyr/zephyr.dts
```

Or open the file:

```bash
less build/zephyr/zephyr.dts
```

You can verify:

- Your node exists.
- Its properties were applied.
- The GPIO controller is correct.
- The GPIO pin is correct.
- The node status is correct.
- Aliases point to the expected nodes.

------

# 1️⃣3️⃣ Verify Your Overlay Was Merged

Suppose your overlay contains:

```dts
/ {
    leds {
        blue_led: led_name {
            gpios = <&hsgpio0 0 GPIO_ACTIVE_LOW>;
        };
    };
};
```

After building, inspect:

```text
build/zephyr/zephyr.dts
```

Search for:

```text
blue_led
```

If the node appears in the generated DTS, the overlay was successfully merged.

Example:

```bash
grep -n "blue_led" build/zephyr/zephyr.dts
```

------

# 1️⃣4️⃣ Checking the Actual Node Path

Devicetree nodes have complete paths.

For example:

```text
/leds/led_name
```

The node may have the label:

```text
blue_led
```

These are different concepts:

```text
Node label:
blue_led

Node name:
led_name

Full node path:
/leds/led_name
```

This distinction is important when using Devicetree macros.

------

# 1️⃣5️⃣ Node Label vs Node Path

Consider:

```dts
/ {
    leds {
        blue_led: led_name {
            gpios = <&hsgpio0 0 GPIO_ACTIVE_LOW>;
        };
    };
};
```

The full path is:

```text
/leds/led_name
```

The node label is:

```text
blue_led
```

You can reference the node by label:

```cpp
#define LED_NODE DT_NODELABEL(blue_led)
```

This is often easier than working with the complete path.

------

# 1️⃣6️⃣ Using `DT_PATH()`

`DT_PATH()` allows you to reference a node using its path.

For:

```text
/leds/led_name
```

you can write:

```cpp
#define LED_NODE DT_PATH(leds, led_name)
```

The syntax follows the hierarchy:

```text
DT_PATH(parent, child)
```

For example:

```text
/leds/led_name
```

becomes:

```cpp
DT_PATH(leds, led_name)
```

------

# 1️⃣7️⃣ Finding the Correct Path

If you are unsure about the node path, inspect:

```text
build/zephyr/zephyr.dts
```

Look for the hierarchy:

```dts
leds {
    led_name {
        ...
    };
};
```

Then construct the path:

```text
/leds/led_name
```

and use:

```cpp
DT_PATH(leds, led_name)
```

------

# 1️⃣8️⃣ Checking Aliases

Aliases provide convenient names for nodes.

For example:

```dts
/ {
    aliases {
        warning-led = &blue_led;
    };
};
```

Now the application can use:

```cpp
#define LED_NODE DT_ALIAS(warning_led)
```

Notice that the Devicetree alias:

```text
warning-led
```

is accessed in C code as:

```text
warning_led
```

because Devicetree macro identifiers use underscores.

------

# 1️⃣9️⃣ Verify Aliases in Generated DTS

Always verify that the alias points to the node you expect.

In:

```text
build/zephyr/zephyr.dts
```

look for:

```dts
aliases {
    warning-led = &blue_led;
};
```

Conceptually:

```text
warning-led
     │
     ▼
 blue_led
     │
     ▼
 /leds/led_name
```

This helps prevent accidentally controlling the wrong hardware.

------

# 2️⃣0️⃣ Complete Example

### Overlay

```dts
/* boards/frdm_rw612.overlay */

 / {
    leds {
        blue_led: led_name {
            gpios = <&hsgpio0 0 GPIO_ACTIVE_LOW>;
            label = "Blue LED";
        };
    };

    aliases {
        warning-led = &blue_led;
    };
};
```

### Application

```cpp
#include <zephyr/drivers/gpio.h>

#define LED_NODE DT_ALIAS(warning_led)

static const struct gpio_dt_spec led =
    GPIO_DT_SPEC_GET(LED_NODE, gpios);
```

The application does not need to know that the LED is connected to a specific GPIO pin.

That information comes from the overlay.

------

# 2️⃣1️⃣ Debugging Workflow

When your overlay doesn't work, follow these steps:

### Step 1 — Check the overlay location

```text
boards/<BOARD>.overlay
```

or:

```text
app.overlay
```

### Step 2 — Check the board name

For example:

```text
frdm_rw612
```

requires:

```text
boards/frdm_rw612.overlay
```

### Step 3 — Build the application

```bash
west build -b frdm_rw612
```

### Step 4 — Inspect the generated DTS

```text
build/zephyr/zephyr.dts
```

### Step 5 — Search for your node

```bash
grep -n "blue_led" build/zephyr/zephyr.dts
```

### Step 6 — Verify the GPIO

Check that the generated DTS contains the expected:

```dts
gpios = <&hsgpio0 0 GPIO_ACTIVE_LOW>;
```

### Step 7 — Verify aliases

Check:

```dts
aliases {
    warning-led = &blue_led;
};
```

### Step 8 — Verify the application macro

For an alias:

```cpp
#define LED_NODE DT_ALIAS(warning_led)
```

For a node label:

```cpp
#define LED_NODE DT_NODELABEL(blue_led)
```

For a node path:

```cpp
#define LED_NODE DT_PATH(leds, led_name)
```

------

# 🧠 Important Concepts

```text
                 Board DTS
                     │
                     ▼
              ┌─────────────┐
              │ Base Hardware│
              └──────┬──────┘
                     │
                  Overlay
                     │
                     ▼
              ┌─────────────┐
              │   Changes   │
              │ Add / Modify│
              └──────┬──────┘
                     │
                     ▼
             Final zephyr.dts
                     │
                     ▼
              Application
```

Remember:

### `boards/<BOARD>.overlay`

Board-specific automatic overlay.

### `app.overlay`

Fallback automatic overlay.

### `DTC_OVERLAY_FILE`

Force a specific overlay.

```bash
west build -- -DDTC_OVERLAY_FILE="my.overlay"
```

### `EXTRA_DTC_OVERLAY_FILE`

Add an extra overlay.

```bash
west build -- -DEXTRA_DTC_OVERLAY_FILE="extra.overlay"
```

### `build/zephyr/zephyr.dts`

The final merged Devicetree generated by the build.

------

#  Quick Reference

| Task                 | Syntax / Command                          |
| -------------------- | ----------------------------------------- |
| Board overlay        | `boards/<BOARD>.overlay`                  |
| Fallback overlay     | `app.overlay`                             |
| Force overlay        | `-DDTC_OVERLAY_FILE="file.overlay"`       |
| Add extra overlay    | `-DEXTRA_DTC_OVERLAY_FILE="file.overlay"` |
| Generated DTS        | `build/zephyr/zephyr.dts`                 |
| Node label           | `DT_NODELABEL(label)`                     |
| Node path            | `DT_PATH(parent, child)`                  |
| Alias                | `DT_ALIAS(alias)`                         |
| Search generated DTS | `grep -n "name" build/zephyr/zephyr.dts`  |

```text
build/zephyr/zephyr.dts
```

> **Think of an overlay as a Devicetree patch that customizes the board for your application.**

---

