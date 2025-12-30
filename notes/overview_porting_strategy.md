# BuildCraft (1.12.2) → Fabric 1.21.1: Porting Overview & Section Index

This document is a **hub** for the multi‑report analysis set. Each subsystem has its own report under `notes/` with two emphases:

1) **Texture mapping** (how the original 1.12.2 textures map onto weird/complex shapes)
2) **Game logic** (core mechanics and how to recreate them in Fabric 1.21.1)

## Reports (per subsystem)

- `notes/core_markers_engines.md`
  - Landmarks/markers (torch‑style textures) + base engine visuals (stage‑based trunk textures)
- `notes/transport_pipes_gates.md`
  - Pipes, wires, gates, pluggables, routing logic, and dynamic rendering strategy
- `notes/energy_engines_power.md`
  - MJ engines (stone/iron/RF/wood/creative), dynamo, oil spring, power stage mechanics
- `notes/factory_machines_fluids.md`
  - Pump, Mining Well, Miner, Tank, Distiller, Heat Exchange, Flood Gate, Chute, Auto‑workbenches
- `notes/builders_quarry_builder.md`
  - Quarry, Builder, Filler, Replacer, Architect Table, Electronic Library
- `notes/silicon_lasers_assembly.md`
  - Assembly/Charging/Advanced Crafting tables, Laser + Laser Table integration
- `notes/robotics_zones_automation.md`
  - Zone Planner + robotics integration points
- `notes/textures_porting_guide.md`
  - Cross‑cutting texture/model porting guidance for “weirdly shaped” blocks

## Existing detailed notes

- `notes/quarry_analysis.md` (deep quarry internals)
- `notes/engine_textures_mapping.md` (engine model + stage texture mapping)
- `notes/torch_marker_mapping.md` (marker torch model + texture mapping)

## How to use these

Assign one report per agent. Each report references **exact file/class locations** in the 1.12.2 source tree and **specific texture/model assets** in `buildcraft_resources/`, plus a Fabric 1.21.1 approach.
