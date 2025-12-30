# Energy: MJ Engines, Dynamo, Oil Spring (1.12.2 → Fabric 1.21.1)

This report covers the **energy subsystem**, focusing on MJ engines, their textures and animation stages, plus auxiliary blocks like the MJ dynamo and oil spring.

Focus areas:
1) **Texture mapping** for animated engines
2) **Game logic** for energy generation, heat, and power stages

---

## A) Engines (Stone / Iron / RF / Wood / Creative)

### 1.12.2 logic: where it lives
- Base engine:
  - `common/buildcraft/lib/engine/TileEngineBase_BC8.java`
- Stone engine:
  - `common/buildcraft/energy/tile/TileEngineStone_BC8.java`
- Iron engine:
  - `common/buildcraft/energy/tile/TileEngineIron_BC8.java`
- RF engine:
  - `common/buildcraft/energy/tile/TileEngineRF.java`
- Wood (Redstone) engine:
  - `common/buildcraft/core/tile/TileEngineRedstone_BC8.java`
- Creative engine:
  - `common/buildcraft/core/tile/TileEngineCreative.java`

### Texture mapping (summary)
Detailed mapping in `notes/engine_textures_mapping.md`.

- Shared animated geometry: `buildcraft_resources/assets/buildcraftlib/models/block/engine_base.json`
- Engine wrappers:
  - `buildcraft_resources/assets/buildcraftcore/models/block/engine_redstone.json`
  - `buildcraft_resources/assets/buildcraftcore/models/block/engine_creative.json`
  - `buildcraft_resources/assets/buildcraftenergy/models/block/engine_stone.json`
  - `buildcraft_resources/assets/buildcraftenergy/models/block/engine_iron.json`
  - `buildcraft_resources/assets/buildcraftenergy/models/block/engine_rf.json`
- Stage textures:
  - `buildcraft_resources/assets/buildcraftlib/textures/blocks/engine/trunk_*.png`
- Chamber texture:
  - `buildcraft_resources/assets/buildcraftlib/textures/blocks/engine/chamber_base.png`

### Game logic highlights
- **Heat → power stage** mapping is in `TileEngineBase_BC8.computePowerStage()`.
- Stage names are used directly in model expressions (`trunk_tex = '#trunk_' + stage`).
- Creative engine overrides stage to BLACK.
- Engine progress drives piston motion (`ENGINE_PROGRESS`).

### Fabric 1.21.1 port strategy
- Implement MJ capability in Fabric (custom energy network or compat layer).
- Use BER rendering for animated engines with texture switching by stage.
- Preserve stage names to avoid texture mapping mismatches.
- Bake static item models with fixed stage/progress for inventory rendering.

---

## B) MJ Dynamo

### 1.12.2 logic
- Tile: `common/buildcraft/energy/tile/TileDynamoMJ.java`
- Purpose: accept energy and output MJ to connected systems (details in tile class).

### Fabric 1.21.1 port strategy
- Implement as a block entity that converts/outputs energy on a defined side.
- Port MJ buffer logic and side configuration if present in tile.

---

## C) Oil Spring

### 1.12.2 logic
- Tile: `common/buildcraft/energy/tile/TileSpringOil.java`
- Generates oil fluid in the world (spawning logic/flow in tile).

### Fabric 1.21.1 port strategy
- Implement as a world gen + block entity combination (if required by modern Fabric).
- Use the same fluid output rates and behavior as 1.12.

---

## Key porting risks / notes
- Engine animation and stage switching is a **core visual identity**; treat as high‑priority.
- MJ behavior must be consistent across subsystems (builders, factory, silicon).
