# Builders: Quarry, Builder, Filler, Replacer (1.12.2 → Fabric 1.21.1)

This report covers the **builders subsystem**. Quarry is the most complex and has a dedicated deep note (`notes/quarry_analysis.md`). This report focuses on system boundaries and texture mapping for the rest of the builders ecosystem.

Focus areas:
1) **Texture mapping** for builder blocks, filler, quarry frame, etc.
2) **Game logic** for construction/mining tasks

---

## A) Quarry (overview)

### 1.12.2 logic
- Tile: `common/buildcraft/builders/tile/TileQuarry.java`
- Block: `common/buildcraft/builders/block/BlockQuarry.java`
- Frame block: `common/buildcraft/builders/block/BlockFrame.java`
- Config: `common/buildcraft/builders/BCBuildersConfig.java`
- Marker integration: `common/buildcraft/core/tile/TileMarkerVolume.java`

**Detailed analysis:** see `notes/quarry_analysis.md`.

### Textures / models
- Model: `buildcraft_resources/assets/buildcraftbuilders/models/block/quarry.json`
- Item model: `buildcraft_resources/assets/buildcraftbuilders/models/item/quarry.json`
- Frame textures: in `buildcraft_resources/assets/buildcraftbuilders/textures/blocks/…` (frame + quarry body)

### Fabric 1.21.1 port strategy
- Implement quarry frame placement + mining order exactly as in `TileQuarry`.
- Recreate laser/drill visuals with BER (use existing textures).

---

## B) Builder + Filler + Replacer

### 1.12.2 logic
- Builder tile: `common/buildcraft/builders/tile/TileBuilder.java`
- Filler tile: `common/buildcraft/builders/tile/TileFiller.java`
- Replacer tile: `common/buildcraft/builders/tile/TileReplacer.java`
- Architect table: `common/buildcraft/builders/tile/TileArchitectTable.java`
- Electronic library: `common/buildcraft/builders/tile/TileElectronicLibrary.java`

### Textures / models
- Builder item model: `buildcraft_resources/assets/buildcraftbuilders/models/item/builder.json`
- Filler item model: `buildcraft_resources/assets/buildcraftbuilders/models/item/filler.json`
- Filler planner item: `buildcraft_resources/assets/buildcraftbuilders/models/item/filler_planner.json`
- Block textures in `buildcraft_resources/assets/buildcraftbuilders/textures/blocks/…`

### Fabric 1.21.1 port strategy
- Rebuild area selection mechanics (likely via markers or blueprints).
- Implement pattern execution (fill, replace, build) as task queues.
- Port architect table for blueprint capture and storage.
- Port electronic library for blueprint storage/sharing.

---

## Key porting risks / notes
- Builder/filler/replacer require strong **task scheduling** and area logic.
- Blueprint formats need compatibility or migration strategy.
