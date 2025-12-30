# BuildCraft Quarry (1.12.2) – Deep Technical Notes + Fabric 1.21.1 Porting Guide

These notes are intentionally exhaustive and reference the **exact files, classes, and key methods** in the 1.12.2 BuildCraft source tree. The goal is to enable a full re-implementation on Fabric 1.21.1 without needing to re-read the original code.

---

## 1) Primary 1.12.2 Quarry Implementation (where everything lives)

### Core classes and responsibilities

- **`common/buildcraft/builders/tile/TileQuarry.java`**
  - Core server-side logic, ticking, frame scanning, mining iteration, power consumption, chunk loading, and state sync.
  - Contains task sub-classes for mining, frame placement, and drill movement.

- **`common/buildcraft/builders/block/BlockQuarry.java`**
  - Block + tile creation, connection properties, frame cleanup on break, advancement triggers.

- **`common/buildcraft/builders/block/BlockFrame.java`**
  - Frame block visuals (connected model), collision boxes, and no drops. Used for the quarry frame itself.

- **`common/buildcraft/builders/BCBuildersConfig.java`**
  - Quarry configuration (frame min height, move both axes, max tasks per tick, power scaling, max mining rate, max frame move rate).

- **`common/buildcraft/core/BCCoreConfig.java`**
  - Global mining depth limit, marker maximum distance, etc.

- **Markers / landmarks** (volume markers build the quarry box)
  - **`common/buildcraft/core/tile/TileMarkerVolume.java`** – marker block entity; implements `ITileAreaProvider`.
  - **`common/buildcraft/core/marker/VolumeSubCache.java`** – manages per-world marker connections.
  - **`common/buildcraft/core/marker/VolumeConnection.java`** – stores marker positions + computes bounding `Box`.
  - **`common/buildcraft/core/marker/VolumeCache.java`** – global marker cache registry.

- **Chunk loading**
  - **`common/buildcraft/lib/chunkload/ChunkLoaderManager.java`**
  - **`common/buildcraft/lib/chunkload/IChunkLoadingTile.java`**
  - Uses Forge chunk tickets; `TileQuarry` implements `IChunkLoadingTile`.

- **Rendering / laser visuals**
  - **`common/buildcraft/builders/client/render/RenderQuarry.java`** – draws lasers, drill, and frame via laser renderer.
  - **`common/buildcraft/builders/client/render/AdvDebuggerQuarry.java`** – debug visualization (less relevant to logic).

- **Misc core utilities used by quarry**
  - **`common/buildcraft/lib/misc/data/Box.java`** – mutable AABB-like structure for area mining + frame bounds.
  - **`common/buildcraft/lib/misc/data/BoxIterator.java`** – iterates mining volume in pseudo-random axis order.
  - **`common/buildcraft/lib/misc/BlockUtil.java`** – break block + compute power cost, fluids, etc.
  - **`common/buildcraft/lib/misc/InventoryUtil.java`** – pushes drops into inventories.
  - **`common/buildcraft/lib/misc/BoundingBoxUtil.java`** – AABB helpers for collision boxes.
  - **`common/buildcraft/lib/misc/VecUtil.java`** – position math.

---

## 2) Quarry block placement flow (frame creation + landmark integration)

**File:** `TileQuarry.java` – method `onPlacedBy(EntityLivingBase placer, ItemStack stack)`

### Step-by-step logic

1. **Determine facing**
   - Reads `BlockBCBase_Neptune.PROP_FACING` from the blockstate at the quarry position.
   - Calculates `areaPos` as `pos.offset(facing.getOpposite())` (block behind quarry)
     - This is where it checks for an `IAreaProvider` (landmark/marker or other area provider).

2. **Check for explicit `IAreaProvider`**
   - If a tile entity at `areaPos` implements `IAreaProvider`, use `provider.min()` / `provider.max()`.
   - Reject if box is too small (`dx < 3` or `dz < 3`).
   - If valid, call `provider.removeFromWorld()` (destroy the landmark structure).

3. **Fallback: scan **Volume markers** (landmarks)**
   - Uses `VolumeCache.INSTANCE.getSubCache(world)`.
   - Iterates all markers (`cache.getAllMarkers()`).
   - For each marker:
     - `VolumeConnection connection = marker.getCurrentConnection()`.
     - Extracts `Box volBox = connection.getBox()`.
     - Guards:
       - `pos.getY() == box.min().getY()` (must align vertically with marker box base).
       - `!box.contains(pos)` (quarry cannot be inside marker box).
       - `box.contains(areaPos)` (quarry must be adjacent to box on the correct side).
       - Size must be at least 3x3.
     - `box2.expand(1)` and then `box2.setMin(box2.min().up())` – expands outward and shifts Y+1.
     - Only if the expanded box is on the edge and contains the quarry, accepts it.
     - Removes the marker from world once selected.

4. **If no valid markers / area provider**
   - Falls back to a **default 11x11 footprint** facing away from the front of the quarry.
   - The exact default bounds are computed per facing:
     - `EAST` (default): `min = pos + (1,0,-5)`, `max = pos + (11,4,5)`
     - `WEST`: `min = pos + (-11,0,-5)`, `max = pos + (-1,4,5)`
     - `SOUTH`: `min = pos + (-5,0,1)`, `max = pos + (5,4,11)`
     - `NORTH`: `min = pos + (-5,0,-11)`, `max = pos + (5,4,-1)`

5. **Frame height sanity**
   - Enforces `BCBuildersConfig.quarryFrameMinHeight` (default 4). If `maxY - minY` is smaller, it raises the max.

6. **World height clamp**
   - If `max` is outside world build height, it shifts both `min` and `max` downward by the same distance.

7. **Set frame & mining boxes**
   - `frameBox` is **exactly** `min..max`.
   - `miningBox` is **inner volume** and extends downwards by `BCCoreConfig.miningMaxDepth`:
     - `miningBox.min = (minX+1, minYAdjusted, minZ+1)`
     - `miningBox.max = (maxX-1, maxY-1, maxZ-1)`
   - `minYAdjusted` is `maxY - 1 - miningMaxDepth`, clamped to world minimum (0 if outside build height).

8. **Call `updatePoses()`**
   - Resets internal frame-logic state, recalculates frame positions, and initializes chunk loading.

### Implications

- The quarry **always** builds its frame from the `frameBox` computed above.
- The mining volume is **strictly inside** the frame, one block inset on X/Z and one block down on Y.
- Landmarks are implemented via marker volume connections, not a special quarry-only block.

---

## 3) Frame generation, frame maintenance, and frame rebuilding

**Relevant fields:**
- `frameBox` – `Box` representing the frame boundaries.
- `framePoses` – ordered list of edge positions where frame blocks should be placed.
- `toCheck`, `frameBreakBlockPoses`, `framePlaceFramePoses` – live tracking of frame block validity.

**Key methods:**
- `getFramePositions()` – produces ordered list of edge blocks by BFS-like traversal around the frame.
- `shouldBeFrame(BlockPos)` – returns `frameBox.isOnEdge(pos)`
- `check(BlockPos)` – validates each frame position, updates `frameBreakBlockPoses` / `framePlaceFramePoses`.
- `updatePoses()` – resets + repopulates frame positions and initial checks.

### How frame positions are computed (`getFramePositions`)

- Starts from the quarry block position and uses a randomized BFS through edge positions.
- It assumes `frameBox` is correct and adjacent to the quarry; otherwise it throws `IllegalStateException`.
- Each iteration:
  - Shuffle the `EnumFacing` list.
  - Add adjacent positions that are `frameBox.isOnEdge` and not yet visited.
  - Collects them in `framePositions` as the **placement order**.
- Has multiple sanity checks:
  - `openSet` size cap.
  - iteration count cap (`frameBox.getBlocksOnEdgeCount()`).

### How the frame is maintained

- On each server tick:
  - `toCheck` list is rotated and scanned (faster once initial pass is done).
  - `check(pos)` determines whether a position is:
    - Correct frame
    - Obstruction that must be broken (`frameBreakBlockPoses`)
    - Air that must be filled with frame (`framePlaceFramePoses`)

- `canIgnoreInFrameBox`:
  - A block is "ignorable" if it is not air and NOT a fluid (flowing or source).
  - If something is ignorable and should not be there, it is **broken** by the quarry.
  - If a space is not ignorable (air or fluid), it is **filled** with `BlockFrame`.

### How frame break/placement tasks happen

- **Break obstructing blocks:**
  - `frameBreakBlockPoses` is processed before any frame placement or mining.
  - Creates `TaskBreakBlock` at those positions.

- **Place missing frame blocks:**
  - Iterates `framePoses` in the generated order.
  - For a position in `framePlaceFramePoses`, creates `TaskAddFrame`.

- **If the quarry is broken (`BlockQuarry.breakBlock`)**:
  - The block iterates `framePoses` and removes actual frame blocks (no drops).

### Summary of frame behavior

- Frame generation is **stateful and ordered**, allowing animated frame-building.
- Frame is **self-repairing**; if blocks are removed, the quarry re-adds them over time.
- Frame blocks are **inert** (no drops, connected visuals).

---

## 4) Mining order and drill movement

**Primary driver:** `TileQuarry.update()`

### Mining iterator

- Uses `BoxIterator` seeded by quarry position to produce a **deterministic but pseudo-random** axis order.
- `createBoxIterator()`:
  - Seed combines X/Y/Z bits.
  - Randomly chooses `EnumAxisOrder` (XZY or ZXY) and axis inversion flags.
  - `BoxIterator(miningBox, AxisOrder, invert = true)`

### Drill position (`drillPos`)

- Stored as `Vec3d`.
- If `boxIterator == null` or `drillPos == null`, it initializes:
  - `boxIterator = createBoxIterator()`
  - Advances until current block is valid to mine
  - `drillPos = miningBox.closestInsideTo(pos)` (starts at quarry position projected into mining box).

### Block selection

Loop logic:

1. **Skip blocks that are air/fluids or not mineable**
   - `canMoveThrough(pos)`:
     - Air or fluid with viscosity <= 1000
   - `canMine(pos)`:
     - Hardness >= 0 and no high-viscosity fluid
   - `canMoveDownTo(pos)`:
     - Ensures every block above the target in the mining column can be moved through.

2. **If drill is not at target**:
   - Creates `TaskMoveDrill` from `drillPos` to target block position.

3. **If drill is at target**:
   - Creates `TaskBreakBlock` for the current block.

### End of mining

- If no valid tasks remain and the area is fully mined:
  - Awards advancement `buildcraftbuilders:diggy_diggy_hole` when size is 64x64.

---

## 5) Task system (power-driven state machine)

**Nested classes inside `TileQuarry`:**

- `Task` (base)
  - Tracks `power` (server) and `clientPower` for rendering.
  - `getTarget()` defines required energy.
  - `addPower()` progresses task; returns **true** when done.
  - `onReceivePower()` called for incremental updates.
  - `finish()` called once target power reached.

### TaskBreakBlock

- **Target power:** `BlockUtil.computeBlockBreakPower(world, breakPos)` (includes mining multiplier).
- **Rate limit:** `BCBuildersConfig.quarryMaxBlockMineRate`
  - Converts to blocks per second, caps per tick power.
- **During progress:**
  - Sends `world.sendBlockBreakProgress(...)` with 0..9 progress.
  - If block disappears early, task finishes immediately.
- **On finish:**
  - Calls `BlockUtil.breakBlockAndGetDrops(WorldServer, pos, diamond_pickaxe, owner, true)`
  - Drops are inserted via `InventoryUtil.addToBestAcceptor(world, pos, null, stack)`
  - If `drillPos == null` (frame break), it destroys without inserting drops.

### TaskAddFrame

- **Target power:** fixed `24 * MjAPI.MJ`.
- **On finish:**
  - If the position is still missing frame and not ignored, sets block to `BCBuildersBlocks.frame`.

### TaskMoveDrill

- **Target power:** `distance(from, to) * 20 MJ`
  - 20 MJ per block distance.
- **Rate limit:** `BCBuildersConfig.quarryMaxFrameMoveSpeed`
  - Max blocks per second, converted to per-tick movement.
- **During progress:**
  - Updates `drillPos` using linear interpolation between `from` and `to`.

### Task scheduling

Inside `update()`:

1. Calculate `max` power for this tick based on battery charge.
2. `maxTasks` = `quarryMaxTasksPerTick` scaled to available power.
3. For each task slot:
   - If `currentTask` exists, feed it power (scaled by `quarryTaskPowerDivisor`).
   - If no `currentTask`, generate one in priority order:
     1. `frameBreakBlockPoses`
     2. `framePlaceFramePoses`
     3. `move drill` or `break block`

---

## 6) Power system and energy usage

- **Battery:** `MjBattery battery = new MjBattery(24000 * MJ)`.
- Uses `MjBatteryReceiver` capability to accept energy.
- **Max power per tick:** `MAX_POWER_PER_TICK = 512 MJ`.
- When battery is less than half full, `max` scales down proportionally (using BigInteger to avoid overflow).

**Additional config tuning:**
- `BCBuildersConfig.quarryMaxTasksPerTick` – limits number of tasks/tick.
- `BCBuildersConfig.quarryTaskPowerDivisor` – power cost increase per additional task.
- `BCBuildersConfig.quarryMaxFrameMoveSpeed` – speed cap for `TaskMoveDrill`.
- `BCBuildersConfig.quarryMaxBlockMineRate` – speed cap for `TaskBreakBlock`.

---

## 7) Chunk loading behavior

- `TileQuarry implements IChunkLoadingTile`.
- **Load type:** `LoadType.HARD` (`TileQuarry.getLoadType`).
- **Chunk selection:** `getChunksToLoad()` returns all chunks covering the **frame box**.
- `ChunkLoaderManager.loadChunksForTile(this)` invoked in `updatePoses()` if frame is initialized.

This ensures:
- The quarry block’s own chunk is **always loaded**.
- All chunks in the quarry frame area are forced loaded when chunk loading is enabled.

---

## 8) Collision and rendering hooks

### Collision

- `TileQuarry.getCollisionBoxes()` creates three AABBs aligned to the drill arms:
  - X arm, Z arm, and vertical drill arm.
  - Only active when `drillPos != null`.
- `BCBuildersEventDist.onGetCollisionBoxesForQuarry` injects these boxes into world collision queries.

### Rendering

- `RenderQuarry` draws lasers representing the frame and drill (uses `LaserRenderer_BC8`).
- Frame lasers render between `frameBox.min/max` and `drillPos`.
- If currently mining, it adjusts drill head offset to simulate bit motion.
- When `drillPos` is null, it draws a static laser box around `frameBox` instead.

---

## 9) Saving, loading, and network sync

### NBT

- `writeToNBT` writes:
  - `frameBox`, `miningBox`, `boxIterator`, `battery`, `currentTask`, `drillPos`, `firstChecked`.
- `readFromNBT` validates:
  - Frame/mining boxes correspond to each other and quarry adjacency.
  - If validation fails, resets boxes and drill position.

### Network

- Uses `NET_RENDER_DATA` to sync `frameBox`, `miningBox`, `drillPos`, and `currentTask` state to clients.
- `readPayload` reconstructs a task based on `EnumTaskType` and updates `currentTask`.

---

## 10) Landmarks (volume markers) detail

### How volume markers work

- **`TileMarkerVolume`** extends a generic marker (`TileMarker<VolumeConnection>`).
- Uses **`VolumeCache`** / **`VolumeSubCache`** to maintain connections.
- Markers connect along axis-aligned lines (no diagonal), within `BCCoreConfig.markerMaxDistance`.
- A **connection** becomes a **box** once at least two markers are linked along different axes (see `VolumeConnection`).

### Connection rules

- **`VolumeConnection.tryCreateConnection`**
  - Ensures a direct line exists between two markers without interference.
- **`VolumeConnection.canAddMarker` / `canMergeWith`**
  - Ensures axes aren’t reused and that adding marker expands or completes the box.
- When a box exists, `VolumeConnection.getBox()` exposes the `Box` bounds used by the quarry.

### TileMarkerVolume -> ITileAreaProvider

- `min()` and `max()` return marker box corners.
- `removeFromWorld()` destroys all markers in the connection.
- `isValidFromLocation()` ensures the quarry is adjacent to the marker box’s edge.

---

# Fabric 1.21.1 Porting Guide (design/implementation plan)

Below is a **direct mapping** of the 1.12.2 logic to Fabric 1.21.1. These are implementation notes, not code.

## A) Core entity/block structure

### 1. Block + BlockEntity

- Create a **QuarryBlock** (equivalent of `BlockQuarry`).
  - Use **`BlockEntityProvider`** to attach a **QuarryBlockEntity**.
  - Use `FACING` property + `BlockState` for orientation.
  - On break, remove all frame blocks in `framePoses`.

- Create a **QuarryBlockEntity** (equivalent of `TileQuarry`).
  - Implement `Tickable` via `BlockEntityTicker`.
  - Maintain all state fields from `TileQuarry`:
    - `frameBox`, `miningBox`, `boxIterator`, `drillPos`, `currentTask`, `framePoses`, `frameBreak/Place` sets.
  - Save state using `NbtCompound`.

### 2. Frame block

- Create **FrameBlock** (equivalent of `BlockFrame`).
  - Use `BlockState` booleans for connected faces.
  - Implement `getOutlineShape` / `getCollisionShape` based on connected directions.
  - Override drops to none.
  - Render as cutout.

### 3. Marker blocks (landmarks)

- Port **Volume Marker** as a `BlockEntity` that stores connections (or leverage world saved data).
- Use a **per-world storage** (Fabric `PersistentState`) to track marker connections:
  - Equivalent to `VolumeSubCache`, `VolumeConnection`, and `VolumeCache`.
  - Store marker positions and computed bounding boxes.

---

## B) Quarry placement behavior

Replicate `TileQuarry.onPlacedBy` semantics:

1. Identify area provider behind the quarry
   - In Fabric, query the block entity at `pos.offset(facing.getOpposite())`.
   - If it implements an `AreaProvider` interface (you create), use its bounds.

2. Search markers
   - Use your **marker cache** to iterate all marker positions.
   - Find a connection whose `Box`:
     - Has matching `Y` at the quarry base.
     - Contains the back position but not the quarry itself.
     - Is big enough (>= 3x3).
     - When expanded and shifted, places the quarry on the edge.

3. If none, compute the default 11x11 area
   - Keep the exact offsets from 1.12 for consistency.

4. Apply minimum frame height, world height clamping, and mining depth
   - `miningMaxDepth` config should be ported.
   - Use `world.getBottomY()` / `world.getTopY()` in 1.21 to clamp.

---

## C) Frame generation / maintenance

### Frame position generation

- Port the BFS in `getFramePositions()`:
  - Randomly shuffle adjacent directions each step.
  - Build ordered list of edge blocks.
  - Preserve size/iteration sanity checks (debug logs instead of exceptions if preferred).

### Frame validation tasks

- Maintain:
  - `frameBreakBlockPoses` for solid blocks that should be removed.
  - `framePlaceFramePoses` for empty/fluid blocks that need frame placement.

- On each tick, iterate a limited number of positions from `toCheck`:
  - If not `firstChecked`, process more (500), else 10.

---

## D) Mining iteration logic

### BoxIterator port

- Implement a `BoxIterator` clone:
  - `min`, `max`, `AxisOrder`, inversion logic, `advance()` semantics.
  - Provide `contains`, `hasVisited`, `moveTo` for live updates when blocks change.

### Drill task logic

- Keep the task system:
  - **MoveDrill** task – consumes energy based on distance.
  - **BreakBlock** task – consumes energy based on block hardness + config.
  - **AddFrame** task – fixed energy cost.

### Filtering blocks

- Match `canMine` / `canMoveThrough` behavior:
  - Treat air + low-viscosity fluids as passable.
  - Deny non-breakable (hardness < 0) and thick fluids.

### Progress updates

- In Fabric, send custom block update packets to sync `drillPos`, `currentTask`, `frameBox`.
- For block break animation, call `world.setBlockBreakingInfo(id, pos, progress)`.

---

## E) Power system

### 1. Energy storage

- Implement your own energy buffer (MJ equivalent), or map to Fabric Energy API.
- Keep:
  - `batteryCapacity` = 24,000 MJ.
  - `MAX_POWER_PER_TICK` = 512 MJ.
  - Power scaling when battery is under half capacity.

### 2. Task throughput scaling

- Port:
  - `quarryMaxTasksPerTick`
  - `quarryTaskPowerDivisor`
  - `quarryMaxFrameMoveSpeed`
  - `quarryMaxBlockMineRate`

---

## F) Item output

- Use Fabric Transfer API (`ItemStorage`) to push drops into adjacent inventories.
- Mimic `InventoryUtil.addToBestAcceptor`:
  - Search adjacent blocks (likely all sides) and insert drops.
  - If none available, spawn items (if you choose to mimic default behavior).

---

## G) Chunk loading

- Fabric 1.21 requires a different chunk loading approach:
  - Use **chunk tickets** (`ServerWorld#addTicket` / `#removeTicket`).
  - Quarry should keep loaded **all chunks covering frameBox**.
  - Provide config option for enabling/disabling chunkloading.

---

## H) Rendering (frame lasers + drill)

- Replace `RenderQuarry` with a `BlockEntityRenderer`.
- Use custom rendering to draw:
  - Frame lasers (lines between frame corners and drill position).
  - Vertical drill line.
- Use `WorldRenderEvents` or `RenderLayer` for translucent beams.
- Sync `drillPos`, `frameBox`, and task progress to client.

---

## I) Network syncing details

- In 1.12, `NET_RENDER_DATA` includes:
  - `frameBox` + `miningBox`
  - `drillPos`
  - `currentTask` + task state

- In Fabric, implement:
  - `toInitialChunkDataNbt()` for initial sync.
  - Custom packet for ongoing task updates, or ticked block entity update packets.

---

## J) Misc and edge cases to carry over

- Validate `frameBox` + `miningBox` on load; reset if invalid.
- If `drillPos` is absurdly far (`distanceSq > 1024^2`), reset it.
- Mining iterator should skip blocks if not `canMoveDownTo` (maintains vertical clearance).
- When the quarry is broken, **all frames are removed**.
- Frame blocks drop nothing.

---

# Quick Reference: Key Data Structures to Port

| System | 1.12 Class | Fabric Port Equivalent |
|--------|------------|------------------------|
| Quarry block | `BlockQuarry` | `QuarryBlock` |
| Quarry tile | `TileQuarry` | `QuarryBlockEntity` |
| Frame block | `BlockFrame` | `FrameBlock` |
| Mining box | `Box` | custom `Box` / `BlockBox` wrapper |
| Mining iteration | `BoxIterator` | custom iterator |
| Markers | `TileMarkerVolume` + caches | marker block entity + `PersistentState` cache |
| Chunk loading | `ChunkLoaderManager` | `ChunkTicket` + `ServerWorld` ticketing |
| Rendering | `RenderQuarry` | `BlockEntityRenderer` |

---

# Suggested Fabric Implementation Checklist (ordered)

1. **Data types** – port `Box`, `AxisOrder`, `BoxIterator` to common module.
2. **Marker system** – implement volume marker blocks and persistent connection cache.
3. **Quarry state** – implement `QuarryBlockEntity` with frame/mining box handling.
4. **Frame blocks** – implement connected frame block with collision and visual logic.
5. **Task system** – `TaskBreakBlock`, `TaskAddFrame`, `TaskMoveDrill` logic + power.
6. **Mining loop** – port `update()` loop with task scheduling.
7. **Energy** – integrate with FE/Fabric Energy, keep MJ scaling.
8. **Chunk loading** – add chunk ticket management for the frame area.
9. **Client rendering** – beam/drill rendering and break animation progress.
10. **Network sync** – block entity sync packets for drill position + tasks.

---

If you need the above expanded further (e.g., exact data flow diagrams, ported pseudo-code, or class skeletons), I can extend these notes.
