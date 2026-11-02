# AROS native port — architecture and roadmap

Status: **proposal / no AROS implementation yet**. Target repository: JJDSNT/pestinha; upstream: mille-okws/pestinha.

## Objective and non-goals

Build a native Pestinha desktop companion for AROS, with **Kitty as an optional AROS character**, while preserving existing Python/PySide6 implementations and upstream synchronization. Initial target: AROS x86-64; design for portability to AROS 68k and other targets. Do not require Python, NumPy, Qt, a network service, or an LLM at runtime. Do not patch Intuition or Wanderer unless an independently reproducible missing capability is demonstrated.

Do not conflate *the character* with *the neural engine*: keep a character-neutral brain, behavioral controllers, OS sensors, rendering backend and persistence. Existing Bryan/Jorge assets and the Kitty artwork require separate licensing review; Pestinha code is GPL-3.0-or-later and assets are not necessarily covered by that license.

## Source references

- `pestinha/core/nervous_system.py`: `Brain` contract (`stimulate`, `step`, `motor_activity`, `snapshot`, `save/load`), NumPy dependency, 470 LIF neurons, 15,939 synapses, 30 sensory channels, 15 motor populations.
- `pestinha/core/{conditioning,emotion,episodic,homeostasis,motor,physics,protesto}.py`: behavioral model and learning.
- `docs/{ARQUITETURA,REDE_NEURAL}.md`: layered design, motor decisions and update clocks.
- AROS `workbench/tools/BoingIconBar/iconbar.c`: existing composited ARGB window with `WA_AlphaSurface` and `WA_InputRegion`.
- AROS `developer/debug/test/intuition/{compwindow,compflicker,windowinputregion}.c`: tests for composition, alpha and input regions.
- AROS `rom/intuition/openwindow.c`: window tags; `rom/graphics/bltbitmapalpha.c`: alpha blitting.

These are implementation references, **not** a guarantee that every graphics driver or AROS target supports compositing.

## Proposed layout

```
ports/aros/
  README.md
  include/             # platform-independent brain, scene, and sensor contracts
  src/
    brain/              # C port of neural algorithms
    behavior/           # controllers, emotions, physics
    platform/           # Intuition, Layers, input, clocks
    graphics/           # composited and fallback renderer
    storage/            # DOS persistence
  tests/                # native and host-side fixtures
  assets/               # redistributable, licensed art only
```

Keep `pestinha/`, `pets/`, `tests/` and existing CI behavior intact. No placeholder build files until the toolchain is selected and verified.

## Platform contract

- `clock_monotonic_ms()`, `sleep_or_wait_until()`: elapsed time, no busy waiting.
- `read_sensors()`: cursor and activity, foreground application name where possible, window rectangles without titles or content. Unsupported sensors return 'unavailable', never fabricated values.
- `render_frame()`: per-pixel alpha when supported; masked/opaque fallback otherwise.
- `set_hit_region()`: mouse events only on the avatar, preserving access to underlying windows.
- `persist_state()`: versioned, validated and atomic save/restore; interrupted writes must not destroy the previous state.
- `request_hide_or_exit()`: always responsive; no focus theft, cursor injection or system-wide blocking.

Preserve time-based simulation semantics: reference intervals are 16 ms frame, 100 ms brain, 250 ms window scan, 500 ms activity scan and 30 s save. Use elapsed-time integration and bound catch-up; do not tie neural correctness to frame rate.

## Milestones and acceptance criteria

### M0 — Baseline and licensing
- Record upstream and fork commit SHAs; identify license terms for every distributed sprite, text and meme.
- Run original Python tests; export deterministic neural fixtures (seed, sensory stream, elapsed times, motor rates, plasticity, serialized state).
- **Done:** reproducible baseline, documented character-asset permissions and reference fixture generator.

### M1 — Native graphics proof of concept
- Build a borderless, non-activating Kitty-shaped demo window on AROS x86-64.
- Exercise `WA_AlphaSurface`, `WA_InputRegion`, composition capability detection and fallback.
- Validate drag, close, focus preservation, click-through outside the sprite, and background window interaction.
- **Done:** screenshots and repeatable checks on a named AROS build/driver; no upstream patch needed.

### M2 — Portable C brain
- Implement the neural model and random generator with explicit seed, numerical precision and update ordering.
- Test LIF/refractory behavior, habituation, lateral inhibition, three-factor learning, conditioning, motor outputs and serialization against Python fixtures.
- Use tolerances for floating-point results; test statistical behavior when exact random-stream equivalence is impractical.
- **Done:** headless Linux-hosted C tests pass and AROS binary builds; memory and CPU measured.

### M3 — AROS sensors and behavior
- Wire native events and window geometry to the sensor abstraction.
- Add physics, expressions, mouse interaction and persisted personality without Qt.
- Respect privacy: no window titles/content, no remote telemetry.
- **Done:** the companion can interact with multiple windows without stealing focus or blocking input.

### M4 — Character packaging and additional targets
- Introduce `kitty.json` and licensed sprite/animation set, while preserving Bryan/Jorge configurations.
- Measure and optimize on x86-64, then validate 68k/ARM feasibility separately.
- **Done:** documented installation, resource usage, target-specific limitations and license inventory.

## Engineering rules

1. Work in small, reviewable commits; keep AROS changes confined to `ports/aros/` and dedicated documentation initially.
2. Preserve compatibility with upstream Pestinha and avoid unrelated Python refactors.
3. Distinguish **confirmed API presence** from **verified runtime behavior** on a given AROS build.
4. Do not silently replace the spiking neural network with a scripted state machine.
5. Do not ship Kitty or Pestinha assets without confirmed redistribution rights.
6. Report build/test evidence and limitations at each milestone.

## Immediate agent task

Implement **M0 and M1 investigation only**: inspect the AROS compositor and BoingIconBar code paths, establish the cross-compilation toolchain, generate the neural reference fixtures, and propose the smallest native alpha-window demo. Do not start a full neural rewrite before the fixture format and graphics feasibility are reviewed.
