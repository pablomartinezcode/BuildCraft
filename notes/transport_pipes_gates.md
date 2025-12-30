# Transport: Pipes, Wires, Pluggables, Gates (1.12.2 → Fabric 1.21.1)

This report covers the **transport subsystem**: pipes, wire colors, gates, filters, and dynamic rendering.

Focus areas:
1) **Texture mapping** for complex pipe shapes and overlays
2) **Game logic** for pipe flow, routing, and gate logic

---

## A) Pipe Rendering & Texture Mapping

### 1.12.2 rendering pipeline
- Renderer entry point: `common/buildcraft/transport/client/render/RenderPipeHolder.java`
  - Uses **FastTESR** rendering, not static block models.
  - Rendering layers:
    1) Wires (`PipeWireRenderer`)
    2) Pluggables (gates, facades, plugs)
    3) Contents (flow renderer + behavior renderer)

### Texture locations
- Pipe block textures & overlays:
  - `buildcraft_resources/assets/buildcrafttransport/textures/blocks/pipes/…`
- Pipe GUI textures:
  - `buildcraft_resources/assets/buildcrafttransport/textures/gui/…`
- Pipe trigger/action icons:
  - `buildcraft_resources/assets/buildcrafttransport/textures/triggers/…`

### Key mapping implications
- Pipes are **not** simply JSON block models; the base shape and connected arms are produced dynamically.
- Lens/overlay textures (e.g., filters, stations) are stacked as separate render layers.

### Fabric 1.21.1 rendering strategy
- Implement a **BER** or Fabric Renderer API pipeline to build quads for:
  - core pipe body
  - connected arms
  - wire overlays (per face + per color)
  - pluggables
  - moving contents (items/fluids/energy visualization)
- Reuse original textures for each pipe type + overlays.

---

## B) Pipe Logic (Flow + Behavior)

### 1.12.2 logic structure
- Holder tile: `common/buildcraft/transport/tile/TilePipeHolder.java`
- Pipe base classes: `common/buildcraft/transport/pipe/*`
- Flows: `common/buildcraft/transport/pipe/*Flow*`
- Behaviors: `common/buildcraft/transport/pipe/*Behaviour*`
- Pluggables: `common/buildcraft/transport/plug/*`
- Wires: `common/buildcraft/transport/wire/*`

### Core behavior outline
- **Connection graph** updates per tick.
- **Flow** handles per‑tick transport of items/fluids/energy.
- **Behaviors** modify flow or apply logic (e.g., sorting, gating).
- **Pluggables** inject GUI‑driven logic (filters, gates, facades).

### Fabric 1.21.1 port strategy
- Rebuild pipe graph updates using Fabric block events and neighbor updates.
- Implement flow simulation using component‑style interfaces:
  - `PipeFlowItems`, `PipeFlowFluids`, `PipeFlowPower` equivalents
- Port behavior hooks to allow pipe variants to add modifiers.
- Port pluggables/gates as per‑face attachable components.

---

## C) Gates, Triggers, and Actions

### 1.12.2 assets and logic
- Statements: `common/buildcraft/transport/statements/*`
- Gate UIs: `common/buildcraft/transport/gui/*`
- Trigger icons: `buildcraft_resources/assets/buildcrafttransport/textures/triggers/*.png`

### Fabric 1.21.1 port strategy
- Implement a **gate UI system** that can assign triggers/actions to pipe slots.
- Map each trigger/action icon using existing PNGs.
- Port logic evaluation to check pipe state (inventory, fluid, power, redstone).

---

## Key porting risks / notes
- Pipe rendering is **dynamic**; ensure compatibility with Fabric’s render thread model.
- Many textures are overlays (filters, lenses) rather than base pipe textures.
- Gate logic needs a robust data model for saving/restoring configurations.
