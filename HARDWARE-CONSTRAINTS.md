# Study Bot — Where We're Stuck (using only the parts we already have)

Parts we already have: ESP32 board, small speaker, 2× 16×2 LCD screens, ISD1820 voice module.

**The problem in one line:** the parts we have are the *wrong type* for the jobs they need to do, and we don't have the basic materials to connect them.

## 1. We can't connect anything
- We have no jumper wires, so no two modules can be linked together.
- We have no breadboard to hold parts.
- We have no soldering iron, so the LCD pins can't even be attached.

## 2. Microphone problem
- We have no proper digital microphone. The only mic is the one built into the ISD1820 module, and it's an old analog type.
- To feed that audio into the ESP32, we'd have to hack an amplified output into a basic ADC pin.
- That audio is low quality, muffled, and noisy — speech recognition would mishear often.
- The signal is also too strong and could damage the ESP32 pin unless we reduce it (needs resistors, which we don't have).

## 3. Speaker / amplifier problem
- We have no amplifier board, so the speaker can only be driven very quietly.
- The amplifier inside the ISD1820 can't be reused — it only plays its own recordings, it can't take audio from the ESP32.
- Driving the speaker straight from the ESP32 gives an 8-bit robotic voice and needs a transistor + resistor to be heard (we don't have them).

## 4. Screen problem
- The LCDs have no I2C adapter, so each one needs ~6 wires plus a contrast dial and a backlight resistor.
- Two screens = 12+ wires, which we don't have.
- Without a contrast dial, the screen usually shows nothing readable.
- These screens use 5V while the ESP32 uses 3.3V, which makes them unreliable.
- We also can't solder their pins on.

## 5. Power problem
- Two screen backlights + speaker + ISD1820 all draw power from the ESP32's tiny USB supply → it can brown out or reset.

## 6. Wake-word problem
- The hands-free wake word needs decent audio. With the hacked, noisy mic it would trigger poorly — so we'd fall back to a push button (which also needs wires).

## 7. Can't test or fix anything
- No multimeter, no wires, no board to build on → no way to test or debug the circuitry.

## Bottom line
Every single part of the device is either blocked by missing wires/tools, or would need a hack that itself requires parts we don't have. As it stands, the device can't be physically built yet.
