# NAM A2-Lite on RP2350: benchmark results

The methodology behind the headline in the [README](README.md): how a NAM (Neural Amp Modeler)
A2-Lite capture went from far over budget to real time on two cores at 300 MHz on the RP2350
(Cortex-M33), with the measurements, including the dead ends, that got it there.

> Terminology: TONE3000's two A2 sizes are A2-Lite (3-channel) and A2-Full (8-channel). The NAM
> engine source predates those names and still calls them nano and standard, so `is_a2_shape` and
> the `a2_fast` code refer to A2-Lite when they say "nano".

## Method
Measured on real RP2350 hardware via the DWT cycle counter (`src/benchmark.cpp`), taking the
minimum of 200 iterations of one 48-frame block. The per-sample cycle count is clock-invariant;
CPU% is against the 48 kHz deadline. The engine is a vendored
[NeuralAmpModelerCore](https://github.com/sdatkinson/NeuralAmpModelerCore) (C++/Eigen) built with
`NAM_ENABLE_A2_FAST` and `NAM_USE_INLINE_GEMM`. For correctness, the device output checksum matches
a host verifier built from the same engine sources, and the dual-core split is bit-exact against a
single core (max|err| = 0). `a2_fast` is data-independent, so these cycle counts hold for any
A2-Lite capture.

## The starting point: 31k cycles per sample

| build            | per-sample cycles | notes |
|------------------|------------------:|-------|
| flash (XIP)      | 31,864            | baseline |
| hot path in SRAM | 31,030            | only 2.6% faster |

| clock   | budget cyc/sample | CPU @48kHz |
|---------|------------------:|-----------:|
| 300 MHz | 6,250             | 496%       |
| 350 MHz | 7,292             | 426%       |
| 400 MHz | 8,333             | 372%       |

At about 31k cycles per sample the A2-Lite does not fit one M33 core at any tested clock, running
roughly 3.7x over budget even at 400 MHz.

### Diagnosis
Disassembled the RAM-resident hot path (`Conv1D::Process`, `_Layer::Process`,
`_LayerArray::ProcessInner`, `WaveNet::_process_condition`):

- No double arithmetic in the hot path. All FP is hardware single-precision (`vfma.f32` /
  `vadd.f32`), with no `__aeabi_d*` soft-float calls. The M33 has only an SP FPU, so this was
  checked first.
- LUTs do not apply. A2-Lite uses LeakyReLU (`x>0?x:slope*x`, inlined), not tanh or sigmoid, so
  there is no transcendental to table.
- XIP cache misses are not the bottleneck. Moving the hot path to SRAM gained only 2.6%; the
  RP2350's 8 KiB XIP cache holds the inner loops.

The cause is the generic Eigen GEMM path. `Conv1D::Process` was dispatching to
`Eigen::generic_product_impl::scaleAndAddTo` for 3-channel matrices. Eigen's general
matrix-product machinery carries large per-call setup and dispatch overhead relative to the few
real MACs in a 3-channel conv, repeated across 23 layers and 48 frames. The Daisy reference figure
(2,965 cycles per sample on a Cortex-M7) comes from the `a2_fast` path, which is compile-time
unrolled with no Eigen general GEMM. The difference is the code path, not the silicon.

## The fix: the a2_fast path
`a2_fast` is the engine's fast path for the fixed A2 shape: compile-time unrolled, no general GEMM.
It engages automatically once the model matches `is_a2_shape` (single 23-layer array, channels = 3,
the fixed kernel/dilation pattern, LeakyReLU, layer-head rechannel conv k = 16).

| engine path   | per-sample cyc | CPU @300 MHz |
|---------------|---------------:|-------------:|
| generic Eigen | 31,030         | 496%         |
| a2_fast       | 8,396          | 134%         |

That is 3.7x faster. Clock-invariant 8,396 cycles against the 48 kHz budget:

| clock   | budget | CPU  |
|---------|-------:|-----:|
| 300 MHz | 6,250  | 134% |
| 350 MHz | 7,292  | 115% |
| 400 MHz | 8,333  | 101% |

A single M33 core reaches about real time at 400 MHz (101%), but 400 MHz is unstable on this board
(see notes), so this is not sufficient on its own.

### SRAM placement of the a2_fast hot path
Tagging the `a2_fast` per-sample functions `.time_critical` (run from SRAM) made it about 2.7%
slower (8,396 to 8,627). The compact, unrolled inner loops already fit the 8 KiB XIP cache, and
running from SRAM adds single-ported SRAM-bank contention between instruction fetch and data.
Reverted. Memory placement is not a useful lever here, the same conclusion as for the generic
engine.

## Dual-core layer pipeline
`a2_fast` is split as a layer pipeline: core1 runs the front layers `[0,K)`, and core0 runs the
back layers `[K,23)` plus the head. They pipeline at 48-frame block granularity through a
double-buffered handoff (residual, the front layers' summed head contribution, and the float
conditioning input) over the SIO FIFO, so block t's back half overlaps block t+1's front half.
This adds one block (about 1 ms) of latency and is bit-exact against a single core.

The engine change this needs is a layer-partition API on `a2_fast`: processing an arbitrary layer
range with externally supplied handoff buffers, with the head accumulation decoupled so each core
owns its own head ring. `process()` is untouched.

Split-point sweep at 300 MHz. The front gets the cheap kernel-6 layers; the two kernel-15
"degridding" layers and the head sit in the back, so the balance lands near K = 14:

| KSPLIT | per-sample cyc | CPU @300 MHz |
|-------:|---------------:|-------------:|
| 12     | 4,891          | 78%          |
| 13     | 4,598          | 74%          |
| 14     | 4,533          | 73%          |
| 15     | 5,177          | 83%          |
| 16     | 5,778          | 92%          |

K = 14 gives 4,533 cycles per sample: 1.85x over single-core `a2_fast`, and 6.85x over the original
generic path. This is real time at every clock:

| clock   | budget | CPU (dual K=14) |
|---------|-------:|----------------:|
| 300 MHz | 6,250  | 73%             |
| 350 MHz | 7,292  | 62%             |
| 400 MHz | 8,333  | 54%             |

The 1.85x, versus an ideal 2x, is synchronization and residual-imbalance overhead. The A2-Lite
runs in real time at 300 MHz with no 400 MHz overclock and about 27% headroom, which leaves the
second core's slack and the clock headroom for the audio path.

## Engine changes (forked, additive, minimal)
The RP2350 changes to NeuralAmpModelerCore are additive (`process()` is untouched) and live in a
[fork](https://github.com/oyama/NeuralAmpModelerCore/tree/add-rp2350-support):

- a2_fast layer-partition API (`NAM/wavenet/a2_fast.{h,cpp}`): process a layer range `[K0,K1)` with
  externally supplied residual, head-sum, and cond buffers, with head accumulation decoupled, so
  the WaveNet stack can be split across cores bit-exactly.
  [Proposed upstream.](https://github.com/sdatkinson/NeuralAmpModelerCore/issues/291)
- Bare-metal portability (`NAM/wavenet/slimmable.{h,cpp}`): a `NAM_SHARED_PTR_ATOMIC_FREE_FUNCS`
  path so the engine compiles without `std::atomic<std::shared_ptr<>>`, which the bare-metal
  toolchain lacks.

## Operational notes
300 MHz at 1.20 V is stable. 400 MHz at 1.20 V hard-faults on this board; above 350 MHz needs
higher VREG, and a boot-time overclock that faults can lock the core and flash interface and
require BOOTSEL recovery. The dual-core pipeline runs in real time at 300 MHz, so the firmware
ships there; the 400 MHz single-core figure above is a measurement, not the operating point.

## Conclusion
The 31k figure was the engine on the generic Eigen GEMM path, which is wrong for a 3-channel conv.
Switching to the `a2_fast` path it already ships brings it to 8,396 cycles per sample (3.7x), and
splitting the layer stack across both M33 cores, the main work this project adds, reaches 4,533
cycles per sample (6.85x overall, 73% CPU at 300 MHz). A production NAM A2-Lite tone runs in real
time on a dual-core Cortex-M33 at a stable 300 MHz / 1.20 V, without the unstable 400 MHz overclock.
