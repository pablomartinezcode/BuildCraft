# Core: Markers + Base Engines (1.12.2 logic → Fabric 1.21.1 port)

This report covers **BuildCraft Core** components that are foundational for other modules:
- **Landmark/Marker blocks** (volume + path markers)
- **Base engine visuals** (redstone/creative engines in core) and shared model/texture logic

Focus areas:
1) **Texture mapping** for marker torches and engine stages
2) **Game logic** for markers and engine power stage selection

---

## A) Markers (Volume + Path)

### 1.12.2 logic: where it lives
- Block registration: `common/buildcraft/core/BCCoreBlocks.java`
  - `markerVolume` → `buildcraftcore:marker_volume`
  - `markerPath` → `buildcraftcore:marker_path`
- Base block class: `common/buildcraft/lib/block/BlockMarkerBase.java`
  - Properties: `BuildCraftProperties.BLOCK_FACING_6` and `BuildCraftProperties.ACTIVE`
  - `getActualState()` sets `ACTIVE` based on `TileMarker.isActiveForRender()`
- Tiles:
  - `common/buildcraft/core/tile/TileMarkerVolume.java`
  - `common/buildcraft/core/tile/TileMarkerPath.java`

### Texture mapping (torch‑style)
- Blockstates:
  - `buildcraft_resources/assets/buildcraftcore/blockstates/marker_volume.json`
  - `buildcraft_resources/assets/buildcraftcore/blockstates/marker_path.json`
- Models:
  - `buildcraft_resources/assets/buildcraftcore/models/block/marker_volume.json`
  - `buildcraft_resources/assets/buildcraftcore/models/block/marker_path.json`
- Geometry parent:
  - `buildcraft_resources/assets/buildcraftcore/models/block/torch_center_lit.json`
- Textures:
  - `buildcraft_resources/assets/buildcraftcore/textures/blocks/marker_volume.png`
  - `buildcraft_resources/assets/buildcraftcore/textures/blocks/marker_path.png`

**Key mapping rule:** marker models inherit a **custom centered torch geometry** and bind a **single texture slot** (`all`). This is not vanilla torch geometry; it’s a custom block model.

### Fabric 1.21.1 port strategy
- Create a `block/torch_center_lit.json` equivalent (same geometry), and use it as the parent for marker models.
- Map textures 1:1 with original PNGs.
- Create standard 1.21.1 blockstate variants for `facing` rotations.
- Preserve the `ACTIVE` property if it affects logic; visuals can remain constant as in 1.12 (no texture swap).

---

## B) Engines (Core: Wood/Redstone + Creative)

### 1.12.2 logic: where it lives
- Base engine logic:
  - `common/buildcraft/lib/engine/TileEngineBase_BC8.java`
- Core engines:
  - `common/buildcraft/core/tile/TileEngineRedstone_BC8.java`
  - `common/buildcraft/core/tile/TileEngineCreative.java`
- Power stage selection:
  - `TileEngineBase_BC8.computePowerStage()`
  - Creative overrides to return BLACK stage

### Texture mapping (stage‑based trunk)
Detailed in `notes/engine_textures_mapping.md`. Highlights:

- Shared model with variable textures:
  - `buildcraft_resources/assets/buildcraftlib/models/block/engine_base.json`
- Core engine wrappers:
  - `buildcraft_resources/assets/buildcraftcore/models/block/engine_redstone.json`
  - `buildcraft_resources/assets/buildcraftcore/models/block/engine_creative.json`
- Trunk textures (power stage):
  - `buildcraft_resources/assets/buildcraftlib/textures/blocks/engine/trunk_*.png`
- Chamber texture:
  - `buildcraft_resources/assets/buildcraftlib/textures/blocks/engine/chamber_base.png`

**Key mapping rule:** `engine_base.json` computes `trunk_tex = '#trunk_' + stage`, so the **engine’s power stage** selects the trunk texture.

### Fabric 1.21.1 port strategy
- Implement a BER (Block Entity Renderer) for engines to replicate animated piston/trunk and stage‑based texture switching.
- Preserve model variables: `stage`, `progress`, `facing`.
- Use original PNGs for `back`/`side` textures and trunk/chamber textures.
- For item rendering, bake a static model snapshot (progress ~0.2, stage BLUE / BLACK for creative).

---

## Key porting risks / notes
- Marker model is **custom geometry**, so don’t rely on vanilla torch JSONs.
- Engine stage selection is logic‑driven; texture mapping is tightly coupled to stage string names.
- These are **core** assets used by other modules (e.g., power networks and markers for quarry frames).
