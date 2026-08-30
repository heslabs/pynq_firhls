# FIR Filter — Vitis HLS — PYNQ-Z2

A fully pipelined FIR filter accelerator built with Vitis HLS and Vivado,
adapted for the **Digilent PYNQ-Z2** board (Zynq-7020, `xc7z020clg400-1`).

Adapted from Michael Schmid's article/repo, originally targeting the AMD
Kria KV260 (Zynq UltraScale+ MPSoC):
- Article: https://www.hackster.io/michi_michi/fpga-fir-filter-hls-kria-kv260-pynq-2eec35
- Original repo: https://github.com/Nunigan/FIR-FIlter_HLS

## What's different from the KV260 version

| | KV260 (original) | PYNQ-Z2 (this project) |
|---|---|---|
| SoC | Zynq UltraScale+ MPSoC | Zynq-7020 |
| Part | e.g. `xck26-sfvc784-2LV-c` | `xc7z020clg400-1` |
| PS block | `zynq_ultra_ps_e` | `processing_system7` |
| Board files | Xilinx KV260 SOM | Digilent `tul.com.tw:pynq-z2:part0:1.0` |
| PL clock in this project | 100 MHz (article overclocks to 250 MHz post-build) | 100 MHz (adjust `CLK_MHZ`, re-time as needed) |

The HLS C++ kernel itself (`src/fir.cpp`) is functionally the same
shift-register FIR design as the article: a `float` version and an
`ap_fixed<W,I>` version (32-bit word, 1-bit integer, matching the article),
both pipelined at `II=1`.

## Layout

```
.
├── Makefile                 # orchestrates the whole flow
├── requirements.txt          # host-side numpy/scipy pin for the venv
├── src/
│   ├── gen_fir.py            # Kaiser-window filter design -> src/fir.h (generated)
│   ├── fir.cpp                # HLS kernel: fir() float, fir_fixed() fixed-point
│   └── fir_tb.cpp             # C-sim testbench vs. scipy.signal.lfilter() golden ref
├── scripts/
│   ├── setup_env.sh           # source Vitis HLS / Vivado settings64.sh
│   ├── setup_venv.sh           # create host-side .venv with numpy/scipy
│   ├── gen_coeffs.sh           # wraps gen_fir.py (auto-activates .venv if present)
│   ├── run_hls.sh / run_hls.tcl        # Vitis HLS csim/csynth/export (PYNQ-Z2 part)
│   ├── run_vivado.sh / build_bd.tcl    # Vivado block design + bitstream (PYNQ-Z2 PS7)
│   ├── package_overlay.sh      # bundle fir.bit/fir.hwh/driver for the board
│   └── clean.sh                # clean; pass --venv to also remove .venv
├── pynq/
│   └── fir_test.py             # runs ON the board: loads overlay, drives IP via MMIO
├── .venv/                      # generated: host-side Python venv (never copy to board)
├── build/                      # generated: HLS project, exported IP, Vivado project
├── output/                     # generated: fir.bit, fir.hwh
└── dist/fir_overlay/           # generated: final bundle to copy to the board
```

## Prerequisites

- Vitis HLS and Vivado (tested layout assumes 2022.2–2024.x; edit
  `scripts/setup_env.sh` for your install path/version).
- Digilent PYNQ-Z2 board files installed in Vivado:
  https://github.com/Digilent/vivado-boards
  (`build_bd.tcl` falls back to a part-only flow with a warning if these
  aren't found — you'll then need to add DDR/clock config and pin
  constraints by hand.)
- Python 3 (for the `venv` module) on the host, for coefficient generation.
- On the board: the standard PYNQ Linux image (has `pynq`, `numpy` preinstalled)
  — no venv needed there, and `.venv/` should never be copied to the board.

## Build flow

```bash
source scripts/setup_env.sh        # puts vitis_hls / vivado on PATH

make venv                          # create .venv/ with numpy/scipy (host-side only)
source .venv/bin/activate          # optional: activate it in your shell too

make coeffs                        # design filter -> src/fir.h
make hls                           # Vitis HLS: csim, csynth, export IP
make vivado                        # Vivado: block design, bitstream, .hwh
make package                       # -> dist/fir_overlay/{fir.bit,fir.hwh,fir_test.py}

# or simply:
make all
```

Note `make coeffs` (and therefore `make hls`/`make vivado`/`make package`/`make all`)
automatically depends on `make venv`, so the venv is created and populated
the first time you run any of them if it doesn't already exist.

Tune the filter or hardware target without editing files:

```bash
make coeffs FS=48000 CUTOFF_HZ=4000 TRANSITION_HZ=1000 SIZE=2048
make hls TOP=fir CLK_NS=8          # try the plain-float kernel at 125 MHz
make vivado CLK_MHZ=100
```

## Deploying to the PYNQ-Z2

```bash
scp -r dist/fir_overlay xilinx@<board-ip>:/home/xilinx/jupyter_notebooks/
ssh xilinx@<board-ip>
cd jupyter_notebooks/fir_overlay
python3 fir_test.py
```

`fir_test.py` loads `fir.bit`/`fir.hwh` via `pynq.Overlay`, allocates
cache-coherent buffers with `pynq.allocate()`, writes their physical
addresses into the HLS IP's AXI-Lite control registers, and toggles
`ap_start`/polls `ap_done` — the same MMIO-driven pattern as the article's
`fir.ipynb`, just re-pointed at the `processing_system7_0` clock/HP-port
names Vivado generates for a Zynq-7000 device instead of a
Zynq UltraScale+ one.

## Notes / things to double check on real hardware

- `build_bd.tcl` wires the FIR IP's two `m_axi` ports (`gmem0`/`gmem1`) into
  `S_AXI_HP0` through a shared AXI interconnect, and its `s_axi_control`
  into `M_AXI_GP0`. For a 1024–2048 sample buffer this is comfortably within
  the HP port's bandwidth at 100 MHz; if you significantly enlarge `SIZE`
  or the sample rate, watch the achieved II/throughput in the HLS report.
- The article overclocks the KV260 PL up to 250 MHz from PYNQ using
  `PL.Clock`. The Zynq-7000 fabric on the PYNQ-Z2 has a lower comfortable
  ceiling — reclock conservatively and re-check timing closure in Vivado
  before trusting a higher `CLK_MHZ` in `fir_test.py`'s timing numbers.
- `fir_test.py` regenerates its own test signal rather than reusing
  `src/fir.h`'s `test_input`/`test_expected` arrays; for an apples-to-apples
  accuracy check against the C-sim golden reference, dump `taps`/`test_input`
  to a `.npy` from `gen_fir.py` and load it on the board instead.
