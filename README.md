# STM32f103-USB-DFU-Bootloader-CDC-Firmware

Firmware for the **STM32F103C8T6** ("Blue Pill") that uses the chip's built-in USB peripheral to do one of three things at boot, depending on the level of a mode pin:

1. **Jump to a user application** already stored in flash.
2. **Enumerate as a USB DFU device** so new firmware can be flashed over USB (no ST-Link or UART adapter needed).
3. **Enumerate as a USB CDC virtual COM port** (serial over USB).

> **Note on the name:** this is *not* a true USB composite device. Only **one** USB class is active per boot. The mode is chosen once, at reset, from a pin. See [Limitations](#limitations-and-known-issues).

---

## Table of contents

- [How it works](#how-it-works)
- [Boot flow](#boot-flow)
- [Hardware](#hardware)
- [Flash memory map](#flash-memory-map)
- [Project layout](#project-layout)
- [Code walkthrough](#code-walkthrough)
- [Building](#building)
- [Flashing an application over DFU](#flashing-an-application-over-dfu)
- [Writing an application for this bootloader](#writing-an-application-for-this-bootloader)
- [Limitations and known issues](#limitations-and-known-issues)

---

## How it works

On reset, `main()` initialises the clocks and GPIO, then reads the mode pin (`boot1`, **PB2**):

| `boot1` (PB2) | Valid app at `0x0800C000`? | Result |
|---|---|---|
| HIGH | Yes | Jump to the application |
| HIGH | No | Start as **DFU** device |
| LOW | n/a | Start as **CDC** virtual COM port |

Both USB modes share one device descriptor table. At runtime the firmware patches the product ID (PID) and a few strings so the host sees two different devices:

| Mode | VID | PID | Product string |
|---|---|---|---|
| DFU | `0x0483` (1155, ST) | `0xDF11` (57105) | `STM32 DownLoad Firmware Update` |
| CDC | `0x0483` (1155, ST) | `0x5740` (22336) | `STM32 Virtual COM Port` |

## Boot flow

```
Reset
  |
  v
HAL_Init -> SystemClock_Config (72 MHz, USB 48 MHz) -> MX_GPIO_Init
  |
  v
Read boot1 (PB2)
  |
  +-- HIGH --> app vector table valid at 0x0800C000?
  |              |-- yes --> set MSP, jump to app reset vector
  |              '-- no  --> DFU mode (usb_mode = 0, PID 0xDF11)
  |
  '-- LOW  --> CDC mode (usb_mode = 1, PID 0x5740)
  |
  v
update_usb_descriptor(pid)      patch PID (and bcdUSB for CDC)
MX_USB_DEVICE_Init()            register DFU or CDC class, start USB
  |
  v
Main loop
  CDC mode: send "test logs!\r\n" every 1 s
  DFU mode: idle (USB interrupts do the work)
```

**Application validity check:** the first word at `0x0800C000` is the app's initial stack pointer. The bootloader accepts the app if `(word & 0x2FFE0000) == 0x20000000`, i.e. the stack pointer points into SRAM.

## Hardware

- **MCU:** STM32F103C8T6
- **Clock:** 8 MHz external crystal (HSE) -> PLL x9 = 72 MHz. USB clock = PLL / 1.5 = 48 MHz.
- **USB pins:** PA11 (D-) and PA12 (D+)
- **Mode pin:** PB2 (`boot1`), configured as a floating input. Tie it HIGH or LOW with a jumper or resistor, since there is no internal pull.

## Flash memory map

```
0x08000000  +--------------------------+
            |  Bootloader (this repo)  |
0x0800C000  +--------------------------+  <- USBD_DFU_APP_DEFAULT_ADD
            |  User application        |
0x0800FBFC  |  (erase range ends here) |
0x0801xxxx  +--------------------------+
```

- Application start: `0x0800C000`
- DFU erase range: `0x0800C000` to `0x0800FBFC` (15 pages of 1 KB)
- That leaves about 16 KB for the application on a 64 KB device.

The bootloader size limit (about 48 KB, `0x08000000`-`0x0800BFFF`) is **not enforced** by the build. Nothing in `platformio.ini` or the linker settings stops the bootloader from growing into the app region.

## Project layout

```
src/
  Core/
    Src/main.c               Boot logic, mode select, app jump, clock config
    Src/stm32f1xx_it.c       Interrupt handlers
    Src/stm32f1xx_hal_msp.c  HAL low-level init
    Inc/main.h               Pin definitions (boot1 = PB2)
  USB_DEVICE/
    App/usb_device.c         Registers DFU or CDC class based on usb_mode
    App/usbd_desc.c          USB descriptors, runtime PID patching
    App/usbd_dfu_if.c        DFU flash backend (erase / write / read)
    App/usbd_cdc_if.c        CDC interface (transmit / receive)
    Target/usbd_conf.h/.c    USB config (APP address, transfer size, low-level driver)
  stm32f103c8t6_cube.ioc     STM32CubeMX project
  EWARM/                     IAR Embedded Workbench project files
platformio.ini               PlatformIO build config
```

## Code walkthrough

### `main.c`
- Declares the app address range: `AddressMyApplicationBegin = 0x0800C000`, `AddressMyApplicationEnd = 0x0800FBFC`.
- `usb_mode` (`0` = DFU, `1` = CDC) is the global that every other file uses to know the current mode.
- Reads `boot1` and picks the path described above. Sets the product, configuration and interface strings, plus the PID.
- Calls `update_usb_descriptor()` then `MX_USB_DEVICE_Init()`.
- Main loop: in CDC mode, calls `CDC_Transmit_FS()` with a test message once per second.

### `usb_device.c`
`MX_USB_DEVICE_Init()` runs `USBD_Init`, registers either `USBD_DFU` (with `USBD_DFU_fops_FS`) or `USBD_CDC` (with `USBD_Interface_fops_FS`), then `USBD_Start`.

### `usbd_desc.c`
- Holds the static device descriptor (VID `0x0483`, default PID `0xDF11`).
- `update_usb_descriptor(val)` overwrites bytes 10-11 (PID). For the CDC PID it also sets `bcdUSB` to `0x0200`.
- String descriptors for product, configuration and interface are read from the global strings set in `main.c`.

### `usbd_dfu_if.c`
Implements the DFU memory callbacks:
- **Init / DeInit:** unlock / lock flash.
- **Erase:** erases the whole app region (`0x0800C000`-`0x0800FBFC`). The address argument is ignored.
- **Write:** programs 32-bit words and reads each one back to verify.
- **Read:** copies bytes from flash.
- **GetStatus:** reports fixed program and erase timeouts (50 ms).

### `usbd_cdc_if.c`
Standard CubeMX CDC interface. `CDC_Receive_FS()` only re-arms the receive buffer, so incoming data is currently discarded.

## Building

The project builds with **PlatformIO** (`platformio.ini`) or **IAR EWARM** (`src/EWARM`).

```ini
[env:genericSTM32F103C8]
platform = ststm32
board = genericSTM32F103C8
framework = stm32cube
upload_protocol = serial
build_flags = -Isrc/Core/Inc -Isrc/USB_DEVICE/App -Isrc/USB_DEVICE/Target -Os
```

```bash
pio run                 # build
pio run -t upload       # upload over serial (first-time flash of the bootloader)
```

The bootloader itself is flashed once over the serial bootloader (or an ST-Link). After that, applications can be updated over USB.

## Flashing an application over DFU

1. Connect **PB2 to 3.3 V** (HIGH) and reset the board. With no valid app it enters DFU mode, enumerating as `0483:df11`.
2. Flash your application binary to `0x0800C000` with `dfu-util`:

   ```bash
   dfu-util -a 0 -s 0x0800C000 -D app.bin
   ```
3. Reset the board with PB2 still HIGH. The bootloader finds a valid app and jumps to it.

On Windows, the DFU device may need a WinUSB driver (for example via Zadig) before `dfu-util` can see it.

> Because of the memory-layout string issue below, `dfu-util` may print a wrong memory map. Passing the address explicitly with `-s` is recommended.

## Using CDC mode

1. Connect **PB2 to GND** (LOW) and reset.
2. The board appears as a COM port (`COMx` on Windows, `/dev/ttyACM0` on Linux).
3. Open it with any serial terminal. You will see `test logs!` once per second.

## Writing an application for this bootloader

Your application must be built to run from `0x0800C000`:

- Set the flash origin to `0x0800C000` in the linker script (and shrink the length accordingly).
- Set the vector table offset to `0x0000C000` (`VECT_TAB_OFFSET` in `system_stm32f1xx.c`, or write `SCB->VTOR` early in `main`).
- Keep the first word of the image a valid initial stack pointer in SRAM, since the bootloader's validity check depends on it.
- Do not assume peripherals are in their reset state. The bootloader jumps without de-initialising clocks, SysTick or interrupts (see below), so your startup code should reconfigure them.

## Limitations and known issues

**Design**
- **Not a true composite device.** One USB class per boot, selected by PB2. A real composite device would need `USBD_MAX_NUM_INTERFACES` greater than 1 and a merged configuration descriptor.
- **Mode switch needs a reset.** There is no DFU_DETACH / runtime-mode switching.
- **Bootloader size is not enforced** (see [Flash memory map](#flash-memory-map)).
- **The CDC demo does nothing useful.** It sends a test string and drops received data.

**Bugs and risks in the current code**

| Area | Issue | Suggested fix |
|---|---|---|
| DFU erase | `MEM_If_Erase_FS` ignores its `Add` argument and erases the full app region on every call. Harmless if the host erases once, wasteful or risky if it erases per page. | Erase only the page containing `Add`. |
| DFU memory map | `FLASH_DESC_STR` uses an STM32F4 sector layout (`016Ka`, `128Kg`). The F103 has 1 KB pages, so `dfu-util` shows a wrong map. | e.g. `@Internal Flash /0x08000000/64*001Kg`. |
| App jump | No `HAL_DeInit`, SysTick disable, NVIC clear, or VTOR setup before jumping. The app can inherit live interrupts and the running PLL. | Add a clean-up step before the jump. |
| Strings | Each string pointer is `malloc`ed and then overwritten by a literal, so the allocation is wasted. | Remove the `malloc` calls. |
| DFU write | Casts `src` to `uint32_t*` and steps by 4. If `Len` is not a multiple of 4 it reads past the buffer. | Pad the final word with `0xFF`. |
| Erase range | `AddressMyApplicationEnd = 0x0800FBFC` leaves the last 1 KB page (`0x0800FC00`+) unerased. Possibly intentional, but undocumented. | Document it, or extend the range. |
| CDC transmit | Return value of `CDC_Transmit_FS` is ignored. It returns `USBD_BUSY` if the previous transfer is still pending. | Check the result and retry. |
| Error handling | `Error_Handler` disables IRQs and spins forever. | Use a watchdog or reset in production. |
| Dead code | RTC and TIM2 init functions are commented out, but `hrtc` and `htim2` globals remain. | Remove them. |

**Naming cleanups worth considering**
- `usb_mode` as `0`/`1` -> an `enum` (`USB_MODE_DFU`, `USB_MODE_CDC`).
- PIDs `57105` / `22336` -> `0xDF11` / `0x5740` defines.
- `boot1` label -> `MODE_SEL`, since PB2 is also the chip's real BOOT1 pin.
- `AddressMyApplicationBegin/End` -> `APP_START_ADDR` / `APP_END_ADDR`.

## License

ST-generated files carry the STMicroelectronics license (see file headers). No project-level license file is present in the repository.