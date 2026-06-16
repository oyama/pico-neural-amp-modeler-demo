# Getting started

Build the firmware, hear it through a USB-audio loopback, swap in your own tone, and run the
benchmark. For the background (the dual-core port, the numbers), see the [README](README.md) and
[`RESULTS.md`](RESULTS.md).

## Quick start: flash the prebuilt firmware

If you just want to hear it, download the latest `pico-nam-*.uf2` from
[Releases](https://github.com/oyama/pico-neural-amp-modeler-demo/releases/latest), hold BOOTSEL
while plugging in the board, and copy the `.uf2` onto the `RP2350` mass-storage volume. Then jump
to [Run it](#run-it--hearing-the-loopback). To build from source or embed your own tone, read on.

## Requirements

- A **Raspberry Pi Pico 2** or any RP2350 board.
- The [Pico SDK](https://github.com/raspberrypi/pico-sdk) and the **Arm GNU toolchain**
  (`arm-none-eabi-gcc`).
- A **host C++ compiler** (clang or gcc). The build runs the `nam2c` model-embedding tool
  natively, so no Python is needed.
- Host tested on **macOS**. The USB device is class-compliant UAC2, with no drivers.

## Build & flash

```bash
git clone --recurse-submodules https://github.com/oyama/pico-neural-amp-modeler-demo.git
cd pico-neural-amp-modeler-demo
export PICO_SDK_PATH=/path/to/pico-sdk
# (if your toolchain isn't on PATH) export PICO_TOOLCHAIN_PATH=/path/to/arm-none-eabi

cmake -S . -B build -DCMAKE_BUILD_TYPE=Release
cmake --build build --target pico_nam_loopback -j
```

Flash `build/pico_nam_loopback.uf2`:

- **BOOTSEL drag-and-drop**: hold BOOTSEL while plugging in the board, then copy the `.uf2` onto
  the `RP2350` mass-storage volume; or
- **picotool**: `picotool load build/pico_nam_loopback.uf2 -f`; or
- **SWD / debug probe**: `openocd -f interface/cmsis-dap.cfg -f target/rp2350.cfg -c 'program build/pico_nam_loopback.elf reset exit'`.

The board enumerates as a USB-audio device named **"Pico NAM"** (stereo, 24-bit, 48 kHz).

## Run it — hearing the loopback

This is a USB-audio **loopback FX**: the host plays audio into the device (its "speaker"), the
device processes it, and returns it as an input (its "mic"). To hear the result, route a source
into "Pico NAM" and monitor "Pico NAM"'s input through your real speakers. Press **BOOTSEL** to
toggle between the two states (LED on for amp, LED off for clean passthrough).

For guitar, feed a DI/interface send into the device's output and monitor its input. For a quick
test you can use any system audio:

### macOS (DAW input-monitoring)

1. Connect the board. In **System Settings → Sound → Output**, select **Pico NAM** (so audio
   plays into the device).
2. In a DAW (GarageBand / Logic / Reaper), create an audio track with **input = Pico NAM**
   (Input 1+2) and turn **Monitor on**; set the DAW's **output to your speakers/headphones**.
3. Play a source (a track, YouTube, your guitar via an interface) and press **BOOTSEL** to A/B
   the amp against clean passthrough.

> Tip: avoid routing the device's input back into its own output (a feedback loop). Monitor it
> on a *different* output (your built-in speakers/headphones).

### Windows (Listen to this device)

1. **Settings → System → Sound**: set **Output** to *Pico NAM* and **Input** to *Pico NAM*.
2. **More sound settings → Recording**, open *Pico NAM* properties → **Listen** tab → check
   **Listen to this device**, set playback to your speakers, **Apply**.
3. Play a source and press **BOOTSEL** to toggle the amp.

## Use your own tone

The repo ships one example A2-Lite model (`example.nam`). To use any A2 tone from
[tone3000.com](https://www.tone3000.com/), point the build at the packed `.nam` as is. The `nam2c`
host tool extracts the A2-Lite (3-channel) sub-model, validates it with the engine's own
`is_a2_shape` (a non-A2 model **fails the build** with a clear error), and embeds the weights:

```bash
cp "Your A2 Tone.nam" example.nam        # or edit nam_set_model(...) in CMakeLists.txt
cmake --build build --target pico_nam_loopback -j
```

Output level is `NAM_OUTPUT_GAIN` in `src/nam_fx.cpp`; the output clamps, giving amp-like clipping
when pushed. Only A2-Lite (3-channel) fits real time. A2-Full and A1 models are rejected by
`is_a2_shape`.

## Benchmark

`nam_bench` measures the dual-core per-sample cycle cost and prints it over USB-CDC (this board
has no UART bridge):

```bash
cmake --build build --target nam_bench -j
# flash it, then read the CDC port (a serial terminal at any baud), e.g. on macOS:
#   screen /dev/cu.usbmodem* 115200
```

Sweep the clock and the dual-core split point:

```bash
cmake -S . -B build -DNAM_SYS_KHZ=350000 -DNAM_KSPLIT=13
cmake --build build --target nam_bench -j
```

`a2_fast` is data-independent, so the cycle count is the same for any A2-Lite model.
