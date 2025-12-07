# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

Sensor Watch is a board replacement for the Casio F-91W wristwatch, powered by a Microchip SAM L22 ARM Cortex M0+ microcontroller. The project includes:
- Low-level watch library for hardware abstraction
- **Movement**: A framework for building watch face applications
- Browser-based emulator for testing

## Build Commands

**Prerequisites**: Install the GNU Arm Embedded Toolchain (`apt install gcc-arm-none-eabi` on Debian/Ubuntu).

### Building Movement firmware (most common)
```bash
cd movement/make
COLOR=GREEN make        # Set COLOR to RED, BLUE, GREEN, or PRO based on board variant
```

### Building for the emulator
```bash
cd movement/make
emmake make             # Requires emscripten installed
python3 -m http.server -d build-sim
# Visit http://localhost:8000/watch.html
```

### Installing to watch
1. Double-tap the reset button (LED pulses red, "WATCHBOOT" drive appears)
2. Run `make install` from the project's make directory

### Clean build
```bash
make clean
```

## Architecture

### Directory Structure
- `watch-library/` - Core hardware abstraction layer
  - `hardware/` - SAM L22 hardware-specific code (HAL, drivers, linker scripts)
  - `shared/` - Code shared between hardware and simulator (drivers, utilities)
  - `simulator/` - Emscripten-based browser simulator
- `movement/` - Watch face application framework
  - `watch_faces/` - Watch face implementations organized by category:
    - `clock/` - Time display faces
    - `complication/` - Feature faces (stopwatch, calculator, games, etc.)
    - `sensor/` - Sensor-reading faces
    - `settings/` - Configuration faces
    - `demo/` - Example/test faces
  - `lib/` - Support libraries (TOTP, astronomy, chess, etc.)
  - `template/` - Template files for new watch faces
- `apps/` - Standalone bare-metal applications (not using Movement)
- `boards/` - Board-specific pin definitions (OSO-SWAT-A1-05, OSO-SWAT-C1-00, etc.)

### Movement Framework

Movement manages a carousel of "watch faces" - screens that display information or provide interactive features. Each face implements four required functions:

```c
void myface_face_setup(movement_settings_t *settings, uint8_t watch_face_index, void **context_ptr);
void myface_face_activate(movement_settings_t *settings, void *context);
bool myface_face_loop(movement_event_t event, movement_settings_t *settings, void *context);
void myface_face_resign(movement_settings_t *settings, void *context);
```

Key concepts:
- `setup`: Called at boot and after deep sleep; allocate state in `context_ptr`
- `activate`: Called when face comes to foreground; set up display
- `loop`: Event-driven main loop; handle `EVENT_TICK`, button events, `EVENT_TIMEOUT`, etc.
- `resign`: Called when face goes off-screen; disable peripherals

### Adding a New Watch Face

1. Create `movement/watch_faces/<category>/<name>_face.c` and `.h`
2. Add include to `movement/movement_faces.h`
3. Add source file to `movement/make/Makefile` SRCS list
4. Add face to `movement/movement_config.h` watch_faces array

Template files are in `movement/template/`. The face struct is defined as:
```c
#define myface_face ((const watch_face_t){ \
    myface_face_setup, \
    myface_face_activate, \
    myface_face_loop, \
    myface_face_resign, \
    NULL, \
})
```

### Configuring Which Faces to Build

Edit `movement/movement_config.h` to modify the `watch_faces[]` array. The first face is the default clock, `MOVEMENT_SECONDARY_FACE_INDEX` sets which faces are accessed via long-press of Mode button.

### Bare-Metal Apps (Non-Movement)

For apps that don't use Movement, copy `apps/starter-project/` and implement:
- `app_init()`, `app_setup()`, `app_loop()`
- `app_wake_from_backup()`, `app_wake_from_standby()`, `app_prepare_for_standby()`

## Hardware Details

- MCU: Microchip SAM L22 (ARM Cortex M0+)
- 10-digit segment LCD plus 5 indicators
- 3 buttons: MODE, LIGHT, ALARM
- Red/green LED backlight (PWM capable)
- 9-pin flex connector for sensor boards (I2C, SPI, UART, GPIO, ADC)
- 32KHz crystal for RTC with alarm
- CR2016 coin cell (~3V nominal)

## Board Variants

Set `COLOR` environment variable when building:
- `RED` - Sensor Watch Lite (OSO-SWAT-B1)
- `GREEN` - Original Sensor Watch (OSO-SWAT-A1-05)
- `BLUE` - Blue LED variant
- `PRO` - Sensor Watch Pro (OSO-SWAT-C1-00)
