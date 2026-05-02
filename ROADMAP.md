# Pokémon Leaf Green NSP — Hoenn Compatibility Project Roadmap

Tracking improvements, next steps, and feasibility analysis for the project that turned the FRLG NSP into a working multi-game wrapper.

**Current state (v2):**
- ✅ Hoenn games (Ruby/Sapphire/Emerald) save correctly via timer-based save chain trigger
- ✅ State setup unlocked via 4 NOPs in `fn 0x56f60`
- ✅ Counter-based periodic save fires every ~60 sec (TST `w11, #0x1FFF`)

---

## Roadmap

### v3: Better save trigger
**Status:** Planned, in progress

**Goal:** Reduce save latency from 60 sec to "essentially instant" without crashing or excessive NAND wear.

**Options being evaluated:**

| Approach | Save latency | NAND wear | Crash risk | Complexity |
|---|---|---|---|---|
| **v3a — tighter timer (5-10 sec)** | 5-10 sec | Moderate | Low | Trivial (1-byte change) |
| **v3b — aggressive timer (~1 sec)** | <1 sec | Heavy | Medium (needs stress test) | Trivial |
| **v3c — subsdk0 chip detection patch** | Instant | Same as FRLG (light) | Low after RE | High (multi-day RE) |

**Feasibility:** All three are feasible. v3a is a single-byte mask change. v3b needs a stress test (play 30 min and watch for the User Break crash signature we saw with full per-tick spam). v3c is the architecturally correct fix and has been mapped at a high level (FlashEmulator at subsdk0+`0x15d6d18`, chip handlers around subsdk0+`0xb43d0`+, per-byte command processor at subsdk0+`0xb9000`/`0xb9070`).

**v3c plan if pursued:**
1. Identify the chip state machine's "save complete" detection inside subsdk0
2. Find what byte sequence FRLG uses (we know it's accepted) vs what Hoenn uses (rejected)
3. Patch the detector to accept both patterns
4. With proper detection, the timer in `save_handler` can be removed entirely — save fires naturally on actual save events, just like FRLG

**Recommendation:** Ship v3a (7.5 sec) first as quick win. Pursue v3c as a second pass if timer-based feels wrong long-term.

---

### Custom branding (icon + title)
**Status:** Feasible. Not yet started.

**Goal:** Make the app appear in the Switch home menu as something other than "Pokémon LeafGreen", reflecting the fact that it now plays multiple games. E.g., name it "GBA Wrapper" with a generic GBA icon, or "Pokémon Hoenn" with the Hoenn-themed art when running Hoenn ROMs.

#### How it works on Switch

App branding lives in the **control partition** of the NSP (separate from `exefs` which is the binary, and `romfs` which is game data). The control partition contains:

| File | Purpose |
|---|---|
| `control.nacp` | Application Control Property — contains TitleName per language, Author, AppVersion, ParentalControl, Save sizes, etc. |
| `icon_AmericanEnglish.dat` | 256×256 JPG-encoded icon for US English |
| `icon_BritishEnglish.dat` | UK English variant |
| `icon_French.dat`, `icon_German.dat`, ... | One per supported language |

Atmosphère LayeredFS supports overriding the control partition by placing replacement files at:
- `/atmosphere/contents/010034d02340e000/control.nacp` (overrides NACP)
- `/atmosphere/contents/010034d02340e000/icon_AmericanEnglish.dat` (overrides US English icon)

The Switch's `qlaunch` (home menu) reads these at boot, falls back to the NSP's bundled versions if not overridden.

#### Tasks

1. **Extract the original control partition** from the NSP. The file `Pokemon Leaf Green [010034D02340E000][v0].nsp` is a PFS0 archive containing the control NCA. We can use `hactool` or `hactoolnet` to dump:
   - `control.nacp` (binary file with structured fields)
   - All `icon_*.dat` files

2. **Modify the NACP**:
   - Edit `TitleName` string for each language slot we care about
   - Optionally edit `Author` to add credit
   - Tools: `nacptool`, hex editor, or Python struct-based editor (the format is documented at switchbrew)

3. **Replace the icon**:
   - Create a 256×256 JPG (Switch is permissive about exact dimensions but officially 256×256 with specific JPEG settings)
   - Save as `icon_AmericanEnglish.dat` (and other languages as needed)

4. **Deploy via LayeredFS** to the SD path above

5. **Reboot Switch** — home menu should display the new icon and title

#### Variants we could ship

- **Generic GBA Wrapper**: Plain GBA-themed icon, name "GBA Player" or "Game Boy Advance"
- **Hoenn-specific**: Use Hoenn cover art (Ruby/Sapphire/Emerald), name "Pokémon Hoenn"
- **Multi-icon based on detected ROM**: This is harder — would require runtime patching of the icon based on which ROM is in romfs. Not standard Switch behavior. Probably skip.

#### Feasibility verdict

**Easy.** The mechanism is well-documented, doesn't require RE, and works via standard Atmosphère LayeredFS file overrides. Estimated effort: 2-4 hours including making a decent-looking icon.

#### Constraints

- Atmosphère's LayeredFS on the control partition is supported but the file paths/format is more brittle than romfs/exefs. Test carefully.
- The ParentalControlData section of the NACP is signed in some ways; modifying it might cause issues. Stick to TitleName/Author/Version.
- Some launchers cache icons; may need to clear cache or reboot fully.

---

### Per-game save profiles
**Status:** Idea. Not yet evaluated.

**Goal:** When users swap LayeredFS ROMs (Emerald → Ruby), their saves shouldn't conflict. Currently, every game uses the same `LeafGreen_e.sav` file.

**Approach:** Hash the ROM bytes at boot, use the hash to derive a save filename. Or expose a "current ROM" filename slot in main's data and write to ROM-specific `.sav` files.

**Feasibility:** Moderate. Requires patching `WriteSaveFile`'s filename argument, which may be hardcoded. Could also be done at the LayeredFS level: a sysmodule that swaps `LeafGreen_e.sav` based on which ROM is loaded.

**Priority:** Low. Most users will play one game at a time and swap ROMs between sessions, manually managing saves.

---

### Cheat / GameShark support
**Status:** Idea.

**Goal:** Support GameShark / CodeBreaker codes for the GBA games being emulated.

**Feasibility:** Subsdk0 might already have GameShark support (since vanilla LeafGreen supports it via Mystery Gift). If so, we just need to find the input mechanism. If not, requires patching memory writes during emulation.

**Priority:** Low.

---

### Save state functionality (mid-game state snapshots)
**Status:** Idea.

**Goal:** Beyond Pokémon's in-game save, support arbitrary save states (full emulator state snapshots) that capture exact CPU/memory/Flash state.

**Feasibility:** Requires deep subsdk0 hooks. Possibly out of scope.

**Priority:** Very low.

---

### Cross-game compatibility testing
**Status:** Pending

**Tests to run:**

- **Vanilla Pokémon LeafGreen** (FRLG) — does our patch *not* break anything? Confirm it still saves normally.
- **Pokémon FireRed** — same as LeafGreen. Should work without our patch (and our patch shouldn't break it).
- **Pokémon Ruby** — confirmed not tested; likely works same as Emerald.
- **Pokémon Sapphire** — same.
- **Pokémon Emerald** — confirmed working ✅
- **FRLG-engine ROM hacks** (e.g., Pokémon Glazed, FireRed Rocket Edition) — should save naturally without our patch.
- **Hoenn-engine ROM hacks** (e.g., Pokémon Glazed v9.4 if Hoenn-based, RSE-engine fan games) — should work with our patch.
- **Non-Pokémon GBA games** (e.g., Mother 3 fan translation, Pokémon Mystery Dungeon: Red Rescue Team, Mario Kart Super Circuit) — varies by save type:
  - SRAM games (Mario Kart, F-Zero) → likely don't go through Flash chip emulator at all → our patch may not help
  - EEPROM games (Pokémon Pinball) → different path entirely
  - FLASH128/FLASH64 games (most RPGs) → should work with our patch

---

### Documentation / scene contribution
**Status:** Pending

**Tasks:**

- ✅ HOENN_SAVE_FIX.md — written
- ✅ SAVE_TRIGGER_ARCHITECTURE.md — written  
- 🔄 ROADMAP.md (this doc) — written
- ⏳ GBAtemp / r/SwitchHacks post — draft when v3 is settled
- ⏳ GitHub repo for the patches and tooling — clean up `tools/` dir, add README

---

### Stability / robustness
**Status:** Ongoing

**Known issues:**

- The User Break crash at `subsdk0+0x17574c` from the early force-spam testing — what triggers it? The post-save commit chain has *some* assertion or pattern check that fails on aggressive saves. Worth understanding even if our v2/v3 timer avoids it.
- ASLR-related: our counter is at `main+0x1d4514`, which is a fixed offset from main's load base. Atmosphère's load base is randomized, but the offset is constant, so ADRP+LDR resolves correctly. No issue here, but worth confirming if anyone sees odd counter behavior.
- Power-off timing: with v2 (60-sec timer), if the user presses Save in-game and immediately power-offs, the save can be lost. v3 reduces this window. v3c eliminates it.

---

### Other ideas (parking lot)

- **Touch input** for Switch's touchscreen (some GBA games support stylus via WarioWare Twisted accelerometer or similar — probably not relevant)
- **Multiplayer / link cable**: Stretch goal. The wrapper has 181 nn::pia networking classes — there's enormous untapped infrastructure here for emulated GBA wireless adapter or link cable. If someone could find the trigger for FRLG's wireless trade UI, we could potentially enable trades between Hoenn games via Switch local wireless. *Speculative.*
- **Frame rate / performance options**: If subsdk0 is throttled to GBA's 60fps, we could potentially overclock to match host FPS. Out of scope for save fix project, but interesting.

---

## Priority order (suggested)

1. **v3a (tighter timer, 5-10 sec)** — quick win, ship today
2. **Custom branding (icon + title)** — feasible, polish, ship this week
3. **Cross-game compatibility testing** — confirms what works/doesn't, no code changes
4. **GBAtemp / Reddit writeup** — share the discovery
5. **v3c (subsdk0 detection patch)** — proper fix, multi-day RE
6. Per-game saves, cheats, etc. — long-term ideas

---

**Generated:** 2026-04-29
**Maintainer:** add yourself when posting
**Status:** Roadmap living document; update as items are completed or descoped.
