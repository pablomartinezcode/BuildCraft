# Torch texture mapping for Landmarks (Volume Markers) and Path Markers (BuildCraft 1.12.2 → Fabric 1.21.1)

## Goal of this note
Document **exactly where and how the torch-style textures are mapped** to the **Landmark (volume marker)** and **Path Marker** blocks in this 1.12.2 repo, and describe a **Fabric 1.21.1**-compatible way to reproduce the same mapping.

---

## 1.12.2: Where the mapping happens

### 1) Block registration and identities
The two relevant blocks are registered in **BuildCraft Core** as:
- **Volume marker (Landmark):** `buildcraftcore:marker_volume`
- **Path marker:** `buildcraftcore:marker_path`

Registration happens in:
- `common/buildcraft/core/BCCoreBlocks.java`:
  - `markerVolume = HELPER.addBlockAndItem(new BlockMarkerVolume(..., "block.marker.volume"));`
  - `markerPath = HELPER.addBlockAndItem(new BlockMarkerPath(..., "block.marker.path"));`
- `common/buildcraft/core/BCCore.java` tag setup:
  - `registerTag("block.marker.volume").reg("marker_volume").model("marker_volume")`
  - `registerTag("block.marker.path").reg("marker_path").model("marker_path")`

This is why you see `marker_volume.json` and `marker_path.json` in the blockstate and model assets.

> **Legacy IDs:**
> `pathMarkerBlock` and `markerBlock` are older registry names used in previous versions. The current 1.12.2 code still declares compatibility IDs via `.oldReg("pathMarkerBlock")` / `.oldReg("markerBlock")`, which is why the resource pack also contains:
> - `buildcraft_resources/assets/buildcraftcore/blockstates/pathMarkerBlock.json`
> - `buildcraft_resources/assets/buildcraftcore/blockstates/markerBlock.json`
>
> Those are **legacy aliases** to point old names at the new models.

---

### 2) Block classes define **properties** used by blockstates
Both `BlockMarkerVolume` and `BlockMarkerPath` inherit from:
- `common/buildcraft/lib/block/BlockMarkerBase.java`

`BlockMarkerBase` defines **block state properties** for markers:
- `BuildCraftProperties.BLOCK_FACING_6` (property name in JSON is `facing`)
- `BuildCraftProperties.ACTIVE` (property name in JSON is `active`)

Key code reference:
- `createBlockState()` returns a container with `BLOCK_FACING_6` and `ACTIVE`.
- `getActualState()` sets `ACTIVE` based on `TileMarker.isActiveForRender()`.

So **every marker block state JSON** must accept:
- `facing = up/down/north/east/south/west`
- `active = true/false`

---

### 3) Blockstate JSON determines the model + rotation
#### A) **Current blockstates used by 1.12.2 assets**
These are the primary files used by the new registry names:

- `buildcraft_resources/assets/buildcraftcore/blockstates/marker_volume.json`
- `buildcraft_resources/assets/buildcraftcore/blockstates/marker_path.json`

Both are **Forge multi-variant** blockstate files (`"forge_marker": 1`).

Example (`marker_path.json`):
```json
{
  "forge_marker": 1,
  "defaults": { "model": "buildcraftcore:marker_path" },
  "variants": {
    "facing": {
      "up": {},
      "down": { "x": 180 },
      "east": { "y": 90, "x": 90 },
      "south": { "y": 180, "x": 90 },
      "west": { "y": 270, "x": 90 },
      "north": { "x": 90 }
    },
    "active": {
      "false": {},
      "true": {}
    }
  }
}
```
**Key points:**
- Uses a **single model** (`marker_path` or `marker_volume`).
- Applies rotation based on `facing`.
- The `active` property does **not** change textures here (it only exists so the blockstate is valid).

#### B) **Legacy blockstates (compatibility)**
- `buildcraft_resources/assets/buildcraftcore/blockstates/pathMarkerBlock.json`
- `buildcraft_resources/assets/buildcraftcore/blockstates/markerBlock.json`

These files map older block names to the newer models.
`pathMarkerBlock.json` also references a **"torch" texture override** with a `led_done` property:
```json
"led_done": {
  "false": { "textures": { "torch": "buildcraftcore:blocks/marker/path" } },
  "true":  { "textures": { "torch": "buildcraftcore:blocks/marker/path_searching" } }
}
```
This is leftover from older versions (pre-1.12 behavior). The textures `blocks/marker/path` and `blocks/marker/path_searching` do **not** exist in this repo, which indicates these legacy files are only used for **old registry compatibility**, not active 1.12 rendering.

---

### 4) Block models define the **torch geometry** and texture slots
#### A) Marker path + volume (current)
- `buildcraft_resources/assets/buildcraftcore/models/block/marker_path.json`
- `buildcraft_resources/assets/buildcraftcore/models/block/marker_volume.json`

Both use the same torch-like **centered** model parent:
```json
{
  "parent": "buildcraftcore:block/torch_center_lit",
  "textures": {
    "all": "buildcraftcore:blocks/marker_path"
  }
}
```
or
```json
{
  "parent": "buildcraftcore:block/torch_center_lit",
  "textures": {
    "all": "buildcraftcore:blocks/marker_volume"
  }
}
```

So the **actual texture mapping** is:
- `marker_path` → `buildcraftcore:textures/blocks/marker_path.png`
- `marker_volume` → `buildcraftcore:textures/blocks/marker_volume.png`

#### B) Torch geometry model
`buildcraft_resources/assets/buildcraftcore/models/block/torch_center_lit.json`

This file defines the **torch geometry** as a small centered stick using a single `"all"` texture. It is not vanilla torch geometry; it’s a **custom model** that draws a vertical centered torch, with UVs mapped to `#all`.

This is the **core of the texture mapping**: you are not using the vanilla torch model directly. Instead:
1. `marker_path` and `marker_volume` models **inherit** `torch_center_lit`.
2. `torch_center_lit` defines geometry and references a **single texture slot**: `#all`.
3. The texture slot is filled by the block models:
   - `marker_path.json` sets `all: buildcraftcore:blocks/marker_path`
   - `marker_volume.json` sets `all: buildcraftcore:blocks/marker_volume`

#### C) Additional unused legacy models
There are older models that reference `"torch"` texture slots and the vanilla torch parents:
- `buildcraft_resources/assets/buildcraftcore/models/block/marker.json`
- `buildcraft_resources/assets/buildcraftcore/models/block/marker_wall.json`
- `buildcraft_resources/assets/buildcraftcore/models/block/marker_path_wall.json`

These are leftovers from the old ID system (`markerBlock`, `pathMarkerBlock`), and refer to missing textures (e.g., `blocks/marker/path_off`). In **1.12.2**, the active assets used by the registry are the `marker_path` and `marker_volume` models (with `torch_center_lit`).

---

### 5) Texture file locations
Current 1.12.2 texture files used by the active models:
- `buildcraft_resources/assets/buildcraftcore/textures/blocks/marker_path.png`
- `buildcraft_resources/assets/buildcraftcore/textures/blocks/marker_volume.png`

These are **single-texture, torch-style sprites** used by the centered torch model.

---

## Summary: 1.12.2 mapping flow (actual assets used)
**Landmark (volume marker)**
1. Block ID: `buildcraftcore:marker_volume` (`BCCoreBlocks`, `BCCore`)
2. Blockstate: `blockstates/marker_volume.json`
3. Model: `models/block/marker_volume.json`
4. Parent: `models/block/torch_center_lit.json`
5. Texture: `textures/blocks/marker_volume.png`

**Path marker**
1. Block ID: `buildcraftcore:marker_path`
2. Blockstate: `blockstates/marker_path.json`
3. Model: `models/block/marker_path.json`
4. Parent: `models/block/torch_center_lit.json`
5. Texture: `textures/blocks/marker_path.png`

---

## Fabric 1.21.1: How to reproduce the same texture mapping

### 1) Keep the same model layering concept
In Fabric 1.21.1, you can re-use the same approach:
- A **custom torch model** (equivalent to `torch_center_lit.json`) provides geometry.
- Each marker block model references that geometry and supplies a **single texture**.

This keeps the **1.12 look** and avoids reliance on vanilla `torch` / `wall_torch` offsets.

### 2) Blockstate JSON in modern format
Forge-specific `"forge_marker": 1` is no longer used.
Instead, create standard `variants` with the `facing` property:

**Example** (`assets/<modid>/blockstates/marker_path.json`):
```json
{
  "variants": {
    "facing=up":    { "model": "<modid>:block/marker_path" },
    "facing=down":  { "model": "<modid>:block/marker_path", "x": 180 },
    "facing=east":  { "model": "<modid>:block/marker_path", "x": 90, "y": 90 },
    "facing=south": { "model": "<modid>:block/marker_path", "x": 90, "y": 180 },
    "facing=west":  { "model": "<modid>:block/marker_path", "x": 90, "y": 270 },
    "facing=north": { "model": "<modid>:block/marker_path", "x": 90 }
  }
}
```
Do the same for `marker_volume.json`.

> **Active state**: If you want to keep the `active` property for future use, use a `multipart` or `variants` file that includes `active=true/false` variants. In the 1.12 assets, `active` does not change textures, so you can safely ignore it unless you intend to use it for a visual change.

### 3) Block model for the markers
Port `torch_center_lit.json` into `assets/<modid>/models/block/torch_center_lit.json`.
Then create block models:

`marker_path.json`:
```json
{
  "parent": "<modid>:block/torch_center_lit",
  "textures": {
    "all": "<modid>:block/marker_path"
  }
}
```

`marker_volume.json`:
```json
{
  "parent": "<modid>:block/torch_center_lit",
  "textures": {
    "all": "<modid>:block/marker_volume"
  }
}
```

### 4) Textures
Copy the 1.12 textures (or re-export them to your new resource pack location):
- `assets/<modid>/textures/block/marker_path.png`
- `assets/<modid>/textures/block/marker_volume.png`

Make sure the texture names and paths match the **model JSON** exactly.

### 5) Block class (Fabric) alignment
In Fabric, you need the block to expose a **6-direction facing** property so the blockstate variants are valid. The equivalent to `BLOCK_FACING_6` is:
- `Properties.FACING`

In your block class (Fabric), implement:
- `appendProperties(StateManager.Builder<Block, BlockState> builder)` to include `Properties.FACING`
- `getPlacementState(ItemPlacementContext ctx)` to set the facing based on placement face
- `rotate` / `mirror` as needed for wrenching or rotation

This matches the 1.12.2 `BlockMarkerBase` logic:
- `getStateForPlacement` chooses the facing.
- `getBoundingBox` changes based on facing (if you want the torch-sized collision box).

### 6) Optional: Use vanilla torch model parents instead
If you do **not** need the centered torch style, you can use vanilla torch parents:
- `minecraft:block/torch`
- `minecraft:block/wall_torch`

Then define separate blockstate variants for `facing=up` vs `facing=north/south/east/west` and point to a floor or wall model. But **this will not match the 1.12 centered torch geometry**; the original BuildCraft model is more centered.

---

## Quick 1.12 → 1.21.1 mapping checklist
- ✅ **Keep** `marker_path` and `marker_volume` as separate blocks.
- ✅ **Reuse** the `torch_center_lit` geometry or create an equivalent in Fabric.
- ✅ **Assign textures** via `"all"` texture slot (`marker_path.png`, `marker_volume.png`).
- ✅ **Rotate** the model using `facing` in blockstate JSON.
- ✅ **Ignore** old `pathMarkerBlock` / `markerBlock` resources unless you need legacy IDs.

---

## File map (1.12.2 source of truth)
**Blocks / properties**
- `common/buildcraft/core/BCCoreBlocks.java`
- `common/buildcraft/core/BCCore.java`
- `common/buildcraft/lib/block/BlockMarkerBase.java`
- `common/buildcraft/core/block/BlockMarkerPath.java`
- `common/buildcraft/core/block/BlockMarkerVolume.java`

**Blockstate JSON**
- `buildcraft_resources/assets/buildcraftcore/blockstates/marker_path.json`
- `buildcraft_resources/assets/buildcraftcore/blockstates/marker_volume.json`
- (legacy) `buildcraft_resources/assets/buildcraftcore/blockstates/pathMarkerBlock.json`
- (legacy) `buildcraft_resources/assets/buildcraftcore/blockstates/markerBlock.json`

**Block models**
- `buildcraft_resources/assets/buildcraftcore/models/block/marker_path.json`
- `buildcraft_resources/assets/buildcraftcore/models/block/marker_volume.json`
- `buildcraft_resources/assets/buildcraftcore/models/block/torch_center_lit.json`
- (legacy) `buildcraft_resources/assets/buildcraftcore/models/block/marker.json`
- (legacy) `buildcraft_resources/assets/buildcraftcore/models/block/marker_wall.json`
- (legacy) `buildcraft_resources/assets/buildcraftcore/models/block/marker_path_wall.json`

**Textures**
- `buildcraft_resources/assets/buildcraftcore/textures/blocks/marker_path.png`
- `buildcraft_resources/assets/buildcraftcore/textures/blocks/marker_volume.png`
