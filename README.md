# More Legacies

**Expanded dynasty legacy system for Crusader Kings III**

A continuation of bravelildragon's [More Legacies](https://steamcommunity.com/sharedfiles/filedetails/?id=2859278694) mod, bundling artwork from I Am Full Truie's [More Legacies Art +](https://steamcommunity.com/sharedfiles/filedetails/?id=2977838440).

## For Players

Looking for mod features and gameplay information? See **[mod/README.md](mod/README.md)** - that's the file that ships with the mod.

## For Developers

This repository uses the [jesec/ck3-mod-base](https://github.com/jesec/ck3-mod-base) structure for CK3 mod development.

### Quick Start

```bash
# Clone the repository
git clone <repository-url> ck3-mod-more-legacies
cd ck3-mod-more-legacies

# Build the mod
bash build.sh

# Install to CK3
bash install.sh
```

### Repository Structure

```
ck3-mod-more-legacies/
├── mod/                                    # All mod content (shipped to users)
│   ├── common/
│   │   ├── dynasty_legacies/               # 10 legacy track definitions
│   │   └── dynasty_perks/                  # 50 perk definitions
│   ├── gfx/interface/                      # Icons and illustrations
│   ├── localization/                       # 9 language translations
│   ├── descriptor.mod                      # Mod metadata
│   ├── thumbnail.png                       # Mod icon
│   ├── CHANGELOG                           # Version history
│   └── README.md                           # User documentation (ships with mod)
│
├── base/                                   # jesec/ck3-mod-base vanilla reference
│   ├── game/                               # CK3 1.18.1.1 vanilla files
│   ├── scripts/                            # Build scripts
│   └── docs/                               # Base system documentation
│
├── out/                                    # Build output (gitignored)
│
├── README.md                               # This file (repo documentation)
├── build.sh / build.bat                    # Build scripts (symlinks to base/scripts/)
└── install.sh / install.bat                # Install scripts (symlinks to base/scripts/)
```

### Building

The build system detects changes in `base/game/` and includes all of `mod/`:

```bash
# Build mod to out/ directory
bash build.sh

# Output structure:
out/
├── common/
├── gfx/
├── localization/
├── descriptor.mod
└── thumbnail.png
```

**Note:** This mod only uses `mod/` directory. No vanilla files are modified, so `base/game/` serves only as reference.

### Installing

```bash
# Interactive install
bash install.sh

# Choose:
# 1. Copy - Copies out/ to CK3 mod directory
# 2. Symlink - Creates symlink (useful for development)
```

**Manual install locations:**
- **Windows**: `Documents\Paradox Interactive\Crusader Kings III\mod\`
- **Linux**: `~/.local/share/Paradox Interactive/Crusader Kings III/mod/`
- **macOS**: `~/Documents/Paradox Interactive/Crusader Kings III/mod/`

### Development Workflow

1. **Edit mod files** in `mod/` directory
2. **Build**: `bash build.sh`
3. **Test** in CK3
4. **Iterate**

**Testing checklist:**
- All 10 legacies visible with appropriate cultures
- Perk tooltips display correctly
- Stat-scaling perks work (prestige/piety/stress levels)
- Localization displays in all supported languages

### File Descriptions

**Mod Content (`mod/`):**
- `common/dynasty_legacies/more_legacies.txt` - 10 legacy track definitions with visibility conditions
- `common/dynasty_perks/more_dynasty_perks.txt` - 50 perk definitions with modifiers
- `gfx/interface/icons/dynasty/*.dds` - 10 small icons (77 KB each)
- `gfx/interface/illustrations/legacy_tracks/*.dds` - 10 large illustrations (758 KB each)
- `localization/*/dynasty_legacies/*.yml` - 9 language files
- `descriptor.mod` - Mod metadata (name, version, supported game version, tags)
- `CHANGELOG` - Version history
- `README.md` - User-facing documentation (ships with mod)

**Development Files:**
- `base/` - Vanilla CK3 reference files (jesec/ck3-mod-base structure)

### Version Compatibility

**Current version:** CK3 1.18.* (Crane)

**Key systems to verify when updating CK3 versions:**
- Prestige/piety level thresholds (`base/game/common/defines/00_defines.txt` lines 154-161)
- Scheme duration constants (`base/game/common/script_values/00_scheme_values.txt` lines 433-439)
- Modifier definitions (`base/game/common/modifier_definition_formats/00_definitions.txt`)
- Cultural traditions (new traditions may need adding to legacy visibility conditions)
- Dread mechanics (`base/game/common/defines/00_defines.txt` lines 120-124)
- Stress system (`base/game/common/defines/00_defines.txt` lines 117-118)

### Naming Conventions

**ID Prefix:** `bld_` (from original author bravelildragon)

All legacy tracks and perks use this prefix:
- Legacy tracks: `bld_chivalry_legacy_track`, `bld_dominance_legacy_track`, etc.
- Perks: `bld_chivalry_legacy_1`, `bld_chivalry_legacy_2`, etc.

### Credits

**Original Mod:**
[More Legacies](https://steamcommunity.com/sharedfiles/filedetails/?id=2859278694) by bravelildragon
- 10 legacy track designs
- 50 dynasty perks
- Game mechanics and balance

**Artwork:**
[More Legacies Art +](https://steamcommunity.com/sharedfiles/filedetails/?id=2977838440) by I Am Full Truie
- 10 legacy track icons
- 10 legacy track illustrations

**Localization:**
- German: positief161
- Spanish: Kordi
- Korean: WKML
- Chinese: Marcus

**Build System:**
[jesec/ck3-mod-base](https://github.com/jesec/ck3-mod-base)

### License

**Mod Content:**
- Original design: bravelildragon
- Artwork: I Am Full Truie
- Integration: This repository

**Base Game Files:**
- `base/game/` © Paradox Interactive
- See `base/LICENSE-GAME-CONTENT`

**Build System:**
- jesec/ck3-mod-base automation
- See `base/LICENSE`

### Links

- **Original More Legacies**: https://steamcommunity.com/sharedfiles/filedetails/?id=2859278694
- **More Legacies Art +**: https://steamcommunity.com/sharedfiles/filedetails/?id=2977838440
- **jesec/ck3-mod-base**: https://github.com/jesec/ck3-mod-base
