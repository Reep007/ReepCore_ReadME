<div align="center">
<img width="5240" height="2600" alt="reepcore_readme_banner_v2" src="https://github.com/user-attachments/assets/8cefb073-2704-4507-840d-39c29217b1ec" />

# ReepCore

### Your machine. Formalized.

*A cohesive, AI-powered Linux environment built around one source of truth.*

---

**Declarative System Management • Local AI • Wayland Desktop • Development Environment • Atomic Rollbacks**

</div>

---

# Screenshots



<img width="2560" height="1440" alt="image" src="https://github.com/user-attachments/assets/b07404ab-37a7-463a-bc89-5f40efdd5e75" />

<img width="2560" height="1440" alt="image" src="https://github.com/user-attachments/assets/127bf7c3-f997-4604-8685-d6a60ccbacae" />

<img width="2563" height="1441" alt="image" src="https://github.com/user-attachments/assets/4251eb26-055d-4b16-b17a-5d07ec33f06f" />

<img width="2562" height="1441" alt="image" src="https://github.com/user-attachments/assets/272a0b05-c2a2-44d9-8a2d-c2e89d86b4d4" />

<img width="2560" height="1440" alt="image" src="https://github.com/user-attachments/assets/6a665b5f-af3c-4790-89d7-2ebe95daa813" />


---

# What is ReepCore?

ReepCore is a complete Linux ecosystem.

Not a dotfiles repository.

Not another Hyprland setup.

Not an AI chatbot.

Not a package manager.

ReepCore brings system management, desktop configuration, development tools, and local AI together into one cohesive environment.

Every component is designed to work together instead of being assembled from unrelated projects.

> **One machine. One philosophy. One source of truth.**

---

# Why?

Most Linux desktops slowly become collections of disconnected tools.

```
Wallpaper Generator
        │
Dotfiles
        │
Launcher
        │
Status Bar
        │
Shell Plugins
        │
AI Assistant
        │
Package Manager
```

Every piece has its own configuration.

Its own cache.

Its own update cycle.

Its own philosophy.

Eventually...

Nobody remembers how it all fits together.

---

ReepCore takes a different approach.

```
                  REEPCORE
                      │
      ┌───────────────┼────────────────┐
      │               │                │
    State          Outputs         Control
      │               │                │
 Profiles       Generated Files      CLI
 Themes          Colors              TUI
 Generations     Cache              Jarvis
 Modules         Runtime            Automation
```

Everything has a single owner.

Everything has one place to live.

Everything works together.

---

# The ReepCore Ecosystem

| Component | Description |
|-----------|-------------|
| 🧠 **REEPCORE** | Declarative system management with generations and rollback |
| 🤖 **Jarvis** | Local AI assistant with deep system awareness |
| 🖥 **ReepCompositor** | Custom Wayland compositor |
| 🎨 **walr** | Wallpaper-driven color engine |
| 🖼 **reepaper** | Native Wayland wallpaper daemon |
| 📂 **reef** | File manager |
| ✏️ **reepedit** | Built-in editor |
| 🚀 **reep-launcher** | Application launcher |
| 📊 **reepfetch** | System renderer aware of REEPCORE state |
| 💻 **reep-shell** | Native status bar |
| ⚡ **zsh-rust-plugins** | Rust-powered shell enhancements |

---

# Meet Jarvis

Jarvis isn't another chat window.

It is part of ReepCore.

It understands your machine because it lives inside your machine.



Jarvis understands:

✔ Running system

✔ Installed packages

✔ Active profiles

✔ Theme state

✔ Generation history

✔ Repository structure

✔ Open files

✔ Embedded editor

✔ Embedded terminal

✔ Security monitoring

✔ Network monitoring

✔ Documentation

✔ Local RAG indexes

✔ Git worktrees

✔ JetBrains MCP

Everything runs locally through Ollama.

No cloud required.

---

# Built for Developers

Development happens without leaving ReepCore.


Included workflows:

- Embedded editor
- Interactive terminal
- Unified diff preview
- Git sandbox worktrees
- Automatic patch review
- Local RAG
- RustRover MCP integration
- AI code review
- Multi-model routing
- Semantic repository search

The goal isn't replacing your IDE.

The goal is making your operating environment understand your projects.

---

# Declarative System Management

Inspired by declarative systems while remaining pure Arch Linux.

```
reepcore switch

reepcore rollback

reepcore migrate

reepcore doctor

reepcore theme apply

reepcore tui
```

Configuration becomes reproducible.

Rollbacks become effortless.

Your machine becomes understandable.

---

# Theme Engine

One wallpaper.

One command.

Everything updates.

---

![Theme](Preview_images/theme_manager.png)

---

walr generates:

- Terminal colors

- GTK themes

- Wayland colors

- Application themes

- Shell colors

- Generated outputs

All managed through ReepCore.

---

# Real-Time Monitoring

Your desktop is also your operations center.

---

![Monitoring](Preview_images/monitoring.png)

---

Monitor:

- CPU
- Memory
- Network
- Services
- ntopng
- Fail2Ban
- Suricata
- Security events

Without leaving the TUI.

---

# Local-First AI

Privacy isn't an option.

It's the default.

✔ Local Ollama

✔ Local embeddings

✔ Local RAG

✔ Local models

✔ Local voice

✔ Local Git

✔ Local terminal

✔ Local editing

No subscriptions.

No APIs required.

---

# Architecture

```
                         ReepCore
                             │
       ┌─────────────────────┼────────────────────┐
       │                     │                    │
   System State         Desktop Stack      Development
       │                     │                    │
 Profiles             ReepCompositor      reepedit
 Modules              walr                Jarvis
 Rollbacks            reepaper            RAG
 Packages             reep-shell          Sandbox
 Generations          launcher            MCP
```

Everything answers one question:

> **Am I State, Output, or Control?**

---

# Quick Start

```bash
git clone https://github.com/Reep007/ReepCore

cd ReepCore

./bootstrap.sh --yes

reepcore init --template reepcore-setup

reepcore switch

sudo reboot
```

---

# Philosophy

ReepCore is intentionally opinionated.

It isn't trying to support every workflow.

It isn't trying to become another Linux distribution.

It isn't trying to replace Arch Linux.

Instead...

It formalizes one complete Linux environment into a cohesive system where every component understands every other component.

That is the idea behind ReepCore.



# Roadmap

- ✅ Declarative system management
- ✅ Atomic rollbacks
- ✅ Module manager
- ✅ Theme engine
- ✅ Wayland compositor
- ✅ Local AI assistant
- ✅ Embedded editor
- ✅ Embedded terminal
- ✅ Git sandbox
- ✅ Patch preview
- ✅ Local RAG
- ✅ Security monitoring
- ✅ JetBrains MCP
- 🚧 Multi-agent workflows
- 🚧 Remote node management
- 🚧 Plugin system

---

<div align="center">

## ReepCore

**Your machine. Formalized.**

*Built with Rust • Powered by Arch Linux • Enhanced by Local AI*

</div>
