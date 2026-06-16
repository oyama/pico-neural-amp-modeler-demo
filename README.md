# pico-neural-amp-modeler-demo

A guitar amp captured as a neural network, running in real time on an RP2350 microcontroller.

A Raspberry Pi Pico 2 (RP2350) runs as a class-compliant USB Audio device and applies a
[Neural Amp Modeler](https://www.neuralampmodeler.com/) A2-Lite capture to your signal in real
time. It needs no driver, no DSP chip, and no extra hardware: plug it in, select it as your audio
interface, and press the BOOTSEL button to A/B the modeled amp against clean passthrough. The
model runs across both Cortex-M33 cores.

https://github.com/user-attachments/assets/7691aff4-b685-411e-9f66-75b73250b5ac

> This repository is a demonstration and technical write-up, not a product. It shows that a
> production NAM A2 tone fits an RP2350, with the firmware, build tooling, and benchmarks to
> reproduce it. Playing a real guitar through it needs an audio codec, which is a separate,
> upcoming project (an Audio In/Out codec kit). See [Scope](#scope--where-it-goes).

## Why it looked too slow at first

Out of the box the model ran at 31,030 cycles per sample, about 10x a Cortex-M7 and roughly 5x
over the 48 kHz budget, which makes the RP2350 look far too slow for the task. The cause was the
NAM engine's generic path: a general-purpose Eigen matrix multiply, which is the wrong tool for a
3-channel convolution.

| code path           | cycles/sample | CPU @ 300 MHz |
|---------------------|--------------:|--------------:|
| generic Eigen       |        31,030 |          496% |
| a2_fast             |         8,396 |          134% |
| a2_fast + dual-core |         4,533 |           73% |

The engine already ships a purpose-built fast path for the fixed A2 shape, `a2_fast`, which is
compile-time unrolled and uses no general GEMM. Switching to it drops the cost to 8,396 cycles, a
3.7x improvement. Splitting the layer stack across both M33 cores, which is the main work this
project adds, brings another 1.85x for a final 4,533 cycles. That is 6.85x overall, enough to run
in real time on two cores at 300 MHz (2× the 150 MHz default, at 1.20 V core), without the
unstable 400 MHz overclock. The split produces bit-exact
output compared to a single core, and `a2_fast` is data-independent, so the per-sample cost is the
same for any A2-Lite capture.

The full methodology, the sweep tables, and the dead ends (SRAM placement, double precision, LUTs)
are in [RESULTS.md](RESULTS.md).

## How it works

![Host audio output runs through Neural Amp Modeler inference on the RP2350 and returns to host audio input](docs/how-it-works.svg)

- The model is a NAM A2-Lite: a small WaveNet with 23 dilated-conv layers, 3 channels, about
  1,871 parameters (roughly 7.5 KB), LeakyReLU activation, and a receptive field near 132 ms. It
  is the small "slim point" of a TONE3000 A2 capture, which packs an A2-Lite and an 8-channel
  A2-Full; only the Lite runs in real time on this class of MCU.
- The compute runs on `a2_fast`, split as a layer pipeline across the two cores: core1 runs the
  front layers, core0 runs the back layers, the head, and USB. One block's back half overlaps the
  next block's front half, so both cores stay busy. Throughput roughly doubles, at the cost of one
  extra block of latency (about 1 ms).
- The interface is a class-compliant UAC2 stereo 24-bit / 48 kHz device named "Pico NAM", so the
  host needs no driver. The IN endpoint is fed exactly one host frame per USB SOF, which keeps the
  audio clean.
- BOOTSEL toggles effect and passthrough, and the LED tracks the state. A `multicore_lockout`
  guards the BOOTSEL read against core1's flash access.

> Naming: TONE3000's two A2 sizes are A2-Lite (3-channel) and A2-Full (8-channel). The NAM engine
> source predates those names and still calls them nano (Lite) and standard (Full), so you will
> see `nano` and `is_a2_shape` in the code referring to A2-Lite.

## Build & run

Prebuilt firmware is on the [latest release](https://github.com/oyama/pico-neural-amp-modeler-demo/releases/latest):
download `pico-nam-*.uf2` and drag it onto the RP2350 in BOOTSEL mode, then skip to selecting "Pico NAM" below.

To build from source, this project uses the [pico-sdk](https://github.com/raspberrypi/pico-sdk).
Refer to the official guide, [Getting Started with Raspberry Pi Pico](https://datasheets.raspberrypi.com/pico/getting-started-with-pico.pdf), to set up your development environment.

```bash
git clone --recurse-submodules https://github.com/oyama/pico-neural-amp-modeler-demo.git
cd pico-neural-amp-modeler-demo
PICO_SDK_PATH=/path/to/pico-sdk cmake -S . -B build -DCMAKE_BUILD_TYPE=Release
cmake --build build --target pico_nam_loopback -j
# drag build/pico_nam_loopback.uf2 onto the RP2350 in BOOTSEL mode
```

Select "Pico NAM" as your audio interface and press BOOTSEL to A/B amp against passthrough. The
full walkthrough (toolchain setup, per-OS monitoring, embedding your own
[TONE3000](https://www.tone3000.com/) tone with `nam2c`, and the benchmark) is in
[GETTING_STARTED.md](GETTING_STARTED.md).

## Under the hood: the RP2350 engine port

The network runs on [NeuralAmpModelerCore](https://github.com/sdatkinson/NeuralAmpModelerCore).
This project's engine changes are additive and minimal (`process()` is untouched) and live in a
[fork](https://github.com/oyama/NeuralAmpModelerCore/tree/add-rp2350-support):

- a layer-partition API on the `a2_fast` path so the WaveNet stack can be split across cores
  (bit-exact against a single core), [proposed upstream](https://github.com/sdatkinson/NeuralAmpModelerCore/issues/291);
- a bare-metal portability flag (`NAM_SHARED_PTR_ATOMIC_FREE_FUNCS`) so the engine compiles
  without `std::atomic<std::shared_ptr<>>`.

Everything else stays in this repo: the dual-core glue (`src/nam_fx.cpp`), the UAC2 layer, the
`nam2c` model embedder (`tools/nam2c.cpp`), and a no-op `<mutex>` shadow for the single-thread toolchain
(`include/compat/`). Build-level adaptations (C++ exceptions, `--whole-archive` so the engine's
parsers self-register) are in `CMakeLists.txt`.

| dependency | license | role |
|---|---|---|
| [NeuralAmpModelerCore](https://github.com/sdatkinson/NeuralAmpModelerCore) ([fork](https://github.com/oyama/NeuralAmpModelerCore)) | MIT | NN engine / `a2_fast` |
| Eigen | MPL2 | linear algebra |
| nlohmann/json | MIT | `.nam` parsing (host-side `nam2c`) |
| Pico SDK / TinyUSB | BSD-3 | RP2350 + USB |

## Scope & where it goes

This repo is the software demonstration layer, a USB-audio loopback FX. The same Pico NAM core
takes on new roles once you add audio hardware around it: an ADC turns it into a USB guitar
interface, and an ADC plus DAC makes it a standalone effect box with no host involved.

![Add an ADC for a USB guitar interface, add a DAC for a standalone pedal](docs/roadmap.svg)

That codec hardware (the Audio In/Out codec kit) is a separate, upcoming project and is
deliberately out of scope here, so this repo stays a software and tooling reference. Also planned
is runtime model loading: today the tone is compiled in, and a USB mass-storage loader would
replace the rebuild-and-reflash step.

## License & credits

- This repo: MIT, see [LICENSE](LICENSE).
- NAM and the A2 architecture: [Steve Atkinson](https://github.com/sdatkinson) /
  [TONE3000](https://www.tone3000.com/).
- Bring your own A2 capture from [tone3000.com](https://www.tone3000.com/).
