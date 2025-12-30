# Robotics: Zone Planner + Automation (1.12.2 → Fabric 1.21.1)

This report covers the **robotics subsystem**, focusing on the Zone Planner block and its role in automation.

Focus areas:
1) **Texture mapping** for zone planner models
2) **Game logic** for zone definition and robot interaction

---

## A) Zone Planner

### 1.12.2 logic
- Tile: `common/buildcraft/robotics/tile/TileZonePlanner.java`
- Block/item models:
  - `buildcraft_resources/assets/buildcraftrobotics/models/block/zone_planner.json`
  - `buildcraft_resources/assets/buildcraftrobotics/models/item/zone_planner.json`

### Texture mapping
- Textures: `buildcraft_resources/assets/buildcraftrobotics/textures/blocks/…`

### Fabric 1.21.1 port strategy
- Implement a zone definition UI that writes region data to the tile.
- Provide hooks for robot pathing/behavior to read zones.
- Use original block textures in the Fabric model JSONs.

---

## Key porting risks / notes
- Zone data must be serializable and compatible with other automation systems.
