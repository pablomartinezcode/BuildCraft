# BuildCraft Engine Texture Mapping (1.12) + Fabric 1.21.1 Strategy

This document records how BuildCraft (1.12-era code in this repo) builds the engine models, maps textures to each engine type, and switches the animated trunk textures by power stage. It also outlines how to reproduce this behavior in Fabric 1.21.1.

---

## 1.12 BuildCraft: Where engine textures and models are defined

### A. Engine blockstate and base “fallback” models

**Engine blockstate (type → model mapping):**
- `buildcraft_resources/assets/buildcraftcore/blockstates/engine.json`
  - `variants.type` maps engine subtype (`wood`, `stone`, `iron`, `creative`, `rf`) to model locations:
    - `buildcraftcore:engine/wood`
    - `buildcraftcore:engine/stone`
    - `buildcraftcore:engine/iron`
    - `buildcraftcore:engine/creative`
    - `buildcraftcore:engine/rf`
  - This is a standard Forge 1.12 blockstate. These models are **simple cube placeholders** used as the baked block model (not the animated TESR model).

**Static cube models for fallback / particles:**
- `buildcraft_resources/assets/buildcraftcore/models/block/engine/wood.json`
- `buildcraft_resources/assets/buildcraftcore/models/block/engine/stone.json`
- `buildcraft_resources/assets/buildcraftcore/models/block/engine/iron.json`
- `buildcraft_resources/assets/buildcraftcore/models/block/engine/creative.json`
- `buildcraft_resources/assets/buildcraftcore/models/block/engine/rf.json`
  - All are `parent: block/cube_all` and use **only the “back” texture** for all faces.
  - Example (`engine/wood.json`):
    - `textures.all = buildcraftcore:blocks/engine/wood/back`
  - These are likely used as the base blockstate model and for particles (Forge 1.12 still requires a blockstate model even when the real render is done by a TESR / FastTESR).

### B. Actual animated engine model (shared geometry)

**Shared engine geometry:**
- `buildcraft_resources/assets/buildcraftlib/models/block/engine_base.json`
  - This is the **real model** used by the TESR-style renderer.
  - It defines **engine geometry + animation variables**:
    - **Variables**
      - `progress_size`: scales/moves the piston/chamber elements
      - `trunk_tex`: `'#trunk_' + stage`
      - `stage_light`: brightness boost for hot stages
    - **Rules**
      - `direction != Facing.UP` → `builtin:rotate_facing` (rotates model based on facing)
    - **Elements / Faces**
      - `base` and `base_moving`
        - use `#back` on up/down and `#side` on the four sides
      - `trunk`
        - uses `trunk_tex` and light `stage_light`
        - UVs map to the colored part of the trunk texture
      - `chamber`
        - uses `#chamber` for the piston chamber strip

**Textures referenced by `engine_base.json`:**
- Trunk (power stage indicator):
  - `buildcraft_resources/assets/buildcraftlib/textures/blocks/engine/trunk_blue.png`
  - `trunk_green.png`
  - `trunk_yellow.png`
  - `trunk_red.png`
  - `trunk_overheat.png`
  - `trunk_black.png` → **alias** to `#trunk_overheat` via the model (`#trunk_black: "#trunk_overheat"`)
- Chamber (piston):
  - `buildcraft_resources/assets/buildcraftlib/textures/blocks/engine/chamber_base.png`

**Key mapping behavior in the model:**
- `trunk_tex = '#trunk_' + stage` means the **texture chosen for the trunk comes directly from the `stage` variable** (a string value; see “Power stage” below).
- `stage_light` boosts emissive light in red/yellow/overheat stages.

### C. Engine-type-specific wrappers (bind #back and #side)

Each engine type “wraps” the base model and defines **which side/back textures** to use.

**BuildCraft Core engines:**
- Redstone (wood) engine:
  - `buildcraft_resources/assets/buildcraftcore/models/block/engine_redstone.json`
  - Binds:
    - `#back = buildcraftcore:blocks/engine/wood/back`
    - `#side = buildcraftcore:blocks/engine/wood/side`
  - Parent: `buildcraftlib:models/block/engine_base`

- Creative engine:
  - `buildcraft_resources/assets/buildcraftcore/models/block/engine_creative.json`
  - Binds:
    - `#back = buildcraftcore:blocks/engine/creative/back`
    - `#side = buildcraftcore:blocks/engine/creative/side`
  - Parent: `buildcraftlib:models/block/engine_base`

**BuildCraft Energy engines:**
- Stone engine:
  - `buildcraft_resources/assets/buildcraftenergy/models/block/engine_stone.json`
  - Binds:
    - `#back = buildcraftenergy:blocks/engine/stone/back`
    - `#side = buildcraftenergy:blocks/engine/stone/side`

- Iron engine:
  - `buildcraft_resources/assets/buildcraftenergy/models/block/engine_iron.json`
  - Binds:
    - `#back = buildcraftenergy:blocks/engine/iron/back`
    - `#side = buildcraftenergy:blocks/engine/iron/side`

- RF engine:
  - `buildcraft_resources/assets/buildcraftenergy/models/block/engine_rf.json`
  - Binds:
    - `#back = buildcraftenergy:blocks/engine/rf/back`
    - `#side = buildcraftenergy:blocks/engine/rf/side`

### D. Where textures live (explicit listing)

**Wood (redstone) engine:**
- `buildcraft_resources/assets/buildcraftcore/textures/blocks/engine/wood/back.png`
- `buildcraft_resources/assets/buildcraftcore/textures/blocks/engine/wood/side.png`

**Creative engine:**
- `buildcraft_resources/assets/buildcraftcore/textures/blocks/engine/creative/back.png`
- `buildcraft_resources/assets/buildcraftcore/textures/blocks/engine/creative/side.png`

**Stone engine:**
- `buildcraft_resources/assets/buildcraftenergy/textures/blocks/engine/stone/back.png`
- `buildcraft_resources/assets/buildcraftenergy/textures/blocks/engine/stone/side.png`

**Iron engine:**
- `buildcraft_resources/assets/buildcraftenergy/textures/blocks/engine/iron/back.png`
- `buildcraft_resources/assets/buildcraftenergy/textures/blocks/engine/iron/side.png`

**RF engine:**
- `buildcraft_resources/assets/buildcraftenergy/textures/blocks/engine/rf/back.png`
- `buildcraft_resources/assets/buildcraftenergy/textures/blocks/engine/rf/side.png`

**Shared engine internals:**
- Trunk textures (power stage):
  - `buildcraft_resources/assets/buildcraftlib/textures/blocks/engine/trunk_blue.png`
  - `trunk_green.png`
  - `trunk_yellow.png`
  - `trunk_red.png`
  - `trunk_overheat.png`
  - `trunk_black.png` (via alias in the model)
- Chamber texture:
  - `buildcraft_resources/assets/buildcraftlib/textures/blocks/engine/chamber_base.png`

### E. How power stage controls the trunk texture

**Power stage source:**
- `common/buildcraft/lib/engine/TileEngineBase_BC8.java`
  - `computePowerStage()` maps heat → stage:
    - `< 0.25` → BLUE
    - `< 0.5` → GREEN
    - `< 0.75` → YELLOW
    - `< 0.85` → RED
    - otherwise → OVERHEAT
- `TileEngineCreative.java` overrides `computePowerStage()` to always return **BLACK**.

**Model variables set by the renderer:**
- `common/buildcraft/core/BCCoreModels.java` and `common/buildcraft/energy/BCEnergyModels.java`
  - Sets `ENGINE_STAGE.value = tile.getPowerStage()`
  - Sets `ENGINE_PROGRESS.value = tile.getProgressClient(partialTicks)`
  - Sets `ENGINE_FACING.value = tile.getCurrentFacing()`

**How stage maps to textures:**
- `engine_base.json` uses `trunk_tex = '#trunk_' + stage`.
- The `stage` string is derived from `EnumPowerStage::getName` in expression handling (`common/buildcraft/lib/misc/ExpressionCompat.java`).
- Therefore, each power stage maps to a trunk texture with a matching suffix:
  - `BLUE` → `trunk_blue.png`
  - `GREEN` → `trunk_green.png`
  - `YELLOW` → `trunk_yellow.png`
  - `RED` → `trunk_red.png`
  - `OVERHEAT` → `trunk_overheat.png`
  - `BLACK` (creative engine) → `trunk_black` alias → `trunk_overheat.png`

### F. How the model is rendered in 1.12 (TESR / FastTESR)

**Render path:**
- The engine blocks use **FastTESR** rendering (not standard block models).
- Base renderer:
  - `common/buildcraft/lib/client/render/tile/RenderEngine_BC8.java`
  - This renders quads from `ModelHolderVariable` and applies light/shading.

**Per-engine renderers:**
- `common/buildcraft/core/client/render/RenderEngineWood.java`
  - calls `BCCoreModels.getRedstoneEngineQuads(...)`
- `common/buildcraft/core/client/render/RenderEngineCreative.java`
  - calls `BCCoreModels.getCreativeEngineQuads(...)`
- `common/buildcraft/energy/client/render/RenderEngineStone.java`
  - calls `BCEnergyModels.getStoneEngineQuads(...)`
- `common/buildcraft/energy/client/render/RenderEngineIron.java`
  - calls `BCEnergyModels.getIronEngineQuads(...)`
- `common/buildcraft/energy/client/render/RenderEngineRF.java`
  - calls `BCEnergyModels.getRfEngineQuads(...)`

**Model holder creation:**
- `BCCoreModels` and `BCEnergyModels` create `ModelHolderVariable` with:
  - `engine_redstone.json`, `engine_creative.json`, `engine_stone.json`, `engine_iron.json`, `engine_rf.json`.
  - These all parent to `engine_base.json`, which contains the animation rules and stage-based trunk texture mapping.

**Item model baking:**
- `BCCoreModels.onModelBake` and `BCEnergyModels.onModelBake` create **item models** from the engine quads.
  - They set `ENGINE_PROGRESS`, `ENGINE_STAGE`, and `ENGINE_FACING` to fixed values (progress 0.2, stage BLUE, facing UP; creative uses stage BLACK).
  - They bake a static item model into the model registry for inventory rendering.

---

## How this works end-to-end (1.12 data flow summary)

1. **Block state** chooses a simple cube model for the engine type (wood/stone/iron/creative/rf).
2. **Tile entity** (`TileEngineBase_BC8`) holds engine state:
   - `powerStage` (based on heat)
   - `progress` (piston stroke, 0→1)
   - `currentDirection` (facing)
3. **FastTESR** (`RenderEngine_BC8`) requests dynamic quads from the model system.
4. **ModelHolderVariable** applies variables (`progress`, `stage`, `direction`) to `engine_base.json`.
5. The model selects textures:
   - **#back / #side** come from per-engine model wrappers.
   - **Trunk** uses `#trunk_<stage>`.
   - **Chamber** uses `chamber_base`.
6. The renderer emits quads with lighting and shading.

---

## Fabric 1.21.1: How to replicate this mapping & animation

Below is a direct translation of the BuildCraft approach into a Fabric 1.21.1 architecture.

### 1) Block and BlockState setup (engine type & facing)

**Use a blockstate with engine type and facing:**
- Properties:
  - `type` → enum like `WOOD`, `STONE`, `IRON`, `CREATIVE`, `RF`
  - `facing` → `Direction`
- You can still provide a simple cube model for particles and fallback, similar to BuildCraft’s blockstate models:
  - Each engine type can map to a cube model that uses the *back* texture as `all` faces (matches 1.12).

**Why this matters:**
- Fabric block models are still used for particles and some rendering contexts. Even with a BER (Block Entity Renderer), Minecraft expects a baked block model for the blockstate.

### 2) BlockEntity state (progress + power stage)

**BlockEntity fields to track:**
- `float progress` (0–1 piston cycle)
- `EnumPowerStage stage` (same mapping as 1.12: BLUE, GREEN, YELLOW, RED, OVERHEAT; optional BLACK for creative)
- `Direction facing`

**Power stage thresholds:**
- Reuse the 1.12 logic from `TileEngineBase_BC8.computePowerStage()`:
  - heat < 0.25 → BLUE
  - heat < 0.5 → GREEN
  - heat < 0.75 → YELLOW
  - heat < 0.85 → RED
  - else → OVERHEAT
- Creative engine: always stage BLACK.

### 3) Texture mapping in Fabric

**Match BuildCraft texture layout:**
- Keep the *same* texture structure: back/side per engine type + shared trunk/chamber textures.

**Textures to preserve:**
- `blocks/engine/<type>/back.png`
- `blocks/engine/<type>/side.png`
- `blocks/engine/trunk_blue.png`
- `blocks/engine/trunk_green.png`
- `blocks/engine/trunk_yellow.png`
- `blocks/engine/trunk_red.png`
- `blocks/engine/trunk_overheat.png`
- `blocks/engine/trunk_black.png` (alias in 1.12; can just use the same asset as overheat if needed)
- `blocks/engine/chamber_base.png`

### 4) Rendering strategy in Fabric 1.21.1

There are two viable approaches. The closest to BuildCraft is a **BlockEntityRenderer** that builds quads dynamically based on `progress`, `stage`, and `facing`.

#### Option A: BlockEntityRenderer (closest to 1.12 TESR)

**How it maps to BuildCraft:**
- `RenderEngine_BC8` ⇒ Fabric `BlockEntityRenderer`.
- `ModelHolderVariable` ⇒ your own code that builds a small mesh (a few cuboids) and picks textures based on stage and engine type.

**Implementation approach:**
1. **Pre-bake the engine geometry** (the same 4 elements: `base`, `base_moving`, `trunk`, `chamber`).
2. On each render call:
   - Compute `progress_size` like in `engine_base.json`.
   - Pick trunk texture: `trunk_<stage>`.
   - Choose `#back` and `#side` textures from `type`.
   - Apply rotation from `facing`.
3. Render with `VertexConsumer` from `RenderLayer.getCutout()` (or `getSolid()` if no alpha).

**Benefits:**
- Full control of animation, lighting, and texture switches.
- Direct mapping to BuildCraft’s logic.

#### Option B: Fabric baked model / dynamic model

**How it maps to BuildCraft:**
- `ModelHolderVariable` is essentially a **dynamic model**. In Fabric you can implement `UnbakedModel`/`BakedModel` or use the Fabric Renderer API to output a mesh per frame.

**Implementation approach:**
- Create a `BakedModel` that reads from a `ModelProperty` (via `BlockEntityRenderData` / `BlockEntityRendererFactory.Context` or `BlockEntityRenderDispatcher`) to access stage/progress and produce quads accordingly.
- More complex but keeps rendering in the standard block model pipeline.

### 5) Exact texture mapping rules (1:1 with 1.12)

**Base elements:**
- `base` and `base_moving`:
  - Up/down faces use the **back texture**.
  - Side faces use the **side texture**.

**Trunk element (center column):**
- Uses **trunk texture based on power stage**:
  - BLUE → `trunk_blue`
  - GREEN → `trunk_green`
  - YELLOW → `trunk_yellow`
  - RED → `trunk_red`
  - OVERHEAT → `trunk_overheat`
  - BLACK (creative) → `trunk_black` (or alias to overheat)

**Chamber element (piston strip):**
- Uses `chamber_base` texture.
- Moves vertically according to `progress_size`.

**Facing / rotation:**
- Rotate the model from UP to `facing` (same as `builtin:rotate_facing`).

### 6) Suggested file layout for Fabric 1.21.1

You can keep the same texture naming to simplify migration:

```
resources/assets/<modid>/textures/blocks/engine/<type>/back.png
resources/assets/<modid>/textures/blocks/engine/<type>/side.png
resources/assets/<modid>/textures/blocks/engine/trunk_blue.png
resources/assets/<modid>/textures/blocks/engine/trunk_green.png
resources/assets/<modid>/textures/blocks/engine/trunk_yellow.png
resources/assets/<modid>/textures/blocks/engine/trunk_red.png
resources/assets/<modid>/textures/blocks/engine/trunk_overheat.png
resources/assets/<modid>/textures/blocks/engine/trunk_black.png
resources/assets/<modid>/textures/blocks/engine/chamber_base.png
```

### 7) Important notes about “when” each texture is used

**Per-engine textures (back/side):**
- **Always used** for the base and base_moving elements.
- Selected by engine type:
  - WOOD → `buildcraftcore:blocks/engine/wood/{back,side}`
  - STONE → `buildcraftenergy:blocks/engine/stone/{back,side}`
  - IRON → `buildcraftenergy:blocks/engine/iron/{back,side}`
  - RF → `buildcraftenergy:blocks/engine/rf/{back,side}`
  - CREATIVE → `buildcraftcore:blocks/engine/creative/{back,side}`

**Trunk textures (power stage):**
- Determined from `TileEngineBase_BC8.powerStage` which is recomputed from heat.
- Used in **every render call** to pick trunk color.

**Chamber texture:**
- Always `chamber_base`.
- Moves with `progress_size` to show piston action.

**Facing:**
- The whole model rotates so the trunk aligns with `currentDirection`.

---

## Quick reference: key files (1.12 code & resources)

**Models:**
- `buildcraft_resources/assets/buildcraftlib/models/block/engine_base.json`
- `buildcraft_resources/assets/buildcraftcore/models/block/engine_redstone.json`
- `buildcraft_resources/assets/buildcraftcore/models/block/engine_creative.json`
- `buildcraft_resources/assets/buildcraftenergy/models/block/engine_stone.json`
- `buildcraft_resources/assets/buildcraftenergy/models/block/engine_iron.json`
- `buildcraft_resources/assets/buildcraftenergy/models/block/engine_rf.json`

**Blockstate:**
- `buildcraft_resources/assets/buildcraftcore/blockstates/engine.json`

**Textures:**
- `buildcraft_resources/assets/buildcraftcore/textures/blocks/engine/*`
- `buildcraft_resources/assets/buildcraftenergy/textures/blocks/engine/*`
- `buildcraft_resources/assets/buildcraftlib/textures/blocks/engine/*`

**Render & model binding:**
- `common/buildcraft/lib/client/render/tile/RenderEngine_BC8.java`
- `common/buildcraft/core/BCCoreModels.java`
- `common/buildcraft/energy/BCEnergyModels.java`
- `common/buildcraft/lib/engine/TileEngineBase_BC8.java`
- `common/buildcraft/core/tile/TileEngineCreative.java`

---

## Summary

BuildCraft 1.12 uses a **single shared animated model** (`engine_base.json`) with variable-driven texture selection and movement. Engine type textures (back/side) are injected by per-engine wrapper models, while the **trunk texture is chosen dynamically based on power stage**. The model is rendered via FastTESR, not the normal block model pipeline. For Fabric 1.21.1, the closest equivalent is a BlockEntityRenderer that constructs the same geometry and switches textures based on type, stage, and progress, preserving the exact texture layout and stage-to-trunk mapping defined in 1.12.
