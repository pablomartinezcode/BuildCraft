# Silicon: Lasers, Assembly, Programming (1.12.2 → Fabric 1.21.1)

This report covers **silicon subsystem** machines: lasers, assembly tables, charging tables, and programming/integration tables.

Focus areas:
1) **Texture mapping** for complex laser/assembly blocks
2) **Game logic** for laser-powered crafting and programming

---

## A) Laser + Laser Table Integration

### 1.12.2 logic
- Laser tile: `common/buildcraft/silicon/tile/TileLaser.java`
- Laser table base: `common/buildcraft/silicon/tile/TileLaserTableBase.java`

### Texture mapping
- Laser/laser table textures in `buildcraft_resources/assets/buildcraftsilicon/textures/blocks/…`
- Models in `buildcraft_resources/assets/buildcraftsilicon/models/block/…`

### Fabric 1.21.1 port strategy
- Implement laser beam rendering (BER) that links laser blocks to target tables.
- Port beam activation, power draw, and target detection logic.

---

## B) Assembly + Charging + Advanced Crafting Tables

### 1.12.2 logic
- Assembly table: `common/buildcraft/silicon/tile/TileAssemblyTable.java`
- Charging table: `common/buildcraft/silicon/tile/TileChargingTable.java`
- Advanced crafting table: `common/buildcraft/silicon/tile/TileAdvancedCraftingTable.java`

### Fabric 1.21.1 port strategy
- Recreate recipe processing using MJ input and progress tracking.
- Implement GUI and slot layout consistent with 1.12 behavior.

---

## C) Programming + Integration Tables

### 1.12.2 logic
- Programming table: `common/buildcraft/silicon/tile/TileProgrammingTable_Neptune.java`
- Integration table: `common/buildcraft/silicon/tile/TileIntegrationTable.java`

### Fabric 1.21.1 port strategy
- Port logic for creating gates/robot instructions.
- Ensure NBT/data storage is compatible with new item types.

---

## Key porting risks / notes
- Laser interactions and beam visuals are key to user experience.
- Table UIs are complex and must align with existing recipe formats.
