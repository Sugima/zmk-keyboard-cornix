# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

# AI-DLC and Spec-Driven Development

Kiro-style Spec Driven Development implementation on AI-DLC (AI Development Life Cycle)

## Project Context

### Paths
- Steering: `.kiro/steering/`
- Specs: `.kiro/specs/`

### Steering vs Specification

**Steering** (`.kiro/steering/`) - Guide AI with project-wide rules and context
**Specs** (`.kiro/specs/`) - Formalize development process for individual features

### Active Specifications
- Check `.kiro/specs/` for active specifications
- Use `/kiro:spec-status [feature-name]` to check progress

## Development Guidelines
- Think in English, but generate responses in Japanese (思考は英語、回答の生成は日本語で行うように)

## Minimal Workflow
- Phase 0 (optional): `/kiro:steering`, `/kiro:steering-custom`
- Phase 1 (Specification):
  - `/kiro:spec-init "description"`
  - `/kiro:spec-requirements {feature}`
  - `/kiro:validate-gap {feature}` (optional: for existing codebase)
  - `/kiro:spec-design {feature} [-y]`
  - `/kiro:validate-design {feature}` (optional: design review)
  - `/kiro:spec-tasks {feature} [-y]`
- Phase 2 (Implementation): `/kiro:spec-impl {feature} [tasks]`
  - `/kiro:validate-impl {feature}` (optional: after implementation)
- Progress check: `/kiro:spec-status {feature}` (use anytime)

## Development Rules
- 3-phase approval workflow: Requirements → Design → Tasks → Implementation
- Human review required each phase; use `-y` only for intentional fast-track
- Keep steering current and verify alignment with `/kiro:spec-status`

## Steering Configuration
- Load entire `.kiro/steering/` as project memory
- Default files: `product.md`, `tech.md`, `structure.md`
- Custom files are supported (managed via `/kiro:steering-custom`)

---

## Project Overview

This repository contains ZMK (Zephyr Mechanical Keyboard) firmware for the Cornix split keyboard - a Corne-inspired 3×6 column-staggered ergonomic keyboard with adjustable tenting. The project provides full split-role configuration, battery power management, and Bluetooth central/peripheral setup.

## Build System

### Primary Build Tool: Just

The project uses **Just** (command runner) for all build operations. All build commands are defined in `Justfile`.

**Environment Variables:**
- `ZMK_LIB_PREFIX`: Path to ZMK library parent directory (default: `zmk_exts`)
  - ZMK base is at `${ZMK_LIB_PREFIX}/zmk/app`

**Common Commands:**
```bash
# List all available commands
just

# Initialize west workspace
just init

# Update west dependencies
just update

# Build firmware for specific target (uses build.yaml)
just build <expr>    # expr: filter expression (e.g., "cornix_left", "dongle", "all")

# List all build targets
just list

# Clean build artifacts
just clean

# Generate keymap visualization
just draw <keyboard>    # keyboard: "cornix" or "cornix42"
```

### Build Configuration

**build.yaml:** Defines all firmware build targets (GitHub Actions matrix)
- Multiple board/shield/snippet combinations
- Generates artifacts for left/right halves, dongle variants, and reset firmware

**Key Targets:**
- `cornix_left_default`: Left half (standalone mode)
- `cornix_left_for_dongle`: Left half (peripheral mode for dongle)
- `cornix_right`: Right half
- `cornix_dongle`: Dongle adapter (nice_nano_v2 or seeeduino_xiao_ble)
- `reset`: Settings reset firmware

### West Workspace Management

**west.yml dependencies:**
- `zmkfirmware/zmk` (main ZMK firmware)
- `urob/zmk-helpers` (ZMK helper utilities)
- `hitsmaxft/zmk-rgbled-widget` (RGB LED support)
- `hitsmaxft/zmk-dongle-display` (Dongle display module)
- `janpfischer/zmk-dongle-screen` (Alternative display module)

**Important:** Never set `ZEPHYR_BASE` environment variable - use `Zephyr_DIR` instead (pointing to `zephyr/share/zephyr-package/cmake/`)

### Nix Development Environment

The project uses Nix flakes for reproducible development environments:

```bash
# Enter development shell
nix develop

# Update Zephyr SDK
just upgrade-sdk
```

**Included tools:** Zephyr SDK (arm-zephyr-eabi), west, cmake, ninja, just, yq, keymap-drawer

## Architecture

### Directory Structure

```
boards/
├── arm/cornix/              # Main board definitions (nRF52840-based)
│   ├── cornix_left.dts      # Left half device tree
│   ├── cornix_right.dts     # Right half device tree
│   ├── cornix_ph_left.dts   # Peripheral left (for dongle mode)
│   ├── cornix_dongle.dts    # Dongle board definition
│   ├── cornix.dtsi          # Common device tree includes
│   ├── cornix-layouts.dtsi  # Physical layout definitions
│   ├── cornix-pinctrl.dtsi  # Pin control mappings
│   ├── cornix_sensors.dtsi  # Battery/sensor configuration
│   └── nrf_e73.dtsi         # nRF52840 E73 module specifics
└── shields/                 # Shield overlays
    ├── cornix_dongle_adapter/     # Dongle matrix/BT adapter
    ├── cornix_dongle_eyelash/     # Eyelash dongle display config
    └── cornix_indicator/          # RGB indicator support

config/
├── cornix.keymap            # Default keymap (52 keys)
├── cornix42.keymap          # 42-key layout variant
└── west.yml                 # West manifest (dependencies)

keymap-drawer/               # Keymap visualization configs
├── cornix.yaml              # Generated keymap data
└── config-cornix.yaml       # Keymap-drawer configuration
```

### Hardware Architecture

**Split Keyboard Topology:**
- **Standalone Mode:** Left half = central (master), right half = peripheral
- **Dongle Mode:** Dongle = central, both halves = peripherals
  - Use `cornix_ph_left` board for left half in dongle mode
  - Use `cornix_left` board for standalone mode

**Flash Partitions:**
- Since v2.3: No-SoftDevice (nosd) layout by default
- SoftDevice partition reduced from 150K to 4K
- Stock RMK firmware used nosd layout

**Connectivity:**
- Bluetooth Low Energy (split communication)
- USB-C (wired mode)
- EC11 encoder support (since v2.2)

### Shield System

The dongle configuration uses a modular shield approach:
1. **cornix_dongle_adapter**: Base shield for matrix and Bluetooth
2. **dongle_display** or **dongle_screen**: Display module (from west dependencies)
3. **cornix_dongle_eyelash**: Display device tree overlay (only if board lacks `zephyr,display`)

Example build configuration:
```yaml
- board: nice_nano_v2
  shield: cornix_dongle_adapter cornix_dongle_eyelash dongle_display
  snippet: studio-rpc-usb-uart nrf52840-nosd
```

## Keymap Development

### Keymap Files
- **config/cornix.keymap**: Full 52-key layout
- **config/cornix42.keymap**: Compact 42-key layout
- Uses urob/zmk-helpers for advanced keymap features

### Visualization Workflow

```bash
# Generate keymap visualization
just draw cornix      # or "cornix42"

# Output: keymap-drawer/cornix.svg
```

This uses the keymap-drawer tool to:
1. Parse ZMK keymap (`-z` flag for ZMK format)
2. Generate intermediate YAML representation
3. Render SVG diagram with layer visualization

## Flashing Workflow

### Initial Flash (from RMK firmware)
1. Flash `reset.uf2` to both halves
2. Flash `cornix_left_*.uf2` to left half
3. Flash `cornix_right.uf2` to right half
4. Reset both halves simultaneously

### Dongle Setup
1. Flash dongle firmware to nice_nano_v2 or seeeduino_xiao_ble
2. Use `cornix_left_for_dongle` (not `cornix_left_default`) for left half
3. Flash right half normally

## GitHub Actions

Automated builds trigger on:
- Push to `boards/*` or `config/*`
- Manual workflow dispatch

Uses official ZMK build workflow: `zmkfirmware/zmk/.github/workflows/build-user-config.yml@main`

## Important Notes

- **No-SoftDevice Warning:** Original Cornix firmware removed SoftDevice. If flashing to a dongle that previously ran SoftDevice firmware, restore SoftDevice first (see `bootloader/README.md`) or use `nrf52840-nosd` snippet
- **RGB Support:** RGB indicator shield (`cornix_indicator`) is available but consumes significant power
- **Rollback:** Stock RMK firmware backups are in `rmkfw/` directory
- **Testing:** ZMK native_posix_64 tests available via `just test <testpath>`
