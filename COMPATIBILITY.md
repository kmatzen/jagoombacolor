# Jagoomba Color — Compatibility Gap Analysis

A comprehensive analysis of what this GB/GBC emulator implements, what it's
missing, and how feasible each gap is to close.

## Test Suite Results

| Test | Result | Notes |
|------|--------|-------|
| Blargg cpu_instrs | **PASS** | All 11 instruction tests |
| Blargg instr_timing | FAIL | RST 38h timing off (cause unclear) |
| Blargg mem_timing | FAIL | Fundamental architecture limitation |
| Blargg mem_timing2 | FAIL | Same |
| cgb-acid2 | **PASS** | Minor BG/OBJ priority eye artifacts |
| EI delay test | **PASS** | Custom test ROM validates 1-instruction delay |
| Sprite limit test | **PASS** | Custom test ROM validates 10/line limit |
| 14 game regression tests | **PASS** | Crystal, Shantae, Zelda DX, Kirby, etc. |

---

## CPU (SM83 / LR35902)

### What works
- All 246 valid base opcodes + 256 CB-prefixed opcodes
- Cycle counts correct for most instructions (validated by `scripts/validate_timing.py`)
- HALT with interrupt wake-up
- DAA (decimal adjust) fully correct
- Interrupt priority ordering (VBlank > STAT > Timer > Serial > Joypad)
- EI 1-instruction delay (interrupts enabled after next instruction completes)

### Gaps

**STOP instruction simplified** — LOW PRIORITY
> Only handles double-speed toggle. Doesn't block waiting for joypad/interrupt
> reset. No known game depends on exact STOP behavior beyond speed switching.

**RST 38h timing** — UNCLEAR
> Blargg's instr_timing test fails specifically on RST 38h. The instruction
> uses `fetch 16` which matches documentation. May be a cycle-counting
> methodology difference rather than a real bug. No test ROM available to
> investigate further.

**Memory timing (mem_timing, mem_timing2)** — NOT FEASIBLE
> These tests verify cycle-accurate memory access timing (e.g., that a read
> from ROM takes exactly N cycles). The emulator uses a per-scanline cycle
> budget rather than per-access timing. Fixing this would require
> restructuring the entire CPU loop, which is the core performance path.

---

## PPU / LCD

### What works
- Per-scanline rendering with HBlank DMA for mid-frame register changes
- LCDC all bits including mid-frame sprite size/enable tracking
- STAT mode flags (cycle-based thresholds, self-modifying for double speed)
- LYC=LY coincidence interrupt with 0→1 edge detection
- Window rendering (WX, WY) with per-scanline buffering
- Sprite rendering with 8x8/8x16 mid-frame tracking
- 10 sprites per scanline limit (sprites hidden if over limit on all scanlines)
- OAM DMA (FF46) with pattern-detection optimization
- GBC HDMA (FF51-FF55): both general and HBlank modes
- VRAM banking (FF4F)
- GBC palette writes (FF68-6B) with auto-increment
- Per-scanline palette DMA3 for games that update palettes every scanline
- Mid-frame scroll register changes via DMA0/DMA1/DMA2

### Gaps

**CGB BG/OBJ priority not pixel-perfect** — HARD
> cgb-acid2 shows minor "eye artifacts" from priority handling. CGB has
> complex priority rules involving per-tile BG priority bits, OBJ priority,
> and LCDC bit 0. The current implementation approximates these using GBA
> hardware layer priorities.
> *Root cause: GBA priority system doesn't map 1:1 to CGB rules. Would need
> per-pixel software compositing, which is too slow.*

**STAT mode 0/2 interrupts may be unreliable** — UNCLEAR
> The HBlank (mode 0) and OAM (mode 2) interrupt code is disabled (`#if 0`
> in lcd.s). Only LYC=LY and VBlank STAT interrupts are active. This may
> affect games that rely on mode-change interrupts for timing.
> *Impact: low — most games use VBlank and LYC, not mode 0/2 interrupts.*

**Per-scanline palette flicker on Hercules GBC** — NOT FEASIBLE
> GBC frame takes ~2 GBA frames to process. Games that update palettes every
> scanline (143+ writes) get half-stale palette data visible as flicker.
> 11 different mitigation strategies attempted (documented in
> KNOWN_ISSUES.md). Root cause is the fundamental 3:1 ARM/GBC cycle ratio.

**No per-dot (sub-scanline) rendering** — NOT FEASIBLE
> Rendering is per-scanline, not per-dot. Mid-scanline effects (raster
> effects within a single scanline) are not supported. Would require
> complete rewrite of the rendering pipeline. Very few GB/GBC games use
> sub-scanline effects.

---

## Audio (APU)

### What works
- All 4 channels mapped: pulse 1 (sweep), pulse 2, wave, noise
- Master volume (NR50), channel output selection (NR51), master enable (NR52)
- Wave RAM (FF30-FF3F) with bank switching

### Architecture

The emulator uses **direct pass-through** to GBA APU hardware rather than
software synthesis. GB sound register writes are mapped 1:1 to equivalent
GBA sound registers. The GBA hardware generates the actual audio.

### Gaps

**No software audio synthesis** — NOT FEASIBLE (by design)
> Software synthesis would consume too many ARM cycles on the GBA's 16MHz
> CPU. The pass-through approach gives acceptable audio quality for free.
> Trade-off: some GB sound quirks (envelope trigger bugs, frame sequencer
> edge cases, length counter reload behavior) are not reproduced.

**Sound channel edge cases** — LOW PRIORITY
> GB APU has obscure behaviors: zombie mode envelope, DAC power-off pops,
> wave channel corruption when restarting. None of these are emulated, but
> they're also rarely used intentionally by games.

---

## Timers

### What works
- DIV (FF04): resets on write
- TIMA/TMA/TAC (FF05-FF07): frequency selection, overflow detection, TMA reload
- Timer interrupt generation (IF bit 2)
- Double-speed mode adjusts timer rates

### Gaps

**DIV is not a true 16-bit counter** — LOW PRIORITY
> Real GB has a 16-bit internal counter; DIV is bits 15-8. Writing DIV
> resets the full 16-bit counter, which can cause a "falling edge" on the
> timer's selected bit, potentially triggering an unexpected TIMA increment.
> Current implementation just stores 0.
> *Impact: extremely obscure; no known game depends on this.*

**No TAC enable-bit edge detection** — LOW PRIORITY
> Disabling the timer at the right moment can trigger a spurious TIMA
> increment on real hardware. Not emulated.

**TIMA overflow delay** — LOW PRIORITY
> Real GB has a 1-cycle window after overflow where TIMA reads as 0 before
> TMA is loaded. Not emulated.

---

## Memory Bank Controllers (MBC)

### What works
- **MBC0**: ROM only (no banking)
- **MBC1**: 5-bit ROM bank + 2-bit upper, mode switching
- **MBC2**: 4-bit ROM bank, built-in 512-nibble RAM
- **MBC3**: 7-bit ROM bank, RTC register selection + latch, software RTC fallback
- **MBC5**: 9-bit ROM bank, rumble bit (conditional compilation)
- **HuC1/HuC3**: Hudson Soft mappers (basic support)
- RAM enable/disable (0x0A magic value)
- SRAM bank switching
- SRAM write-through persistence (no compressed save system needed)

### Gaps

**MBC4/MBC6/MMM01 are stubs** — LOW PRIORITY
> These rare mappers have init functions but no banking logic. Very few
> games use them (MBC6: Net de Get, MMM01: Momotarou Collection 2).

**MBC7 accelerometer** — NOT FEASIBLE
> MBC7 includes an ADXL202E accelerometer for tilt controls (Kirby Tilt 'n'
> Tumble). Would need to map to GBA gyroscope hardware (which most GBAs
> don't have) or simulate via button input.

---

## GBC-Specific Features

### What works
- GBC detection (bit 7 of header byte 0x143)
- VRAM banking (FF4F): two 8KB banks
- WRAM banking (FF70): eight 4KB banks
- Color palettes (FF68-6B): BCPS/BCPD/OCPS/OCPD with auto-increment
- Double-speed mode (FF4D): STOP-triggered switch
- HDMA (FF51-FF55): general + HBlank DMA
- DMG compatibility mode (GBC registers disabled for DMG games)
- Automatic DMG palette selection from game hash database (76 games)
- SGB border + palette support (parallel to GBC)

### Gaps

**Infrared port (FF56) not implemented** — LOW PRIORITY
> No-op read/write. Used by a handful of games for local communication
> (Pokemon Gold/Silver mystery gift, Mission Impossible). Would need GBA IR
> hardware or a network protocol.

**CGB-only: undocumented registers** — LOW PRIORITY
> FF6C (OPRI), FF72-FF77 (undocumented) are not handled. Very few games
> use them.

---

## Serial / Link Cable

**Status: Stubbed** — HARD
> Serial data writes (FF01) are ignored. Reads return 0xFF (no device).
> Serial interrupt is triggered after one scanline when internal clock is
> enabled. This means:
> - No link cable multiplayer
> - No printer support
> - Games that require serial handshake may hang
>
> *Implementing real serial would need GBA link cable hardware or wireless
> adapter support. Game-specific serial stubs (e.g., always return "no
> partner") could prevent hangs.*

---

## Architectural Constraints

These are fundamental to running a GB/GBC emulator on GBA hardware:

| Constraint | Impact |
|------------|--------|
| **IWRAM: 32KB, ~97% used** | Limited room for new per-scanline code |
| **All 4 GBA DMA channels allocated** | No spare DMA for new features |
| **ARM/GBC cycle ratio ~3:1** | GBC frame spans ~2 GBA frames |
| **16MHz ARM7TDMI** | No room for software audio synthesis or per-pixel compositing |
| **GBA APU ≈ GB APU** | Sound pass-through works but can't emulate quirks |
| **GBA has 4 BG layers + OBJ** | Priority mapping is approximate, not exact |

---

## Remaining Gaps (all low priority or not feasible)

The easy and moderate items have been addressed. What remains:

| Gap | Difficulty | Notes |
|-----|-----------|-------|
| STOP joypad wait | LOW PRIORITY | No known game depends on it |
| RST 38h timing | UNCLEAR | Matches docs; no test ROM available |
| Memory timing | NOT FEASIBLE | Would need per-access cycle counting |
| CGB BG/OBJ priority | HARD | Needs per-pixel compositing |
| STAT mode 0/2 interrupts | UNCLEAR | Disabled intentionally; low impact |
| Per-scanline palette desync | NOT FEASIBLE | Fundamental cycle ratio |
| Sub-scanline rendering | NOT FEASIBLE | Complete rewrite needed |
| Software audio synthesis | NOT FEASIBLE | No CPU budget |
| Sound edge cases | LOW PRIORITY | Rarely used by games |
| DIV 16-bit counter | LOW PRIORITY | No known game depends on it |
| TAC edge detection | LOW PRIORITY | Extremely obscure |
| TIMA overflow delay | LOW PRIORITY | Extremely obscure |
| MBC4/MBC6/MMM01 | LOW PRIORITY | Very few games |
| MBC7 accelerometer | NOT FEASIBLE | No hardware |
| Infrared port | LOW PRIORITY | Handful of games |
| Serial / link cable | HARD | Needs GBA link hardware |
