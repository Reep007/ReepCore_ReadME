# ReepCore

**Your complete, opinionated Arch + ReepCompositor system — formalized.**

This is not a library, package, or general-purpose tool. This is **your machine, formalized**.

## What Is This?

ReepCore is a monorepo containing:

- **REEPCORE**: The spine — declarative system management with atomic rollbacks
- **reepcompositor**: Custom Wayland compositor — tiling, keybinds, ReepCore theme integration
- **reepaper**: Native Wayland wallpaper daemon and CLI
- **walr**: Color engine — generates themes from wallpapers
- **reep-shell**: Status bar for ReepCompositor (wlr-layer-shell)
- **reep-launcher**: App launcher — terminal UI or Wayland overlay (`--gui`, `--power`)
- **reef**: File manager (Iced UI)
- **reepfetch**: Terminal renderer — displays system state with REEPCORE awareness
- **reepedit**: In-TUI editor (used by Jarvis and the ReepCore TUI)
- **zsh-rust-plugins**: Shell enhancements — Rust-powered autosuggestions and syntax highlighting
- **Jarvis**: Local AI assistant in `reepcore tui` (Ollama-backed; see [REEPCORE/docs/jarvis.md](REEPCORE/docs/jarvis.md))

Theme boot/cache and wallpaper management live in **`reepcore theme restore`**, **`reepcore theme refresh`**, and **`reepcore tui`** (Theme Manager) — not a separate wallpaper app.

## Philosophy

**REEPCORE is the root of authority.**

Everything answers one question: "Am I part of state, output, or control?"

- **State** (`~/.local/state/reepcore/`): Generations, profile, outputs (colors.sh, compositor.sock)
- **Control**: Tools that modify state (REEPCORE CLI / TUI, walr)

## Prerequisites

Before installing, ensure you have:

- **Rust** (with `cargo`): Required to build `reepcore`, `walr`, and companion tools
  - On Arch Linux: `sudo pacman -S rust`
  - Or via rustup: `curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh`

The installation script will check for `cargo` and provide helpful error messages if it's missing.

## Quick Start

### For Fresh Arch Linux Installations

```bash
# Clone
git clone https://github.com/Reep007/ReepCore
cd ReepCore

# Bootstrap (installs paru, builds reepcore + tools, optional init)
./bootstrap.sh --yes

# Initialize REEPCORE with complete template (default: reepcore-setup)
reepcore init --template reepcore-setup

# Review configuration
cat ~/.config/reepcore/reepcore.yaml

# Test (dry run)
reepcore switch --dry-run

# Apply configuration
reepcore switch

# Reboot to apply changes and log into ReepCompositor
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
# Use this if you just want reepfetch, walr, reepcompositor, etc.
# without the full REEPCORE system management
./install.sh
```

See [BOOTSTRAP_GUIDE.md](BOOTSTRAP_GUIDE.md) for detailed instructions.

## Structure

```
ReepCore/
├── REEPCORE/
│   ├── bin/
│   │   ├── reepfetch           # Terminal renderer
│   │   └── reepedit            # TUI editor
│   ├── tools/
│   │   ├── walr/               # Color engine (vendored)
│   │   ├── reepcompositor/     # Wayland compositor
│   │   ├── reepaper/           # Wallpaper daemon
│   │   ├── reep-shell/         # Status bar
│   │   ├── reep-launcher/      # App launcher
│   │   ├── reef/               # File manager
│   │   └── zsh-rust-plugins/   # Shell enhancements (vendored)
│   ├── src/                    # REEPCORE core (Rust)
│   ├── templates/              # Profile templates (reepcore-setup)
│   └── docs/                   # Jarvis, intent-v1, security, etc.
│
├── dotfiles/                   # Stow-style configs deployed by reepcore switch
├── bootstrap.sh                # Full bootstrap (paru + build + init)
├── install.sh                  # Build and install binaries only
└── README.md                   # This file
```

See [REEPCORE/README.md](REEPCORE/README.md) for the full CLI reference, configuration guide, and screenshots.

## Components

### REEPCORE
**Role**: Philosophy + glue  
**Owns**: State, outputs, orchestration  
**Docs**: [REEPCORE/README.md](REEPCORE/README.md)

### reepcompositor
**Role**: Custom Wayland compositor (ReepCompositor session)  
**Reads**: walr colors, ReepCore generation/theme state  
**Location**: `REEPCORE/tools/reepcompositor/`

### walr
**Role**: Color generation engine  
**Input**: Wallpaper image  
**Output**: `colors.json` and app theme files  
**Location**: `REEPCORE/tools/walr/`

### reepaper
**Role**: Native Wayland wallpaper daemon and CLI  
**Location**: `REEPCORE/tools/reepaper/`

### reep-shell
**Role**: Status bar for ReepCompositor (wlr-layer-shell)  
**Location**: `REEPCORE/tools/reep-shell/`

### reep-launcher
**Role**: App launcher — fuzzy search, `--gui` overlay, `--power` session menu  
**Location**: `REEPCORE/tools/reep-launcher/`

### reef
**Role**: File manager (Iced UI)  
**Location**: `REEPCORE/tools/reef/`

### reepfetch
**Role**: Terminal display of system state  
**Reads**: `~/.local/state/reepcore/generations/*` (current + history) and ReepCore-managed outputs  
**Location**: `REEPCORE/bin/reepfetch`

### reepedit
**Role**: In-TUI editor for Jarvis and ReepCore screens  
**Location**: `REEPCORE/bin/reepedit`

### zsh-rust-plugins
**Role**: Shell enhancements  
**Features**: Fish-like autosuggestions and real-time syntax highlighting  
**Binaries**: `zsh-rust-suggest`, `zsh-rust-highlight`, `zsh-rust-daemon`  
**Location**: `REEPCORE/tools/zsh-rust-plugins/`

### Jarvis
**Role**: Local AI assistant (Ollama) inside `reepcore tui`  
**Features**: Chat, reepedit integration, embedded shell, RAG, voice  
**Docs**: [REEPCORE/docs/jarvis.md](REEPCORE/docs/jarvis.md)

## Installation

The `install.sh` script:

1. Creates `~/.local/state/reepcore/outputs/` for generated files
2. Copies binaries to `~/.local/bin/` (survives repo move/removal)
3. Builds and installs: `reepcore`, `reepfetch`, `reepedit`, `walr`, `reef`, `reepaper`, `reep-launcher`, `reepcompositor`, `reep-shell`, and `zsh-rust-plugins`
4. Optionally installs the ReepCompositor Wayland session file for display managers

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

# Boot restore and cache refresh
reepcore theme restore
reepcore theme refresh
```

### Start ReepCompositor
```bash
# Display manager: select "ReepCompositor" at login
# Nested test session:
reepcompositor --session
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
reepcore tui         # Interactive profile manager (+ Jarvis AI assistant)
reepcore doctor      # Health check
```

## Why This Structure?

1. **Clear Authority**: REEPCORE owns state, not individual tools
2. **Separation**: Engine (walr) ≠ Compositor (reepcompositor) ≠ Control surface (reepcore TUI/CLI) ≠ Renderer (reepfetch)
3. **Explicit Dependencies**: Everything under one roof
4. **Atomic Operations**: Theme and system changes feel atomic
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
