# DMG-Scratch

**A cycle-accurate hardware reconstruction of the Nintendo Game Boy DMG-01 (DMG-CPU B) implemented entirely in native Scratch 3.0 blocks.**

[![Mooneye 18/18](https://img.shields.io/badge/Mooneye%20GB-18%2F18-success)](docs/technical-report.md)
[![DMG Acid2](https://img.shields.io/badge/DMG%20Acid2-100%25-success)](docs/technical-report.md)
[![License: MIT](https://img.shields.io/badge/License-MIT-blue.svg)](LICENSE)

---

## Overview

**DMG-Scratch** is not an emulator in the conventional sense. It is a full digital logic reconstruction of the Game Boy DMG-01 hardware running strictly inside Scratch blocks:

- **100% Native Scratch Primitives**: Uses only Scratch variables, 1-indexed lists, basic arithmetic/comparison operators, and warp custom procedures.
- **Zero External Dependencies**: No JavaScript, No WebAssembly, No extensions, and No Foreign Function Interfaces (FFI) inside the machine core.
- **Cycle-Accurate Single Master Loop**: Advances in discrete 4.194304 MHz T-states (dots). Exactly 4 dots = 1 M-cycle; exactly 70,224 dots = 1 frame.
- **Verified Against Physical Hardware Standards**:
  - 100% Pass (23,040 / 23,040 pixels) on Matt Currie's **DMG Acid2** PPU benchmark.
  - Complete pass across Mooneye-GB acceptance suites (Timer 13/13, OAM DMA 3/3, Boot registers, STAT IRQ blocking).
  - Commercial game verification running full sessions of **Tetris** and **Super Mario Land** with real-time interactive controls.

---

## Play in TurboWarp

The compiled project file [`machine-base.sb3`](machine-base.sb3) is ready to load directly into [TurboWarp](https://turbowarp.org/editor).

### ROM Loading & Startup Flow
The release `.sb3` comes with **Super Mario Land** pre-bundled directly into its internal ROM memory lists:
1. Open [TurboWarp Editor](https://turbowarp.org/editor).
2. Click **File -> Load from your computer** and select `machine-base.sb3`.
3. Enable **Turbo Mode** (Shift + click the Green Flag, or toggle in Advanced settings).
4. Click the **Green Flag** — the machine boots immediately into the game!

*(To run another game, ROM bytes can be loaded into `[ROM_Bank_0]`, `[ROM_Bank_X]`, and `[ROM]` lists before running, or re-bundled using the build toolchain).*

### Controls

| Game Boy Button | Keyboard Key |
| :--- | :--- |
| **D-Pad Up** | `Up Arrow` |
| **D-Pad Down** | `Down Arrow` |
| **D-Pad Left** | `Left Arrow` |
| **D-Pad Right** | `Right Arrow` |
| **Button A** | `Z` |
| **Button B** | `X` |
| **Start** | `E` |
| **Select** | `Space` or `S` |

---

## Screenshots & Visual Parity

Screenshots captured directly from the native Scratch framebuffer output:

| Title Screen | Gameplay |
| :---: | :---: |
| ![Super Mario Land Title](demo-artifacts/sml_title.png) | ![Super Mario Land Gameplay](demo-artifacts/sml_gameplay.png) |
| ![Tetris Title](demo-artifacts/tetris_title.png) | ![Tetris Gameplay](demo-artifacts/tetris_gameplay.png) |

---

## Known Limitations

Transparent engineering and honest limitations build trust:
- **Model B Instruction Write Placement**: Multi-cycle CPU instructions execute their bus writes at the beginning of their M-cycle window rather than their exact intra-instruction M-cycle position (e.g. M3 for `LDH`). As a result, 5 Mooneye PPU tests calibrating intra-instruction micro-timing offsets fail by design.
- **APU Verification Scope**: The 4-channel sound synthesizer is verified via internal unit tests and commercial game execution; external APU acceptance suites (e.g. Blargg dmg_sound) have not yet been evaluated.
- **Gameplay Throughput**: Full active gameplay runs at ~24.5 FPS in TurboWarp (~0.41× real hardware speed) due to the overhead of per-dot FIFO processing and 4-channel audio synthesis in native blocks. Early menus run at ~87 FPS.
- **Reference Emulator Comparison**: Differential execution against SameBoy was not run in the automated harness; verification relies on physical hardware test ROMs (Mooneye, Blargg, Acid2).

---

## Architectural Highlights

- **The Master Dot Loop**: All subsystems (SM83 CPU, PPU pixel fetcher, APU audio mixer, Timer, DMA, Serial, Joypad matrix) execute synchronously inside a single `step_system_tick` warp block.
- **Fast Bitwise Operations**: Implemented using O(1) memory-mapped 64 KiB lookup tables (`LUT_AND`, `LUT_OR`, `LUT_XOR`) to overcome Scratch's lack of bitwise operators.
- **Dot-by-Dot PPU & FIFO Pipeline**: Mode 2 (OAM Search), Mode 3 (Pixel Transfer with dynamic fine-scrolling and sprite penalty delays), Mode 0 (HBlank), and Mode 1 (VBlank).
- **Silicon-Accurate Glitches**: Exact DMG hardware falling-edge timer glitches, STAT write glitches, and 5 M-cycle interrupt dispatch sequences modeled from physical hardware schematics.

---

## Documentation

- **[Technical Report](docs/technical-report.md)** — Comprehensive 8,000-word engineering whitepaper covering timing theory, subsystem mechanics, memory architecture, and test results.
- **[Design Notes](docs/design-notes.md)** — Architectural summary of the 15 key engineering decisions and silicon quirks modeled.

---

## License

This project is licensed under the [MIT License](LICENSE).
