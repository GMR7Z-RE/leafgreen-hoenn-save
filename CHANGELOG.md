# Changelog

All notable changes to this project will be documented in this file.

## [v1] — 2026-05-02 (WIP)

Initial public release.

### Added
- IPS patch for Pokémon Leaf Green NSP `main` (build ID `a8dd9bcc29745ae5be9635a11a42dcbd90071181`).
- Bypass of FRLG-engine state-setup gates so save state initializes for any GBA cartridge.
- Periodic save flush (~7.5 seconds) that writes the emulated GBA Flash buffer to `LeafGreen_e.sav` while the game is running.
- Technical writeup, save trigger architecture note, and `subsdk0` investigation notes under `docs/`.
- Roadmap covering the proper save trigger fix, custom icon/title, and link cable / wireless networking experiments.

### Confirmed working
- Pokémon Leaf Green NSP (USA, Title ID `010034D02340E000`)
- Pokémon Emerald (USA), 16 MB GBA ROM
- Pokémon Emerald (Europe), 16 MB GBA ROM

### Known limitations
- Saves are only persisted by a periodic timer, so users must wait ~7-8 seconds after pressing Save before powering off.
- Home menu still shows "Pokémon Leaf Green" with the original icon (custom branding planned for a future version).
- Ruby, Sapphire, non-USA NSP versions, and Japanese Hoenn ROMs are unconfirmed (expected to work, reports welcome).
