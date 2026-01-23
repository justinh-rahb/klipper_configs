# Klipper Configurations

This repository contains Klipper configuration files for various 3D printers, optimized for improved user experience and feature parity.

## Printer Environments

### [Ender-3 V3 SE](file:///Users/justinh/Development/github.com/justinh-rahb/klipper_configs/e3v3se)
The Ender-3 V3 SE configuration has been significantly overhauled to align with the KE stock config.
- **Unified Macros**: Consolidated gcode macros in `gcode_macro.cfg`.
- **Persistent Z-Offset**: Saves Z-offset to `variables.cfg` and reapplies it automatically on boot.
- **Advanced Features**: Includes `M600` support with UI prompts, `screws_tilt_adjust` calibration, and `timelapse` support.
- **Parameter System**: Centralized machine parameters in `printer_params.cfg`.

### [Ender-3 V3 KE](file:///Users/justinh/Development/github.com/justinh-rahb/klipper_configs/e3v3ke)
Standard configurations for the Ender-3 V3 KE, serving as the benchmark for the SE's alignment.

### [Snapmaker U1 (lava)](file:///Users/justinh/Development/github.com/justinh-rahb/klipper_configs/u1)
A custom configuration for the Snapmaker U1, internally referred to as "lava", featuring an extended configuration structure.
- **Extended Configs**: Specialized configurations located in `u1/extended/`.

## Repository Structure

```text
.
├── e3v3ke/         # Ender-3 V3 KE Configuration
├── e3v3se/         # Ender-3 V3 SE Configuration
└── u1/             # Snapmaker U1 Configuration
    ├── extended/   # Specialized config includes
    └── ...
```

## Getting Started

Each printer directory contains its own `printer.cfg` and supporting files.