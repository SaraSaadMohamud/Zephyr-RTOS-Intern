# Zephyr RTOS — Shell

## 1. Overview

Zephyr Shell is an interactive command-line interface (CLI) that lets users interact with an embedded device at runtime. It supports command execution, command history, tab completion, custom command registration, logging integration, and multiple communication backends.

## 2. Key Features

- Multiple shell instances and backends, such as Serial and Telnet, when configured.
- Tab completion for commands.
- Command history.
- Static and dynamic command registration.
- Built-in commands such as `clear`, `history`, `shell`, and `log`, depending on enabled configuration and modules.
- Integration with Zephyr's Logging subsystem.
- ANSI escape codes for terminal cursor control and colors.

## 3. Enabling the Shell

The minimal configuration is:

```conf
CONFIG_SHELL=y
```

The Shell typically uses a configured default backend. On many boards this is a serial console, but the actual backend depends on the board and application configuration.

### Additional options

```conf
CONFIG_SHELL_CMDS=y
CONFIG_SHELL_TAB=y
CONFIG_SHELL_TAB_AUTOCOMPLETION=y
CONFIG_SHELL_HISTORY=y
CONFIG_SHELL_LOG_BACKEND=y
```

These options enable additional command support, tab handling/autocompletion, history, and logging integration. Availability and dependencies can vary by Zephyr version and application configuration.

## 4. Shell Backends

| Backend       | Configuration / Setup                                      | Typical use                                    |
| ------------- | ---------------------------------------------------------- | ---------------------------------------------- |
| UART / Serial | `CONFIG_SHELL_BACKEND_SERIAL=y`                            | Local debugging through a serial terminal      |
| USB CDC ACM   | `cdc-acm-console` snippet, if supported                    | USB virtual serial port                        |
| Telnet        | `CONFIG_NETWORKING=y` and `CONFIG_SHELL_BACKEND_TELNET=y`  | Remote shell over a network                    |
| SEGGER RTT    | `CONFIG_USE_SEGGER_RTT=y` and `CONFIG_SHELL_BACKEND_RTT=y` | Communication through a compatible debug probe |
| BLE NUS       | `nus-console` snippet, if supported                        | Wireless shell over Bluetooth LE               |
| Dummy         | `CONFIG_SHELL_BACKEND_DUMMY=y`                             | Testing or programmatic use                    |

Not every backend is available on every board. Each backend may require specific hardware, networking support, snippets, or additional configuration.

## 5. UART / Serial Backend

Enable the Shell and serial backend:

```conf
CONFIG_SHELL=y
CONFIG_SHELL_BACKEND_SERIAL=y
```

Connect using a serial terminal such as `minicom`:

```bash
minicom -D /dev/ttyACM0 -b 115200
```

Or use `screen`:

```bash
screen /dev/ttyACM0 115200
```

The device path and baud rate are examples; verify the actual port and console settings for the target board.

To list common serial device paths on Linux:

```bash
ls /dev/ttyACM* /dev/ttyUSB* 2>/dev/null
```

## 6. USB CDC ACM Backend

USB CDC ACM can expose a virtual serial port over USB, avoiding the need for a separate UART adapter when the board supports USB device mode.

Example build command for a supported board and application:

```bash
west build -b <board> -S cdc-acm-console <app>
```

After flashing and connecting the appropriate USB port, the device may appear as `/dev/ttyACM*` on Linux. The required USB connector and configuration depend on the board.

## 7. Telnet Backend

Basic configuration:

```conf
CONFIG_NETWORKING=y
CONFIG_SHELL=y
CONFIG_SHELL_BACKEND_TELNET=y
```

After networking is configured and the target has an IP address, connect using the actual target address and configured port:

```bash
telnet <device-ip> 4242
```

The address is a placeholder; verify the port used by the selected Zephyr version and configuration.

**Security note:** Telnet does not encrypt traffic by default. Avoid exposing it to untrusted networks.

## 8. SEGGER RTT Backend

Basic configuration:

```conf
CONFIG_USE_SEGGER_RTT=y
CONFIG_SHELL=y
CONFIG_SHELL_BACKEND_RTT=y
CONFIG_SHELL_BACKEND_SERIAL=n
```

RTT requires a compatible debug probe and host-side tools. Disable the serial backend only if it is not needed.

## 9. Minimal Shell Mode

For resource-constrained devices, minimal mode reduces the default feature set:

```conf
CONFIG_SHELL=y
CONFIG_SHELL_MINIMAL=y
```

Re-enable only the features required by the application. The exact feature set and dependencies should be checked against the Zephyr version being used.

## 10. Useful Shell Commands

Depending on the enabled configuration and modules, useful commands may include:

```text
help
clear
history
kernel uptime
kernel version
```

- `help`: lists available commands.
- `clear`: clears the terminal display.
- `history`: displays previously entered commands when history is enabled.
- `kernel uptime`: displays the system uptime when the relevant command is enabled.
- `kernel version`: displays the Zephyr version when the relevant command is enabled.

The available commands depend on the Shell configuration and enabled subsystems.

## 11. Troubleshooting Checklist

If the Shell prompt does not appear:

1. Confirm `CONFIG_SHELL=y` is enabled.
2. Confirm the intended backend is enabled.
3. Check the board's console and UART/USB routing.
4. Verify the correct Linux serial device path and baud rate.
5. Ensure no other terminal program is using the same serial port.
6. Rebuild and flash the correct application and board configuration.
7. Check Kconfig dependencies and build output for warnings or errors.

## 12. Practical Exercise: Shell on STM32F4 Discovery

For a first experiment on a supported STM32F4 Discovery setup, start with:

```conf
CONFIG_SHELL=y
CONFIG_SHELL_BACKEND_SERIAL=y
CONFIG_SHELL_CMDS=y
CONFIG_SHELL_HISTORY=y
CONFIG_SHELL_TAB=y
CONFIG_SHELL_TAB_AUTOCOMPLETION=y
```

Build the application from the Zephyr workspace:

```bash
west build -b stm32f4_disco app -p always
```

Flash the resulting build using the workflow configured for your board, then connect through the configured serial interface and try:

```text
help
kernel uptime
kernel version
```

If the prompt is unavailable, check the board's actual console/backend configuration before assuming that the Shell is using a particular UART or USB port.

## 13. Summary

Zephyr Shell provides a runtime CLI for inspecting and controlling an embedded application. Start with the serial backend, verify the available commands, and add features or alternative backends only when needed. For small devices, minimal mode can help reduce resource usage. Always verify backend support and Kconfig dependencies for the specific Zephyr version and board.

----

# 2.Part2: Zephyr RTOS Shell Commands 

## Overview

The Zephyr RTOS Shell provides an interactive command-line interface (CLI) that allows users to interact with an embedded application at runtime.

Developers can use the Shell to:

- Execute application-specific commands.
- Display information and diagnostic messages.
- Inspect runtime parameters.
- Organize commands into hierarchical structures using subcommands.
- Parse command-line arguments.
- Debug and test embedded applications without recompiling for every operation.

This guide covers the fundamental concepts of implementing and registering Shell commands in Zephyr RTOS.

## Table of Contents

- [1. Shell Command Hierarchy](#1-shell-command-hierarchy)
- [2. Command Handler Signature](#2-command-handler-signature)
- [3. Shell Print Functions](#3-shell-print-functions)
- [4. Registering a Root Command](#4-registering-a-root-command)
- [5. Defining Static Subcommands](#5-defining-static-subcommands)
- [6. Registering a Command Tree](#6-registering-a-command-tree)
- [7. Testing Commands in the Shell](#7-testing-commands-in-the-shell)
- [8. Important Notes](#8-important-notes)
- [9. Summary](#9-summary)

## 1. Shell Command Hierarchy

Zephyr Shell commands are organized into a **tree hierarchy**.

Each command belongs to a specific level in the tree:

- **Root command (Level 0):** The top-level entry point.
- **Subcommand (Level 1 or higher):** A command associated with a parent command.
- **Command handler:** The function executed when a command is invoked.
- **Subcommand set:** A collection of subcommands associated with a root command.

Each node in the command tree can have a handler, child commands, or both.

### Example: Command Tree

```
demo                    <- Root command (Level 0)
├── ping                <- Subcommand (Level 1)
└── params               <- Subcommand (Level 1)
```

Commands are executed by specifying the root command followed by the desired subcommand.

For example:

```
uart:~$ demo ping
uart:~$ demo params
```

**Key concept:** The root command groups related operations under a common name, making the Shell interface easier to organize and maintain.

## 2. Command Handler Signature

Every Shell command handler follows a common function signature.

```
static int cmd_handler(const struct shell *sh,
                       size_t argc,
                       char **argv)
{
    ARG_UNUSED(argc);
    ARG_UNUSED(argv);

    shell_print(sh, "Hello from the shell!");

    return 0;
}
```

### Understanding the Parameters

| Parameter | Description                                                |
| --------- | ---------------------------------------------------------- |
| `sh`      | Pointer to the Shell instance used to execute the command. |
| `argc`    | Number of arguments provided to the handler.               |
| `argv`    | Array of strings containing the command arguments.         |

### Understanding the Return Value

The handler returns an integer status code.

```
return 0;
```

A return value of `0` conventionally indicates successful execution.

Other return values can be used to indicate errors, depending on the command implementation.

### Understanding `argc` and `argv`

Consider the following command:

```
uart:~$ demo parse hello world
```

The handler receives the command arguments through `argc` and `argv`.

For this example, the arguments exposed to the handler are:

```
argc = 2

argv[0] = hello
argv[1] = world
```

**Important:** The command tokens used to select the root command and subcommand are not included in the handler's argument array. The handler receives the arguments associated with its invocation.

If a handler does not use a parameter, `ARG_UNUSED()` can be used to indicate that the parameter is intentionally unused.

## 3. Shell Print Functions

Zephyr provides several functions for displaying messages through the Shell.

These functions help developers distinguish normal output, informational messages, warnings, and errors.

### 3.1 `shell_print()`

Use `shell_print()` to display ordinary text or formatted output.

```
shell_print(sh, "Hello from Zephyr Shell!");

shell_print(sh, "Temperature: %d C", 25);
```

Output:

```
Hello from Zephyr Shell!
Temperature: 25 C
```

### 3.2 `shell_info()`

Use `shell_info()` to display informational messages.

```
shell_info(sh, "Application initialized successfully");
```

This is useful for communicating application status and successful operations.

### 3.3 `shell_warn()`

Use `shell_warn()` to display warnings.

```
shell_warn(sh, "Battery level is low");
```

Warnings help identify conditions that require attention but may not prevent execution.

### 3.4 `shell_error()`

Use `shell_error()` to display error messages.

```
shell_error(sh, "Failed to initialize the sensor");
```

This is useful for reporting failures and unexpected conditions.

### 3.5 `shell_fprintf()`

Use `shell_fprintf()` when you need to specify a Shell output color or category explicitly.

```
shell_fprintf(sh, SHELL_INFO, "Value: %d\n", 42);
```

The `SHELL_INFO` parameter selects the informational output category.

### Comparison

| Function          | Purpose                                                 |
| ----------------- | ------------------------------------------------------- |
| `shell_print()`   | General text and formatted output.                      |
| `shell_info()`    | Informational messages.                                 |
| `shell_warn()`    | Warning messages.                                       |
| `shell_error()`   | Error messages.                                         |
| `shell_fprintf()` | Output using an explicitly selected Shell message type. |

**Note:** The actual appearance and colors depend on the Shell backend and its configuration.

## 4. Registering a Root Command

A root command is a top-level command that can be executed directly from the Shell.

To implement a root command, define a handler and register it using `SHELL_CMD_REGISTER()`.

### Example: Version Command

```
#include <zephyr/shell/shell.h>

static int cmd_version(const struct shell *sh,
                       size_t argc,
                       char **argv)
{
    ARG_UNUSED(argc);
    ARG_UNUSED(argv);

    shell_print(sh, "App version: 1.0.0");

    return 0;
}

SHELL_CMD_REGISTER(version,
                   NULL,
                   "Show app version",
                   cmd_version);
```

### Understanding `SHELL_CMD_REGISTER()`

The macro has the following general structure:

```
SHELL_CMD_REGISTER(
    command_name,
    subcommand_set,
    help_description,
    handler
);
```

| Argument           | Description                                                  |
| ------------------ | ------------------------------------------------------------ |
| `command_name`     | Name used to invoke the command.                             |
| `subcommand_set`   | Pointer to the command's subcommand set, or `NULL` if it has none. |
| `help_description` | Description displayed by Shell help.                         |
| `handler`          | Function executed when the command is invoked, or `NULL` when the command serves as a container for subcommands. |

### Running the Command

After building and flashing the application, execute:

```
uart:~$ version
```

Expected output:

```
App version: 1.0.0
```

The command is now available as a top-level Shell command.

## 5. Defining Static Subcommands

Static subcommands allow developers to group related operations under a single root command.

For example, a `demo` command can contain:

- `salute`: Prints a greeting.
- `parse`: Displays the arguments passed to the command.

Each subcommand has its own handler.

### 5.1 Greeting Handler

```
static int cmd_demo_salute(const struct shell *sh,
                           size_t argc,
                           char **argv)
{
    ARG_UNUSED(argc);
    ARG_UNUSED(argv);

    shell_print(sh, "Good Afternoon");

    return 0;
}
```

This handler prints a greeting when the user executes `demo salute`.

### 5.2 Argument Parsing Handler

```
static int cmd_demo_parse(const struct shell *sh,
                          size_t argc,
                          char **argv)
{
    for (int i = 0; i < argc; i++) {
        shell_print(sh, "argv[%d] = %s", i, argv[i]);
    }

    return 0;
}
```

This handler iterates through the supplied arguments and prints each argument with its index.

For example:

```
uart:~$ demo parse hello world
```

Output:

```
argv[0] = hello
argv[1] = world
```

The example demonstrates how a Shell handler can process user-supplied strings.

## 6. Registering a Command Tree

After implementing the handlers, create a static subcommand set and register it under the root command.

### 6.1 Create the Subcommand Set

```
SHELL_STATIC_SUBCMD_SET_CREATE(
    sub_demo,

    SHELL_CMD(salute,
              NULL,
              "Print a greeting",
              cmd_demo_salute),

    SHELL_CMD(parse,
              NULL,
              "Display command arguments",
              cmd_demo_parse),

    SHELL_SUBCMD_SET_END
);
```

### Understanding the Macros

```
**SHELL_STATIC_SUBCMD_SET_CREATE()**
```

Creates a statically defined collection of subcommands.

In this example, the collection is named `sub_demo`.

```
**SHELL_CMD()**
```

Defines an individual command within the subcommand set.

Its general structure is:

```
SHELL_CMD(
    command_name,
    subcommand_set,
    help_description,
    handler
);
```

For the `salute` and `parse` commands, the second argument is `NULL` because neither command has its own child commands.

```
**SHELL_SUBCMD_SET_END**
```

Marks the end of the static subcommand set.

### 6.2 Register the Root Command

```
SHELL_CMD_REGISTER(
    demo,
    &sub_demo,
    "Demo commands",
    NULL
);
```

Here:

- `demo` is the root command.
- `&sub_demo` points to the associated subcommand set.
- `"Demo commands"` is the root command's help description.
- `NULL` means that the root command does not have its own handler.

When the user executes `demo salute`, the Shell dispatches the request to `cmd_demo_salute()`.

When the user executes `demo parse hello world`, the Shell dispatches the request to `cmd_demo_parse()`.

### 6.3 Complete Example

The following code combines the handlers, subcommand set, and root command registration.

```
#include <zephyr/shell/shell.h>
#include <zephyr/sys/util.h>

/* Print a greeting. */
static int cmd_demo_salute(const struct shell *sh,
                           size_t argc,
                           char **argv)
{
    ARG_UNUSED(argc);
    ARG_UNUSED(argv);

    shell_print(sh, "Good Afternoon");

    return 0;
}

/* Display the arguments supplied to the command. */
static int cmd_demo_parse(const struct shell *sh,
                          size_t argc,
                          char **argv)
{
    for (int i = 0; i < argc; i++) {
        shell_print(sh, "argv[%d] = %s", i, argv[i]);
    }

    return 0;
}

/* Define the static subcommand set. */
SHELL_STATIC_SUBCMD_SET_CREATE(
    sub_demo,
    SHELL_CMD(salute,
              NULL,
              "Print a greeting",
              cmd_demo_salute),
    SHELL_CMD(parse,
              NULL,
              "Display command arguments",
              cmd_demo_parse),
    SHELL_SUBCMD_SET_END
);

/* Register the root command. */
SHELL_CMD_REGISTER(
    demo,
    &sub_demo,
    "Demo commands",
    NULL
);
```

This example provides a simple, reusable pattern for implementing hierarchical Shell commands in Zephyr RTOS.

## 7. Testing Commands in the Shell

After compiling, flashing, and starting the Zephyr Shell, execute the following commands.

### Test 1: Greeting Subcommand

Command:

```
uart:~$ demo salute
```

Output:

```
Good Afternoon
```

Explanation:

The Shell identifies `demo` as the root command and `salute` as its subcommand. It then executes `cmd_demo_salute()`.

### Test 2: Argument Parsing

Command:

```
uart:~$ demo parse hello world
```

Output:

```
argv[0] = hello
argv[1] = world
```

Explanation:

The Shell passes `hello` and `world` to the `cmd_demo_parse()` handler through `argv`.

### Test 3: Shell Help

Use the built-in help command to inspect the registered commands.

```
uart:~$ help
```

You can also request help for the root command:

```
uart:~$ demo
```

The available output depends on the Shell configuration and registered commands.

## 8. Important Notes

### 8.1 Required Header

Include the Shell API header when using Shell command registration and output functions.

```
#include <zephyr/shell/shell.h>
```

The `ARG_UNUSED()` macro is provided by Zephyr's utility headers.

```
#include <zephyr/sys/util.h>
```

### 8.2 Shell Configuration

Ensure that Shell support is enabled in your application's `prj.conf`.

A typical UART Shell configuration is:

```
CONFIG_SHELL=y
CONFIG_SERIAL=y
CONFIG_CONSOLE=y
CONFIG_UART_CONSOLE=y
```

The exact configuration requirements depend on your board, chosen Shell backend, and existing application configuration. Your board must also have a suitable UART and console configuration.

### 8.3 Handler Parameters

Use `ARG_UNUSED()` for unused handler parameters to avoid compiler warnings.

```
ARG_UNUSED(argc);
ARG_UNUSED(argv);
```

Do not mark parameters as unused if you need to access them.

### 8.4 Command Registration

Use `SHELL_CMD_REGISTER()` for root commands.

Use `SHELL_STATIC_SUBCMD_SET_CREATE()` to define a static collection of subcommands.

Use `SHELL_CMD()` to define individual commands within that collection.

Use `SHELL_SUBCMD_SET_END` to terminate the collection.

### 8.5 Root Commands With and Without Handlers

A root command can be registered with a handler:

```
SHELL_CMD_REGISTER(version,
                   NULL,
                   "Show app version",
                   cmd_version);
```

Or it can serve as a container for subcommands:

```
SHELL_CMD_REGISTER(demo,
                   &sub_demo,
                   "Demo commands",
                   NULL);
```

The first example executes a handler directly. The second organizes related operations under a shared command name.

## 9. Summary

| Concept                   | Macro or Function                  | Purpose                                         |
| ------------------------- | ---------------------------------- | ----------------------------------------------- |
| Root command registration | `SHELL_CMD_REGISTER()`             | Registers a top-level command.                  |
| Static subcommand set     | `SHELL_STATIC_SUBCMD_SET_CREATE()` | Groups related subcommands.                     |
| Individual subcommand     | `SHELL_CMD()`                      | Defines a command and its handler.              |
| End of subcommand set     | `SHELL_SUBCMD_SET_END`             | Marks the end of the static command collection. |
| General output            | `shell_print()`                    | Prints text and formatted values.               |
| Informational output      | `shell_info()`                     | Displays informational messages.                |
| Warning output            | `shell_warn()`                     | Displays warnings.                              |
| Error output              | `shell_error()`                    | Displays errors.                                |
| Explicit output category  | `shell_fprintf()`                  | Prints using a selected Shell message type.     |
| Unused parameters         | `ARG_UNUSED()`                     | Indicates intentionally unused parameters.      |

### Key Takeaways

1. Zephyr Shell commands can be organized into a hierarchical tree.
2. Root commands provide the entry point for command execution.
3. Subcommands group related operations under a common root.
4. Each command handler receives a Shell instance, an argument count, and an argument array.
5. Shell output functions provide a consistent way to display messages and diagnostic information.
6. Static subcommand registration provides a straightforward way to build structured command-line interfaces for embedded applications.

**Result:** You can now create root commands, implement command handlers, process user arguments, and organize multiple operations under a single Zephyr Shell command.

---

# 3.Part3: Zephyr RTOS Shell - Dynamic Commands and Advanced Features

## Overview

The Zephyr RTOS Shell provides an interactive command-line interface (CLI) for communicating with embedded applications.

In addition to static commands and subcommands, Zephyr supports advanced features that make Shell interfaces more flexible and easier to use.

This guide explores:

- Dynamic subcommands generated at runtime.
- Registering commands that can change while the application is running.
- Adding, executing, listing, and removing dynamic commands.
- Automatic argument validation using `SHELL_CMD_ARG`.
- Dictionary commands using `SHELL_SUBCMD_DICT_SET_CREATE`.
- Accessing parent command information.
- Wildcard support.
- Option parsing with `sys_getopt`.
- Obscured input for sensitive information.

## Table of Contents

- [1. Dynamic Commands](#1-dynamic-commands)
- [2. Creating Dynamic Subcommands](#2-creating-dynamic-subcommands)
- [3. Registering Dynamic Commands](#3-registering-dynamic-commands)
- [4. Using Dynamic Commands in the Shell](#4-using-dynamic-commands-in-the-shell)
- [5. Argument Validation with ](#5-argument-validation-with-shell_cmd_arg)`SHELL_CMD_ARG`
- [6. Dictionary Commands](#6-dictionary-commands)
- [7. Parent Command Access](#7-parent-command-access)
- [8. Wildcard Support](#8-wildcard-support)
- [9. Option Parsing with ](#9-option-parsing-with-sys_getopt)`sys_getopt`
- [10. Obscured Input](#10-obscured-input)
- [11. Summary](#11-summary)

## 1. Dynamic Commands

### What Are Dynamic Commands?

Dynamic commands are Shell commands whose names or available entries can change while the application is running.

Unlike static commands, which are defined at compile time, dynamic commands allow an application to create or remove command entries at runtime.

For example, an application might allow users to add commands dynamically:

```
uart:~$ dynamic add hello
uart:~$ dynamic add world
```

The available commands can then be displayed or accessed through the Shell.

### Static vs. Dynamic Commands

| Feature           | Static Commands                    | Dynamic Commands                                |
| ----------------- | ---------------------------------- | ----------------------------------------------- |
| Definition        | Compile time                       | Entries can change at runtime                   |
| Command names     | Usually fixed                      | Can be added or removed                         |
| Memory management | Typically fixed structures         | Requires storage for dynamic entries            |
| Flexibility       | Suitable for predefined operations | Suitable for runtime-configurable command lists |
| Typical use cases | System control, diagnostics        | Runtime registries, configurable operations     |

**Important:** Dynamic commands do not necessarily require dynamic memory allocation. An application can use a fixed-size array and update its entries at runtime.

This approach is particularly useful in embedded systems where predictable memory usage is important.

## 2. Creating Dynamic Subcommands

Zephyr provides `SHELL_DYNAMIC_CMD_CREATE()` to define a dynamic command set.

The application supplies a callback that returns the command entry corresponding to a requested index.

### 2.1 Define Storage for Dynamic Commands

```
static char dynamic_cmd_buffer[10][50];
static uint8_t dynamic_cmd_cnt;
```

These variables define the storage used by the dynamic command implementation.

| Variable             | Description                                                  |
| -------------------- | ------------------------------------------------------------ |
| `dynamic_cmd_buffer` | Stores up to 10 command names, with space for 50 characters per entry. |
| `dynamic_cmd_cnt`    | Tracks the number of currently registered dynamic commands.  |

This implementation uses a fixed-size buffer rather than allocating memory dynamically.

The maximum number of stored commands is 10.

Each command name must fit into its corresponding buffer, including the terminating null character.

### 2.2 Implement the Dynamic Command Callback

```
static void dynamic_cmd_get(size_t idx,
                            struct shell_static_entry *entry)
{
    if (idx < dynamic_cmd_cnt)
    {
        entry->syntax = dynamic_cmd_buffer[idx];
        entry->handler = NULL;
        entry->subcmd = NULL;
        entry->help = "Dynamic command.";
    }
    else
    {
        entry->syntax = NULL;
    }
}
```

### Understanding the Callback

The callback receives two parameters:

- `idx`: The index of the requested command.
- `entry`: A pointer to the Shell entry structure that must be populated.

When `idx` refers to an existing command, the callback provides its information.

| Structure member | Purpose                                     |
| ---------------- | ------------------------------------------- |
| `syntax`         | Command name or syntax.                     |
| `handler`        | Handler function associated with the entry. |
| `subcmd`         | Pointer to child commands, if any.          |
| `help`           | Help description displayed for the command. |

In this example:

```
entry->syntax = dynamic_cmd_buffer[idx];
```

The command name comes from the corresponding buffer entry.

```
entry->handler = NULL;
entry->subcmd = NULL;
```

The dynamic entry has neither a direct handler nor its own child command set in this implementation.

```
entry->help = "Dynamic command.";
```

The command receives a descriptive help message.

When the requested index is outside the current command list:

```
entry->syntax = NULL;
```

This signals the end of the available dynamic command entries.

### 2.3 Create the Dynamic Command Set

```
SHELL_DYNAMIC_CMD_CREATE(m_sub_dynamic, dynamic_cmd_get);
```

This macro creates a dynamic subcommand set named `m_sub_dynamic`.

The Shell calls `dynamic_cmd_get()` to retrieve the available entries.

**Key concept:** The callback allows the Shell to discover the current command list without requiring every possible command to be defined individually at compile time.

## 3. Registering Dynamic Commands

Dynamic commands can be combined with ordinary static subcommands.

In this example, the root command is `dynamic`, and its static subcommands provide operations for managing the dynamic command list.

### 3.1 Define the Command Tree

```
dynamic
├── add <name>
├── execute <dynamic_command>
├── remove <name>
└── show
```

The static commands provide the management interface, while the dynamic command set supplies the entries that can change at runtime.

### 3.2 Register the Static Management Commands

```
SHELL_STATIC_SUBCMD_SET_CREATE(
    m_sub_dynamic_cmds,

    SHELL_CMD(add,
              NULL,
              "Add a dynamic command.",
              cmd_dynamic_add),

    SHELL_CMD(execute,
              &m_sub_dynamic,
              "Execute a command.",
              cmd_dynamic_execute),

    SHELL_CMD(remove,
              &m_sub_dynamic,
              "Remove a command.",
              cmd_dynamic_remove),

    SHELL_CMD(show,
              NULL,
              "Show all commands.",
              cmd_dynamic_show),

    SHELL_SUBCMD_SET_END
);
```

### Understanding the Management Commands

| Command   | Purpose                                                      |
| --------- | ------------------------------------------------------------ |
| `add`     | Adds a command name to the dynamic command registry.         |
| `execute` | Provides access to the dynamic command entries.              |
| `remove`  | Provides access to the dynamic command entries for removal operations. |
| `show`    | Displays the currently registered command names.             |

The handlers `cmd_dynamic_add`, `cmd_dynamic_execute`, `cmd_dynamic_remove`, and `cmd_dynamic_show` must be implemented separately.

The dynamic command set is associated with the `execute` and `remove` entries in this example.

**Important:** Attaching a dynamic command set to a subcommand makes the dynamic entries available beneath that subcommand. It does not automatically implement adding, executing, or removing commands; those behaviors depend on the corresponding handlers.

### 3.3 Register the Root Command

```
SHELL_CMD_REGISTER(
    dynamic,
    &m_sub_dynamic_cmds,
    "Dynamic command demo.",
    NULL
);
```

The registration defines `dynamic` as the root command and associates it with the static management commands.

The `NULL` handler indicates that the root command itself does not have a direct handler.

The Shell can now expose the command hierarchy through its command registration mechanism.

## 4. Using Dynamic Commands in the Shell

Consider the following example session:

```
uart:~$ dynamic add hello
uart:~$ dynamic add world
uart:~$ dynamic execute <Tab>
hello  world
```

### Explanation

**Step 1: Add a command**

```
uart:~$ dynamic add hello
```

The `cmd_dynamic_add()` handler should validate the supplied name, ensure that capacity is available, and store the name in `dynamic_cmd_buffer`.

**Step 2: Add another command**

```
uart:~$ dynamic add world
```

The second name is stored as another entry in the command registry.

**Step 3: Request command completion**

```
uart:~$ dynamic execute <Tab>
```

The Shell can use the dynamic command callback to discover the available entries and offer command completion.

The exact behavior depends on the command implementation and the Shell's completion configuration.

### Important Implementation Considerations

When implementing the management handlers, ensure that:

1. The command count never exceeds the array capacity.
2. Names are copied safely and are always null-terminated.
3. Duplicate command names are handled consistently.
4. Removal updates the registry without leaving stale entries.
5. Command entries remain valid while the Shell uses them.
6. Concurrent updates are synchronized if commands can be modified by multiple threads.

For a fixed-size registry, `dynamic_cmd_cnt` must always represent the number of valid entries.

## 5. Argument Validation with `SHELL_CMD_ARG`

Zephyr provides `SHELL_CMD_ARG()` for defining commands with explicit argument-count constraints.

This allows the Shell to validate the number of arguments before invoking the command handler.

### 5.1 General Syntax

```
SHELL_CMD_ARG(
    command_name,
    subcommand_set,
    help_description,
    handler,
    mandatory_arguments,
    optional_arguments
);
```

The final two arguments specify the expected number of mandatory and optional arguments.

### 5.2 Example

```
static int cmd_set_level(const struct shell *sh,
                         size_t argc,
                         char **argv)
{
    ARG_UNUSED(argc);

    shell_print(sh, "Level: %s", argv[1]);

    return 0;
}

SHELL_CMD_ARG(
    level,
    NULL,
    "Set a level: level <value>",
    cmd_set_level,
    1,
    0
);
```

This example defines a command named `level` that requires one argument.

Usage:

```
uart:~$ level high
```

Expected output:

```
Level: high
```

Because the command requires one mandatory argument and allows no optional arguments, the Shell can reject invocations with an incorrect argument count.

### Why Use Automatic Argument Validation?

Argument-count validation helps prevent handlers from accessing arguments that were not supplied.

For example, without proper validation, accessing `argv[1]` when no argument exists could cause incorrect behavior.

**Note:** Argument-count validation does not validate the contents of an argument. If a command expects an integer, valid range, or specific string, the handler should still validate that value.

## 6. Dictionary Commands

Dictionary commands provide a way to associate command strings with predefined values or operations.

Zephyr provides `SHELL_SUBCMD_DICT_SET_CREATE()` for defining dictionary-style subcommand sets.

This approach is useful when several textual options correspond to values understood by the application.

### Conceptual Example

Suppose an embedded application supports three operating modes:

```
mode
├── idle
├── active
└── maintenance
```

Each entry can be associated with an application-defined value.

For example:

| Command       | Application value |
| ------------- | ----------------- |
| `idle`        | `0`               |
| `active`      | `1`               |
| `maintenance` | `2`               |

The application can use these values to select a mode without manually comparing every command string in its handler.

### General Pattern

The Zephyr dictionary-subcommand API uses entries that associate command names with values.

The exact macro arguments and entry format should match the API version and definitions in your Zephyr source tree.

Consult the Shell header:

```
include/zephyr/shell/shell.h
```

Search for the macro definition:

```
grep -n -A30 -B10 "SHELL_SUBCMD_DICT_SET_CREATE" \
    /opt/zephyrproject/zephyr/include/zephyr/shell/shell.h
```

Adjust the path to match your Zephyr installation.

**Key takeaway:** Dictionary commands are useful when the command vocabulary is predefined and each entry maps to an application-specific value.

## 7. Parent Command Access

In a hierarchical command tree, a child command may need information associated with its parent.

For example:

```
device
├── status
├── start
└── stop
```

The `status`, `start`, and `stop` handlers belong to the `device` command group.

Depending on the Shell API and the registration mechanism, a handler may need to identify its invocation context or retrieve information associated with a parent entry.

### Why Is Parent Context Useful?

Parent command context can help when:

- Multiple commands share common configuration.
- A command belongs to a particular device or subsystem.
- A handler needs to distinguish between similar command groups.
- A command hierarchy represents a collection of configurable objects.

### Important Note

Parent-command access depends on the specific Zephyr Shell API and version.

Do not assume that a handler can directly access a parent command's private data through `argv` or the Shell instance.

For reliable implementation, consult the relevant API documentation and the command-entry definitions in `shell.h`.

## 8. Wildcard Support

Wildcards are symbols that can represent patterns rather than literal characters.

In command-line interfaces, common wildcard examples include:

| Wildcard | Typical meaning                  |
| -------- | -------------------------------- |
| `*`      | Matches zero or more characters. |
| `?`      | Matches one character.           |

For example, a pattern such as `sensor*` might match names including:

```
sensor1
sensor2
sensor_temp
```

### Wildcards in Zephyr Shell

Wildcard support must be distinguished from ordinary Shell command completion.

A Shell may provide completion for registered commands without automatically supporting wildcard expansion.

If wildcard matching is required, verify the capabilities of your Zephyr version and the relevant command implementation.

If the required behavior is not provided by the Shell, the application can implement its own pattern-matching logic inside a handler.

For example, a handler could accept a pattern and filter a registry of device names.

**Security consideration:** Validate wildcard patterns carefully when they can select devices, files, or operations with different access privileges.

## 9. Option Parsing with `sys_getopt`

The `sys_getopt` API provides option-parsing functionality for processing command-line options.

Options allow commands to accept flags or parameters in a structured format.

### Example Command Syntax

```
uart:~$ device --verbose --timeout 100
```

In this example:

- `--verbose` requests additional diagnostic output.
- `--timeout 100` supplies a timeout value.

The exact syntax supported depends on the parser and the application's command-line conventions.

### Why Use Option Parsing?

Option parsing is useful when commands have multiple configurable parameters.

For example:

```
uart:~$ network --verbose --retry 3
```

Possible options include:

| Option      | Purpose                    |
| ----------- | -------------------------- |
| `--verbose` | Enable additional logging. |
| `--retry`   | Specify the retry count.   |
| `--timeout` | Specify a timeout.         |

### Implementation Notes

Before using `sys_getopt`, check the relevant headers and API documentation in your Zephyr version.

Option parsing should be implemented with attention to:

1. Unknown options.
2. Missing option values.
3. Invalid numeric values.
4. Duplicate options.
5. Out-of-range values.
6. Interaction between positional arguments and options.

For numeric options, convert the input carefully and validate the resulting value before using it.

**Key takeaway:** Option parsing provides a structured way to handle flags and configurable parameters in more complex Shell commands.

## 10. Obscured Input

Obscured input prevents typed characters from being displayed normally while the user enters sensitive information.

This feature is useful for commands that request data such as:

- Passwords.
- Authentication tokens.
- PINs.
- Other sensitive credentials.

### Example Interaction

A conceptual authentication command might look like this:

```
uart:~$ login
Password:
```

Instead of displaying the entered characters, the interface obscures the input.

### Why Is Obscured Input Important?

Displaying sensitive input can expose credentials to anyone who can see the terminal.

Obscuring input reduces this particular risk.

However, obscured input alone does not provide secure credential storage or secure authentication.

### Security Recommendations

- Avoid printing passwords or tokens in diagnostic messages.
- Do not store sensitive credentials in ordinary command buffers longer than necessary.
- Validate credentials without exposing them through logs.
- Use appropriate authentication and authorization mechanisms.
- Be aware that terminal history, logs, memory dumps, and transport security can introduce additional risks.

**Important:** Confirm the precise API and configuration needed for obscured input in your Zephyr version before implementing it.

## 11. Summary

| Feature                            | Purpose                                                      |
| ---------------------------------- | ------------------------------------------------------------ |
| Dynamic commands                   | Allow command entries to change at runtime.                  |
| `SHELL_DYNAMIC_CMD_CREATE()`       | Defines a dynamic command set using a callback.              |
| `SHELL_STATIC_SUBCMD_SET_CREATE()` | Defines a static set of subcommands.                         |
| `SHELL_CMD_REGISTER()`             | Registers a root command.                                    |
| `SHELL_CMD_ARG()`                  | Defines a command with explicit argument-count constraints.  |
| `SHELL_SUBCMD_DICT_SET_CREATE()`   | Defines dictionary-style subcommand entries.                 |
| Parent command access              | Supports operations that depend on command hierarchy or invocation context. |
| Wildcards                          | Allow pattern-based selection when supported or implemented. |
| `sys_getopt`                       | Provides option-parsing functionality.                       |
| Obscured input                     | Helps prevent sensitive characters from being displayed during input. |

### Key Takeaways

1. Dynamic command sets allow the available command names to change while an application is running.
2. Dynamic command callbacks provide the Shell with the current command entries.
3. Static commands can manage a dynamic registry by adding, removing, listing, and selecting entries.
4. `SHELL_CMD_ARG()` supports argument-count validation before handler execution.
5. Dictionary commands are useful for mapping predefined strings to application values.
6. Parent context and wildcard behavior depend on the relevant APIs and implementation.
7. Option parsing is useful for commands with multiple flags and parameters.
8. Obscured input is an important usability feature when collecting sensitive information.

**Conclusion:** These features extend the Zephyr Shell beyond simple static commands, making it possible to develop more flexible, configurable, and user-friendly command-line interfaces for embedded applications.

---

# 4.Part4: Zephyr RTOS Shell: Built-in Commands and Custom Driver Integration

## Overview

The Zephyr RTOS Shell provides an interactive command-line interface (CLI) for inspecting system state, testing peripherals, and interacting with application-specific functionality.

Many Zephyr subsystems provide built-in Shell commands that can be enabled through Kconfig. Developers can also expose custom drivers through the Shell, allowing them to control hardware and inspect its state without implementing a separate testing application for every operation.

This guide covers:

- Built-in Shell commands for Zephyr subsystems.
- Enabling subsystem Shell support through Kconfig.
- Integrating a custom driver with the Shell.
- Registering custom commands and subcommands.
- Building and flashing the application.
- Testing the driver through a UART Shell.
- Best practices for writing safe and maintainable Shell commands.

## Table of Contents

- [1. Built-in Subsystem Commands](#1-built-in-subsystem-commands)
- [2. Enabling Shell Support](#2-enabling-shell-support)
- [3. Demo: Exposing a Custom Driver](#3-demo-exposing-a-custom-driver)
- [4. Registering Custom Shell Commands](#4-registering-custom-shell-commands)
- [5. Building and Flashing](#5-building-and-flashing)
- [6. Live Testing](#6-live-testing)
- [7. Shell Best Practices](#7-shell-best-practices)
- [8. Summary](#8-summary)

## 1. Built-in Subsystem Commands

Many Zephyr subsystems provide built-in Shell commands for interacting with hardware and system services.

Instead of implementing every diagnostic command from scratch, developers can enable the appropriate subsystem Shell configuration.

### Available Subsystem Shell Options

| Subsystem   | Kconfig Option               | Purpose                                                      |
| ----------- | ---------------------------- | ------------------------------------------------------------ |
| GPIO        | `CONFIG_GPIO_SHELL=y`        | Provides Shell commands for GPIO operations.                 |
| I2C         | `CONFIG_I2C_SHELL=y`         | Provides commands for interacting with supported I2C controllers and devices. |
| Sensor      | `CONFIG_SENSOR_SHELL=y`      | Provides commands for inspecting supported sensor devices and channels. |
| Flash       | `CONFIG_FLASH_SHELL=y`       | Provides commands for interacting with supported flash devices. |
| ADC         | `CONFIG_ADC_SHELL=y`         | Provides commands for inspecting and testing ADC functionality. |
| File System | `CONFIG_FILE_SYSTEM_SHELL=y` | Provides commands for working with supported file systems.   |
| Kernel      | `CONFIG_KERNEL_SHELL=y`      | Enables available kernel-related Shell functionality.        |

**Note:** The exact commands and supported operations depend on the Zephyr version, enabled subsystem features, device drivers, and hardware configuration. Some options may require additional dependencies.

### Why Are Built-in Commands Useful?

Built-in subsystem commands help developers:

- Inspect peripheral configuration.
- Test hardware during development.
- Diagnose device initialization problems.
- Validate subsystem functionality interactively.
- Reduce the amount of custom diagnostic code required.

For example, GPIO Shell support can help test a GPIO device independently of the application's main functionality.

However, the availability of a subsystem's Shell commands does not guarantee that every device or operation is supported.

## 2. Enabling Shell Support

Subsystem Shell functionality is typically enabled through the application's `prj.conf` file.

### Example Configuration

```
# Enable the Shell
CONFIG_SHELL=y

# Enable UART console support
CONFIG_SERIAL=y
CONFIG_CONSOLE=y
CONFIG_UART_CONSOLE=y

# Enable subsystem Shell commands as needed
CONFIG_GPIO_SHELL=y
CONFIG_I2C_SHELL=y
CONFIG_SENSOR_SHELL=y
CONFIG_FLASH_SHELL=y
CONFIG_ADC_SHELL=y
CONFIG_FILE_SYSTEM_SHELL=y
CONFIG_KERNEL_SHELL=y
```

This configuration illustrates how to enable the listed features. You do not need to enable every subsystem in every application.

For a real project, enable only the features required by your hardware and application.

### Important Considerations

1. Ensure the underlying subsystem is enabled.
2. Verify that the board supports the required peripheral.
3. Enable the appropriate device driver.
4. Check Kconfig dependencies when a configuration option cannot be enabled.
5. Verify the available commands using the Shell's help functionality.

For example, enabling GPIO Shell support does not automatically configure a GPIO pin or implement application-specific motor control.

## 3. Demo: Exposing a Custom Driver

### Objective

The goal of this demonstration is to reuse the previous L06 Sandbox project and expose a custom motor driver through the Zephyr Shell.

The application will provide commands to:

- Turn the motor ON.
- Turn the motor OFF.
- Display the GPIO configuration and driver state.

Instead of changing the application source code every time the motor needs to be tested, the developer can execute commands directly from the Shell.

### Project Information

**Project path:**

```
/home/mariano/zephyrproject/apps/sandbox/app
```

**Target board:**

```
nucleo_f429zi
```

**Custom driver example:**

The demonstration uses GPIOB pin 7 to represent the motor control output.

The actual GPIO configuration and motor-control implementation must match the custom driver's initialization and the hardware connected to the board.

### Expected Command Structure

```
motor
├── on
├── off
└── status
```

| Command        | Description                                      |
| -------------- | ------------------------------------------------ |
| `motor on`     | Turns the motor ON through the custom driver.    |
| `motor off`    | Turns the motor OFF through the custom driver.   |
| `motor status` | Displays the GPIO configuration and motor state. |

This structure groups related motor operations under a single root command.

## 4. Registering Custom Shell Commands

Zephyr allows developers to register custom commands using Shell registration macros.

The following example registers a `motor` root command with three subcommands.

### 4.1 Define the Command Handlers

Each subcommand needs a handler that performs the corresponding operation.

A typical handler follows this structure:

```
static int cmd_motor_on(const struct shell *sh,
                        size_t argc,
                        char **argv)
{
    ARG_UNUSED(argc);
    ARG_UNUSED(argv);

    /* Call the custom driver to turn the motor ON. */

    shell_print(sh, "Motor state set to ON");

    return 0;
}
```

The handler receives:

- `sh`: The Shell instance used for output.
- `argc`: The number of arguments passed to the handler.
- `argv`: The command arguments.

The handler should invoke the actual driver operation before reporting success. The example above illustrates the command structure; the driver call must be supplied by the application.

### 4.2 Define the OFF Handler

```
static int cmd_motor_off(const struct shell *sh,
                         size_t argc,
                         char **argv)
{
    ARG_UNUSED(argc);
    ARG_UNUSED(argv);

    /* Call the custom driver to turn the motor OFF. */

    shell_print(sh, "Motor state set to OFF");

    return 0;
}
```

This handler should instruct the driver to turn the motor off.

### 4.3 Define the Status Handler

```
static int cmd_motor_status(const struct shell *sh,
                            size_t argc,
                            char **argv)
{
    ARG_UNUSED(argc);
    ARG_UNUSED(argv);

    /*
     * Query the actual GPIO configuration and driver state.
     * Replace the example output with values obtained
     * from the application or custom driver.
     */

    shell_print(sh, "GPIO: port=GPIOB pin=7, state=OFF");

    return 0;
}
```

The status handler should report the actual state obtained from the driver or application.

**Important:** The example output is illustrative. A production implementation should not hard-code `OFF` if the motor's state can change at runtime.

### 4.4 Create the Static Subcommand Set

```
#include <zephyr/shell/shell.h>
#include <zephyr/sys/util.h>

SHELL_STATIC_SUBCMD_SET_CREATE(
    sub_motor,

    SHELL_CMD(on,
              NULL,
              "Turn motor ON",
              cmd_motor_on),

    SHELL_CMD(off,
              NULL,
              "Turn motor OFF",
              cmd_motor_off),

    SHELL_CMD(status,
              NULL,
              "Show GPIO and driver state",
              cmd_motor_status),

    SHELL_SUBCMD_SET_END
);
```

#### Understanding the Macros

```
**SHELL_STATIC_SUBCMD_SET_CREATE()**
```

Creates a statically defined collection of subcommands.

```
**SHELL_CMD()**
```

Defines an individual subcommand, its help description, and its handler.

```
**SHELL_SUBCMD_SET_END**
```

Marks the end of the subcommand list.

Each command has a `NULL` child-command set because `on`, `off`, and `status` do not contain additional subcommands.

### 4.5 Register the Root Command

```
SHELL_CMD_REGISTER(
    motor,
    &sub_motor,
    "Custom driver test commands",
    NULL
);
```

This registration creates the root command `motor` and associates it with the three subcommands.

The root handler is `NULL` because the root command is used to organize the child commands rather than execute an operation directly.

### 4.6 Required Driver Integration

Registering Shell commands only creates the command interface. It does not automatically connect those commands to the motor hardware.

For a complete implementation:

1. Initialize the custom driver.
2. Configure the motor-control GPIO.
3. Implement the ON and OFF driver operations.
4. Call the appropriate driver functions from the command handlers.
5. Obtain the actual driver state in the status handler.
6. Handle driver errors and report them through the Shell.

This separation keeps the Shell interface independent of the details of hardware control.

## 5. Building and Flashing

After configuring the application and implementing the command handlers, build and flash the project.

### Step 1: Navigate to the Application Directory

```
cd /home/mariano/zephyrproject/apps/sandbox/app
```

### Step 2: Build the Application

```
west build -b nucleo_f429zi
```

The build process compiles the application, resolves Kconfig settings, generates the device tree, and links the final firmware.

If the build fails, inspect the reported compiler errors and configuration warnings before proceeding.

### Step 3: Flash the Firmware

```
west flash
```

The flashing process depends on the board's supported programming interface and the available flashing tools.

### Step 4: Open the Shell

Connect to the board's configured UART console using an appropriate serial terminal.

After the application starts, the Zephyr Shell should display its prompt if Shell support and the UART backend are configured correctly.

**Note:** The project path above belongs to the original lecture environment. Replace it with your own project directory when reproducing the demonstration.

## 6. Live Testing

Once the application is running, test the registered commands through the Shell.

### Test 1: Inspect Motor State

Command:

```
uart:~$ motor status
```

Example output:

```
GPIO: port=GPIOB pin=7, state=OFF
```

This command should report the configured GPIO and the motor driver's current state.

### Test 2: Turn the Motor ON

Command:

```
uart:~$ motor on
```

Example output:

```
Motor state set to ON
```

The handler should invoke the driver's ON operation before displaying the success message.

### Test 3: Turn the Motor OFF

Command:

```
uart:~$ motor off
```

Example output:

```
Motor state set to OFF
```

The handler should invoke the driver's OFF operation before reporting success.

### Test 4: Display Command Help

Command:

```
uart:~$ motor -h
```

Example output:

```
motor - Custom driver test commands
```

The available help output may include additional information about the registered subcommands, depending on the Zephyr version and Shell configuration.

### Expected Test Sequence

```
uart:~$ motor status
GPIO: port=GPIOB pin=7, state=OFF

uart:~$ motor on
Motor state set to ON

uart:~$ motor status
GPIO: port=GPIOB pin=7, state=ON

uart:~$ motor off
Motor state set to OFF

uart:~$ motor status
GPIO: port=GPIOB pin=7, state=OFF
```

The final status values should reflect the actual state maintained by the custom driver.

**Hardware safety:** Before testing motor commands, verify the GPIO voltage levels, motor-driver wiring, power supply, and emergency stop arrangements. Never connect a motor directly to a microcontroller GPIO pin.

## 7. Shell Best Practices

Following good Shell implementation practices improves reliability, maintainability, and system responsiveness.

### 7.1 Use Shell Output Functions

Use Zephyr's Shell output functions rather than `printf()` inside Shell command handlers.

```
shell_print(sh, "Operation completed");
shell_warn(sh, "Warning: low battery");
shell_error(sh, "Driver initialization failed");
```

These functions integrate output with the Shell instance.

Avoid using `printf()` as the default mechanism for command output because it may bypass the Shell's configured output path and formatting behavior.

### 7.2 Keep Command Handlers Short

Command handlers should primarily validate input, invoke the appropriate application or driver operation, and report the result.

Avoid performing long-running operations directly inside the Shell thread.

For example, a handler that starts an extended motor test can submit work to a worker thread and return promptly.

This helps keep the Shell responsive to additional commands.

### 7.3 Validate Arguments

Use `SHELL_CMD_ARG()` when the command requires a specific number of arguments.

This allows the Shell to validate argument counts before calling the handler.

For example, a command that sets a configurable motor speed should validate the supplied argument and ensure the requested speed is within the permitted range.

Argument-count validation does not replace validation of argument contents.

### 7.4 Add Help Descriptions

Provide useful descriptions for every command and subcommand.

For example:

```
SHELL_CMD(status,
          NULL,
          "Show GPIO and driver state",
          cmd_motor_status)
```

Clear help strings make the interface easier to discover and use.

### 7.5 Consider `CONFIG_SHELL_MINIMAL`

Zephyr provides `CONFIG_SHELL_MINIMAL` for minimal Shell configurations.

This option is worth considering when reducing the Shell's footprint for a production application.

However, a minimal configuration may limit available features. Review the Kconfig documentation for your Zephyr version and verify that the required command functionality remains available.

### 7.6 Never Call Shell Output Functions from an ISR

Do not call Shell print functions directly from interrupt service routines (ISRs).

Instead, an ISR can signal a worker thread or enqueue an event. The thread can then process the event and report relevant information through the Shell.

This avoids performing potentially unsuitable output operations in interrupt context.

### 7.7 Avoid Blocking the Shell Thread

Long delays, blocking operations, and lengthy hardware tests can prevent the Shell from processing additional commands promptly.

Use worker threads, work queues, or other appropriate asynchronous mechanisms for long-running tasks.

If a command waits for a hardware operation, use a bounded timeout and provide a clear status or error result.

### 7.8 Terminate Every Static Subcommand Set

Always include:

```
SHELL_SUBCMD_SET_END
```

at the end of a static subcommand set.

This marks the end of the command definitions and is required by the static subcommand registration pattern.

### 7.9 Handle Driver Errors Correctly

Do not report success before confirming that the driver operation succeeded.

For example:

```
int ret = motor_driver_set_state(true);

if (ret < 0) {
    shell_error(sh, "Failed to turn motor ON: %d", ret);
    return ret;
}

shell_print(sh, "Motor state set to ON");

return 0;
```

Here, `motor_driver_set_state()` is an illustrative function name; replace it with the API exposed by your actual driver.

Checking return values prevents the Shell from reporting successful hardware operations when the underlying driver has failed.

### 7.10 Follow Least Privilege

Shell commands that control motors, erase flash, modify configuration, or perform other sensitive operations should be exposed only in appropriate application configurations.

Consider whether production firmware should include development-only diagnostic commands, and ensure safety-critical operations have appropriate access controls.

## 8. Summary

### Built-in Shell Support

Zephyr provides subsystem Shell integrations for functionality such as GPIO, I2C, sensors, flash, ADC, file systems, and kernel diagnostics.

These features can simplify peripheral testing and system inspection during development.

### Custom Driver Integration

A custom driver can be exposed through a structured Shell command hierarchy:

```
motor
├── on
├── off
└── status
```

The implementation consists of:

1. Defining command handlers.
2. Creating a static subcommand set.
3. Registering the root command.
4. Connecting the handlers to the custom driver.
5. Building and flashing the firmware.
6. Testing the commands through the UART Shell.

### Key Takeaways

- Enable only the subsystem Shell features required by the application.
- Use Shell registration macros to organize custom commands.
- Keep Shell command handlers short and responsive.
- Validate command arguments and driver return values.
- Use Shell output functions for command messages.
- Avoid calling Shell output functions from ISR context.
- Use asynchronous processing for lengthy operations.
- Include descriptive help text and terminate static subcommand sets correctly.
- Verify that reported hardware states reflect actual driver state.

**Conclusion:** The Zephyr Shell is a valuable development and diagnostic tool. By combining built-in subsystem commands with custom driver commands, developers can inspect hardware, validate driver functionality, and control application behavior through a consistent command-line interface.

----

