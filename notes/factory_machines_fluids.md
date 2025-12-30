# Factory: Pumps, Tanks, Refining, Automation (1.12.2 → Fabric 1.21.1)

This report covers the **factory subsystem** with machines that handle fluids, mining, and automation.

Focus areas:
1) **Texture mapping** for multiblock tanks and machine models
2) **Game logic** for fluid movement, refining, and automation

---

## A) Pumps, Mining Wells, Miners

### 1.12.2 logic: where it lives
- Pump: `common/buildcraft/factory/tile/TilePump.java`
- Mining Well: `common/buildcraft/factory/tile/TileMiningWell.java`
- Miner: `common/buildcraft/factory/tile/TileMiner.java`

### Textures / models
- Pump model: `buildcraft_resources/assets/buildcraftfactory/models/block/pump.json`
- Mining Well model: `buildcraft_resources/assets/buildcraftfactory/models/block/mining_well.json`
- Textures: `buildcraft_resources/assets/buildcraftfactory/textures/blocks/*`

### Fabric 1.21.1 port strategy
- Port pump logic to drain fluids from world and output to pipes/tanks.
- Port mining well/miner logic for vertical drilling and block extraction.
- Preserve animations via BER if needed (moving drill components).

---

## B) Tanks (multiblock visuals + fluid storage)

### 1.12.2 logic
- Tile: `common/buildcraft/factory/tile/TileTank.java`
- Multiblock joining logic is in tile + blockstate updates.

### Textures / models
- Models:
  - `buildcraft_resources/assets/buildcraftfactory/models/block/tank.json`
  - `buildcraft_resources/assets/buildcraftfactory/models/block/tank_joined_below.json`
- Item model: `buildcraft_resources/assets/buildcraftfactory/models/item/tank.json`

### Fabric 1.21.1 port strategy
- Recreate multiblock adjacency detection and dynamic model switching.
- Ensure fluid rendering inside tank blocks (custom rendering or block entity renderer).
- Reuse original tank textures with updated blockstate logic.

---

## C) Distiller / Heat Exchange (Refinery chain)

### 1.12.2 logic
- Distiller tile: `common/buildcraft/factory/tile/TileDistiller_BC8.java`
- Heat exchange tile: `common/buildcraft/factory/tile/TileHeatExchange.java`
- Recipe registry:
  - `common/buildcraft/lib/recipe/RefineryRecipeRegistry.java`
  - Accessed via `BuildcraftRecipeRegistry.refineryRecipes`

### Textures / models
- Distiller models:
  - `buildcraft_resources/assets/buildcraftfactory/models/block/distiller.json`
  - `buildcraft_resources/assets/buildcraftfactory/models/tiles/distiller.json`
- Refinery model:
  - `buildcraft_resources/assets/buildcraftfactory/models/block/refinery.json`

### Fabric 1.21.1 port strategy
- Port refinery recipe logic to Fabric recipe registries.
- Implement heat exchange linking (start/middle/end sections) as in `TileHeatExchange`.
- Recreate distiller processing logic and fluid IO.

---

## D) Flood Gate, Chute, Auto‑workbenches

### 1.12.2 logic
- Flood Gate: `common/buildcraft/factory/tile/TileFloodGate.java`
- Chute: `common/buildcraft/factory/tile/TileChute.java`
- Auto‑workbenches:
  - `common/buildcraft/factory/tile/TileAutoWorkbenchItems.java`
  - `common/buildcraft/factory/tile/TileAutoWorkbenchFluids.java`

### Fabric 1.21.1 port strategy
- Flood gate: controlled fluid placement and world fill behavior.
- Chute: gravity-based item dropping and directional control.
- Auto‑workbenches: crafting with inventory/fluid inputs + output buffering.

---

## Key porting risks / notes
- Many factory machines are **fluid‑heavy** and require accurate throughput + IO behavior.
- Multiblock tank visuals need careful model/state updates to avoid desync.
