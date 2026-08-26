# West — Zephyr's Python-Based Build Orchestrator

`west` is Zephyr's **Python-based meta-tool and build orchestrator**. It simplifies the development workflow by managing multi-module Zephyr workspaces, integrating with **CMake + Ninja**, and providing convenient commands for building, flashing, debugging, and working with supported boards.

## What is West?

West acts as a single command-line interface for common Zephyr development tasks. Instead of manually running several tools, developers can use `west` to manage the complete application workflow.

### Key Responsibilities

- **Repository Management** — Manages Zephyr's multi-module workspaces and manifest dependencies.
- **Build System Integration** — Integrates with **CMake** and **Ninja** to configure and build applications.
- **Flashing** — Programs the compiled firmware onto supported development boards.
- **Debugging** — Launches debugger sessions for embedded targets.
- **Logging & Convenience** — Provides a unified command-line interface for common development operations.
- **Extensibility** — Supports custom commands and extensions for project-specific workflows.

## How West Works

```text
                 ┌─────────────────────┐
                 │     Developer       │
                 └──────────┬──────────┘
                            │
                            ▼
                    ┌───────────────┐
                    │     west      │
                    │ Python-based  │
                    │  orchestrator │
                    └───────┬───────┘
                            │
             ┌──────────────┼──────────────┐
             ▼              ▼              ▼
       ┌───────────┐  ┌────────────┐  ┌────────────┐
       │ Workspace │  │ CMake +    │  │ Flash /    │
       │ Management│  │ Ninja      │  │ Debug Tools│
       └───────────┘  └────────────┘  └────────────┘
```

West does not replace CMake or Ninja. Instead, it provides a convenient layer that coordinates these tools and other Zephyr development utilities.

## Essential West Commands

| Command              | Purpose                                    |
| -------------------- | ------------------------------------------ |
| `west init`          | Initialize a Zephyr workspace              |
| `west update`        | Fetch or update all manifest dependencies  |
| `west build`         | Configure and build a Zephyr application   |
| `west flash`         | Flash the built image to a supported board |
| `west debug`         | Launch a debugger session                  |
| `west boards`        | List supported Zephyr boards               |
| `west zephyr-export` | Register the Zephyr CMake package          |

## Command Examples

### 1. Initialize a Workspace

```bash
west init ~/zephyrproject
```

Creates a new West workspace and prepares it for managing Zephyr and its related modules.

### 2. Fetch Dependencies

```bash
cd ~/zephyrproject
west update
```

Downloads and synchronizes the repositories specified by the Zephyr manifest.

### 3. Build an Application

```bash
west build -b <board> <application>
```

For example:

```bash
west build -b nucleo_f401re samples/hello_world
```

Internally, the build process integrates with **CMake** for configuration and **Ninja** for executing the build.

### 4. Flash the Application

After a successful build:

```bash
west flash
```

West uses the appropriate flashing mechanism for the selected board and programs the generated firmware.

### 5. Start a Debug Session

```bash
west debug
```

Launches a debugger session using the configured debug infrastructure.

### 6. List Supported Boards

```bash
west boards
```

Displays the boards supported by the installed Zephyr version.

### 7. Register Zephyr with CMake

```bash
west zephyr-export
```

Registers the Zephyr CMake package so that external CMake projects can locate and use Zephyr.

## West + CMake + Ninja

The relationship between the tools can be summarized as:

```text
             west build
                 │
                 ▼
              CMake
                 │
        Configure the project
                 │
                 ▼
              Ninja
                 │
          Execute the build
                 │
                 ▼
          Firmware Image
                 │
          ┌──────┴──────┐
          ▼             ▼
      west flash    west debug
          │             │
          ▼             ▼
        Board        Debugger
```

This makes `west` a convenient entry point while allowing Zephyr to continue using established build tools underneath.

## West Workspace

A typical Zephyr workspace managed by West can contain:

```text
zephyrproject/
├── .west/
├── zephyr/
├── modules/
├── tools/
└── <other repositories>
```

The **manifest** defines the repositories and revisions that make up the workspace. Running:

```bash
west update
```

ensures that the required modules are fetched according to that manifest.

## Extensibility

West is designed to be extensible. In addition to its built-in commands, Zephyr modules and projects can provide **custom West commands** for specialized development workflows.

This makes West useful not only as a build wrapper but also as a general development orchestration tool for Zephyr-based projects.

## Quick Workflow

A typical Zephyr workflow looks like this:

```bash
# Initialize workspace
west init ~/zephyrproject

# Enter workspace
cd ~/zephyrproject

# Fetch dependencies
west update

# Build application
west build -b <board> <application>

# Flash firmware
west flash

# Start debugging
west debug
```

# Zephyr Application Topologies

Zephyr supports different project layouts depending on the complexity and purpose of the application. The three common approaches are **Freestanding Applications**, **Repository Applications**, and **Workspace Applications**.

------

## 1. Freestanding Application

A freestanding application is a simple application located outside the Zephyr repository.

```text
<home>/
├── zephyrproject/
│   ├── .west/
│   │   └── config
│   ├── zephyr/
│   └── ...
│
└── my-app/
    ├── CMakeLists.txt
    ├── prj.conf
    └── src/
        └── main.c
```

### Structure

- `CMakeLists.txt` — Defines how the application is built.
- `prj.conf` — Contains Zephyr Kconfig configuration.
- `src/main.c` — Contains the application source code.
- `zephyrproject/` — Contains the Zephyr installation and West workspace.

### When to Use

Freestanding applications are suitable for:

- Quick experiments
- Learning Zephyr
- Small and simple applications
- Testing individual Zephyr features

This is usually the easiest topology for getting started with Zephyr.

------

## 2. Repository Application

A repository application keeps the application inside a repository that also contains a Zephyr workspace configuration.

```text
my-project/
├── .west/
│   └── config
│
├── app/
│   ├── CMakeLists.txt
│   ├── prj.conf
│   ├── west.yml
│   └── src/
│       └── main.c
│
├── zephyr/
├── bootloader/
├── modules/
└── <vendor/private-repos>/
```

### Structure

- `.west/config` — West workspace configuration.
- `app/` — Contains the application.
- `CMakeLists.txt` — Application build configuration.
- `prj.conf` — Application Kconfig configuration.
- `west.yml` — Manifest describing workspace dependencies.
- `src/main.c` — Application source code.
- `zephyr/` — Zephyr repository.
- `bootloader/` — Bootloader components when required.
- `modules/` — Additional Zephyr modules.
- `<vendor/private-repos>/` — Vendor-specific or private repositories.

### When to Use

Repository applications are useful when:

- Contributing to Zephyr samples or tests
- Working with projects that are maintained together with Zephyr-related repositories
- Managing application-specific dependencies
- Keeping project source and Zephyr components under one repository structure

------

## 3. Workspace Application

A workspace application uses a complete West workspace to manage Zephyr, modules, bootloaders, and potentially multiple applications.

```text
my-project/
├── .west/
│   └── config
│
├── app/
│   ├── CMakeLists.txt
│   ├── prj.conf
│   ├── west.yml
│   └── src/
│       └── main.c
│
├── zephyr/
├── bootloader/
├── modules/
└── <vendor/private-repos>/
```

The important concept is that **West manages the complete workspace**, rather than only building a single application.

A workspace can contain:

- Zephyr itself
- Application repositories
- External modules
- Bootloaders
- Vendor SDKs
- Private repositories
- Multiple related applications

### When to Use

Workspace-based development is recommended for:

- Production projects
- Custom hardware and boards
- Projects with multiple applications
- Projects with multiple external modules
- Vendor-specific dependencies
- Large embedded systems

------

## Topology Comparison

| Topology         | Main Purpose                                      | Typical Use                           |
| ---------------- | ------------------------------------------------- | ------------------------------------- |
| **Freestanding** | Simple application outside Zephyr                 | Learning, experiments, small projects |
| **Repository**   | Application integrated with a repository          | Zephyr samples, tests, contributions  |
| **Workspace**    | Complete multi-repository development environment | Production and complex projects       |

------

## Recommended Approach

For beginners, a **Freestanding Application** is usually the simplest way to learn Zephyr because the application contains only the files required to build and configure it.

For projects that need closer integration with Zephyr repositories, a **Repository Application** provides a more structured environment.

For serious embedded development, the **Workspace Application** topology is recommended because it scales better and allows West to manage Zephyr, modules, bootloaders, vendor repositories, and multiple applications.

### Recommendation

> **Use Freestanding for learning and quick experiments.**
> **Use Repository topology when contributing to Zephyr or maintaining closely related repositories.**
> **Use Workspace topology for real-world production projects, custom boards, and multi-application systems.**



# West Manifest (`west.yml`)

A **West manifest** defines the repositories that belong to a Zephyr workspace. It tells West **which projects to fetch, where to get them, which revision to use, and which dependencies to import**.

## Example `west.yml`

```yaml
manifest:
  version: "1.0"  # Manifest schema version

  projects:
    - name: zephyr
      url: https://github.com/zephyrproject-rtos/zephyr
      revision: v4.2.0  # Pinned Git tag

      import:
        name-allowlist:
          - cmsis
          - hal_nxp

        path-prefix: deps  # Put imported dependencies inside deps/
```

## Explanation

### `version`

```yaml
version: "1.0"
```

Specifies the **West manifest schema version**.

------

### `projects`

```yaml
projects:
  - name: zephyr
```

Defines the repositories that West should manage.

In this example, the workspace includes the **Zephyr repository**.

------

### `name`

```yaml
name: zephyr
```

The name used by West to identify this dependency.

------

### `url`

```yaml
url: https://github.com/zephyrproject-rtos/zephyr
```

Specifies the Git repository URL.

------

### `revision`

```yaml
revision: v4.2.0
```

Pins the project to a specific Git revision.

This can be:

- A Git tag
- A branch
- A commit SHA

Using a specific version such as `v4.2.0` helps ensure that all developers use the same Zephyr version.

------

## Importing Zephyr Dependencies

Zephyr itself has a manifest containing many additional repositories and modules.

The `import` section controls which of these dependencies West imports.

### Using `name-allowlist`

```yaml
import:
  name-allowlist:
    - cmsis
    - hal_nxp
```

This tells West to import **only the dependencies listed**:

- `cmsis`
- `hal_nxp`

This is useful when the project needs only a small subset of Zephyr's modules.

------

### Using `path-prefix`

```yaml
path-prefix: deps
```

Places imported dependencies under the `deps/` directory.

The resulting workspace may look like:

```text
my-project/
├── app/
│   ├── CMakeLists.txt
│   ├── prj.conf
│   ├── west.yml
│   └── src/
│       └── main.c
│
├── zephyr/
│
└── deps/
    ├── cmsis/
    └── hal_nxp/
```

------

## Fetching All Zephyr Dependencies

If you omit the `name-allowlist`, West imports all projects defined by Zephyr's manifest.

For example:

```yaml
manifest:
  version: "1.0"

  projects:
    - name: zephyr
      url: https://github.com/zephyrproject-rtos/zephyr
      revision: v4.2.0
      import: true
```

Then running:

```bash
west update
```

will fetch Zephyr and its full set of manifest dependencies, which can include **many repositories and modules**.

## Why Use an Allowlist?

Using:

```yaml
name-allowlist:
  - cmsis
  - hal_nxp
```

provides several benefits:

- Faster initial setup
- Less disk space usage
- Fewer repositories to clone
- Smaller workspace
- Only required dependencies are downloaded

## Summary

```text
west.yml
    │
    ├── version
    │     └── Defines manifest schema version
    │
    └── projects
          │
          ├── name
          ├── url
          ├── revision
          │
          └── import
                │
                ├── name-allowlist
                │     └── Fetch only selected dependencies
                │
                └── path-prefix
                      └── Control dependency location
```

> **Without `name-allowlist`, West imports all projects defined by Zephyr's manifest. With `name-allowlist`, you can create a smaller, more focused workspace containing only the dependencies required by your project.**

---

# Zephyr LED Blink Example

This example demonstrates how to control an LED using the **Zephyr GPIO API** and **DeviceTree**, while periodically toggling the LED every second and reporting its state through the Zephyr logging system.

## Source Code

```c
#include <zephyr/drivers/gpio.h>
#include <zephyr/kernel.h>
#include <zephyr/logging/log.h>

#define SLEEP_TIME_MS 1000

/* The devicetree node identifier for the "led0" alias. */
#define LED_NODE DT_ALIAS(led0)

static const struct gpio_dt_spec led = GPIO_DT_SPEC_GET(LED_NODE, gpios);

LOG_MODULE_REGISTER(main, LOG_LEVEL_INF);

int main(void)
{
    bool led_state = true;

    if (!gpio_is_ready_dt(&led)) return 0;

    if (gpio_pin_configure_dt(&led, GPIO_OUTPUT_ACTIVE) < 0) return 0;

    while (1) {
        if (gpio_pin_toggle_dt(&led) < 0) return 0;

        led_state = !led_state;
        LOG_INF("LED state: %s", led_state ? "ON" : "OFF");

        k_msleep(SLEEP_TIME_MS);
    }

    return 0;
}
```

## How the Code Works

### 1. Include Zephyr APIs

```c
#include <zephyr/drivers/gpio.h>
#include <zephyr/kernel.h>
#include <zephyr/logging/log.h>
```

The application uses three main Zephyr components:

| Header     | Purpose                          |
| ---------- | -------------------------------- |
| `gpio.h`   | GPIO configuration and control   |
| `kernel.h` | Kernel APIs such as `k_msleep()` |
| `log.h`    | Zephyr logging system            |

------

### 2. Define the Delay

```c
#define SLEEP_TIME_MS 1000
```

The LED state changes every **1000 ms**, which is equivalent to **1 second**.

------

### 3. Access the LED Through DeviceTree

```c
#define LED_NODE DT_ALIAS(led0)
```

Instead of hard-coding a GPIO port and pin, the application uses the DeviceTree alias `led0`.

This makes the application more portable across different boards.

The actual GPIO configuration comes from the board's DeviceTree.

------

### 4. Create a GPIO Device Specification

```c
static const struct gpio_dt_spec led =
    GPIO_DT_SPEC_GET(LED_NODE, gpios);
```

`GPIO_DT_SPEC_GET()` extracts the GPIO information from the DeviceTree and creates a `gpio_dt_spec` containing the information required to control the LED.

Conceptually:

```text
DeviceTree
    │
    │ led0 alias
    ▼
LED_NODE
    │
    ▼
GPIO_DT_SPEC_GET()
    │
    ▼
gpio_dt_spec
    │
    ├── GPIO controller
    ├── GPIO pin
    └── GPIO flags
```

------

### 5. Enable Logging

```c
LOG_MODULE_REGISTER(main, LOG_LEVEL_INF);
```

Registers a logging module named `main` and enables messages at the `INFO` level and above.

The application can therefore use:

```c
LOG_INF("LED state: %s", ...);
```

instead of using standard C functions such as `printf()`.

------

### 6. Check GPIO Readiness

```c
if (!gpio_is_ready_dt(&led))
    return 0;
```

Before using the GPIO, the code checks whether the corresponding GPIO device is ready.

If the device is not ready, the application exits.

------

### 7. Configure the LED as an Output

```c
if (gpio_pin_configure_dt(&led, GPIO_OUTPUT_ACTIVE) < 0)
    return 0;
```

Configures the LED GPIO as an output.

`GPIO_OUTPUT_ACTIVE` also initializes the pin in its active state according to the GPIO configuration.

------

### 8. Toggle the LED

```c
gpio_pin_toggle_dt(&led);
```

This Zephyr GPIO API toggles the current state of the LED:

```text
ON  → OFF
OFF → ON
```

The application also maintains its own state variable:

```c
led_state = !led_state;
```

This allows the current state to be printed to the log.

------

### 9. Log the LED State

```c
LOG_INF("LED state: %s", led_state ? "ON" : "OFF");
```

The ternary operator converts the Boolean state into a readable message:

```text
LED state: ON
LED state: OFF
LED state: ON
LED state: OFF
...
```

------

### 10. Sleep for One Second

```c
k_msleep(SLEEP_TIME_MS);
```

The Zephyr kernel puts the current thread to sleep for **1000 ms**.

Unlike a busy-wait loop, this allows the CPU/thread to sleep while waiting.

The overall execution is therefore:

```text
        ┌──────────────────────┐
        │      Application     │
        └──────────┬───────────┘
                   │
                   ▼
          Check GPIO ready
                   │
                   ▼
        Configure LED as output
                   │
                   ▼
             Toggle LED
                   │
                   ▼
             Log LED state
                   │
                   ▼
          Sleep for 1000 ms
                   │
                   │
                   └──────────────┐
                                  │
                                  ▼
                            Toggle again
```

## Key Zephyr Concepts Demonstrated

This small application introduces several important Zephyr concepts:

- **DeviceTree** — Hardware configuration is obtained from `led0`.
- **GPIO API** — The LED is controlled through Zephyr's portable GPIO interface.
- **Kernel API** — `k_msleep()` provides timed thread sleeping.
- **Logging subsystem** — `LOG_INF()` provides structured application logging.
- **Board portability** — The application does not directly hard-code the GPIO port or pin.

## Expected Behavior

After flashing the application to a supported board, the LED should continuously toggle approximately once every second:

```text
LED ON
   ↓
1 second
   ↓
LED OFF
   ↓
1 second
   ↓
LED ON
   ↓
...
```

The console/log output should look similar to:

```text
[00:00:01.000] <inf> main: LED state: OFF
[00:00:02.000] <inf> main: LED state: ON
[00:00:03.000] <inf> main: LED state: OFF
[00:00:04.000] <inf> main: LED state: ON
```

## Build and Run

From the Zephyr workspace:

```bash
west build -b <board> .
west flash
```

For example:

```bash
west build -b <your_board> .
west flash
```

The exact board name can be found using:

```bash
west boards
```

## Important Note

The application expects the selected board to provide a DeviceTree alias named:

```text
led0
```

If `DT_ALIAS(led0)` is not defined for the selected board, the application will not be able to obtain the LED GPIO specification.

This is one of the key advantages of Zephyr's DeviceTree system: **the application code can remain hardware-independent while the board-specific GPIO configuration is defined separately.**