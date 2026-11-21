# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

A port of Arnaud Carré's LSP (LightSpeed Player, originally 68k/Amiga — https://github.com/arnaud-carre/LSPlayer) to ARM2/ARM250 on the Acorn Archimedes. It is a single bare-metal ARM assembly program, not a library: it takes over the machine from RISC OS, emulates the Amiga Paula chip in software, and streams the mix into the MEMC sound DMA buffers.

Source comments are predominantly in French; labels are French too (`boucle` = loop, `voies` = voices/channels, `remplissage` = fill, `ecart` = gap, `depassement` = overrun, `bouclage` = looping). Keep that convention when editing.

## Build

`vasmarm_std_win32.exe` (vasm, ARM backend, oldstyle/GNU-ish syntax) is checked into the repo. The README's "needs msys2+vasm" refers to that toolchain; the bundled exe works directly from PowerShell.

Build order matters: `lsp.asm` `.incbin`s `build\fiqrmi2.bin`, so the FIQ handler must be assembled first, and `build\` must exist.

```
mkdir build
.\vasmarm_std_win32.exe -L compileFIQ2.txt -linedebug -m250 -Fbin -o build\fiqrmi2.bin fiqRM2.asm
.\vasmarm_std_win32.exe -L compiletest.txt -linedebug -m250 -Fbin -opt-adr -o build\test.bin lsp.asm
```

`-m250` (ARM250 target) is required — the code uses 26-bit PSR-in-PC idioms (`TEQP PC,#...`) that later cores reject.

`make.bat` is stale: it invokes `clear`, references `vectorball2.asm` (absent), and copies to another developer's absolute paths (`C:\Users\Eric\...`, `C:\Archi\Arculator_V2.0_Windows\hostfs`). Use it as a hint for the emulator drop step, not as a working script.

## Running

There are no tests. The output `test.bin` is an absolute binary linked at `.org 0x8000` — copy it into an Archimedes emulator's hostfs with RISC OS filetype `ff8` (absolute), e.g. `hostfs\test,ff8` in Arculator, and `*Run` it. `rmrebuild.asm` similarly produces a standalone at `0x8000`.

## Build-time configuration (top of `lsp.asm`, lines ~84-120)

These `.equ`s drive `.if` blocks throughout the file — changing one silently changes which code paths assemble:

- `DEBUG` — `0` = full takeover, no OS, custom FIQ/VSync handler and raw keyboard scan; `1` = leave RISC OS running, use `OS_Byte` VSync and key reads, allocate extra screen DA. Use `DEBUG=1` when bisecting mixer/player bugs, `DEBUG=0` for the real thing.
- `nombre_de_voies` — `4` or `8`. Everything for voices 5-8 is a *hand-duplicated* copy of the 4-voice code suffixed `_voies_5a8` (second player state, second bank, second Paula register block, second mixer). **Any change to a 4-voice routine must be mirrored into its `_voies_5a8` twin.**
- `frequence_replay` — `20833` / `31250` / `62500` Hz. Selects `nb_octets_par_vbl` (bytes per VBL per channel), the VIDC sound-frequency microsecond value, and which `table_frequences_*.s` gets included.
- `ecart_sample` (1024) — the guard gap appended after each instrument so the mixer can overrun a sample end without bounds-checking inside the inner loop.
- `FLAG_DMACON` — whether Amiga DMACON channel-enable bits are honoured.

## Architecture

**Data flow:** LSP-format music data (`.lsmusic` byte+word streams) and sample bank (`.lsbank`) are `.incbin`'d at the tail of `lsp.asm` (`LSPMusic`, `LSPBank`). Currently "SOTB - EXPLORATION".

**Init pipeline** (`main`, called in this order — order is load-bearing):
1. `LSP_PlayerInit_standard` — parse LSP header, relocate the instrument table into `LSP_InstrumentInfo`, set up `LSPVars_standard`.
2. `convertion_samples_instruments_en_mu_law` — convert signed 8-bit Amiga samples to the VIDC μ-law-ish encoding via `lin2logtab` (built by `create_table_lin2log`).
3. `ajout_ecart_entre_instruments_et_copie_repetitions` — respace samples with `ecart_sample` bytes between them, pre-copying loop data into the gap, then relocate every offset that pointed at the old layout.
4. `mise_a_jour_offsets_repetition` — fix up loop start/length offsets after the respacing.
5. `inverse_bytes_stream_words` — byte-swap the 16-bit word stream once at init so the per-tick player can read it natively (Amiga is big-endian, ARM here is little-endian).

**Per-frame loop** (`boucle`), one iteration per VBL:
- `LSP_Player_standard` — the LSP tick. This is the direct 68k→ARM transliteration of the original: a byte stream of opcodes indexes a code table, patching a block of *virtual* Paula registers (`Paula_registers_external`: LCH/LEN/PER/VOL per channel, plus `DMACon`).
- `Paula_remplissage_DMA_416` — the mixer. Converts virtual Paula period/volume into 20.12 fixed-point increments (via `table_increments_frequence`) held in `Paula_registers_internal`, then jumps into `expanded_filling_routine`: a `.rept nb_octets_par_vbl` fully-unrolled block that, per output byte, does 4× (sample fetch by `[R1, Rn, lsr #12]`, increment, volume-subtract, saturate, pack) and stores one 32-bit word of 4 interleaved channel bytes. Register allocation for that loop is documented in a comment block at `expanded_filling_routine` — it uses every register including R13/R14, so it runs with a return address stashed in R10 and is entered/left by `ldr pc`, not `bl`.
- After mixing, per-channel end/loop handling (`test_depassement_canal_*` / `pas_de_bouclage_canal_*`) wraps the fixed-point positions.
- `set_dma_dma1` + `swap_pointeurs_dma_son` — program MEMC Sstart/SendN (`0x3600000` base) with the just-filled buffer and flip to the other one. Double-buffered; buffers are carved out of the screen dynamic area past `fin_de_la_memoire_video`.
- `wait_VBL` + `scankeyboard` (or the RISC OS equivalents under `DEBUG=1`); space exits.

The `SWI 22` / `MOVNV R0,R0` … `teqp r15,#0` pairs scattered through the loop are supervisor-mode entry/exit around border-colour writes to VIDC `0x3400000` — these are raster-time profiling markers, not logic.

**Interrupt layer:** With `DEBUG=0` the program claims FIQ from RISC OS (`XOS_ServiceCall 0x0C`), saves the FIQ bank and IOC IRQ masks, and overwrites the vector at `0x18`. The handler body lives in a *separate* assembly unit — `fiqRM2.asm` (`.org 0x18`, cycle-counted, padded with `nop`s to a fixed layout, magic `"rSTm"` at `&FC`) — assembled to a raw binary and `.incbin`'d at `fiqbase`. That is why the two-step build exists. `fiqRM.asm` is the fuller Rasterman-style variant (per-line colour-0/vstart/vend table driven by Timer 1); `fiqRM2.asm` is the stripped VSync-only version that the player actually uses. Offsets like `FIQ_notHSync` at `&C0` and `position_ligne_hsync` at `&E4` are hard-coded addresses referenced from the installer — moving anything in those files breaks the caller.

`rmrebuild.asm` is a standalone reimplementation/rebuild of the RasterMan raster-effects module (it `.incbin`s a RISC OS module `97,ffa` and `build\fiqrmi.bin` from `fiqRM.asm`); it is not part of the player build and its inputs are not all present in the repo.

`swis.h.asm` is the shared constant header: RISC OS SWI numbers plus RasterMan (`0x47e80+`) and QTM (`0x47e40+`) SWIs.

## Constraints worth remembering

- 26-bit ARM. Mode/flags live in the top and bottom bits of PC; `TEQP PC,#...` followed by `MOV R0,R0` is the mode switch idiom and the `nop` is mandatory (pipeline/register-bank sync).
- The FIQ handler and the mixer inner loop are cycle-budgeted; the comment columns in `fiqRM*.asm` are instruction counts and byte offsets. Adding an instruction there is a behavioural change.
- The unrolled mixer costs `nb_octets_par_vbl` × ~21 instructions of code space (~35 KB at 20833 Hz); raising `frequence_replay` grows the binary proportionally.
