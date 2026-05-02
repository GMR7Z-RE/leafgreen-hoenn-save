# Subsdk0 Deep Dive — Toward v3c Native Save Detection

Investigation log for finding the proper architectural fix for Hoenn saves: making subsdk0 fire `event_id = 0x4C` natively for Hoenn games (eliminating the need for the timer workaround).

---

## Key discovery — the cross-module bridge

We've located the **exact code in subsdk0 that fires `event_id = 0x4C`** to main. It's a callback dispatch pattern:

```asm
ldr x0, [x_obj, #0x10]    ; x0 = main's callback object (set up at init)
mov w1, #0x4C              ; w1 = save event ID
ldr x8, [x0]                ; x8 = vtable
ldr x8, [x8]                ; x8 = vtable[0]
blr x8                       ; call into main's BroadcastEvent equivalent
```

**Six fire sites in subsdk0** all do this:
- `0x22e4a4`
- `0x22e5b8`
- `0x22e718`
- `0x22e844`
- `0x22e9c8`
- `0x22eb28`

The first 5 are inside `fn 0x22e2ac`. The 6th is inside `fn 0x22eb08`. Both are consecutive vtable entries:

```
subsdk0+0x146db08: fn 0x22e2ac  ← contains fire sites 1-5
subsdk0+0x146db10: fn 0x22eb08  ← contains fire site 6
```

These are **HLE (high-level emulation) instruction handlers** — slots in a per-IR-opcode handler vtable. The JIT dispatches to whichever slot matches the IR opcode being compiled.

## What fn 0x22e2ac actually does

Disassembled and decoded:

```c
fn_0x22e2ac(this, ins_ctx) {
    // ins_ctx fields seen accessed:
    // [+0x58] = instruction opcode kind (compared against 0x12, 0x75, 0xd0, 0xd1, 0x117)
    // [+0x60] = some operand index/offset
    // [+0x64] = operand attribute bits (masks against 0x70000000, 0x60000000, etc.)
    // [+0x68] = secondary operand attribute
    // [+0x6c] = operand 1 info
    // [+0x70] = operand 2 info
    // [+0x73] = flags byte
    // [+0x74] = operand 3 info
    
    // The function evaluates the IR instruction's operand kinds + attributes
    // and decides whether to fire event 0x4C.
    
    // 6 different conditional paths each end in:
    callback = this->ptr_10;
    callback->vtable[0](callback, 0x4C);   // ← FIRE SAVE EVENT
    
    // Then additional state-update events:
    callback->vtable[0x348/8](callback);     // = vtable[105]
    callback->vtable[0xe58/8](callback, 0/1); // = vtable[459]
    // ... and more vtable methods on the callback
}
```

Each of the 6 fire paths corresponds to a different combination of operand kinds:
- Operand attribute mask `0x70000000` checks (operand class 1, 2, 3 ranges)
- Comparisons against constants `0x12, 0x75, 0xd0, 0xd1, 0x117` (IR opcode codes)
- Bit checks on `0x6c, 0x70, 0x74` operand info fields

## What this means for our v3c fix

**Hypothesis**: subsdk0 has HLE detection that recognizes FRLG's save routine machine code during JIT compilation. The detection is pattern-matching on IR operand combinations specific to FRLG's save engine. Hoenn's save routine, written in different code, produces different IR operand combinations that don't match any of fn 0x22e2ac's 6 fire paths.

**To make Hoenn save natively**, we have a few options:

1. **Force fn 0x22e2ac to ALWAYS fire**. If fn 0x22e2ac is called even for Hoenn (just with mismatched conditions), we can patch it to unconditionally take one of the fire paths. Risk: false fires for non-save instructions.

2. **Patch fn 0x22e2ac's conditions to be more permissive**. NOP some `b.ne`/`b.hi` gates to widen acceptance.

3. **Call fn 0x22e2ac externally**. If Hoenn doesn't even trigger the JIT to dispatch to this vtable slot, no patch inside fn 0x22e2ac will help — we'd need to wire up a separate path.

## Open questions

1. Is fn 0x22e2ac even *called* for Hoenn? (The vtable dispatch goes through specific IR opcodes; Hoenn might never produce those.)
2. If called, which condition path fails for Hoenn? (Different operand attribute mask? Different opcode constant?)
3. How is the JIT triggered to dispatch THIS vtable slot vs others? (Is there an upstream IR-translation step that picks the slot?)

## Next steps

### Immediate test (low cost)
Deploy a UDF `#0` trap at subsdk0+0x22e2ac entry. If Hoenn boots and the game crashes, fn 0x22e2ac IS called for Hoenn — meaning patching its internal conditions is viable. If it doesn't crash, the function isn't reached, and we need to investigate the JIT IR pipeline upstream.

Subsdk0 build_id: `cd3cc7e5082efeca93bd5d39ff3c17b94d63ea50`
IPS path: `/atmosphere/exefs_patches/<any_name>/cd3cc7e5082efeca93bd5d39ff3c17b94d63ea50.ips`
IPS contents (NSO offset 0x22e2ac → file offset 0x22e2ac + 0x100 = 0x22e3ac):
```
PATCH
22 e3 ac      ; offset (3-byte BE)
00 04         ; length
00 00 00 00   ; UDF #0
EOF
```

### If trap fires (fn called for Hoenn)
- Sub-trap at each of the 6 condition gates inside fn 0x22e2ac
- Identify which `b.eq`/`b.ne`/`b.hi` is taken for Hoenn
- That branch is the discriminator at the JIT IR layer
- Patch it permissively

### If trap doesn't fire (fn never called for Hoenn)
- The JIT pipeline doesn't dispatch to this vtable for Hoenn
- Need to investigate what IR opcodes Hoenn produces vs FRLG
- This means tracing through subsdk0's IR translator (likely megafunctions in `0xb9070`+ region with the `116dc48` jump table we saw)
- Days of additional RE

## Architecture notes (reusable for future RE)

### subsdk0 cross-module communication
subsdk0 doesn't import any `nn::os::*` (no SystemEvent, no svc-based sync). The bridge is **callback function pointers** set up at init time.

Subsdk0's `_init` at `0x10` reads ~100 callback function pointers from `subsdk0+0x15c4xxx` area (BSS) — these are set by main during `nn::ro::LoadModule` or equivalent. After init, subsdk0 calls main by:
- `ldr x0, [<some_state_obj>, #0x10]` → callback dispatcher object (set at init)
- `ldr x8, [x0]` → vtable
- `ldr x8, [x8, #N]` → vtable[N/8]
- `blr x8` → call

The dispatcher object's vtable[0] is the "broadcast event" entry, called with `(this, event_id)`. main's wrapper code maps `event_id` to its broadcast machinery → broadcast_loop → save_handler → save_dispatch → WriteSaveFile.

### Vtable layouts found
- IR instruction handler table at subsdk0+0x146da... (slot per IR opcode)
- Chip database at subsdk0+0x145ffc0 (98 chips × 40 bytes, 3 fn ptrs each at +0x10/+0x18/+0x20)
- FlashEmulator instance at runtime address subsdk0+0x15d6d18 (constructor at fn 0xd600)

### Discovered functions
| Address | Notes |
|---|---|
| `subsdk0+0xd600` | FlashEmulator constructor; sets fields at 97 distinct offsets |
| `subsdk0+0x83b70` | Shared chip dispatcher (counter-based gate) |
| `subsdk0+0xb9000` / `0xb9070` | Per-byte chip command handler (state machine) |
| `subsdk0+0x14ea40` | Chip[0]'s dedicated handler (loops calling 0xb9000) |
| `subsdk0+0x14f020` | Chip[1] and chip[27]'s handler |
| `subsdk0+0x22e2ac` | **HLE save event dispatcher (5 fire sites) ★** |
| `subsdk0+0x22eb08` | **HLE save event dispatcher (1 fire site) ★** |
| `subsdk0+0x110f30` | Cross-module BL site to main+0x2a30 (MainTickEntry) |

---

**Status**: Investigation in progress. Major finding (the actual bridge) located. Next step is the trap test to determine if patching the existing function vs. wiring up a new path is needed.

**Generated**: 2026-05-01
