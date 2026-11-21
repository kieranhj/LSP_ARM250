# LSP_ARM250 — Technical Analysis

A cross-referenced study of this repository against:

- **LSP (Light Speed Player)** v1.26 by Arnaud Carré / Leonard-Oxygene — `C:\Users\khcon\OneDrive\Amiga\Repos\LSPlayer`
- **RasterMan** v0.30 by Steve Harrison (vasm port) — `C:\Users\khcon\OneDrive\Archie\Repos\rasterman-vasm`
- **QTM (QTheMusic)** v1.45 source by Steve Harrison — `…\arculator\hostfs\Steve\qtmv1-45\Source` (BBC BASIC V, detokenised for this analysis)
- **RISC OS Programmer's Reference Manual** — `C:\Users\khcon\OneDrive\Archie\Manuals\www.riscos.com`, chiefly *Vol 4 ch. 72 The Sound system*, *Vol 5a ch. 132 Sound*, and *The hardware* (VIDC/MEMC/IOC)

---

## 1. Executive summary

This project is a port of the Amiga's fastest MOD replay engine to the Acorn Archimedes. It works, it plays a real module (Shadow of the Beast — Exploration), and it demonstrates a genuinely interesting idea: because LSP pre-compiles a `.mod` offline into a stream of *Paula register writes*, the runtime cost of "playing the music" collapses to almost nothing, and what remains is purely the cost of *emulating Paula* — which on the Archimedes means emulating four DMA channels that VIDC will happily provide in hardware.

The port is an unfinished research prototype, not a product:

- It is a **standalone bare-metal demo**, not a module or library. It takes over FIQ/IRQ, disables RISC OS, and runs its own main loop. There is no API for a third party to call.
- **8-voice mode does not assemble** (missing `8canaux2.lsbank`, and `LSPMusic_4a8voies` has no data at all).
- It is **locked to LSP file format v1.03**; current LSPConvert (v1.26) emits a header the init code will misparse.
- The mixer is **fully unrolled to ~40 KB of code** and is roughly cycle-comparable to QTM's, not faster.
- `make.bat` is broken and refers to another developer's machine.

Where it beats QTM is the *tick*, not the *mix*. Where it loses is integration, generality and code size. Section 8 sets out what to do about that.

One conclusion is worth stating up front, because it governs how useful this work can be: **on the Archimedes the tick is not where the time goes.** The DMA fill costs roughly 10–30× the interpreter it replaces, so LSP's headline Amiga result — an 8× faster tick — is worth about 1 % of an ARM2. Section 9 works this through and sets out where the cycles actually are.

---

## 2. How LSP works

### 2.1 The core idea

Protracker replay is expensive because the player is an *interpreter*. Every tick it walks four channels, decodes effect commands, applies vibrato/portamento/tremolo/arpeggio state machines, and only then writes Paula's registers. QTM's `QTM_Play` is exactly this — a large, well-optimised ARM implementation of that interpreter (`set_setspeed`, `vibrato_speed`, `tportaspeed`, the whole `nextFX` dispatch).

LSP does the interpretation **once, offline, on a PC**. `LSPConvert.exe` loads the `.mod` (via Martin Cameron's *micromod*), runs the entire song through a Protracker-accurate simulation, and records — for every tick of the song — only the *changes* that actually reach Paula's hardware registers. The runtime player therefore contains **no effect engine at all**. There is no concept of vibrato in the player; vibrato appears in the data as a period value that happens to change every tick.

The consequence, from the LSP README's own Amiga 500 measurements:

| Player | Average tick | Peak tick |
|---|---|---|
| Protracker | ~7 scanlines | — |
| LSP Standard | 0.83 scanline | 2.27 scanlines |
| LSP Insane | 0.46 scanline | 1.34 scanlines |
| LSP Micro | 1.54 scanlines | 3.68 scanlines |

Standard-mode player code is under 512 bytes.

### 2.2 The data format

`LSPConvert` emits two files (three with `-insane`):

- **`.lsmusic`** — the score: header, instrument table, code table, and two byte-oriented streams. Loadable into any memory.
- **`.lsbank`** — the sample bank, byte-identical in layout to what Paula will DMA. On Amiga this must be chip RAM.
- **`<name>_insane.asm`** — with `-insane`, a *generated player source file* specialised to that one module.

Both files start with the same 32-bit unique ID so the player can refuse a mismatched pair (`LSP_MusicInit` in `LightSpeedPlayer.asm` does exactly this).

The v1.26 `.lsmusic` header, from `LSPEncoder::ExportScore` (`src/LSPEncoder.cpp:1155`):

```
'LSP1'          4   magic ('LSPm' in micro mode)
uniqueId        4   CRC of MOD content + converter version + convert params
major, minor    2   format version
flags           2   bit0 = getpos supported, bit1 = setpos supported; also the relocation guard byte
bpm             2   initial BPM
escRewind       2   \
escSetBpm       2    > escape code values, assigned dynamically from the first unused code
escGetPos       2   /
frameCount      4   song length in ticks
instrCount      2
instruments     12 × instrCount   { u32 sampleOffset, u16 lenWords, u32 repeatOffset, u16 repLenWords }
codeTableSize   2
codeTable       2 × codeTableSize
seqCount        2   (+ 8 bytes per seq entry when -setpos)
wordStreamSize  4
byteStreamLoop  4
wordStreamLoop  4
<word stream><byte stream>
```

### 2.3 The two-stream trick

The runtime state is one byte per tick, read from the **byte stream**. That byte indexes the **code table**, yielding a 16-bit *command word* whose bits are a bitmap of "which registers changed this tick":

```
bit 15..12   channel D,C,B,A : restart from loop point (repeat)
bit 11..8    channel D,C,B,A : new volume    (data follows in the BYTE stream)
bit  7..4    channel D,C,B,A : new period    (data follows in the WORD stream)
bit  3..0    channel D,C,B,A : new instrument (delta follows in the WORD stream)
             — the low 4 bits double as the DMACON enable mask
```

Values are then pulled from whichever stream matches their width: 8-bit volumes from the byte stream, 16-bit periods and instrument deltas from the word stream. Splitting by width keeps the word stream naturally 16-bit-aligned (the 68000 `move.w (a0)+` needs this) and improves compressibility. A command word of 0 means "nothing changed" — the single commonest case, and it early-outs in about a dozen instructions.

A byte-stream value of `0` is an escape meaning "add 256 and read the next byte", which is how more than 255 distinct command words are addressed. Three special command values are reserved for rewind, set-BPM and get-position; in v1.26 their numeric values are stored in the header, because the encoder assigns them from the first unused code (`LSPEncoder.cpp:355`).

Instrument numbers are stored as **signed deltas** against the previous instrument used, in units of 12 bytes (the instrument record size), which is why the original keeps `m_lspInstruments` biased by −12 — "to win 2 cycles in insane player :)".

### 2.4 Insane and micro modes

`-insane` generates a player where the code table is replaced by *actual code*: each command word becomes a straight-line subroutine with no bit tests at all, jumped to through a table. Up to ~30 KB of generated player per module, in exchange for another ~2× speedup. `-micro` reorders the data into 16 separate per-parameter streams to maximise Shrinkler compression, at the cost of speed.

This repo uses **Standard** mode. `LSP_PlayerInit_Insane` and `LSPVars_Insane` exist as stubs (`lsp.asm:2244`, `:2761`) but are unused.

---

## 3. The Archimedes sound system, and why Paula does not map onto it

### 3.1 Paula

Paula is four independent DMA channels. Per channel: a start pointer, a length in words, a 16-bit period (a *divider* on the 3.546895 MHz PAL colour clock), and a 6-bit volume. Hardware fetches the sample, converts at the programmed rate through a **linear** 8-bit signed DAC, multiplies by volume, and sums the four channels into two stereo outputs. When a channel reaches its length it automatically restarts from the *loop* pointer/length that were latched during playback. DMACON enables/disables channels; re-enabling restarts a channel from the top.

Everything a MOD needs — per-channel pitch, per-channel volume, hardware looping, hardware mixing — is in silicon.

### 3.2 VIDC1 / MEMC1

The Archimedes has none of that. From the PRM (*The Sound system*, *The hardware*):

- **One** DMA stream, not four. MEMC's sound DAG has a sound pointer, a current-end register, and a next-start/next-end pair. When the pointer hits the end, MEMC swaps to the "next" buffer and IOC raises an interrupt. That is the entire hardware channel model.
- Multi-channel is **time-division multiplexing of one stream**. For *n* ∈ {1,2,4,8} channels the bytes are interleaved at *n*-byte intervals in the buffer, and VIDC demultiplexes them into *n* stereo image registers. Four channels means bytes 0,4,8… are channel 1, bytes 1,5,9… are channel 2, and so on.
- **There is no pitch.** The output rate is global — one *sample period* for all channels, set by `Sound_Configure` R2 in µs. Resampling is entirely the software's problem.
- **There is no volume.** Amplitude scaling is entirely the software's problem.
- **There is no looping.** The software refills the buffer every period.
- The DAC is **not linear**. Each byte is an 8-bit *signed logarithm* in VIDC format — a µ-law-like companded value. The PRM only ever describes it as "an eight bit signed logarithm, ready for direct output to the VIDC chip", and provides `Sound_SoundLog` (linear→log) and `Sound_LogScale` (log→volume-scaled log) to produce it.
- The DMA buffer must fit in one MEMC page; the RISC OS default is 208 bytes/channel per fill, giving a 0.9984 cs buffer period and a ~100.16 Hz fill interrupt.
- Default output period is 6 µs → **48 µs sample period → 20.833 kHz per channel**. The floor is 4 µs → 32 µs → 31.25 kHz, below which DMA contention with the video bandwidth becomes a problem.
- Stereo position is per *channel*, not per sample: eight `Sound_Stereo` image registers, −127 (full left) to +127 (full right). With four channels, image registers pair up 0+4, 1+5, 2+6, 3+7 — which is why this code programs all eight even in 4-voice mode.

### 3.3 The one place where the Archimedes wins

Because VIDC multiplexes up to **eight** hardware channels while Paula has four, and because the Archimedes' per-channel rate does not drop as channels are added (only the byte output period halves), an 8-channel MOD is *natively* playable here in a way it is not on stock Paula. That is exactly what the abandoned `_voies_5a8` path was reaching for.

### 3.4 The critical consequence for this port

**No summing is required.** A conventional software mixer on a linear DAC must add channels together, worry about headroom and clipping, and generally work in 16 bits. Here, "mixing" is nothing but *interleaving bytes into the right lanes*, and volume is a *subtraction in the log domain*.

That last point is the linchpin. The log→linear table shipped with QTM (`LOGdata`, generated by `LOGtable` via `Sound_SoundLog`) decodes as:

```
index:   0    1    2    3  ...  250  251  252  253  254  255
value:   0   -1    0   -1  ...  111 -108  115 -112  119 -116  127 -120
```

Even indices are positive, odd indices negative, magnitude rising monotonically with index. **Bit 0 is the sign bit; bits 1–7 are the companded magnitude.** Therefore subtracting an *even* constant from a log byte attenuates it while leaving the sign intact — a multiply becomes a subtract. Both QTM and this port use precisely that, and both are careful to keep the attenuation even. QTM's build code says so explicitly:

```
AND R0,R0,#%11111110   ;stops odd/even changes (ie. +/- sample changes)
```

and this port's `table_volume` (`lsp.asm:4281`) contains only even bytes, 0xFE (silence) down to 0x00 (unattenuated), indexed by Amiga volume 0–64.

---

## 4. Implementation details

### 4.1 French glossary

| Label / comment | Meaning |
|---|---|
| `boucle`, `boucler`, `bouclage` | loop / to loop / looping (sample repeat) |
| `déboucler`, "unlooped" | *unrolled* (as in loop unrolling) — the printed banner "unlooped" means *unrolled* |
| `voie`, `voies_5a8` | voice/channel; "voices 5 to 8" |
| `canal A/B/C/D` | channel A–D |
| `remplissage` | filling (buffer fill) |
| `écart` / `ecart_sample` | gap / the guard gap inserted after each sample |
| `dépassement` | overrun (sample pointer past the end) |
| `répétition` | repeat / loop section of a sample |
| `mélangé`, `mixage` | mixed / mixing |
| `attente`, `boucle_attente` | wait / wait loop |
| `nombre_de_voies` | number of voices |
| `nb_octets_par_vbl` | bytes per VBL |
| `fréquence_replay` | replay rate |
| `increment_frequence` | pitch increment |
| `table_frequences` | frequency (period→increment) table |
| `mise à jour` | update |
| `sauvegarde` | save / backup |
| `ancien vecteur` | old (system) vector |
| `pile_regs` | register stack/save area |
| `ligne` | line (scanline) |
| `couleur` | colour |
| `fin`, `début` | end, start |
| `saute` | skip |
| `essai`, `tentative` | attempt |
| `calé entre 0x18 et 0x58` | "wedged between 0x18 and 0x58" (the FIQ code's address budget) |
| `on gère le dmacon` | "we handle DMACON" |

### 4.2 Build-time configuration

All in `lsp.asm:84–120`, driving `.if` blocks throughout:

| Symbol | Effect |
|---|---|
| `DEBUG` | `0` = full hardware takeover, custom FIQ/VSync, raw keyboard. `1` = RISC OS stays up, `OS_Byte` VSync and key reads. |
| `nombre_de_voies` | 4 or 8. 8 selects a *hand-duplicated* second copy of every routine, suffixed `_voies_5a8`. |
| `frequence_replay` | 20833 / 31250 / 62500 Hz → selects `nb_octets_par_vbl` (416/624/1248), the VIDC µs value (48/32/16) and which `table_frequences_*.s` is included. |
| `ecart_sample` | 1024. Guard gap after each sample. |
| `FLAG_DMACON` | Whether DMACON channel-enable bits are honoured. Currently **0**. |

The rate arithmetic is exact rather than approximate: 416 bytes × 50.0801282 Hz = 20 833.33 Hz, matching a 48 µs sample period. This only holds because the program refills per *VSync* on a TV-rate mode, not per sound interrupt.

### 4.3 The virtual Paula

`Paula_registers_external` (`lsp.asm:4175`) is a literal software model of Paula's register block, 0x14 bytes per channel — location, length, period, volume, data. `Paula_registers_internal` at +0x50 adds what the hardware would have kept privately: current sample offset in 20.12 fixed point, end offset, and the increment.

The LSP tick writes only the *external* half. The mixer reads both and maintains the internal half. This is a clean separation and is the main reason the tick could be transliterated from 68k almost line-for-line.

### 4.4 Init pipeline

Order is load-bearing (`main`, `lsp.asm:~330`):

1. **`LSP_PlayerInit_standard`** — parse the `.lsmusic` header, copy the instrument table into `LSP_InstrumentInfo` **expanded from 12-byte to 16-byte records**, and set up `LSPVars_standard`. The 16-byte stride matters: it turns the instrument index multiply into a shift, and `table_12_16` (`lsp.asm:2226`) converts the format's ×12 deltas into ×16 offsets by table lookup.
2. **`convertion_samples_instruments_en_mu_law`** — walk the entire `.lsbank` and replace every byte with `lin2logtab[byte]`, in place. `lin2logtab` is built at init by calling `Sound_SoundLog` 256 times (`create_table_lin2log`). After this the bank is no longer linear PCM; it is VIDC log data ready for direct DMA.
3. **`ajout_ecart_entre_instruments_et_copie_repetitions`** — the interesting one. See below.
4. **`mise_a_jour_offsets_repetition`** — for instruments whose repeat length is 1 word (Protracker's "no loop" idiom), redirect the repeat pointer at the shared `silence` block.
5. **`inverse_bytes_stream_words`** — byte-swap every 16-bit word in the word stream once, at init, converting the format's big-endian values to little-endian so the tick can read them natively.

### 4.5 The `ecart` (gap) trick

This is the port's neatest idea and its most fragile.

A software mixer that must check "have I run off the end of the sample?" *inside* the inner loop pays for that test on every output byte. `ajout_ecart_entre_instruments_et_copie_repetitions` (`lsp.asm:2311`) avoids it entirely:

1. Every sample in the bank is pushed apart so that a **1024-byte gap** follows it (copied backwards from the end of the bank, one instrument at a time, relocating every affected offset in `LSP_InstrumentInfo` as it goes).
2. The gap is then **pre-filled with the sample's own loop section, repeated as many times as fits**.

The mixer can therefore overrun the nominal end of any sample by up to 1024 bytes and still read musically correct data. The end-of-sample test moves *out* of the inner loop and happens once per channel per frame (`test_depassement_canal_*`), where the accumulated overrun is reduced modulo the loop length by repeated subtraction and folded back into the fixed-point position.

Cost: `ecart_sample * 31` = 31 744 bytes are reserved unconditionally at `LSPBank_end`, regardless of how many instruments the module actually has (SOTB has six). The gap must also be ≥ the largest possible single-frame advance: 416 output samples at up to ~170× playback ratio is far more than 1024 bytes, so an extreme pitch on a short sample could still run past the gap.

The author's own `; bug !!!!!!!!!!!!!!!!!!!!!!!!!!!` marker sits at `fin_copie_repetition_sample_dans_ecart` (`lsp.asm:2436`), above a commented-out `str R6,[R11,#8]` — the repeat-offset writeback that step 4 partially compensates for.

### 4.6 The tick

`LSP_Player_standard` (`lsp.asm:1249`) is a faithful ARM transliteration of `LSP_MusicPlayTick`. The 68k `add.b d0,d0 / bcc` chain — shift the command word left and branch on the carry — has no ARM equivalent, so it becomes a walking mask:

```arm
mov  R13,#0b1000000000000000
tst  R10,R13
beq  <skip>
...
mov  R13,R13,lsr #1        ; next bit
```

Sixteen of these, in the same order as the original. Register roles: R0 = current stream pointer, R1 = `LSPVars`, R6 = virtual Paula base, R10 = command word, R13 = walking mask, R11 = DMACON nibble.

Two ARM-specific details are worth calling out:

**Unaligned halfword reads.** Periods and instrument deltas are read with `ldr R8,[R0],#2` followed by `mov R8,R8,lsl#16 / mov R8,R8,asr#16`. On 26-bit ARM, `LDR` from an address with A[1:0] = 2 returns the word **rotated right by 16**, so the wanted halfword lands in bits 0–15 either way. This is correct but depends entirely on ARM2/ARM3 rotate-on-unaligned-load semantics, and it reads up to two bytes past the end of the stream.

**Hardcoded escape codes.** The tick compares against literal `0xFFFF` (rewind) and `0xF00F` (set BPM):

```arm
mov R8,#0xFFFF
cmp R10,R8
beq r_rewind_Player_standard
```

The original reads these from `m_escCodeRewind` / `m_escCodeSetBpm` in the file header, because the encoder assigns them dynamically. See §4.9.

### 4.7 The mixer

`Paula_remplissage_DMA_416` (`lsp.asm:2989`) sets up, then jumps into `expanded_filling_routine` (`lsp.asm:4308`) — a `.rept nb_octets_par_vbl` block that is **fully unrolled**, 416 copies of a 24-instruction body. Per output frame it produces one 32-bit word containing one byte for each of the four channels:

```arm
ldrb  R0,[R1,R2,lsr #12]     ; fetch ch A at 20.12 fixed-point offset
add   R2,R2,R3               ; advance by increment
subs  R0,R0,R13,lsr#24       ; attenuate (log-domain subtract)
movmi R0,#0                  ; clamp to silence
mov   R7,R0,lsl #24          ; pack into lane 0
... (three more channels, ORRed into lanes 1..3) ...
str   R7,[R14],#nombre_de_voies
```

All four volumes are packed into the four bytes of R13, which is why channels B/C/D need an extra `mov R4,R13,lsl#n` to extract theirs. Every register including R13 and R14 is in use, so the routine is entered and left by `ldr pc` with the return address in R10 — not `bl`.

Because every register is consumed, there is no room for a loop counter, which is the real reason for the full unroll. At 20833 Hz that is 416 × 96 = **39 936 bytes of mixer**. At 62500 Hz it would be ~120 KB.

Pitch comes from `table_frequences_20833.s`: a period-indexed table of 20.12 increments, `increment = (PaulaClock / period) / replayRate × 4096`. Entry 1 is 697 348 ≈ 170.25 × 4096, consistent with 3 546 895 / 20 833.33. Periods 0 and 1 are clamped.

### 4.8 Hardware takeover and the RasterMan lineage

With `DEBUG=0`:

- `install_vsync` (`lsp.asm:3777`) claims FIQ from RISC OS via `OS_ServiceCall` 0x0C, saves the FIQ register bank and the IOC IRQ A/B masks, and overwrites the vector at 0x18.
- The handler body is a **separate assembly unit**, `fiqRM2.asm`, assembled `.org 0x18` to a raw binary and `.incbin`'d at `fiqbase` — hence the two-stage build.
- `remove_VBL` restores everything and releases FIQ with service call 0x0B.

`fiqRM2.asm` and `fiqRM.asm` are **direct derivatives of RasterMan's `fiq_code.asm`**. The comparison is unambiguous: identical instruction-by-instruction cycle/offset comments, the same `LDRB R8,[R14,#0x14+0]` with the same "***BUG to v0.13*** v0.14 read &14+0 was reading status at &10" annotation, the same `FIQ_notHSync` at &C0 and `FIQ_tempstack` at &D0, the same `vsyncreturn = 7168+16-1-48+7 = 7142` derivation. Only the trailing magic differs — RasterMan writes `"RstM"`, this writes `"rSTm"`.

`fiqRM.asm` retains RasterMan's per-scanline raster machinery (colour-0 / vstart / vend table driven by IOC Timer 1). `fiqRM2.asm` strips it to VSync-only and is what the player actually uses. `rmrebuild.asm` is a separate, standalone rebuild of the RasterMan module itself; it is not part of the player build and some of its inputs (`97,ffa`) are not in the repo.

### 4.9 Format version lock

`LSP_PlayerInit_standard` skips `LSP1`, the unique ID, and two version bytes, then reads BPM, then instrument count. Compare against the v1.26 layout in §2.2: modern files have a **flags/relocation word, three escape codes and a 4-byte frame count** between the version and the instrument count, plus a **`seqCount` word** before the stream sizes.

Hexdump of the bundled module confirms which format is in play:

```
SOTB - EXPLORATION.lsmusic:
4c 53 50 31  cd d0 a5 ef  01 03  00 7d  00 06  00 00 00 04 ...
'L  S  P  1' uniqueId     v1.03  bpm=125 instr=6  instrument[0]…

Example/rink-a-dink.lsmusic (LSPlayer repo, current):
4c 53 50 31  e6 58 e4 6f  01 17  00 01  00 7d  00 1d 00 1e 00 1f  00 00 25 35  00 1c ...
'L  S  P  1' uniqueId     v1.23  flags   bpm=125 esc codes ×3      frameCount   instr=28
```

So: **the port parses LSP format v1.03 and only v1.03.** The hardcoded `0xFFFF`/`0xF00F` escape codes are consistent with that era, when they were fixed constants rather than header fields. Feeding it a file from current LSPConvert will read the flags word as BPM, the BPM as the instrument count, and diverge from there.

### 4.10 The main loop

`boucle` (`lsp.asm:885`), once per VBL:

1. `LSP_Player_standard` — the tick.
2. `Paula_remplissage_DMA_416` — resample, attenuate and interleave 416 × 4 bytes.
3. Per-channel loop/end fixups.
4. `set_dma_dma1` — program MEMC `Sstart` (0x3680000) and `SendN` (0x36A0000) from the just-filled buffer's physical address.
5. `swap_pointeurs_dma_son` — flip to the other buffer.
6. `wait_VBL`, `scankeyboard`; space exits.

The DMA buffers are carved out of the screen dynamic area: `OS_ChangeDynamicArea` grows it by 16384 bytes and the buffers sit past `fin_de_la_memoire_video`, 8192 bytes apart. Physical addresses for MEMC come from `OS_FindMemMapEntries`.

The `SWI 22` / `MOVNV R0,R0` … `teqp r15,#0` pairs scattered through the loop are supervisor-mode entry/exit around border-colour writes to VIDC 0x3400000. They are **raster-time profiling markers**, not logic — they produce the coloured bands the banner text refers to.

### 4.11 Bugs and rough edges found

| Location | Issue |
|---|---|
| `lsp.asm:4372`, `:4395` | 8-voice mode cannot assemble: `LSPMusic_4a8voies` has **no** `.incbin` (all commented), and `LSPBank_voies_5a8` `.incbin`s `8canaux2.lsbank`, which is not in the repo. |
| `lsp.asm:1275,1279` | Escape codes hardcoded `0xFFFF`/`0xF00F` instead of read from the header. |
| `lsp.asm:1986` | Header parser assumes v1.03; no flags word, no escape codes, no frame count, no `seqCount`. |
| `lsp.asm:2828` | `set_my_stereos` loops `R0` = 1…8 indexing an **8-byte** table, so index 8 reads one byte past `mes_stereos` into `mes_stereos_8voies`. Works by luck. |
| `lsp.asm:4384` | `.skip ecart_sample*31` reserves 31 KB for 31 instruments regardless of the actual count. |
| `lsp.asm:2436` | Author-flagged `; bug !!!` — repeat-offset writeback commented out. |
| `lsp.asm:1395` etc. | `ldr` used for 16-bit reads; correct on ARM2/3 by rotate-on-unaligned, but reads past the stream end and would break on any core with alignment faults enabled. |
| `lsp.asm:85` | `FLAG_DMACON = 0`: DMACON channel enable/disable is ignored, so a note that Protracker would have silenced keeps looping in the gap instead. |
| `make.bat` | Calls `clear`; references `vectorball2.asm` (absent); copies to `C:\Users\Eric\…` and `C:\Archi\Arculator_V2.0_Windows\hostfs`. |
| everywhere | 8-voice support is hand-duplicated copy-paste (`_voies_5a8`), not parameterised. Every fix must be applied twice. |

---

## 5. Cost analysis

Approximate ARM2/ARM250 timings: data-processing 1 cycle (2 with a register-specified shift), `LDR`/`LDRB` 3, `STR`/`STRB` 2, taken branch 3.

**This port, per 4-channel output frame** (`expanded_filling_routine`, one `.rept` iteration):

| | cycles |
|---|---|
| channel A: `ldrb`,`add`,`subs`,`movmi`,`mov` | 3+1+1+1+1 = 7 |
| channels B, C, D: `ldrb`,`add`,`mov`,`subs`,`movmi`,`orr` | 3×8 = 24 |
| `str R7,[R14],#4` | 2 |
| loop overhead | 0 (fully unrolled) |
| **total** | **≈33** |

× 416 frames = **≈13 700 cycles per VBL**. On a 12 MHz ARM250 that is ≈1.1 ms, or **≈5.7 % of a 20 ms frame** — which matches the author's own on-screen figures ("5.1% to 7.7% / 16 to 24 pixels"), so the estimate is calibrated against his border-colour measurements.

**QTM, same workload** (`QTM_DMA.bas:261` `a3_anyvolume`, 8× unrolled, one channel at a time):

| | cycles |
|---|---|
| per byte: `ldrb`,`subs`,`movmi`,`strb`,`add` | 3+1+1+2+1 = 8 |
| loop overhead `cmp`+`blt` per 8 bytes | +0.5 |
| **per byte per channel** | **≈8.5** |
| × 4 channels | **≈34** |

Essentially identical. **The unrolled mixer buys nothing over QTM's 8× unroll** — the cycles LSP saves by doing one `str` instead of four are spent on the `mov`/`orr` lane-packing. And QTM has fast paths this port lacks:

- `a3_fullvolume` (`QTM_DMA.bas:307`) drops `subs`/`movmi`: 6 cycles/byte/channel → **≈24** per frame, a 30 % saving whenever a channel is at full volume.
- `a3_clearchannel` fills silent channels with a tight store loop rather than resampling silence.
- Channels that have genuinely ended are skipped entirely via `chanendflags`.

The port's advantage is elsewhere and it is real: **the tick**. QTM's `QTM_Play` is a complete Protracker interpreter running at ≥100 Hz — CIA emulation, per-channel effect dispatch, vibrato/tremolo/portamento tables, `set_setspeed`. This port's tick is a stream decode that early-outs in a dozen instructions on a no-change tick and never exceeds a few hundred instructions. That is where LSP's design pays.

---

## 6. LSP_ARM250 versus QTM

| | LSP_ARM250 | QTM v1.45 / v1.49b |
|---|---|---|
| Author, status | abandoned prototype (Dec 2021) | Steve Harrison, 1993–2023, actively maintained; v1.49b released 2023 |
| Form factor | bare-metal standalone binary at `&8000` | RISC OS relocatable module with a full SWI API |
| Coexists with RISC OS | no (`DEBUG=0`); optional with `DEBUG=1` | yes |
| Input format | pre-converted `.lsmusic` + `.lsbank` (PC toolchain required) | plays `.mod` files directly, filetype &CB6 |
| Formats | whatever LSPConvert accepts (Protracker `.mod`) | M.K., FLT4/FLT8, 4CHN/6CHN/8CHN, CD61/CD81 — 4/6/8-channel Protracker/FastTracker/StarTrekker |
| Channels | 4 (8-voice path broken) | 4, 6, 8 |
| Effect engine | none at runtime — pre-baked offline | full Protracker interpreter at runtime |
| Runtime song control | none (no seek, no pattern jump, no per-channel mute) | position/pattern/tempo/volume control, VU levels, playing-time scan, jukebox |
| Rate | 20833 / 31250 / 62500 Hz, assembly-time constant | 48 µs / 24 µs (41.6 kHz) selectable at runtime |
| Resampling | nearest-neighbour, 20.12 fixed point | nearest-neighbour, 20.12 fixed point |
| Volume | log-domain subtract, even-only table | log-domain subtract, even-only table (identical technique) |
| Mixer shape | fully unrolled ~40 KB, all 4 lanes packed per word | 8× unrolled per channel, with full-volume / silent / ended fast paths |
| CPU tuning | one code path | separate ARM2 / ARM3 paths, cache detection via `MRC CP15`; v1.49b builds unrolled loops at runtime |
| Fill trigger | VSync, in the foreground, one ~1.1 ms burst | SoundDMA interrupt at ~100 Hz, in IRQ |
| Module size | 184 740-byte binary including music and 40 KB of unrolled mixer | 26 KB module (v1.49b; 63 KB at v1.45) |
| Portability | ARM2/ARM250, 26-bit only | Arthur 0.30 → 32-bit RISC OS 5, ARM2 → modern |
| Provenance | LSP © Arnaud Carré; FIQ code derived from RasterMan © Steve Harrison — neither credited in-repo | © Steve Harrison 1993–2013, source published with a licence |

### The RasterMan connection

RasterMan and QTM are designed to work together, and this is the context the project is really competing in. RasterMan's `rastersound` path (`rasterman.asm:1336`) takes over FIQ, then on VSync *calls QTM's DMA fill handler directly*: `QTM_Debug` hands back the handler address and QTM's R12, `QTM_DMABuffer` hands back the buffer addresses, and RasterMan then programs MEMC's `Sstart`/`SendN` itself (`rasterman.asm:1413`, byte-for-byte the same sequence as this repo's `set_dma_dma1`). This requires a "QTM SE" special-edition build.

So the de-facto Archimedes demo stack is **RasterMan for rasters + QTM SE for music**, and this project is an attempt to replace the QTM half with something cheaper — while borrowing RasterMan's FIQ core for the other half.

---

## 7. What the project actually proves

1. **The LSP data format ports cleanly.** The tick transliterates from 68k to ARM almost line-for-line, and the format's design (bitmapped command words, split byte/word streams, delta instrument indices) survives the move intact.
2. **Paula's semantics can be emulated at acceptable cost on VIDC** — because VIDC's multi-channel DMA does the "mixing" for free and its log DAC turns volume into a subtract.
3. **The saving is in the tick, not the mix.** Any Archimedes MOD player pays roughly the same ~30 cycles per 4-channel output frame; LSP's contribution is eliminating the interpreter that runs on top of that.
4. **Full unrolling is the wrong trade here.** 40 KB of mixer buys no measurable advantage over QTM's 8× unrolled loop, and blocks the higher replay rates it was presumably meant to enable.

---

## 8. Recommendations for taking this forward

### Tier 1 — make it buildable and reproducible

1. **Replace `make.bat`.** Two steps, relative paths, create `build/` first:
   ```
   vasmarm_std_win32.exe -L compileFIQ2.txt -linedebug -m250 -Fbin -o build\fiqrmi2.bin fiqRM2.asm
   vasmarm_std_win32.exe -L compiletest.txt  -linedebug -m250 -Fbin -opt-adr -o build\test.bin lsp.asm
   ```
   Make the emulator-copy step optional and parameterised.
2. **Add credits and licensing.** LSP is © Arnaud Carré (MIT-ish, see the LSPlayer repo); the FIQ handler is derived from RasterMan © Steve Harrison. Neither is currently acknowledged in this repo. This matters if anyone else is to build on it.
3. **Pin the toolchain.** Record which LSPConvert version produced the bundled `.lsmusic` (v1.03 format), or ship it.

### Tier 2 — unblock the format

4. **Rewrite `LSP_PlayerInit_standard` against the current header.** Parse the flags word, read `escRewind`/`escSetBpm`/`escGetPos` into `LSPVars` and compare against those instead of the hardcoded `0xFFFF`/`0xF00F`, skip `frameCount`, and skip `seqCount` (plus its 8-byte entries when set). This is perhaps 40 instructions of work and it unlocks every module anyone converts from now on, including `-shrink` (smaller banks) and the 1.26 loop-volume fix.
5. **Honour the relocation guard byte** so `MusicInit` is idempotent, as the original does.

### Tier 3 — fix or remove 8-voice

6. Either supply the missing `8canaux2.lsbank` / `8canaux2.lsmusic` and repair the path, **or** delete `_voies_5a8` entirely. Half of `lsp.asm` is a copy-paste duplicate that doubles the maintenance cost of every change and currently does not assemble.
7. If 8 voices are kept, **parameterise instead of duplicating**: vasm macros or `.rept` over a channel-base symbol. This is the single largest structural improvement available — the file would roughly halve.

### Tier 4 — make the mixer competitive

8. **Drop the full unroll to 8× or 16×**, matching QTM. Frees ~37 KB, costs ~0.5 cycles per byte, and makes 31250/62500 Hz practical rather than code-size-prohibitive.
9. **Add QTM's fast paths.** A full-volume variant (skip `subs`/`movmi`) is worth ~30 % on any channel at volume 64, and silent/ended channels should be memset rather than resampled. These are the cheapest real wins available.
10. **Compute the guard gap from the actual instrument count**, not a hardcoded 31, and size it from the maximum single-frame advance rather than a fixed 1024.
11. **Enable `FLAG_DMACON`** so channel stop/restart behaves like Protracker rather than relying on the gap to paper over it.

### Tier 5 — turn it into something reusable

This is the decision that determines whether the project has a future. Three viable shapes:

12. **A RasterMan-compatible sound handler.** RasterMan already calls an external DMA fill routine with a fixed register convention (R9 = param block, R10 = DMA end, R11 = channel stride, R12 = DMA start; see `rasterman_1` at `rasterman.asm:1385`). Exposing `Paula_remplissage_DMA_*` behind that convention would let LSP drop straight into the RasterMan slot QTM SE occupies today — the single highest-leverage change in this list, because it puts the player where Archimedes demos actually consume music.
13. **A RISC OS module** with `LSP_Load` / `LSP_Start` / `LSP_Stop` / `LSP_SetSampleSpeed` mirroring QTM's SWI shape, registering a channel handler through `Sound_Configure` and filling on the SoundDMA interrupt. More work, but it makes the player usable from BASIC and from applications, and removes the hardware takeover.
14. **A linkable object** (`-Felf` with `vlink`, both already in the RasterMan repo's `bin/`) for demo authors who want it inline. Lowest effort, narrowest audience.

### Tier 6 — worth investigating

15. **8-channel MODs.** VIDC gives eight hardware channels at the same per-channel rate; Paula gives four. A working `_voies_5a8` would let this play FLT8/8CHN modules that Paula can only manage by software-mixing pairs. That is a capability QTM has and Amiga LSP structurally cannot — a genuine reason for this port to exist rather than being a slower QTM.
16. **An ARM "insane" backend for LSPConvert.** The converter already emits a per-module generated 68k player; an ARM emitter would produce the same specialisation for this target. The data format already carries everything needed. This is where LSP's remaining headroom is.
17. **Move the fill off VSync.** Filling in one ~1.1 ms foreground burst per frame couples music to frame rate and wastes the FIQ that has already been claimed. Filling incrementally from the sound interrupt (as QTM does) or from the raster FIQ would decouple them and remove the tearing risk when a frame overruns.
18. **Measure rather than estimate.** The border-colour instrumentation is already in place; a `-DPROFILE` build that reports cycles for tick and mix separately would make every optimisation above verifiable instead of argued.

---

## 9. Question: could the LSP frontend replace QTM's MOD interpreter, keeping QTM's backend, to speed up 4-channel MODs on a stock 8 MHz ARM2?

Short answer: **no — that combination attacks roughly 1 % of the cost.** The reasoning matters more than the verdict, because it explains why LSP's headline result does not transfer to this platform at all.

### 9.1 Where the cycles actually are on ARM2

The DMA fill workload is **fixed by the hardware configuration**, not by the player: 4 channels × 20 833 Hz = **83 333 bytes/sec**, whatever sits on top of it.

QTM's ARM2 inner loop (`QTM_DMA.bas:731`, `a2_anyvolume`, unrolled 208× at assembly time via `FOR arm2dma=1 TO dma`):

```arm
LDRB  R6,[R0,R1,ASR#12]   ; 3
SUBS  R6,R6,R3            ; 1
MOVMI R6,#0               ; 1
STRB  R6,[R12],R11        ; 2
ADD   R1,R1,R2            ; 1     = 8 clocks per byte per channel
```

| | per second | share of 8 MHz ARM2 |
|---|---|---|
| **Mixer** — 83 333 bytes × 8 clocks | ≈670 k clocks | ≈8 % nominal; **≈10–15 %** allowing for ARM2+MEMC1 running ~4–4.7 MIPS, and more still in a high-bandwidth screen mode |
| **Interpreter** — ~50 ticks/sec (125 BPM, speed 6) × 400–2000 clocks/tick | ≈20–100 k clocks | **≈0.25–1.25 %** |

**Ratio: roughly 10:1 to 30:1 in favour of the mixer.**

These are derived from instruction counts and PRM timings, not measured. The *ordering* is safe; the magnitudes should be confirmed before acting on them (see §9.5).

### 9.2 Why LSP's win does not transfer

On the Amiga, Paula **is** the mixer — resampling, volume scaling and summing all happen in silicon. The interpreter is therefore essentially 100 % of the CPU cost of playing music, which is precisely why LSP's 8× reduction of the tick is transformative there (Protracker ~7 scanlines → LSP 0.83).

On the Archimedes, VIDC provides channel multiplexing and a log DAC but **no pitch, no volume, no looping**. The mixer is ~90–95 % of the cost. LSP optimises the part that is already nearly free on this platform.

Swapping QTM's interpreter for an LSP stream decoder would therefore save **at most ~1 % of ARM2**, while giving up:

- direct `.mod` loading (an offline PC conversion step per tune),
- 6- and 8-channel formats (FLT8, 8CHN, 6CHN, CD61/CD81),
- runtime position/pattern/tempo/volume control, VU levels, playing-time scan,
- and the maintained, module-shaped integration that makes QTM usable at all.

### 9.3 What QTM's backend already does

Worth establishing before optimising, because it is more thorough than its reputation suggests:

- **Fully unrolled mixers generated at assembly time**, with separate ARM2 and ARM3 paths and ARM3 cache detection via `MRC CP15`. (v1.49b moved unrolling to *runtime* construction — this is where 63 KB became 26 KB.)
- **`a2_fullvolume` / `a3_fullvolume`** drop the `SUBS`/`MOVMI` pair → 6 clocks/byte instead of 8, whenever a channel is at maximum volume.
- **`a2_clearchannel`** fills silent channels with a tight store loop rather than resampling silence.
- **`chanendflags`** skips genuinely-finished channels entirely.
- Log-domain volume subtract with an even-only table; precomputed period→increment tables (`arcperiods`); Amiga→Arc pitch conversion done once at note-set time.

For the straightforward algorithm, QTM's mixer is close to optimal. For reference, `LSP_ARM250`'s mixer costs ~33 clocks per 4-channel output frame against QTM's ~32 (§5).

### 9.4 Where the remaining headroom actually is

Ranked by expected magnitude. All of these need prototyping and measurement.

**1. Exploit increment < 1 — the largest opportunity.** MOD samples are typically 8–16 kHz source material played out at 20.8 kHz, so the resampling increment is usually below 1.0 and **the same source byte is fetched and re-scaled 2–3 times consecutively**. A DDA / conditional-load restructure fetches only when the index genuinely advances:

```arm
ADDS   R1,R1,R2        ; 1   accumulate; C set when the integer part advances
LDRCSB R6,[R0],#1      ; 3 when taken, 1 when skipped
...clamp...
STRB   R6,[R12],R11    ; 2
```

A skipped `LDRB` costs 1 clock instead of 3. The fiddly part is the flag interaction between the conditional load, the `SUBS`-based clamp and the accumulate — `SUBS` would need to not clobber the accumulate's carry, and `MOVMI` must not act on stale flags when the load is skipped. Solvable with a second register holding the last scaled value, at the cost of some of the saving.

**2. Lane-packed output with persistent pre-shifted lanes.** Hold each channel's scaled byte *already shifted into its output lane*, `ORR` four together, one `STR` per frame: 4 clocks for 4 bytes versus 8 for four `STRB`s. Combined with (1), a lane is only recomputed when that channel's index advances.

Note this is **not** what `LSP_ARM250` does — it recomputes every lane every frame and therefore breaks even (§5). The saving only materialises if the lane values persist across frames.

Register pressure is the blocker: 4 lanes + 4 accumulators + 4 increments + 4 base pointers + volumes + destination exceeds the register file. Needs volume packing (as LSP does) and probably FIQ-mode banked registers.

**3. Word fetch with shift extraction.** At small increments one `LDR` covers 4 source bytes, usable for ~10 output bytes; extraction via `AND`/`LSR` costs 2 clocks against 3 for `LDRB`.

**4. Screen mode / video bandwidth.** On ARM2 this may dominate every code-level tweak. MEMC gives video DMA priority, so the mixer's wall-clock cost is substantially worse in a 256-colour mode than at 2 bpp. If the objective is "more CPU left for the demo", the bandwidth budget is a bigger lever than the inner loop.

### 9.5 The one LSP idea that *does* help QTM's backend

Not the frontend — the **`ecart` guard gap** (§4.5, `lsp.asm:2311`).

Pre-pad every sample offline with a gap pre-filled with its own loop data. The mixer can then overrun the nominal sample end freely, which means:

- no end-of-sample test anywhere in the inner loop;
- **no per-fill split** — QTM currently dispatches through `a2_chanctrl` → `a2_endanyvol` / `a2_repanyvol` / `a2_clearchannel` / `a2_not_ending` and splits each fill around the sample boundary. With a guard gap, every fill becomes one straight unrolled block plus a single cheap modulo fixup per channel per fill.

On its own this is a per-fill saving, not a per-byte one, so the direct gain is modest. Its real value is as a **precondition for (1) and (2)**: a branch-free inner loop is what makes the DDA and lane-packing restructures tractable at all.

So the productive hybrid is *not* "LSP frontend + QTM backend". It is **QTM kept whole, plus an offline sample-preparation pass borrowed from LSP** — log conversion, guard-gap padding, loop pre-copy — which then unlocks a restructured inner loop.

### 9.6 Measure first

The instrumentation already exists in this repository: the border-colour markers around `LSP_Player_standard` and `Paula_remplissage_DMA_416` (§4.10). Placing the same markers around QTM's `a2_DMAsound` entry and its `CIAhandler` entry, then running under Arculator configured as an 8 MHz ARM2, would give the real split directly.

Practical note: the QTM v1.45 source is tokenised BBC BASIC V, so this means either working in a detokenised form and re-tokenising, or patching the assembled module.

**Conclusion:** the LSP frontend is the wrong lever for ARM2 performance. Keep QTM's interpreter, keep QTM's backend, and spend the effort on the resampling inner loop — with LSP's offline sample preparation as the enabler rather than the goal.

---

## 10. File map

| File | Role |
|---|---|
| `lsp.asm` | Everything: init, LSP tick, Paula model, mixer, DMA, FIQ install, and the embedded music data |
| `fiqRM2.asm` | VSync-only FIQ handler, `.org 0x18`; assembled to `build/fiqrmi2.bin` and `.incbin`'d by `lsp.asm`. Derived from RasterMan `fiq_code.asm` |
| `fiqRM.asm` | Fuller RasterMan-style FIQ with per-line colour/vstart/vend table via Timer 1. Not used by the player build |
| `rmrebuild.asm` | Standalone rebuild of the RasterMan module. Independent of the player; some inputs missing |
| `swis.h.asm` | RISC OS SWI numbers plus RasterMan (0x47E80+) and QTM (0x47E40+) SWI constants |
| `table_frequences_{20833,31250,62500}.s` | Period → 20.12 increment tables, one per replay rate |
| `SOTB - EXPLORATION.{lsmusic,lsbank}` | Bundled module, LSP format v1.03 |
| `vasmarm_std_win32.exe` | vasm 1.8k, ARM backend 0.4h, std syntax — the assembler |
| `make.bat` | Stale; see Tier 1 |
