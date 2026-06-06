# Goose Controller

> Bluetooth Xbox controller bridge for RoboGoose. Runs on a dedicated Arduino Nano ESP32 that pairs with an Xbox controller using the [Bluepad32](https://bluepad32.readthedocs.io/en/latest/) library and forwards inputs to the RoboGoose Mega over a serial line.

This is one of three repositories in the [RoboGoose](https://github.com/cse-450-ROBO-GOOSE) project. See the main [CanControl-fork](https://github.com/cse-450-ROBO-GOOSE/CanControl-fork) repository for the Mega firmware and the full system documentation.

---

## What it does

The Nano sits between an Xbox wireless controller and the RoboGoose Mega. It does three things on a continuous loop:

1. Maintains a Bluetooth connection with a paired Xbox controller
2. Reads stick positions, trigger values, button states, and D-pad state from Bluepad32
3. Packs those values into a comma-separated text line and sends it to the Mega over UART at 9600 baud, roughly 50 times per second

The Mega parses each line as it arrives and uses the values to drive the motors, light the LEDs, play sounds, and everything else. The Nano itself does no game logic or decision-making; it's purely a translation layer from Bluetooth HID to a simple text protocol the Mega can read.

This split exists because the Arduino Mega doesn't have native Bluetooth and Bluepad32 only runs on ESP32-class chips. Rather than rewrite the Mega firmware for an ESP32, we let a small dedicated board handle the Bluetooth side and feed the result to the Mega as plain serial data.

---

## Hardware

- **Arduino Nano ESP32** as the Bluetooth host
- **One-way serial connection to the RoboGoose Mega** (Nano TX wired to Mega Serial1 RX on pin 19)
- Common ground with the Mega and shared 5V power from the REV Mini Power Module

The serial connection is one-way only. The Nano never reads anything from the Mega. If you ever need bidirectional communication (for example, to send rumble feedback to the controller based on Mega state), you'd need to wire the Nano's RX pin to a Mega TX pin and add a parser on the Nano side.

---

## Serial protocol to the Mega

Each line the Nano sends to the Mega is a comma-separated list of 8 integers followed by a newline:

```
lx,ly,rx,ry,brake,throttle,buttons,dpad\n
```

| Field | Range | Meaning |
|-------|-------|---------|
| `lx` | -512 to 512 | Left stick X (negative = left, positive = right) |
| `ly` | -512 to 512 | Left stick Y (negative = up, positive = down) |
| `rx` | -512 to 512 | Right stick X (used for steering) |
| `ry` | -512 to 512 | Right stick Y (unused on Mega side) |
| `brake` | 0 to 1023 | LT analog trigger (used as a modifier flag) |
| `throttle` | 0 to 1023 | RT analog trigger (triggers blue eyes) |
| `buttons` | 0 to 65535 | Button bitmask |
| `dpad` | 0 to 15 | D-pad bitmask |

**Button bitmask values (Bluepad32 format):**

| Button | Bit | Decimal |
|--------|-----|---------|
| A | 0 | 1 |
| B | 1 | 2 |
| X | 2 | 4 |
| Y | 3 | 8 |
| LB | 4 | 16 |
| RB | 5 | 32 |
| LS click | 8 | 256 |
| RS click | 9 | 512 |

**D-pad bitmask values:**

| Direction | Bit | Decimal |
|-----------|-----|---------|
| Up | 0 | 1 |
| Down | 1 | 2 |
| Right | 2 | 4 |
| Left | 3 | 8 |

Multiple buttons or directions can be active simultaneously and are OR'd together. If `buttons = 17`, that's LB (16) + A (1) pressed at the same time.

Bits 6, 7, and 10+ are reserved by Bluepad32 for buttons we don't use (View, Menu, Xbox home, etc.). See the [Bluepad32 documentation](https://bluepad32.readthedocs.io/en/latest/) for the full bit mapping if you want to add support for additional buttons.

---

## How it works

When the Nano boots, Bluepad32 initializes the ESP32's Bluetooth stack and enters discoverable mode automatically. The Nano then waits in its main loop for a controller to pair.

Bluepad32 uses a callback-driven model. When a controller connects or sends input, Bluepad32 fires a callback with the new data, and the callback stores the latest values into global variables. The main loop runs independently and, on its own interval, reads those globals, formats them into a CSV line, and writes the line out over serial.

This means the Nano never blocks waiting for controller input. The serial line gets sent at a steady rate regardless of whether the controller has moved, which makes the Mega's parsing logic simpler (it can assume packets arrive on a predictable cadence and doesn't have to handle gaps).

---

## Pairing the Xbox controller

After the Nano is powered on:

1. Hold the **sync button on the back of the Xbox controller** (next to the charging port) until the Xbox logo on the front starts flashing rapidly
2. The Nano picks up the controller within a few seconds and the Xbox logo goes solid
3. From this point on, the Nano is sending CSV input packets to the Mega and the goose is drivable

If the controller doesn't pair, the most common causes are dead AA batteries in the controller, the Nano not actually running the firmware (see the bootloader gotcha below), a ground wire disconnected, or the controller already being paired to a different device (Xbox console, PC, phone) and not entering pairing mode properly.

---

## Build and flash

Open `goose_controller.ino` in the Arduino IDE.

### Required setup

Bluepad32 isn't a normal library. It's a full board package that replaces the standard ESP32 one for this Nano. Setup takes a few extra steps the first time.

1. **Follow the Bluepad32 Arduino setup tutorial** at [bluepad32.readthedocs.io/en/latest/plat_arduino](https://bluepad32.readthedocs.io/en/latest/plat_arduino/). This walks through adding the Bluepad32 board URL to your Arduino IDE preferences and installing the package through the Boards Manager.

2. **Configure the board settings** in the Arduino IDE:
   - **Tools > Board > esp32_bluepad32 > Arduino Nano ESP32**
   - **Tools > Pin Numbering > GPIO number** (NOT "By Arduino pin"). The pin numbers in this code reference GPIO numbers directly, so the wrong setting will cause every pin to map incorrectly.
   - **Tools > USB Mode > Debug mode**. There's a known bug in the Bluepad32 library where serial output doesn't work correctly in the default USB mode.

### The bootloader gotcha

The first time you try to upload, you'll almost certainly get an error along the lines of:

```
dfu-util: No DFU capable USB device available
```

This is the Nano ESP32 not being in its bootloader mode. To fix it:

1. **Double-tap the reset button** on the Nano ESP32. The timing matters: there needs to be a brief pause between presses (roughly half a second). Too fast and it won't register as a double-tap; too slow and the first press will just reset normally.
2. The onboard LED should start a **slow green breathing effect**. That's the indicator you're in bootloader mode.
3. **Upload the code while the LED is breathing.** The breathing effect will stop once the upload completes.
4. **Tap reset once more** to start the freshly-flashed firmware.

If you don't get the breathing LED on the first try, repeat the double-tap. This bootloader behavior is finicky and sometimes takes a few attempts.

---

## Adding new controller features

If you want to use a button or stick axis we currently ignore (View, Menu, Xbox home, beyond what's already there), the steps are:

1. **Identify the Bluepad32 field** that holds the value you want, from their [API documentation](https://bluepad32.readthedocs.io/en/latest/)
2. **Add the value to the CSV output** in the main loop, either as a new field or packed into an existing bitmask
3. **Update the Mega's CSV parser** in `parseCSV()` in `main.cpp` to read the new field
4. **Add a handler in the Mega firmware** for whatever action the new input should trigger

Adding a new field at the end of the CSV is backward-compatible only if you also update the Mega parser at the same time; the Mega rejects malformed lines so changing the field count without updating both sides will break input entirely until you reflash both boards.

---

## Known issues

- **First-time bootloader is finicky.** Once the firmware is flashed, the Nano runs reliably on every power cycle. The pain is only on the initial flash (see bootloader gotcha above).
- **Pairing latency.** It can take a few seconds for the Nano to pick up a controller in pairing mode. Be patient before assuming something's wrong.
- **No reconnect handling.** If the controller drops the Bluetooth connection mid-use (battery dies, goes out of range), the Nano keeps sending the last known input values to the Mega. The Mega will continue trying to drive based on stale data. A future improvement would be detecting the disconnect and sending a "zero everything" packet so the goose stops gracefully.
- **Other Controllers Connecting.** Other Xbox controllers can potentially pair to the Nano as a unique 'Player 2'. Fix this by limiting max players in Bluepad or utilizing MAC address stuff.

---

## Related repositories

- [CanControl-fork](https://github.com/cse-450-ROBO-GOOSE/CanControl-fork) — main Mega firmware, motor control, anti-tip, audio playback, full system documentation
- [goose_soundboard](https://github.com/cse-450-ROBO-GOOSE/goose_soundboard) — WiFi web soundboard and live telemetry interface running on a separate Nano ESP32
