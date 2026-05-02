# Why FRLG Saves Instantly, But Hoenn Needs a Timer

A focused note on the architectural reason my v2 Hoenn save patch is a *workaround*, not a true fix — and what a true fix would look like.

---

## The observation

In vanilla LeafGreen on the wrapper:
- Press Save in-game → "Save complete!" → leave the game (close, power off, sleep, etc.) → boot → **Continue is there, instantly**.

With my v2 patch on Hoenn (Emerald via LayeredFS):
- Press Save in-game → "Save complete!" → leave the game **immediately** → boot → **Continue might NOT be there**.
- Press Save in-game → wait up to 60 seconds → leave the game → boot → **Continue is there**.

This is unusual. Why does the FRLG case feel "instant" while the Hoenn case feels "delayed"?

---

## The natural FRLG save chain (no patches needed)

When you press Save in vanilla FireRed/LeafGreen:

1. **FRLG game code** writes Pokémon save sectors to GBA Flash address space (`0x0E000000+`) using a specific Macronix-style command sequence:
   - Erase sector: `0xAA → 0x5555`, `0x55 → 0x2AAA`, `0x80 → 0x5555`, `0xAA → 0x5555`, `0x55 → 0x2AAA`, `0x30 → sector_addr`
   - Write byte: `0xAA → 0x5555`, `0x55 → 0x2AAA`, `0xA0 → 0x5555`, `byte → addr`
   - Repeat for all 14 sectors of save data
2. **subsdk0's GBA Flash chip emulator** intercepts these writes (since `0x0E000000+` is mem-mapped Flash). Its internal state machine recognizes the pattern as "save in progress → save complete".
3. When the state machine reaches "save complete", subsdk0 sets a flag in **shared memory** (allocated by main, given to subsdk0 at init time).
4. Main's per-tick callback code (the one called by subsdk0+0x110f30 → main+0x2a30) polls that flag every game tick.
5. When main sees the flag, it generates a `BroadcastEvent` with `event_id = 0x4C`.
6. `BroadcastEvent` (main+0x1f020) walks the 256-slot subsystem table.
7. Slot 0x400 holds the SaveSubsystem instance; its `vtable[3]` is `0x575a8` (a 2-instruction thunk → `b 0x57014`).
8. `save_handler` (main+0x57014) dispatches event 0x4C internally to its handler at `0x571cc`.
9. That handler reads `ptr_e8`, derefs to get the save_dispatch instance, calls:
   - `save_dispatch (main+0x5d930)` → reads buffer pointer, builds args
   - `WriteSaveFile (main+0x4a968)` → `fopen("LeafGreen_e.sav", "wb"); fwrite(buffer, 128KB, ...); fclose`
   - `save_commit (main+0x4a310)` → `nn::fs::Commit("save")` → flush to NAND
10. `.sav` is on the save partition. Power-off doesn't lose it because Switch's `nn::fs::Commit` is durable.

**Total latency: a few hundred ms after "Save complete!" UI message. Effectively instant.**

---

## Why this fails for Hoenn games

Hoenn games (Ruby, Sapphire, Emerald) write to the same mem-mapped Flash region but use a **different chip's command sequence** (Sanyo LE26FV10N1TS-10, with `FLASH1M_V102/103` save type):

- Different chip ID returned to JEDEC ID query
- Slightly different timing/ordering on writes
- Different "ready/busy" polling pattern

subsdk0's chip state machine watches for one specific pattern (FRLG's). When Hoenn writes its sectors:

- The Flash buffer in subsdk0's heap **does** get the writes (data is correct in emulated memory)
- But the chip state machine **never reaches "save complete"** because Hoenn's command sequence doesn't match
- The shared-memory flag stays cleared
- Main's per-tick polling never sees a save event
- `BroadcastEvent` is never invoked with `event_id = 0x4C`
- `save_handler` is called per-frame with `event_id = 1` (the generic per-tick event), which immediately exits via the `b.lo 0x57588` early-return inside save_handler
- The Hoenn save data sits in subsdk0's Flash buffer forever, never written to `.sav`
- Leaving the game (closing it, sleeping the console in a way that ends the process, or powering off) → buffer gone → next boot, no save data, no Continue

This was confirmed empirically by trapping at the save chain entry points: with vanilla LeafGreen + a UDF trap at `WriteSaveFile`, the trap fires on save-press. With Emerald via LayeredFS + the same UDF trap, it stays silent forever — even when "Save complete!" is displayed in-game.

The Hoenn save UI is a lie from main's perspective: from the Hoenn game's view, Flash was written successfully (because subsdk0 honors the writes); from main/subsdk0's bridge view, no save event ever fired.

---

## Two layers of blockage

For Hoenn to save, **two separate things must happen** that don't naturally for non-FRLG games:

### Layer 1: state setup
The save subsystem instance has a field `ptr_e8` (and `ptr_d0->[+0xa0]`) that must point to a valid save_dispatch context. For FRLG-engine games, `fn 0x56f60` populates these via `fn 0x55f54`. For Hoenn, four early-return gates check `byte_5d ≥ 0xa` (a "save engine version" byte) and `byte_55 == 'B'`, all of which fail for Hoenn → setup short-circuited → `ptr_e8 = NULL`, `ptr_d0[+0xa0] = 0x7fffffffffffffff` (poison sentinel).

### Layer 2: save trigger event
Even with state set up, there's no `event_id = 0x4C` event being broadcast for Hoenn (because subsdk0's chip state machine doesn't fire its FRLG-pattern detection). So `save_handler` would just keep returning early on per-tick events.

**My v1/v2 patch addresses both layers**:
- 4 NOPs in `fn 0x56f60` bypass the gates → ptr_e8 / ptr_d0[+0xa0] get populated even for Hoenn (Layer 1 fixed)
- Counter logic in `save_handler` artificially forces `event_id = 0x4C` periodically → save chain fires (Layer 2 fixed via timer, not subsdk0 detection)

---

## Why my v2 patch is a workaround, not a true fix

The truly elegant fix would be at **Layer 2 in subsdk0**: modify subsdk0's chip state machine to also recognize Hoenn's Sanyo-style command pattern as "save complete". One small patch (perhaps 1-2 instructions) in the right location of subsdk0's chip emulator would give:

- **Instant saves** (driven by actual save events, just like FRLG)
- **Zero risk of save loss** from exit timing — leave the game whenever you want
- **Zero changes to save_handler** (the timer in v2 is no longer needed)

My v2 patch instead artificially fires the save chain **every ~60 seconds** by hijacking `save_handler`'s prologue. This works because:

- Whatever's in subsdk0's Flash buffer at the moment the patch fires is what gets written to `.sav`
- After a player presses Save in-game, Flash buffer contains valid save data
- Within 60 seconds, the timer fires, captures Flash → writes `.sav` → commits to NAND

### Trade-offs of the workaround

**Pros:**
- Works for ANY game with any save command pattern (Hoenn, Mother 3, Mystery Dungeon, Pokémon Pinball — anything that writes valid GBA Flash data)
- Doesn't require deep RE of subsdk0's 18 MB chip state machine
- Single IPS file in `main.elf`, no subsdk0 modification needed

**Cons:**
- **60-second granularity**: if you save in-game and leave the game (close, switch titles, sleep, or power off) within 60 sec, the save is lost
- **Wasted work**: even if you didn't save in-game, the patch fires the full WriteSaveFile + Commit cycle every minute, writing 128 KB to NAND. Minor wear over long sessions.
- **Not "feel-instant"**: FRLG saves are sub-second; Hoenn saves under this patch have up-to-60-second latency

---

## What a "true" fix would require

To find subsdk0's FRLG-pattern detection and patch it to also accept Hoenn patterns, I'd need:

1. **Identify the chip state machine** in subsdk0. I've already mapped:
   - FlashEmulator instance at heap address `0x15d6d18`
   - Constructor at subsdk0+`0xd600`
   - 98 chip definitions (40 bytes each) starting at subsdk0+`0x145ffc0` in `.data`
   - Per-chip dispatcher pattern: shared front-end at `0x83b70`, chip-specific tail call (e.g., `0x14ea40` for chip[0])
   - Per-byte chip command handler at `0xb9000` → `0xb9070`

2. **Trace from the per-byte handler to the "save complete" signaler**. The signaler must:
   - Detect a sequence-end pattern (Hoenn: typically `0x10` to `0x5555` for chip-erase complete, or completion of all 14 sector writes)
   - Set a flag in shared memory that main polls

3. **Patch the detector to also recognize Hoenn's pattern**. This could be:
   - Loosening the pattern match (e.g., accepting both Macronix and Sanyo command IDs)
   - Adding a parallel detector for Hoenn's specific sequence

This is meaningful work — subsdk0 is 18 MB of x64-decompiled-then-recompiled-as-AArch64 emulator code with no symbols, no RTTI for save-related classes, and 8493 indirect function calls through 849 distinct callback addresses. Realistically a few weeks of focused Ghidra-driven analysis.

The v2 timer workaround was the pragmatic choice. Whether it's *enough* depends on how you feel about the trade-offs.

---

## Decision matrix

| Approach | Save latency | Save reliability | Work needed | Risk |
|---|---|---|---|---|
| **v2 timer (current)** | Up to 60 sec | High (very rare edge case) | ✅ Done | Low |
| **v2 with shorter interval (10 sec)** | Up to 10 sec | High | Trivial (change one immediate) | Low |
| **subsdk0 Hoenn detection patch** | Instant (FRLG-equivalent) | Maximum | Significant (multi-day RE) | Medium |
| **Live with v2** | 60 sec | High | None | None |

---

**Generated:** 2026-04-29
**Status:** Architectural note explaining why v2 is a workaround. Decision pending on whether to invest in subsdk0 deep-dive for the "true fix".
