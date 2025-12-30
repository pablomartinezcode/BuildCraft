# Texture Porting Guide (Weird Shapes) — BuildCraft 1.12.2 → Fabric 1.21.1

This report is a **cross‑cutting texture mapping guide** for blocks with unusual shapes or dynamic rendering.

Key goal: **preserve original textures** while re‑creating geometry and animation behavior in 1.21.1 Fabric.

---

## 1) Engine Models (Animated Piston + Trunk)

### Texture sources
- `buildcraft_resources/assets/buildcraftlib/textures/blocks/engine/` (trunk stages + chamber)
- Engine back/side textures:
  - `buildcraftcore/textures/blocks/engine/wood/…`
  - `buildcraftcore/textures/blocks/engine/creative/…`
  - `buildcraftenergy/textures/blocks/engine/stone/…`
  - `buildcraftenergy/textures/blocks/engine/iron/…`
  - `buildcraftenergy/textures/blocks/engine/rf/…`

### Mapping rule
- Trunk texture depends on power stage (blue/green/yellow/red/overheat/black).
- Stage selection is logic‑driven (`TileEngineBase_BC8.computePowerStage()`).

### Fabric approach
- Use BER with animated quads and dynamic texture binding.
- Keep stage strings consistent so trunk texture lookup works.

---

## 2) Markers (Torch‑style)

### Texture sources
- `buildcraftcore/textures/blocks/marker_path.png`
- `buildcraftcore/textures/blocks/marker_volume.png`

### Mapping rule
- Marker models inherit a **custom torch geometry** (`torch_center_lit.json`) with a single texture slot.

### Fabric approach
- Recreate the custom geometry JSON.
- Map `all` texture slot to the original PNGs.

---

## 3) Pipes (Dynamic geometry + overlays)

### Texture sources
- `buildcrafttransport/textures/blocks/pipes/…`
- Overlays/lenses: same folder
- Wire colors + triggers:
  - `buildcrafttransport/textures/triggers/…`

### Mapping rule
- Pipes are rendered dynamically; textures are layered.

### Fabric approach
- Use BER/Renderer API to assemble quads per connection and per overlay.
- Keep overlay ordering consistent (base → wire → pluggable → flow).

---

## 4) Quarry (Frames + Lasers + Drill)

### Texture sources
- `buildcraftbuilders/textures/blocks/…`
- Frame textures + quarry body

### Mapping rule
- Frame blocks use dedicated textures; laser/drill visuals are rendered dynamically.

### Fabric approach
- Use BER for drill/laser effects.
- Keep frame block as a distinct block model with original textures.

---

## 5) Tanks (Joined multiblock)

### Texture sources
- `buildcraftfactory/textures/blocks/…`

### Mapping rule
- Adjacent tanks switch models (e.g., joined below).

### Fabric approach
- Use blockstate or model variants to swap between joined/standalone textures.
- Add fluid rendering (custom renderer or block entity renderer).

---

## 6) Distiller/Refinery (Static + Tile Anim)

### Texture sources
- `buildcraftfactory/textures/blocks/…`
- Models:
  - `buildcraftfactory/models/block/distiller.json`
  - `buildcraftfactory/models/block/refinery.json`

### Fabric approach
- Use static models where possible; add BER for animated pieces if needed.

---

## Summary: Texture Porting Checklist

- [ ] Preserve **original PNGs** from `buildcraft_resources/`.
- [ ] Identify dynamic renderers (engine, pipes, quarry, tanks).
- [ ] Rebuild custom JSON geometry (marker torch, engine base).
- [ ] Validate UVs and rotations against original blockstates.
