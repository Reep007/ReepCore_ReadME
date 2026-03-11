# ReepCore

**A declarative, generational system orchestrator for Arch Linux with live theme synchronization and complete desktop integration.**

---

## What Is ReepCore?

ReepCore is a complete system management framework that brings declarative configuration, generational state management, and live theme synchronization to Arch Linux. Unlike traditional configuration managers that focus on one aspect of the system, ReepCore orchestrates your entire computing environment—from packages and services to dotfiles and desktop theming—as a cohesive, reproducible whole.

Think NixOS's declarative approach meets Ansible's flexibility, with the visual cohesion of a carefully curated rice, all managed through a single, native binary.

---

## The Vision

Most Linux systems are assembled from disparate tools:
- Package managers install software
- Stow or Git manages dotfiles  
- Systemd handles services
- Pywal generates themes
- Each compositor, terminal, and editor has its own config

**ReepCore integrates all of this.** One YAML file. One command. One coherent system.

But more importantly: **changes propagate instantly**. Set a new wallpaper, and watch your compositor borders, terminal colors, shell prompt, text editor, and GTK theme update in real-time—no restarts, no manual steps, no broken states.

---

## The Problem

Traditional Linux configuration management has fragmentation:

**Package Management:**
- Official repos, AUR, Flatpak, and Git repos all require different tools
- No unified way to declare "these are my packages"
- Dependency conflicts break systems

**Dotfile Management:**
- Stow, chezmoi, yadm, bare Git repos—many approaches, no standard
- Manual deployment after changes
- No rollback when configs break

**Theme Management:**
- Pywal generates colors, but you manually template 15+ config files
- Changes require restarting every app
- No synchronization between compositor, terminal, shell, etc.

**System State:**
- No generations—if something breaks, manual recovery
- No health checks—problems discovered after the fact
- No migration path from existing systems

**ReepCore solves all of this with a unified, declarative approach.**

---

## Architecture

### Core Principles

1. **Declarative Configuration**: Define your system's desired state in YAML
2. **Generational State**: Every change creates a new generation (instant rollback)
3. **Live Synchronization**: Theme changes propagate across all applications instantly
4. **Modular Profiles**: Compose your system from reusable building blocks
5. **Health Validation**: Continuous monitoring ensures system integrity

### The Stack

```
┌─────────────────────────────────────────────────────────┐
│                    USER INTERFACE                        │
├─────────────────────────────────────────────────────────┤
│  TUI                    │ CLI Commands                   │
│  • Main Menu            │ • reepcore switch              │
│  • Profile Manager      │ • reepcore theme               │
│  • Theme Manager        │ • reepcore doctor              │
│  • File Browser         │ • reepcore migrate             │
│  • Built-in Editor      │ • reepcore rollback            │
│  • App Launcher         │ • reepcore edit                │
│  • Package Manager      │                                │
│  • Kernel Manager       │                                │
└─────────────────────────────────────────────────────────┘
                            ↓
┌─────────────────────────────────────────────────────────┐
│                  ORCHESTRATION LAYER                     │
├─────────────────────────────────────────────────────────┤
│  Profile System         │ Generation Manager             │
│  • YAML parsing         │ • State snapshots              │
│  • Template expansion   │ • Rollback support             │
│  • Dependency resolution│ • Verification                 │
└─────────────────────────────────────────────────────────┘
                            ↓
┌─────────────────────────────────────────────────────────┐
│                    EXECUTION LAYER                       │
├─────────────────────────────────────────────────────────┤
│  Package Management     │ Dotfile Deployment             │
│  • pacman (official)    │ • Stow-based symlinking        │
│  • AUR (yay/paru)       │ • Manual copy fallback         │
│  • Flatpak              │ • Conflict detection           │
│  • Git repos            │                                │
│                         │                                │
│  Service Management     │ Theme Engine (walr)            │
│  • systemd enable/start │ • Color extraction             │
│  • User services        │ • Template generation          │
│  • Mask/disable         │ • Live reload                  │
└─────────────────────────────────────────────────────────┘
                            ↓
┌─────────────────────────────────────────────────────────┐
│                   DESKTOP INTEGRATION                    │
├─────────────────────────────────────────────────────────┤
│                         │                                │
│                         │ • walr-native terminal         │
│  • Live walr borders    │ • Kitty graphics protocol      │
│  • 6+ tiling layouts    │                                │
│                         │                                │
│  Shell Enhancement      │ Text Editor                    │
│  • zsh-rust-highlight   │ • reepedit (built-in)          │
│  • zsh-rust-suggest     │ • walr theming                 │
│  • Neo prompt           │ • Visual mode                  │
└─────────────────────────────────────────────────────────┘
```

---

## Features

### Declarative System Configuration

Define your entire system in a single YAML file:

```yaml
system:
  hostname: "desktop"
  timezone: "America/New_York"
  
profiles:
  - desktop
  - dev
  - media

packages:
  official:
    - firefox
    - kitty
    - neovim
  aur:
    - yay
    - spotify
  flatpak:
    - com.slack.Slack
  git:
    - url: "https://github.com/user/tool"
      path: "~/.local/bin/tool"

dotfiles:
  - name: "nvim"
    source: "~/.dotfiles/nvim"
    target: "~/.config/nvim"
  - name: "zsh"
    source: "~/.dotfiles/zsh/.zshrc"
    target: "~/.zshrc"
    method: copy

services:
  enabled:
    - docker
    - bluetooth
  user:
    - dunst
```

### Profile System

Build modular, composable configurations:

**Base Profiles** (built-in templates):
- `desktop` — Window manager, compositor, desktop apps
- `dev` — Development tools, languages, IDEs
- `gaming` — Steam, Wine, performance tweaks
- `media` — Audio/video production, streaming
- `server` — Headless services, monitoring

**Custom Profiles**: Create your own, combine multiple profiles, override package lists.

**Profile Composition**: Automatically merges package lists, resolves conflicts, manages dependencies.

### Generational State Management

Every `reepcore switch` creates a new generation:

```
Generation 10 (current) ← You are here
├─ Packages: firefox, brave ... (500 total)
├─ Dotfiles: zsh, waybar, ...
├─ Services: docker, bluetooth
└─ Snapshot: /snapshots/gen-10

Generation 9
├─ Packages: firefox, vim, ... (498 total)
└─ Can rollback instantly
```

**Rollback**: `reepcore rollback` restores previous generation (packages, dotfiles, services)

**Snapshots**: Optional Btrfs filesystem snapshots for complete system state

**Verification**: Hash-based integrity checks ensure no silent corruption

### Theme Engine (walr)

Automatic color extraction and live theme propagation:

**How It Works:**
1. Set wallpaper (via Theme Manager or `walr ~/wallpaper.png`)
2. walr extracts 10 colors using k-means clustering
3. Generates themed configs for 15+ applications
4. All apps reload instantly—no restarts needed

**Supported Applications:**
- Terminal colors (kitty, alacritty)
- Shell prompt (Neo powerline)
- Syntax highlighting (zsh-rust-highlight)
- Text editor (reepedit)
- Status bars (waybar, i3status-rust)
- GTK themes (gtk-3.0, gtk-4.0)
- Application launchers (rofi, wofi)
- Notification daemon (dunst, mako)
- And more...

**Cache System**: SHA256-based caching prevents redundant regeneration. Paranoid verification (3-layer hash checks) ensures correct colors.

### Full-Featured TUI

A complete terminal interface built with ratatui:

**Views:**
- **Main Menu** — Animated dashboard with system stats (CPU, RAM)
- **Profile Manager** — Toggle profiles, view packages, apply changes
- **Theme Manager** — Preview wallpapers (with image rendering), apply themes
- **Package Manager** — Search, install, remove packages (official + AUR)
- **File Browser** — Navigate filesystem, preview files/images, Trash support
- **Text Editor** — Built-in editor with syntax awareness, visual mode, walr theming
- **App Launcher** — Launch .desktop applications by category
- **Kernel Manager** — Install, remove, pin kernels per generation
- **Generations** — View history, rollback, create snapshots

**Features:**
- Mouse support (scroll, click)
- Live system stats (CPU, RAM usage)
- Animated borders and spinners
- Password input (sudo integration)
- Image preview (Kitty/halfblocks protocol)
- Keyboard shortcuts everywhere
- Embedded terminal panel for command output

### Built-in Text Editor (reepedit)

A nano-style editor integrated into the TUI:

**Features:**
- UTF-8 support with proper char boundary handling
- Visual line selection mode (vim-style)
- Yank/paste operations
- Line numbers with auto-scroll
- Syntax-aware commenting (accent colors)
- Unsaved changes dialog
- walr theme integration
- F2 to save, F3/Esc to exit

**Access:**
- From TUI: File Browser → select file → press `e`
- From shell: `reepcore edit ~/.zshrc` or `reepedit ~/.zshrc`

### Package Management

Unified interface for all package sources:

**Official Repositories** (pacman):
- Fast, reliable, verified packages
- Automatic dependency resolution

**AUR** (via yay/paru):
- Build from source automatically
- Handle AUR dependencies

**Flatpak**:
- Sandboxed applications
- Cross-distro compatibility

**Git Repositories**:
- Clone and install custom tools
- Automatic updates on `switch`

**Conflict Detection**: Warns about packages from multiple sources, suggests resolutions.

**Performance Optimization**: Concurrent downloads, HashSet lookups (500x faster than sequential pacman calls).

### Health Monitoring

Continuous system validation via `reepcore doctor`:

**Checks:**
- ✓ Package integrity (verify all installed)
- ✓ Dotfile state (check symlinks, file modifications)
- ✓ Service status (ensure enabled services are running)
- ✓ Generation consistency (verify snapshots match state)
- ✓ Theme synchronization (walr colors match applications)
- ✓ Kernel status (check boot entries, initramfs)

**Auto-repair**: Can fix broken symlinks, restart failed services, regenerate configs.

### Migration Wizard

Import your existing system into ReepCore:

**Scans:**
- Explicitly installed packages (`pacman -Qe`)
- Active systemd services
- Common dotfile locations (~/.config, ~/.zshrc, etc.)
- Detected desktop environment/window manager

**Generates:**
- Initial `reepcore.yaml` with current state
- Profile suggestions based on installed packages
- Service list for systemd management

**One-command migration**: Go from manual system to declarative management.

### Kernel Management

Fine-grained control over kernel versions:

**Features:**
- Install multiple kernel packages (linux, linux-lts, linux-zen)
- Pin kernels per generation (survive updates)
- Verify initramfs integrity
- Automatic bootloader integration (GRUB, systemd-boot)
- Remove old kernels safely

**Generation-aware**: Each generation can have different kernel pins.

### File Browser

Integrated file manager with advanced features:

**Navigation:**
- Bookmarks: Home, Config, Profiles, Generations, Wallpapers, Root, Trash, Templates
- Vim-style keybindings (j/k/h/l)
- Type-to-filter

**Operations:**
- Copy/Paste (with cross-device support)
- Delete to Trash (XDG Trash spec compliant)
- Empty Trash
- Edit files (opens built-in editor)

**Preview:**
- Text files: Show content
- Directories: Show file listing
- Wallpapers: Render image (Kitty graphics protocol)

**Special Views:**
- Templates bookmark shows only profile YAMLs
- Trash shows .trashinfo metadata

### App Launcher

Launch applications from the TUI:

**Features:**
- Parses .desktop files from standard locations
- Category filtering (Accessories, Development, Games, Graphics, Internet, etc.)
- Text search (fuzzy matching)
- Icon mapping (common icon names)
- Terminal detection (launches terminal apps in kitty/alacritty/foot/wezterm)

**Integration**: Launches apps detached (won't block TUI).

---

## Integration: How It All Works Together

### The Workflow

1. **Define System** (reepcore.yaml)
   ```yaml
   profiles: [desktop, dev]
   packages:
     official: [firefox, neovim]
   ```

2. **Apply Configuration** (`reepcore switch`)
   - Reads YAML
   - Resolves profiles and dependencies
   - Installs packages (concurrent downloads)
   - Deploys dotfiles (Stow symlinking)
   - Enables/starts services
   - Creates generation snapshot
   - Validates health

3. **Change Theme** (via TUI or `reepcore theme`)
   - User selects wallpaper
   - walr extracts 10 colors
   - Generates configs for all applications
   - Applications reload instantly via file watching

4. **Verify System** (`reepcore doctor`)
   - Checks packages, dotfiles, services
   - Reports inconsistencies
   - Offers auto-repair

5. **Rollback If Needed** (`reepcore rollback`)
   - Restores previous generation
   - Reverts packages, dotfiles, services
   - System returns to known-good state

### Live Theme Synchronization

The secret to instant theme updates:

```
User sets wallpaper
         ↓
walr extracts colors → saves to ~/.config/walr/colors.json
         ↓
Applications watch colors.json (inotify)
         ↓
┌────────────────────────────────────────────┐
│       │
│  Terminal    → Reloads palette (via escape)│
│  Shell       → Regenerates prompt          │
│  Editor      → Reloads theme colors        │
│  Status Bar  → Updates bar/module colors   │
│  GTK         → Reloads theme CSS           │
└────────────────────────────────────────────┘
         ↓
Everything updates in <100ms, no restarts
```

**Key Insight**: All applications read from **one source of truth** (`colors.json`). Changes propagate automatically.

### The ReepCore Ecosystem

ReepCore is part of a larger integrated stack:

**Core System Manager:**
- `ReepCore` — Declarative orchestration (this project)

**Theme Engine:**
- `walr` — Color extraction and templating

**Desktop Environment:**
- `reepterm` — Terminal emulator with Kitty graphics protocol (in progress)
- `reepedit` — Text editor (built into ReepCore TUI)

**Shell Enhancement:**
- `zsh-rust-highlight` — Syntax highlighting in Rust
- `zsh-rust-suggest` — Autosuggestions in Rust
- `Neo` — Powerline prompt with walr colors

**All built in Rust. All walr-themed. All working together seamlessly.**

---

## Technical Details

### Language & Dependencies

**Language:** Rust (stable)

**Key Dependencies:**
- `tokio` — Async runtime for concurrent operations
- `ratatui` — Terminal UI framework
- `serde` — YAML parsing and serialization
- `anyhow` — Error handling with context
- `colored` — Terminal color output
- `notify` — File system watching (for live reload)
- `reqwest` — HTTP client (for git repos, updates)

**Build:** Single binary, no runtime dependencies (except system tools: pacman, systemd, stow)

### File Structure

```
~/.config/reepcore/
├── reepcore.yaml          # Main system configuration
├── profiles/              # Profile YAML files
│   ├── desktop.yaml
│   ├── dev.yaml
│   └── custom.yaml
├── generations/           # Generation state
│   ├── current            # Symlink to active generation
│   ├── 10/                # Generation 10 state
│   │   ├── packages.txt
│   │   ├── dotfiles.json
│   │   └── services.json
│   └── 9/                 # Generation 9 (rollback target)
└── cache/                 # Build cache, AUR packages

~/.config/walr/
└── colors.json            # Current theme colors (shared by all apps)

~/.dotfiles/               # Your dotfiles repository
├── nvim/
├── zsh/
├── waybar/
└── ...
```

### Performance Optimizations

**Concurrent Package Installation:**
- Official packages: Batch pacman calls
- AUR packages: Parallel builds (respects CPU cores)
- Flatpak: Background installation

**Smart Caching:**
- walr: SHA256-based cache (skip regeneration if wallpaper unchanged)
- Package metadata: In-memory HashSet (500x faster than subprocess calls)
- Template rendering: Only regenerate changed applications

**Efficient Dotfile Deployment:**
- Stow: Symlinks (instant, no copies)
- Change detection: Only re-stow modified packages

---

## Design Philosophy

### Declarative But Flexible

Unlike NixOS (pure functional, immutable), ReepCore is **pragmatic**:
- YAML is human-readable and version-controllable
- Imperative escape hatches when needed (manual package install still works)
- Doesn't fight the system—works with Arch's rolling release model

### Generational But Not Immutable

Generations provide safety without rigidity:
- Root filesystem is mutable (fast, familiar)
- Generations track *configuration* state, not filesystem state
- Rollback restores packages/dotfiles/services, not entire FS
- Optional Btrfs snapshots for paranoid users

### Integrated But Modular

Components work together but aren't tightly coupled:
- walr can be used standalone
- Compositor works without ReepCore
- Profiles are just YAML files (easy to share)
- Editor is embedded but accessible via `reepcore edit`

### Opinionated But Configurable

**Opinionated defaults:**
- Rust for system tools (performance, safety)
- Wayland over X11 (future-proof)
- Stow for dotfiles (simple, reliable)
- Btrfs for snapshots (CoW, compression)

**But configurable:**
- Bring your own compositor/terminal/editor
- Use any dotfile method (manual copy supported)
- Works with or without walr
- Profile system allows infinite customization

---

## Why ReepCore?

### For Arch Users

You already love Arch for its simplicity and control. But managing everything manually gets tedious:
- Remembering which AUR packages you installed
- Keeping dotfiles in sync across machines
- Theming 15+ apps consistently
- Recovering from broken updates

**ReepCore gives you NixOS-level reproducibility without leaving Arch.**

### For Ricing Enthusiasts

You've spent hours perfecting your setup. Every color matches. Every config is pristine.

**ReepCore makes that reproducible and maintainable:**
- One wallpaper change updates everything
- Share profiles with others (or across your machines)
- Never lose your perfect setup to a botched update

### For Developers

You need a reliable development environment:
- Consistent across machines
- Quick to provision (new laptop? `reepcore switch`)
- Safe to experiment (rollback if something breaks)

**ReepCore is infrastructure-as-code for your desktop.**

### For System Administrators

Managing personal machines shouldn't require different tools than managing servers:
- Declarative configuration (like Ansible)
- Health monitoring (like monitoring stacks)
- Rollback capability (like deployment pipelines)

**ReepCore brings DevOps practices to desktop management.**

---

## Status & Roadmap

### Current Status

**Stable & Production-Ready:**
- ✅ Core orchestration (switch, rollback, doctor)
- ✅ Profile system (templates, composition, custom)
- ✅ Package management (official, AUR, Flatpak, Git)
- ✅ Dotfile deployment (Stow, manual copy)
- ✅ Service management (systemd integration)
- ✅ Theme engine (walr integration)
- ✅ Full TUI (11 views, 2000+ LOC)
- ✅ Built-in editor (reepedit)
- ✅ File browser (preview, Trash, operations)
- ✅ Migration wizard (import existing system)
- ✅ Health monitoring (doctor command)
- ✅ Generation management (rollback, snapshots)

**Companion Projects:**
- ✅ walr (theme engine) — Stable
- ✅ zsh-rust-highlight — Stable
- ✅ zsh-rust-suggest — Stable
- ✅ Neo prompt — Stable

---

## Technical Requirements

**Operating System:**
- Arch Linux (or Arch-based: Manjaro, EndeavourOS, etc.)

**Architecture:**
- x86_64 (primary)
- aarch64 (untested but should work)

**Display Server:**
- Wayland (for full integration)
- X11 (partial support, some features unavailable)

**Filesystem:**
- Any (Ext4, XFS, etc.)
- Btrfs (recommended for snapshot support)

**Dependencies:**
- pacman (package manager)
- systemd (init system)
- stow (dotfile deployment)
- Optional: yay or paru (AUR helper)
- Optional: flatpak (sandboxed apps)

---

## Philosophy

ReepCore is built on these principles:

**1. Integration Over Isolation**
Don't just manage packages. Manage the entire system as a cohesive whole.

**2. Declarative Over Imperative**
Define what you want, not how to get there.

**3. Safety Over Speed**
Generations and rollback ensure you can always recover.

**4. Quality Over Quantity**
One excellent feature beats ten half-baked ones.

**5. Rust Over Scripts**
Type safety, performance, and reliability matter.

**6. Users Over Features**
If it's not usable, it's not done.

---

## Comparison

### vs. NixOS
| Feature | ReepCore | NixOS |
|---------|----------|-------|
| Declarative config | ✅ | ✅ |
| Generational rollback | ✅ | ✅ |
| Package reproducibility | ⚠️ (Arch rolling) | ✅ (pinned) |
| Learning curve | Low | High |
| Flexibility | High (imperative escape) | Low (pure functional) |
| Live theme sync | ✅ | ❌ |
| Filesystem | Mutable | Immutable |

### vs. Ansible
| Feature | ReepCore | Ansible |
|---------|----------|---------|
| Declarative config | ✅ | ✅ |
| Idempotent | ✅ | ✅ |
| Rollback | ✅ | ❌ |
| Package management | ✅ (built-in) | ⚠️ (via modules) |
| Theme integration | ✅ | ❌ |
| TUI | ✅ | ❌ |
| Language | Rust | Python |

### vs. Dotfile Managers (Stow, Chezmoi, yadm)
| Feature | ReepCore | Stow/Chezmoi |
|---------|----------|--------------|
| Dotfile deployment | ✅ | ✅ |
| Package management | ✅ | ❌ |
| Service management | ✅ | ❌ |
| Theme sync | ✅ | ❌ |
| Rollback | ✅ | ❌ |
| Scope | Full system | Dotfiles only |

---


# walr — wallpaper → theme (Rust)

```
walr/
├─ Cargo.toml
├─ README.md
├─ LICENSE
├─ src/
│  ├─ main.rs
│  ├─ cli.rs
│  ├─ config.rs
│  ├─ palette/
│  │   ├─ mod.rs
│  │   ├─ extract.rs
│  │   └─ lch_kmeans.rs
│  ├─ templates.rs
│  └─ apply.rs
├─ templates/
│  ├─ alacritty.yml.hbs
│  ├─ kitty.conf.hbs
│  ├─ hyprland.conf.hbs
│  ├─ waybar.css.hbs
│  ├─ wofi.css.hbs
│  ├─ dunstrc.hbs
│  ├─ gtk.css.hbs
│  ├─ prompt.sh.hbs
│  └─ ohmyposh.json.hbs
└─ examples/
   └─ waybar-config.jsonc    ← CREATE THIS         
   
```
# 🎨 walr

A blazingly fast wallpaper-based theme generator written in Rust.

Companion to [reepfetch](https://github.com/Reep007/reepfetch).

## Features

- 🚀 **Fast**: Written in Rust for maximum performance
- 🎨 **Smart Color Extraction**: Uses perceptually uniform LCH color space with k-means clustering
- 📦 **Multiple Templates**: Alacritty, Waybar, GTK, and more
- 👁️ **Watch Mode**: Automatically regenerate themes when wallpaper changes
- 🔄 **Auto-Apply**: Optionally reload applications after generating themes
- 🎯 **Zero Config**: Works out of the box with sensible defaults

## Installation

# From source
git clone https://github.com/Reep007/walr
cd walr && cargo install --path .
```

## Quick Start

```bash
# Generate theme from wallpaper
walr /path/to/wallpaper.jpg

# Generate and apply immediately
walr /path/to/wallpaper.jpg --apply

# Watch mode - auto-regenerate on wallpaper change
walr /path/to/wallpaper.jpg --watch --apply

# Custom number of colors
walr wallpaper.jpg --colors 8

# Custom output directory
walr wallpaper.jpg --output ~/.config/my-theme
```

## Usage

```
walr [OPTIONS] <PATH>

Arguments:
  <PATH>  Path to wallpaper image or directory to watch

Options:
  -c, --colors <N>       Number of colors to extract [default: 16]
  -o, --output <PATH>    Output directory for generated files
  -a, --apply            Apply theme immediately (reload apps)
  -w, --watch            Watch mode - regenerate on wallpaper change
  -h, --help             Print help
  -V, --version          Print version
```

## How It Works

1. **Color Extraction**: Resizes image for performance, then uses k-means clustering in perceptually uniform LCH color space
2. **Template Generation**: Applies extracted colors to Handlebars templates
3. **Theme Application**: Optionally reloads configured applications

## Generated Files

By default, reepwal generates these files in `~/.config/walr/`:

- `colors.json` - All extracted colors in JSON format
- `alacritty.yml` - Alacritty terminal colors
- `kitty.conf` - Kitty terminal colors
- `hyprland.conf` - Hyprland colors (borders, shadows, groups)
- `waybar.css` - Waybar styling
- `wofi.css` - Wofi launcher styling
- `dunstrc` - Dunst notification colors
- `gtk.css` - GTK theme colors
- `prompt.sh` - Custom zsh/bash powerline prompt
- `ohmyposh.json` - Oh My Posh theme (optional)
- `waybar-config.jsonc` - Example Waybar config (ready to use)

## Configuration

Link generated files to your app configs:

```bash
# Alacritty
ln -sf ~/.config/walr/alacritty.yml ~/.config/alacritty/colors.yml

# Kitty
echo "include ~/.config/walr/kitty.conf" >> ~/.config/kitty/kitty.conf

# Hyprland
echo "source = ~/.config/walr/hyprland.conf" >> ~/.config/hypr/hyprland.conf

# Waybar
# Add to waybar/style.css:
@import "~/.config/walr/waybar.css";

# Copy the example config:
cp ~/.config/walr/waybar-config.jsonc ~/.config/waybar/config.jsonc

# Reload waybar:
pkill -SIGUSR2 waybar

# Wofi
wofi --style ~/.config/walr/wofi.css
# Or symlink:
ln -sf ~/.config/walr/wofi.css ~/.config/wofi/style.css

# Dunst
ln -sf ~/.config/walr/dunstrc ~/.config/dunst/dunstrc
# Then restart dunst:
killall dunst && dunst &

# GTK
ln -sf ~/.config/walr/gtk.css ~/.config/gtk-3.0/gtk.css

# Zsh Prompt (recommended)
echo "source ~/.config/walr/prompt.sh" >> ~/.zshrc

# Bash Prompt (also works)
echo "source ~/.config/walr/prompt.sh" >> ~/.bashrc

# Oh My Posh (if you prefer)
eval "$(oh-my-posh init zsh --config ~/.config/walr/ohmyposh.json)"
```

## Custom Templates

Create your own templates using Handlebars syntax:

```handlebars
/* My custom template */
background: {{special.background}};
foreground: {{special.foreground}};
color0: {{colors.color0}};
```

### Custom Shell Prompt

The generated `prompt.sh` works with both **zsh** and **bash** and includes:

- 📁 Current directory in accent color
- 🌿 Git branch with status (changes, ahead/behind)
- 🦀 Auto-detects: Rust, Node, Python, Go projects
- ⏱️ Execution time (for commands >1s)
- ✓/✗ Status indicator
- 🎨 Powerline-style separators

No external dependencies needed!

## Comparison with pywal

| Feature | reepwal | pywal |
|---------|---------|-------|
| Language | Rust | Python |
| Speed | 🚀🚀🚀 | 🚀 |
| Color Space | LCH (perceptual) | RGB |
| Watch Mode | ✅ | ❌ |
| Dependencies | Single binary | Python + PIL |

## Supported Applications

- ✅ Alacritty
- ✅ Kitty
- ✅ Hyprland
- ✅ Waybar
- ✅ Wofi
- ✅ Dunst
- ✅ GTK
- ✅ Zsh/Bash (custom powerline prompt)



## Credits

**Author:** Reep    https://www.youtube.com/@reep0074

**Built With:**
- Rust programming language
- ratatui (TUI framework)
- Smithay (Wayland compositor library)
- The Arch Linux community

**Inspired By:**
- NixOS (declarative configuration)
- Pywal (theme generation)
- ML4W My Linux four Work ✅


*"A system should be more than the sum of its parts. ReepCore makes it so."*
