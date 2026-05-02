# leafgreen-hoenn-save

Make Pokémon **Hoenn** games (Ruby / Sapphire / Emerald) save persistently when loaded via LayeredFS into the official Pokémon Leaf Green NSP on Nintendo Switch.

> **Status: v1 (WIP)** — confirmed working, but uses a periodic save flush that requires waiting **~7-8 seconds** after pressing Save before leaving the game (closing it from the home menu, switching titles, sleep, or powering off). A future version aims to remove that wait once the proper save trigger is identified.

---

## Demo

Quick demo on a real Switch — saving inside Pokémon Emerald (loaded via LayeredFS into the Leaf Green NSP), exiting the game, booting back in, and Continue is there with the save loaded.

<video src="assets/demo.mp4" controls width="400" muted playsinline>
  Your browser does not support inline video playback.
  <a href="assets/demo.mp4">Click here to download the demo video.</a>
</video>

> If the embedded player above doesn't show on GitHub, [click here to play `assets/demo.mp4` directly](assets/demo.mp4).

---

## The problem

If you swap a Hoenn ROM into the Pokémon Leaf Green NSP via LayeredFS, the game runs fine. The in-game save UI even tells you "Save complete!". But the moment you power off and reboot, **there is no Continue option** — your save is gone.

That's because the Leaf Green wrapper's save subsystem is hard-coded for the FRLG game engine. Two things go wrong for Hoenn:

1. **State setup gates** — A function in `main` that initializes the save context bails early when it detects a non-FRLG save engine version, leaving internal save state uninitialized.
2. **Save trigger never fires** — The GBA emulator core (`subsdk0`) only recognizes FRLG's specific Flash command sequence. Hoenn uses a different chip command pattern, so no save event is ever broadcast to `main`.

This patch fixes both.

## What v1 does

A single IPS patch applied to `main.elf` at boot:

- **4 NOPs** in the state-setup function bypass the FRLG-engine gates so save state initializes for any game.
- **10 instructions** added at the `save_handler` prologue install a counter that periodically force-fires the save chain (~every 7.5 seconds) so whatever the emulator has buffered in Flash gets flushed to `LeafGreen_e.sav`.

Result: full Hoenn save data (14 sectors, magic `0x08012025`, valid checksums) gets written to disk. Switch reads it on next boot. Continue appears. Save loads.

## Compatibility

**Confirmed working:**
- Pokémon Leaf Green NSP (USA, Title ID `010034D02340E000`)
- Pokémon Emerald (USA) — 16 MB GBA ROM
- Pokémon Emerald (Europe) — 16 MB GBA ROM

**Should work, not yet confirmed:**
- Ruby / Sapphire (same Hoenn engine architecture)
- Other regional NSP versions
- Japanese Hoenn ROMs

If you test something not on the confirmed list, please open an Issue — would help verify the patch is region-agnostic.

## Requirements

- Nintendo Switch with **Atmosphère** CFW (any recent version)
- Pokémon Leaf Green NSP installed (Title ID `010034D02340E000`)
- A 16 MB Hoenn GBA ROM, placed at:
  ```
  /atmosphere/contents/010034D02340E000/romfs/LeafGreen_e.gba
  ```

> **Note:** No ROMs or copyrighted game binaries are distributed with this project. You must provide your own legally-obtained ROM.

## Installation

1. Download the latest release zip from the [Releases](../../releases) page.
2. Extract it at the **root of your Switch SD card**. The included `atmosphere/` folder will merge with your existing one — no files are overwritten.
3. After extraction you should see:
   ```
   <SD root>/atmosphere/exefs_patches/leafgreen-hoenn-save-v1/
       a8dd9bcc29745ae5be9635a11a42dcbd90071181.ips
   ```
4. Boot the game and play normally.

> **About the IPS filename:** `a8dd9bcc...ips` is the build ID of the Leaf Green `main` NSO. Atmosphère matches IPS files to NSOs by build ID. **Do not rename the file** — renaming silently disables the patch.

## Usage

1. Play your Hoenn game normally.
2. Save in-game when you want to save.
3. **Wait ~7-8 seconds** before leaving the game in any way — closing it from the home menu, switching to another title, putting the Switch to sleep, or powering off.
4. Boot back into the game — your save is there under "Continue".

If you exit the game immediately after pressing Save, the save will not have been captured yet. The patch flushes the emulated Flash buffer to disk on a periodic timer (~7.5s) **while the game is running**, so the wait applies to any path that ends the game process — not just power-off.

## Why the wait?

In vanilla LeafGreen, saves are instant because `subsdk0` recognizes FRLG's Flash command pattern and broadcasts a save event immediately. Hoenn uses a different pattern, so the event never fires.

The "right" fix is to extend `subsdk0`'s pattern detector to also recognize the Hoenn command sequence. That's a deeper subsdk0 reverse engineering project (~18 MB of un-symboled emulator code) and is on the roadmap for a future version. The timer is the workaround in the meantime.

See [`docs/SAVE_TRIGGER_ARCHITECTURE.md`](docs/SAVE_TRIGGER_ARCHITECTURE.md) for the full architectural breakdown.

## Documentation

- [`docs/TECHNICAL_WRITEUP.md`](docs/TECHNICAL_WRITEUP.md) — full reverse-engineering writeup: save chain map, all NSO addresses, every dead end, the actual patch source
- [`docs/SAVE_TRIGGER_ARCHITECTURE.md`](docs/SAVE_TRIGGER_ARCHITECTURE.md) — why FRLG saves natively but Hoenn needs the timer; decision matrix for v1 vs. proper fix
- [`docs/SUBSDK0_INVESTIGATION.md`](docs/SUBSDK0_INVESTIGATION.md) — leads on the proper subsdk0 fix (HLE save handler at `0x22e2ac`, 6 fire sites for event `0x4C`)
- [`ROADMAP.md`](ROADMAP.md) — what's next
- [`CHANGELOG.md`](CHANGELOG.md) — version history

## Roadmap

- A future version that fires saves on the actual save event instead of a timer (instant saves, no wait)
- Custom icon and home-menu title (currently still shows as "Pokémon Leaf Green")
- Per-game save profiles
- Investigate link cable / wireless features (Hoenn ↔ FRLG trades through Sevii Islands code) — long-shot, no promises

See [`ROADMAP.md`](ROADMAP.md) for details.

## Reporting issues

Please open an Issue if you:
- Test a Hoenn ROM not on the confirmed list (Ruby, Sapphire, JP regions, hacks…)
- Test a non-USA Leaf Green NSP version
- Hit a crash or save corruption

Helpful info to include: NSP region, ROM region/version, Atmosphère version, Hoenn ROM size, what you did before the issue.

## Contributing

If you have `subsdk0` reverse-engineering experience and want to help nail the proper save trigger fix, get in touch — the leads are in [`docs/SUBSDK0_INVESTIGATION.md`](docs/SUBSDK0_INVESTIGATION.md). Pull requests welcome.

## Support

If this saved you a headache and you want to support the project:

- Buy Me a Coffee: **https://buymeacoffee.com/gmr7z**

The patch and writeups stay free regardless. It just helps justify the late nights spent on the proper-fix and networking experiments.

## Contact

For any inquiry, please open an Issue, reply to the GBATemp/Reddit threads, or email **gmr7z.re@gmail.com** — happy to answer any question.

## Disclaimer

This project does not distribute any Nintendo, Game Freak, or Pokémon Company software. The IPS file in this repository contains only original patch bytes (NOPs and a small counter routine) authored by the project. You must provide your own legally-obtained ROM and own a legitimate copy of the Pokémon Leaf Green NSP. Use at your own risk — back up your saves.

## License

[MIT](LICENSE)
