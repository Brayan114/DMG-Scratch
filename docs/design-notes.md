# Architectural Design Notes: DMG-Scratch

This document highlights 15 key architectural and hardware engineering decisions made during the reconstruction of the Nintendo Game Boy DMG-01 (DMG-CPU B) in native Scratch 3.0 blocks.

---

### 1. Master Clock Loop Frame Pacing
Scratch scripts that run in 'warp mode' (without screen refresh) execute synchronously within the engine's current frame. To maintain deterministic hardware timing without stalling the host engine or throttling to 60 dots/second:
- The outer Stage script runs a frame loop aligned with the host display refresh.
- Each frame calls an inner loop of exactly **70,224 dots** (456 dots × 154 scanlines).
- Every hardware subsystem (CPU, PPU, APU, Timer, DMA, Serial, Joypad) steps synchronously within this single master clock.

### 2. Programmatic JSON AST Compilation
Hand-authoring a project containing 19,000+ Scratch blocks and complex 16-bit decoding matrices is infeasible and error-prone. The machine is defined and generated via a Node.js AST builder that compiles modular JavaScript hardware definitions directly into standard Scratch 3.0 project.json schemas.

### 3. Register Pair Synthesis and Strict F-Masking
Register pairs (AF, BC, DE, HL) are not stored as duplicate 16-bit variables. They are synthesized dynamically using getter/setter custom blocks. Furthermore, all writes to register F enforce hardware Invariant 1 (eg_F mod 16 == 0), ensuring the low 4 unused flag bits are permanently zeroed.

### 4. Hybrid I/O Register Architecture
High-frequency registers (such as LCDC, STAT, SCX, SCY, LY, LYC, DIV, TIMA, TAC, IF, IE) are stored as individual scalar variables for instantaneous O(1) Scratch variable access. Infrequent registers reside in a 128-element IO_REGS list. Reads and writes route via a bifurcated address decoder that prevents dual-storage desynchronization.

### 5. Memory-Mapped Bitwise Lookup Tables
Scratch 3.0 lacks native bitwise operators (AND, OR, XOR, bit shifts). Mathematical workarounds (e.g. division and modulo) are computationally prohibitive in inner loops. The architecture uses pre-generated 65,536-entry 1-indexed Scratch lists (LUT_AND, LUT_OR, LUT_XOR) that compute 8-bit bitwise logic in a single O(1) list lookup.

### 6. SM83 CPU Instruction Decoding via Dispatch Jump Tables
All 256 base opcodes and 256 CB-prefixed opcodes are decoded using binary-partitioned custom blocks. This avoids deep nested if-else cascades and allows Scratch to branch to specific opcode handlers in minimal evaluation steps.

### 7. Interrupt Controller & 5 M-Cycle Hardware Dispatch
Interrupt servicing models the physical 5 M-cycle silicon pipeline:
- M1: IRQ sample and address bus idle.
- M2: Stack prep (SP unchanged).
- M3: High byte of PC pushed to stack.
- M4: Low byte of PC pushed to stack.
- M5: Vector target loaded into PC, IME cleared, and the dispatched bit in IF cleared strictly on M5.

### 8. Timer Subsystem & Falling-Edge Glitch Logic
The timer counter increments synchronously with an internal 16-bit master counter. Frequency selection uses a hardware multiplexer whose output signal feeds a falling-edge detector. Resetting DIV or modifying TAC bits during specific counter phases reproduces exact silicon hardware glitches (instantaneous TIMA increments).

### 9. Two-Cycle Timer Overflow Pipeline
TIMA overflow from  does not immediately trigger an interrupt. For 1 M-cycle (Cycle A), TIMA reads . On the subsequent M-cycle (Cycle B), TIMA reloads from TMA and asserts the Timer IRQ. Writes during Cycle A cancel the reload; writes to TIMA during Cycle B are ignored.

### 10. OAM DMA 161 M-Cycle Transfer & Bus Lockout
Writing to  initiates a 161 M-cycle transfer (1 setup M-cycle + 160 copy M-cycles) that copies 160 bytes from the source page to OAM at 1 byte per M-cycle. During this transfer, access to external bus regions is blocked while HRAM remains executable.

### 11. Dot-by-Dot PPU State Machine & Dynamic Mode 3 Penalties
The PPU does not advance in scanline batches. It steps dot-by-dot through Mode 2 (OAM Search, 80 dots), Mode 3 (Drawing, 172+ dots base), Mode 0 (HBlank), and Mode 1 (VBlank). Mode 3 duration adjusts dynamically according to fine scroll ( mod 8$), window activation, and sprite fetch penalties.

### 12. PPU Pixel FIFO Multiplexing
Background, window, and sprite pixels are streamed into internal 8-pixel FIFO lists. Mode 3 advances only when the background FIFO contains sufficient pixels. Sprites are fetched during Mode 3, matched by X-coordinate, and merged based on priority and palette attributes.

### 13. STAT Interrupt OR-Gate & Rising-Edge Glitch
DMG STAT interrupt sources (LY=LYC, Mode 2, Mode 1, Mode 0) are combined through a hardware OR-gate. An interrupt fires only on a rising edge (0 -> 1) of the combined line. Writing to STAT during Mode 1 causes a temporary 1 M-cycle write glitch where all condition select lines are high, triggering spurious STAT interrupts as observed on DMG silicon.

### 14. 512 Hz APU Frame Sequencer & 4-Channel Synthesis
The Audio Processing Unit implements all four sound channels (Pulse 1 with sweep, Pulse 2, 32-sample Wave, and LFSR Noise). Timing is driven by a 512 Hz frame sequencer clocked from the master timer counter, stepping length counters (256 Hz), frequency sweep (128 Hz), and volume envelopes (64 Hz).

### 15. Standard Active-Low Joypad Matrix & Interactive Keyboard Polling
The joypad registers () model the authentic active-low 2×4 matrix (P14 selects D-Pad, P15 selects action buttons). Keyboard input from native Scratch sensing blocks is mapped directly to the matrix:
- **Right / Left / Up / Down**: Arrow keys
- **A / B**: Z / X
- **Start**: E
- **Select**: Space / S
Transitions on the matrix input lines trigger the high-to-low falling edge Joypad interrupt ().
