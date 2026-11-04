# PlatformPHP: Architecture and Usage Model

## Introduction

PlatformPHP is a software platform (a collection of interconnected libraries) for ESP32 microcontrollers. It combines a command-line shell (ESPShell), a PHP-like scripting engine (PH8), and a built-in HTTP server.

Repositories:

- https://github.com/vvb333007/PH8
- https://github.com/vvb333007/espshell

The goal is to provide a ready-to-use environment for developing IoT devices and automation systems. Most application logic can be written in PHP while retaining direct access to the capabilities of C/C++ and ESP-IDF.

The platform is designed for two use cases:

1. Using the ready-made PlatformPHP firmware without writing any custom C/C++ code.
2. Extending the platform with a custom Arduino sketch, which is compiled together with the base firmware but runs as a separate, parallel task.

## What Does It Look Like?

PlatformPHP provides a ready-made software foundation for an embedded device. The firmware includes:

```text
ESPShell + PH8 + HTTP server + system components
```

The user gets working firmware, a filesystem, a command-line shell, a PHP environment, and a web server. Application scripts can be stored as PHP files and executed without rebuilding the firmware.

When PHP and the existing system APIs are not sufficient, the user can add a custom C/C++ sketch. The sketch does not replace ESPShell, PH8, or the HTTP server, nor does it turn them into parts of the sketch. Instead, it is an additional task running in parallel with the base platform and interacting with it through the provided interfaces.

PHP code can call functions implemented in the sketch's C/C++ code when needed. Conversely, the sketch's C/C++ code can call functions and methods defined in PHP scripts.

## PlatformPHP Components

### 1. ESPShell

ESPShell is a command-line shell and system environment for device management.

It provides interactive access to the device, command execution, GPIO control, networking, signals, and other system capabilities. ESPShell also serves as one of the primary tools for platform administration and diagnostics.

ESPShell is not mandatory for user applications: the device can execute PHP scripts and handle HTTP requests without continuous operator interaction. The shell can be disabled when it is not needed.

### 2. PH8

PH8 is a compact PHP-like scripting engine designed to execute application logic directly on the ESP32.

It can be used to create automation scripts, event handlers, networking logic, and application programs. Scripts are stored in the device's filesystem and can be modified without rebuilding the main firmware.

PH8 supports registering functions implemented in C. This allows PHP scripts to access hardware features, system services, and libraries without having to reimplement them in PHP.

This means that low-level functionality can be implemented in C/C++, while the application logic that uses it can be written in PHP.

### 3. Built-in HTTP Server

The HTTP server allows the device to provide web interfaces and handle HTTP requests directly on the ESP32.

PHP scripts can act as server-side request handlers. This makes it possible to implement control pages, APIs, configuration forms, data handlers, and other web functionality without requiring an external server.

### 4. Filesystem and Storage

The filesystem is used to store PHP scripts, configuration files, application data, and other files. Scripts can be updated independently of the firmware.

In addition to FAT, LittleFS, and SPIFFS, PlatformPHP supports TARFS, an immutable filesystem recommended for read-only data such as system scripts, icons, HTML files, and similar resources.

TARFS supports `mmap()`, which makes it extremely fast (10–30 times faster than LittleFS and FATFS).

https://github.com/vvb333007/tarfs/blob/main/docs/FAQ_RU.md

### 5. ESP32 System Environment

The platform runs on top of the ESP32 software stack, including FreeRTOS, Arduino, and ESP-IDF.

Functions from the corresponding APIs can be called directly from PHP code.

## Default Operation: Shell + PHP + HTTP

In its basic configuration, the user installs the ready-made PlatformPHP firmware. All the main components are already integrated and working together.

After startup, the system shell, PHP environment, and built-in HTTP server are available. The user can place scripts and application files in the filesystem (for example, using the shell), configure which programs should run, and use the device as a standalone network or IoT controller.

Simple automation tasks do not require writing any code at all — neither C nor PHP. The shell provides commands for declaring conditions and defining actions to be performed when those conditions are met.

See: [ESPShell — Aliases and Events (Russian documentation)](https://vvb333007.github.io/espshell/html/Aliases_And_Events.ru.html)

For most application logic changes, simply editing PHP files is sufficient. There is no need to rebuild or reflash the firmware every time a script is changed.

PHP scripts can be executed from the shell command line or through HTTP requests.

PHP scripts run concurrently, each in its own virtual machine (VM).

A startup script can be configured to run automatically when the device boots.

System services — scripts started once and kept running permanently — can also be written in PHP. These scripts have access to all low-level ESP-IDF and Arduino API calls.

## Adding a Custom C/C++ Sketch

When the platform's built-in capabilities are not sufficient, the user can add a custom Arduino sketch.

The sketch includes the PlatformPHP interfaces:

```cpp
#include "espshell.h"
#include "ph8.h"
```

This makes the platform APIs required for interaction with ESPShell and PH8 available to the sketch.

The sketch can use the Arduino API, ESP-IDF, FreeRTOS, third-party libraries, and custom C/C++ code, subject to the capabilities of the selected board and build configuration.

The important point is that the user sketch does not replace the base firmware and is not part of the PHP engine. It is a separate, parallel task (process) running as part of the same firmware.

The execution model looks like this:

```text
One ESP32 firmware image
|
+-- PlatformPHP
|    +-- ESPShell
|    +-- PH8 and its VMs
|    +-- HTTP server
|    +-- system services
|
+-- User sketch
     +-- setup()/loop() or its own task logic
     +-- user drivers and libraries
     +-- custom hardware and networking functions
```

The base platform continues to operate independently of the user sketch. The sketch runs alongside it, performing its own tasks — for example, reading sensors, controlling peripherals, implementing a specialized protocol, or performing computations.

This approach eliminates the need to put the entire application into one large sketch or modify the existing platform just to add a specific hardware feature.

## Interaction Between the Sketch and PHP

A sketch can provide PH8 with functions implemented in C/C++. Once registered, these functions become available to PHP scripts just like ordinary callable functions.

For example, the user can implement sensor reading in C/C++, while a PHP script calls the provided function, processes its result, and uses it in a web interface or automation logic.

An illustrative PHP example:

```php
$value = sensor_read();
echo "Sensor: $value";
```

Here, `sensor_read()` is a function provided by the user's native code. The PHP script does not need to know how the sensor works internally or how it is connected.

Interaction can also work in the opposite direction: the sketch's C/C++ code can call PHP functions and methods through the PH8 interface.

```cpp
ph8_vm_call_function();
ph8_vm_call_class_method();
```
