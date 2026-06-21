# ReepCore

**Your complete, opinionated Arch + ReepCompositor system — formalized.**

This is not a library, package, or general-purpose tool. This is **your machine, formalized**.

## What Is This?

ReepCore is a monorepo containing:

- **REEPCORE**: The spine — declarative system management with atomic rollbacks
- **walr**: Color engine — generates themes from wallpapers
- **wallpaper_app**: Pointer docs only; theme boot/cache via `reepcore theme restore` / `reepcore theme refresh`; UI is `reepcore tui`
- **reepfetch**: Terminal renderer — displays system state with REEPCORE awareness
- **zsh-rust-plugins**: Shell enhancements — Rust-powered autosuggestions and syntax highlighting

## Philosophy

**REEPCORE is the root of authority.**

Everything answers one question: "Am I part of state, output, or control?"

- **State** (`~/.local/state/reepcore/`): Generations, profile, outputs (colors.sh, compositor.sock)
- **Control**: Tools that modify state (REEPCORE CLI / TUI, walr)

## Prerequisites

Before installing, ensure you have:

- **Rust** (with `cargo`): Required to build `reepcore`, `walr`, and `zsh-rust-plugins`
  - On Arch Linux: `sudo pacman -S rust`
  - Or via rustup: `curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh`

The installation script will check for `cargo` and provide helpful error messages if it's missing.

## Quick Start

### For Fresh Arch Linux Installations

```bash
# Clone
git clone https://github.com/Reep007/ReepCore
cd ReepCore

# Bootstrap (installs paru and REEPCORE)
./bootstrap.sh --yes

# Initialize REEPCORE with complete template
reepcore init --template reepcore-setup

# Review configuration
cat ~/.config/reepcore/reepcore.yaml

# Test (dry run)
reepcore switch --dry-run

# Apply configuration
reepcore switch

# Reboot to apply changes
sudo reboot
```

### For Existing Systems

```bash
# Clone
git clone https://github.com/Reep007/ReepCore
cd ReepCore

# Option 1: Use REEPCORE (recommended)
# Bootstrap if you don't have paru/REEPCORE yet
./bootstrap.sh --yes

# Migrate existing system to REEPCORE
reepcore migrate

# Review and customize generated config
cat ~/.config/reepcore/reepcore.yaml

# Apply
reepcore switch

# Option 2: Install tools only (without REEPCORE)
# Use this if you just want reepfetch, walr, zsh-rust-plugins
# without the full REEPCORE system management
./install.sh
```

See [BOOTSTRAP_GUIDE.md](BOOTSTRAP_GUIDE.md) for detailed instructions.

## Structure

```
ReepCore/
├── reepcore/
│   ├── bin/
│   │   └── reepfetch           # Terminal renderer
│   ├── tools/
│   │   ├── walr/               # Color engine (vendored)
│   │   ├── wallpaper_app/      # Theme CLI pointers (README only)
│   │   └── zsh-rust-plugins/   # Shell enhancements (vendored)
│   ├── src/                    # REEPCORE core (Rust)
│   └── templates/              # Profile templates
│
├── install.sh                  # Installation script
└── README.md                   # This file
```

See [`REEPCORE/README_ARCHITECTURE.md`](REEPCORE/README_ARCHITECTURE.md) for detailed architecture.

## Components

### REEPCORE
**Role**: Philosophy + glue  
**Owns**: State, outputs, orchestration  
**Docs**: [REEPCORE/README.md](REEPCORE/README.md)

### walr
**Role**: Color generation engine  
**Input**: Wallpaper image  
**Output**: `colors.json`  
**Status**: Vendored in `reepcore/tools/walr/`

### wallpaper_app
**Role**: Short README only — boot restore and cache refresh live in **`reepcore theme restore`** and **`reepcore theme refresh`**  
**Interactive theme / wallpaper**: `reepcore tui` (Theme Manager) or `reepcore theme …`  
**Location**: `reepcore/tools/wallpaper_app/README.md`

### reepfetch
**Role**: Terminal display of system state  
**Reads**: `~/.local/state/reepcore/generations/*` (current + history) and ReepCore-managed outputs  
**Location**: `reepcore/bin/reepfetch`

### zsh-rust-plugins
**Role**: Shell enhancements  
**Features**: Fish-like autosuggestions and real-time syntax highlighting  
**Binaries**: `zsh-rust-suggest`, `zsh-rust-highlight`, `zsh-rust-daemon`  
**Location**: `reepcore/tools/zsh-rust-plugins/`

### reef
**Role**: File manager (Iced UI)  
**Location**: `reepcore/tools/reef/`

### reepaper
**Role**: Native Wayland wallpaper daemon and CLI  
**Location**: `reepcore/tools/reepaper/`

### reep-shell
**Role**: Status bar for ReepCompositor (wlr-layer-shell)  
**Location**: `reepcore/tools/reep-shell/`

## Installation

The `install.sh` script:

1. Creates `~/.local/state/reepcore/outputs/` for generated files
2. Copies binaries to `~/.local/bin/` (survives repo move/removal)
3. Builds and copies all `REEPCORE/tools/*` user binaries (walr, reef, reepaper, reep-launcher, reepcompositor, reep-shell, zsh-rust-plugins) plus `reepcore`
4. Sets up runtime environment

**Runtime vs Source:**
- **Source**: Lives in `ReepCore/REEPCORE/` (this repo)
- **Runtime state**: All state under `~/.local/state/reepcore/` (generations, backups, outputs, profile)
- **Runtime binaries**: Copied to `~/.local/bin/`

## Usage

### Change Wallpaper & Theme
```bash
# Recommended: Use ReepCore TUI (unified interface)
reepcore tui
# Navigate to Theme Manager for wallpaper selection and profile management

# Via REEPCORE CLI (non-interactive)
reepcore theme apply --wallpaper ~/Wallpaper/nord.jpg --profile soft
```

### View System Info
```bash
reepfetch  # Displays REEPCORE state with walr colors
```

### Enable Shell Enhancements
After running `./install.sh`, add to your `~/.zshrc`:
```bash
# zsh-rust-plugins
source /path/to/ReepCore/REEPCORE/tools/zsh-rust-plugins/zsh/config.zsh
source /path/to/ReepCore/REEPCORE/tools/zsh-rust-plugins/zsh/zsh-rust-suggest.zsh
source /path/to/ReepCore/REEPCORE/tools/zsh-rust-plugins/zsh/zsh-rust-highlight.zsh
```

### Manage System
```bash
reepcore switch      # Apply configuration
reepcore rollback    # Rollback to previous generation
reepcore tui         # Interactive profile manager
```

## Why This Structure?

1. **Clear Authority**: REEPCORE owns state, not individual tools
2. **Separation**: Engine (walr) ≠ Control surface (reepcore TUI/CLI) ≠ Renderer (reepfetch)
3. **Explicit Dependencies**: Everything under one roof
4. **Atomic Operations**: Theme changes feel atomic
5. **Easy Debugging**: `state/`, `outputs/`, `renderer/` — no guesswork

## Important Note

**This repo is your machine, formalized.**

Do not:
- Make it "installable" as a package
- Chase universality
- Support multiple distros

This is:
- Your complete, opinionated system build
- Others can fork if they want

## License

GPL-3.0

## Credits

Built with Rust, inspired by NixOS, powered by Arch Linux.

Created by [@Reep007](https://github.com/Reep007)


*"A system should be more than the sum of its parts. ReepCore makes it so."*
