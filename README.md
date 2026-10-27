
# CCSDS Telemetry Frame Synchronizer

Hardware frame synchronizer for CCSDS telemetry: it locks onto the 32-bit attached sync marker (`0x1ACFFC1D`) in a noisy serial bit stream, survives bit slips, and extracts frames as bytes. Built for the **SPEC'26 Electronics Problem** (NED University, Karachi): a Python model, Verilog RTL with a self-checking testbench, and an FPGA timing report.

**Team (Electronics, Batch 2025):** Syeda Zainab Asim, Syeda Abiyha Fatima, Arisha Ali Khan

## Status

| Level | Deliverable | Status |
|---|---|---|
| 1 | Python model: correlation and threshold detection over simulated AWGN bit errors | Done |
| 2 | Verilog correlator, lock FSM, frame extractor, testbench with bit slips | Done |
| 3 | Quartus synthesis and timing for Cyclone V, target 50 MHz | Done |
| 3 | Virtual channel (VCID) frame extraction | Done |

## How it works

- **Marker:** ASM `0x1ACFFC1D`, 32 bits (CCSDS 131.0-B). Frame = marker + 1115 bytes = 8952 bits.
- **Correlator:** a 32-bit shift register, XOR with the marker and a popcount give a registered match count, one bit per clock. Every bit offset is checked, so bit slips cannot hide a marker.
- **Lock FSM:**
  - `SEARCH`: marker with at most 3 bit errors, any offset
  - `CHECK`: marker must repeat exactly one frame later, at most 6 errors
  - `LOCK`: frames are output as bytes with start and end markers
  - `FLYWHEEL`: one expected marker missed, frame timing keeps running
  - two misses in a row: loss of lock, back to `SEARCH`
- **Parameters** (in `frame_synchronizer.v`): `FRAME_BITS = 8920`, `SEARCH_ERR = 3`, `LOCK_ERR = 6`, `MAX_MISS = 2`. These are our design choices from the Level 1 results, not values taken from the CCSDS recommendations.

## Repository layout

```
level1.py                     Level 1 Python model, saves level1_results.png
level2/
  rtl/sync_correlator.v       32-bit sliding correlator
  rtl/frame_sync_fsm.v        lock FSM and frame extractor
  rtl/frame_synchronizer.v    top level
  tb/tb_frame_sync.v          self-checking testbench
  run_all.py                  stream generator, Python reference model, test runner
  frame_synchronizer.sdc      20 ns (50 MHz) clock constraint for Quartus
docs/                         technical document, slides, screenshots
```

## Quick start

**Level 1 (Python 3, numpy, matplotlib)**

```
pip install numpy matplotlib
python level1.py
```

Prints the detection table and saves `level1_results.png`.

**Level 2 (simulation)**

```
cd level2
python run_all.py --model-only      # generates test streams and the reference-model results
python run_all.py                   # also compiles and runs the RTL (needs Icarus Verilog on PATH)
```

With Questa/ModelSim, run `python run_all.py --model-only`, then in the simulator transcript:

```
cd <path>/level2
do vectors/run_modelsim.do
```

Six scenarios are run: clean, noisy (1% BER), idle gaps, bit slip, destroyed markers, and pure noise. Each prints `PASS` only if the extracted frames and the lock counts match the golden data and the Python reference model.

**Level 3 (Quartus Prime Lite)**

Create a project with `frame_synchronizer` as top level, add the three files in `level2/rtl/` and `frame_synchronizer.sdc`, then run the full compilation.

## Results

**Level 1 (300 markers per point, hard-decision BPSK over AWGN)**

| Threshold T | Detected at 0 dB Eb/N0 | False alarm per bit position |
|---|---|---|
| 0 | 6.3% | 2.3e-10 |
| 2 | 52% | 1.2e-7 |
| 4 | 88% | 9.7e-6 |
| 6 | 98.7% | 2.7e-4 |

A higher threshold detects markers in noise but allows more false alarms, which is why the RTL verifies a marker one frame later.

**Level 3 (Quartus Prime Lite 25.1, Cyclone V 5CGXFC7C7F23C8)**

| Item | Value |
|---|---|
| Clock constraint | 20 ns (50 MHz) |
| Fmax (slow 1100 mV 85 C) | 122.67 MHz |
| Setup slack (85 C / 0 C) | 11.848 ns / 11.402 ns |
| Hold slack (85 C) | 0.461 ns |
| Logic utilization | 58 / 56,480 ALMs (under 1%) |
| Registers, pins | 84, 20 of 268 |
| Memory, DSP, PLL | 0 |

Pin locations are not assigned because no board was targeted; this does not affect the resource or core timing results.

## Limitations

- Virtual channel extraction is not implemented.
- Hard-decision input only (no soft decisions).
- The first frame after the first marker is not output (it falls in `CHECK`).
- Fixed frame length; no inverted-polarity handling, derandomizer or Reed-Solomon decoding.
- Not tested on a physical board; input and output pin timing is unconstrained.

## References

- CCSDS 131.0-B, TM Synchronization and Channel Coding
- CCSDS 132.0-B, TM Space Data Link Protocol
- CCSDS 130.1-G, TM Synchronization and Channel Coding: Summary of Concept and Rationale
