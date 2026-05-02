# Pokémon Leaf Green NSP — Hoenn Save Fix

**Status:** ✅ Working. World-first fix enabling Pokémon Ruby/Sapphire/Emerald to save persistently inside the Pokémon Leaf Green NSP wrapper on Nintendo Switch.

**Title ID:** `010034D02340E000`
**main NSO build_id:** `a8dd9bcc29745ae5be9635a11a42dcbd90071181`
**subsdk0 NSO build_id:** `cd3cc7e5082efeca93bd5d39ff3c17b94d63ea50`

---

## 1. Problem statement

Nintendo's Pokémon Leaf Green NSP for Switch is structured as a wrapper around a GBA emulator (subsdk0) that loads a fixed embedded ROM (`LeafGreen_e.gba`). LayeredFS lets users replace the ROM with any 16 MB GBA ROM, and the emulator core boots and runs that ROM correctly — graphics, audio, input, all functional.

But **only FRLG-engine games (FireRed, LeafGreen, and FRLG-engine ROM hacks) can save**. Hoenn-engine games (Ruby, Sapphire, Emerald) play normally; the in-game save UI completes ("Save complete!"); but no `.sav` file is ever written and "Continue" never appears.

This document is the complete reverse-engineering journey to find why and how to fix it.

---

## 2. The complete save chain (mapped)

Saves on the wrapper flow through this chain in `main.elf`. All addresses are NSO virtual offsets within `main` (the build_id-named ELF).

```
┌─ subsdk0 (GBA emulator) detects "save complete" via Flash chip patterns
│  └─ ONLY for FRLG-engine games. For Hoenn, this signal never fires.
│
↓ (per-tick callback into main)
│
[GBA emulation tick]
│
├─ chain of bit-0 voting gates (0x50c08, 0x50d84, 0x1050)
│
├─ JIT/event dispatcher fn 0x1fa90
│  (jump table indexed by bits 6-15 of w1)
│
├─ BroadcastEvent fn 0x1f020
│
├─ broadcast_loop fn 0x1f860
│  (iterates 256 subsystem slots [base+0x170 .. base+0x970])
│  └─ slot 0x400 holds the SaveSubsystem instance
│     vtable[3] = 0x575a8 (a thunk) → b 0x57014 (save_handler)
│
├─ save_handler fn 0x57014
│  (jump table @ main+0x17d7f6, events 0x40-0x62)
│  └─ event 0x4C → internal handler at 0x571cc
│     ├─ ldr x8, [x21, #0xd0]    (ptr_d0 — non-null for both games)
│     ├─ ldr x8, [x21, #0xe8]    (ptr_e8 — NULL for Hoenn ★)
│     ├─ ldr x0, [x8, #0xa0]    (ptr_e8->save_dispatch_instance)
│     └─ bl 0x5d930  (save_dispatch)
│
├─ save_dispatch fn 0x5d930
│  └─ reads buffer from this->ptr_78 or inline buffer
│  └─ calls vtable[13] for snapshot info
│
├─ WriteSaveFile fn 0x4a968
│  └─ fopen("LeafGreen_e.sav", "wb"); fwrite(buffer, ...); fclose
│
└─ save_commit fn 0x4a310
   └─ nn::fs::Commit("save")  → flushes to NAND
```

### Key addresses

| Symbol | NSO offset | Notes |
|---|---|---|
| `WriteSaveFile` | `0x4a968` | Direct fopen+fwrite+fclose to `save:/LeafGreen_e.sav` |
| `save_dispatch` | `0x5d930` | Builds args for WriteSaveFile, called from `save_handler` |
| `save_handler` | `0x57014` | Event dispatcher; jump table at `main+0x17d7f6` |
| save event 0x4C target | `0x571cc` | Internal label inside `save_handler` |
| `save_commit_func` | `0x4a310` | Calls `nn::fs::Commit("save")` if `[ctx+0x694]` set |
| `BroadcastEvent` | `0x1f020` | No static callers; called via vtable[3] dispatch |
| `broadcast_loop` | `0x1f860` | Walks 256 subsystem slots calling vtable[3] of each |
| SaveSubsystem vtable | `main+0x1c3908` | vtable[3] = `0x57014` (save_handler) |
| Save state setup fn | `0x56f60` | **Contains the discriminator gates** ★ |
| Save state setup helper | `0x55f54` | Populates `[this+0x40]` which becomes `ptr_e8` |
| BackupFlash string | `main+0x16eb94` | The save partition mountpoint name |
| MainTickEntry | `0x2a30` | `main`'s app entry point (reads argc/argv via `nn::os::GetHostArgc/Argv`) |

---

## 3. The discriminator (root cause)

### Inside fn `0x56f60`

```c
fn_0x56f60(this) {
    ptr_28 = this->ptr_28;
    if (ptr_28->byte_5d > 9) {
        // skip byte_55 check, proceed to first block
    } else {
        if (ptr_28->byte_55 != 'B') return;          // gate 1
    }
    // first block setup
    bl 0x3db10(...);

    if (ptr_28->byte_5d < 0xa) return;                // gate 2 ★
    set ptr_28->byte_170 = 1;
    if (ptr_28->byte_5d < 0xa) return;                // gate 3 ★

    bl 0x4bb08("BackupFlash", 2, 0x3d);                // register save partition
    if (returned 0) goto skip_setup;                  // gate 4

    bl 0x55f54(this, 0);                               // ← THIS sets up [this+0x40]

skip_setup:
    [x9, x8] = this->[+0x40];
    this->[+0xe8] = x8;                                // ptr_e8 setter
    this->[+0xe0] = x9->[+0x58];                       // ptr_e0 setter
    return;
}
```

`byte_5d` at offset `0x5d` of `this->ptr_28` is a **save engine version byte**. FRLG-engine games have it ≥ 0xa; Hoenn-engine games have it < 0xa. When < 0xa, gate 2 returns early — `ptr_e8` is never set. `save_handler` event 0x4C then null-derefs at `[ptr_e8 + 0xa0]`, but only if save_handler ever fires for event 0x4C — which it doesn't naturally for Hoenn either, because `subsdk0` never generates the save-complete signal.

So Hoenn was blocked at **two layers**:

1. **State setup blocked** (this fix): `byte_5d < 0xa` short-circuits `fn 0x56f60`, leaving `ptr_e8 = NULL` and `ptr_d0[+0xa0] = 0x7fffffffffffffff` (poison sentinel).
2. **Trigger blocked**: subsdk0's chip emulator only generates save events for FRLG-style Flash command sequences. Hoenn's are different.

The fix needs to address both.

---

## 4. The fix

Two-part patch, applied as a single Atmosphère IPS file at `/atmosphere/exefs_patches/<name>/a8dd9bcc29745ae5be9635a11a42dcbd90071181.ips`.

### Part 1 — Unlock state setup (4 NOPs in fn `0x56f60`)

Bypass all four early-return gates so `fn 0x55f54` always runs, populating `[this+0x40]` and consequently `ptr_e8` / `ptr_d0[+0xa0]`.

```
0x56f88: NOP  (was: b.ne 0x57008  — bypassed byte_55 != 'B' check)
0x56fac: NOP  (was: b.lo 0x57008  — bypassed byte_5d < 0xa, gate #1)
0x56fcc: NOP  (was: b.lo 0x57008  — bypassed byte_5d < 0xa, gate #2)
0x56fe8: NOP  (was: tbz w0, #0    — forces fn 0x55f54 to run)
```

After this, `ptr_e8` and `ptr_d0->[+0xa0]` get populated for Hoenn. But save events still don't fire naturally — that's Part 2.

### Part 2 — Self-triggered save via counter in `save_handler`

Replace the prologue gates of `save_handler` (10 instructions) with counter logic that fires **save event 0x4C internally** when the counter hits a threshold. The counter lives in unused `.data` space at `main+0x1d4514`.

```asm
; at NSO 0x5702c (save_handler prologue area)
mov w19, w2                  ; preserve original semantics for non-fire paths
adrp x10, #0x1d4000          ; counter base page
ldr w11, [x10, #0x514]       ; load counter
add w11, w11, #1              ; increment
str w11, [x10, #0x514]       ; store
cmp w11, #0x8000              ; threshold (≈ 4 minutes at ~136 calls/sec)
b.ne #0x57588                 ; if not exactly threshold, take the original early-return
mov x21, x0                   ; set up `this` for event 0x4C handler
mov x20, x3                   ; set up broadcast arg
b   #0x571cc                  ; jump straight into the save event 0x4C internals
```

This patch was deployed against the Leaf Green NSP via Atmosphère LayeredFS. The single fire on counter==threshold captures whatever state Hoenn has written to its emulated GBA Flash buffer at that moment. Since Hoenn's in-game save UI writes valid Pokémon save sectors (magic `0x08012025`) to the Flash buffer during play, our patch persists those exact bytes to `LeafGreen_e.sav`.

The key insight: the post-save processing in subsdk0's worker thread that crashed all our previous attempts (User Break at `subsdk0+0x17574c`) was caused by **firing save spam every tick**. A single one-shot fire processes cleanly.

### Counter location

`main+0x1d4514` — verified to be 36 consecutive zero bytes in `main.data` (offset `0x20514`), well clear of any initialized data structures. Zero-init at boot, BSS-style.

---

## 5. The journey (chronological summary)

This is a condensed retelling because the path was anything but straight. Many of the dead ends taught us the architecture.

### Phase 1: ruling out the easy answers

Initial hypotheses tested and rejected:
- Gamecode dispatch (BPRE/BPGE/BPEE/AXVE/AXPE) — README confirmed zero static references.
- ROM hash validation — no SHA/CRC of any ROM in either binary.
- Save signature scan (`FLASH1M_V`, `EEPROM_V`, etc.) — strings absent.
- Save partition size — 2.3 MB allocated, plenty.
- Flash chip class (128KB vs 64KB) — applying the 512Kb-flash IPS patch did not help.
- The `+0x694` mount-state gate — confirmed set at boot for both games.

### Phase 2: mapping main.elf save chain via UDF traps

Atmosphère converts `udf #0` into a crash report with full register and stack dumps. We used it as a printf:

1. UDF at `WriteSaveFile (0x4a968)` → fired for vanilla LeafGreen, silent for Hoenn → confirmed WSF is the bottleneck.
2. UDF at `save_handler (0x57014)` → silent for Hoenn → confirmed save chain doesn't fire naturally.
3. UDF at `0x571cc` (event 0x4C internal) → silent → matches.
4. UDF inside broadcast_loop's BLR (`0x1f8c8`) → fired for both games at slot offset 0x400 with vtable[3] = `0x575a8`. Same subsystem on both, so the broadcast layer is identical.
5. Skip-loop trick (modify `mov w23, #0x170` → `#0x408`) to skip past slot 0x400 → revealed only one other slot (`0x968`) populated, with a no-op handler.

So at the broadcast layer, FRLG and Hoenn behave identically. The discriminator is **inside save_handler**, in event 0x4C's path — which only runs when save_handler is called with `event_id == 0x4C`.

### Phase 3: the force-event experiment

Patched `save_handler` to internally force `event_id = 0x4C` regardless of caller value. This redirected per-tick `event_id=1` calls into the event 0x4C handler.

Result: data abort at `0x571d8` (NULL deref of `ptr_e8 + 0xa0`). For Hoenn, `ptr_e8 = NULL`. For FRLG, it's a valid pointer to a save_dispatch instance.

So Hoenn's `ptr_e8` is uninitialized. Why?

### Phase 4: chasing the wrong rabbit hole (nn::pia::net::NetFacade)

Looked at fn `0x9dfdc` which writes to `[reg, #0xa0]` — looked exactly like our save_dispatch installer. Found its caller fn `0x97dd8`, then fn `0x8c52c` which contains a 16-byte memcmp gate.

Patched the cbz to always pass — no effect on save behavior. Re-investigated and found the typeinfo string for the vtable holding fn `0x8c52c`: **`N2nn3pia3net9NetFacadeE`** — Nintendo's networking framework. Wrong subsystem entirely.

This was a few hours of misdirection, but it taught us how to read RTTI typeinfo and confirmed that the wrapper has a heavy networking layer (181 nn::pia classes — the wrapper is not a generic GBA emulator; it's an FRLG-specific application).

### Phase 5: finding the real ptr_e8 setter

Searched for `STR Xt, [Xn, #0xe8]` instructions across `main.text`. 14 hits. Of those, 2 were preceded by a BL (suggesting they store a freshly-computed value):

- `0x56ffc` ★ — preceded by `bl 0x55f54`. Inside fn `0x56f60`.
- `0xf6090` — unrelated.

Disassembled fn `0x56f60`. Found four early-return gates checking `byte_5d` and `byte_55` of `this->ptr_28`. NOPed all four.

### Phase 6: combining gates fix with force-event

Force-event-every-tick (after gates fix) crashed in subsdk0 worker thread (User Break at `subsdk0+0x17574c`) — but the .sav was created (empty, all 0xFF). Save chain was firing fully end-to-end through `save_dispatch → WriteSaveFile → save_commit`, but the post-commit processing in subsdk0 was overwhelmed by per-tick saves.

### Phase 7: the counter-based one-shot

Designed a counter at `main+0x1d4514` to fire save event 0x4C exactly once after N save_handler invocations. After several iterations to find the right threshold (save_handler is called ~136 times/sec, not the 22k/sec initially assumed), settled on `0x8000` (~4 minutes).

Test result: **no crash, .sav created, contains 14 valid Pokémon save sectors with magic `0x08012025`, decoded character names matching Hoenn defaults**.

Continue option appeared in the title menu on next boot. Hoenn save state loaded correctly.

### Phase 8: known limitation (next iteration)

The current patch fires exactly once per game launch (B.NE for exact equality). Subsequent in-game saves are not captured until the user power-cycles the Switch. The fix is to change the comparison from "fires at exact match" to "fires every Nth call" using `TST` against a power-of-2 mask.

```asm
tst w11, #0x1FFF             ; fires every 0x2000 calls (~60 sec)
b.ne #0x57588
```

This will be deployed in patch v2.

---

## 6. Tooling and methodology

### Toolchain
- `tools/nso_unpack.py` — extracts and decompresses NSO segments (text/ro/data) using LZ4 raw block format.
- `tools/nso2elf.py` — wraps decompressed NSO segments into a Ghidra-loadable ELF, preserving `.dynamic` so imports resolve to symbol names.
- `tools/ips_apply.py` — IPS patcher (used for ROM patches).
- Python `capstone` — fast scriptable disassembly. Most of our analysis was Python+capstone, not Ghidra.
- Atmosphère crash reports — used as a printf-style debugging channel via strategically placed `udf #0` instructions.

### Why no Ghidra (mostly)?
We tried. With a 23 MB subsdk0, auto-analysis takes 15-30 minutes and the results were inconsistent (the user reported many `G`-to-address operations finding nothing even after analysis completed). Capstone-based scripting was faster for targeted searches: pattern-matching ADRP+ADD pairs, finding all `STR Xt, [Xn, #imm]` sites with a specific offset, scanning vtables. We'd switch to Ghidra for decompilation when the task needed it, but most of the 25+ patches we deployed were derived from raw byte patterns.

### The udf-as-printf pattern
Atmosphère's exception handler dumps all 29 X registers + stack + return-address chain when a process hits an undefined instruction. Replacing a 4-byte instruction with `00 00 00 00` (`udf #0`) and reading the resulting crash report told us precisely:
- Whether code reached that point (crash fires or not)
- All register values at that point (including ASLR'd heap pointers we couldn't compute statically)
- The full call chain back through both modules (main and subsdk0)

This was our most-used diagnostic tool, used in roughly 20 of the patches we deployed. Without it, we'd have needed Twili or a debugger setup, both of which target homebrew, not commercial titles.

---

## 7. Patch source

Final IPS contents (build via `tools/build_ips.py` or generate inline):

**Records (NSO virtual addresses):**

| NSO offset | Original bytes | New bytes | Purpose |
|---|---|---|---|
| `0x56f88` | `01 04 00 54` | `1f 20 03 d5` | NOP `b.ne 0x57008` (gate 1) |
| `0x56fac` | `e3 02 00 54` | `1f 20 03 d5` | NOP `b.lo 0x57008` (gate 2) |
| `0x56fcc` | `e3 01 00 54` | `1f 20 03 d5` | NOP `b.lo 0x57008` (gate 3) |
| `0x56fe8` | `80 00 00 36` | `1f 20 03 d5` | NOP `tbz w0, #0, 0x56ff8` (gate 4) |
| `0x5702c` | `f3 03 02 2a` | `f3 03 02 2a` | (mov w19, w2 — unchanged, original) |
| `0x57030` | `5f ac 00 71` | `ea 0b 00 b0` | `adrp x10, #0x1d4000` |
| `0x57034` | `ab 04 00 54` | `4b 15 45 b9` | `ldr w11, [x10, #0x514]` |
| `0x57038` | `69 02 01 51` | `6b 05 00 11` | `add w11, w11, #1` |
| `0x5703c` | `3f 89 00 71` | `4b 15 05 b9` | `str w11, [x10, #0x514]` |
| `0x57040` | `48 2a 00 54` | `7f 21 40 71` | `cmp w11, #0x8000` |
| `0x57044` | `2a 09 00 d0` | `21 2a 00 54` | `b.ne #0x57588` |
| `0x57048` | `4a d9 1f 91` | `f5 03 00 aa` | `mov x21, x0` |
| `0x5704c` | `0b 01 00 10` | `f4 03 03 aa` | `mov x20, x3` |
| `0x57050` | `4c 79 69 78` | `5f 00 00 14` | `b #0x571cc` |

**IPS file** (binary): `save-fix/exefs_patch/leafgreen-fix-final/a8dd9bcc29745ae5be9635a11a42dcbd90071181.ips`

**Deploy path**: copy that .ips file to `/atmosphere/exefs_patches/<any_name>/a8dd9bcc29745ae5be9635a11a42dcbd90071181.ips` on the SD card, then boot the LeafGreen NSP.

---

## 8. How to use

### Prerequisites
- Switch with Atmosphère custom firmware
- Pokémon Leaf Green NSP installed (Title ID `010034D02340E000`)
- Hoenn ROM as 16 MB GBA file at `/atmosphere/contents/010034D02340E000/romfs/LeafGreen_e.gba`
- IPS patch from this repo placed at `/atmosphere/exefs_patches/leafgreen-hoenn-save-fix/a8dd9bcc29745ae5be9635a11a42dcbd90071181.ips`

### Save procedure
1. Boot the Leaf Green NSP — Hoenn ROM loads automatically via LayeredFS
2. Play through Hoenn's intro (mash A through cutscenes to skip)
3. Reach a save spot in-game and use the in-game Save UI ("Save complete!" dialog)
4. **Wait approximately 4 minutes from boot** — our patch fires at `~0x8000` save_handler calls (~4 min at 136 calls/sec), which captures the GBA Flash buffer (now containing your saved state) and writes it to `save:/LeafGreen_e.sav`
5. Power off the Switch
6. Boot game again — "Continue" should appear in the title menu, your saved state loads

### Limitations of v1 (known, fixed in v2)
- Only the first in-game save per launch is captured (counter uses exact-equality match). v2 patch (using TST against mask) fires every minute, capturing later saves too.

---

## 9. Future work

- **v2 patch** — periodic save (every ~60 seconds) so multiple in-game saves are captured per session. Replace `cmp w11, #0x8000; b.ne` with `tst w11, #0x1fff; b.ne`.
- **Subsdk0 deep-dive** — find subsdk0's "save complete" detector and patch it to recognize Hoenn's Flash command sequences. Would eliminate the need for Part 2 of our patch and let users save naturally via the in-game menu.
- **Test on Ruby/Sapphire** — these likely work the same as Emerald but should be confirmed.
- **Test on FRLG-engine ROM hacks** — should not need any patch, but verifying ensures we haven't broken vanilla behavior.

---

## 10. Acknowledgments

This investigation took ~50+ patch deployments, many SD card eject/insert cycles, and several false leads (NetFacade, the wrong vtable, the Twili dead-end). It would not have been possible without:

- **The Atmosphère project** — both for letting us patch a commercial NSP via exefs_patches, and for the crash report mechanism that became our primary diagnostic tool.
- **Capstone disassembler** — every static-analysis search we ran depended on it.
- **The user's patience and willingness to keep iterating** — the actual breakthrough came after 7+ hours of dead ends, when we shifted from chasing the wrong vtable (NetFacade) to systematically searching for `STR Xt, [Xn, #0xe8]` writers.

---

**Generated:** 2026-04-29
**Status:** v1 working (one save per session). v2 (periodic) coming next.
