# Three.js Skill Package — DECISIONS

## Architectural Decisions

### D-001: Research-First Methodology
**Date:** 2026-03-20
**Decision:** Use the proven 7-phase research-first methodology from Blender, ERPNext, and Draw.io packages.
**Rationale:** This approach produces higher-quality, deterministic skills because research precedes creation. Proven across 120+ skills in prior packages.

### D-002: Three.js r160+ as Baseline
**Date:** 2026-03-20
**Decision:** Target Three.js r160+ as the minimum supported version, using ES module imports exclusively.
**Rationale:** r160+ represents the modern Three.js era with stable ES module support, WebGPU renderer availability, and deprecation of legacy patterns. Older versions (pre-r150) used global `THREE` namespace and CommonJS — these patterns are obsolete.

### D-003: React Three Fiber as First-Class Integration
**Date:** 2026-03-20
**Decision:** Include React Three Fiber 8.x and Drei as first-class citizens with a dedicated implementation skill, not just a secondary mention.
**Rationale:** R3F is the dominant way to use Three.js in React applications. Over 27k GitHub stars. Drei provides essential abstractions (OrbitControls, Environment, PresentationControls) that most R3F projects use. A dedicated skill prevents the common mistake of mixing imperative Three.js with R3F's declarative model.

### D-004: WebGPU Renderer Included as Experimental
**Date:** 2026-03-20
**Decision:** Include a dedicated WebGPU renderer skill covering TSL (Three Shading Language) node-based materials, marked as experimental.
**Rationale:** Three.js WebGPU renderer is actively developed and represents the future of browser 3D rendering. Developers need guidance on when to use WebGPU vs WebGL, TSL node materials vs GLSL shaders, and browser compatibility. Marking it experimental sets correct expectations while providing actionable patterns.

### D-005: Maximum Atomization of Skills
**Date:** 2026-03-20
**Decision:** Split skills into the smallest coherent units. Prefer more smaller skills over fewer larger skills.
**Rationale:** User preference for engineering-quality atomic skills. Smaller skills are more precise triggers, easier to maintain, and compose better. This may increase the skill count from ~19 to ~25+.

### D-006: Custom Shaders as Separate Skill
**Date:** 2026-03-20
**Decision:** ShaderMaterial/RawShaderMaterial/GLSL gets its own skill (`threejs-syntax-shaders`), separate from `threejs-syntax-materials`.
**Rationale:** Custom shaders is a large topic (uniforms, varyings, GLSL syntax, ShaderChunk, onBeforeCompile) that would bloat the materials skill past the 500-line limit. Atomic skill principle.

### D-007: R3F and Drei Split into Two Skills
**Date:** 2026-03-20
**Decision:** Split React Three Fiber into `threejs-impl-react-three-fiber` (core R3F) and `threejs-impl-drei` (Drei helpers).
**Rationale:** R3F core (Canvas, hooks, event system, JSX mapping) is substantial enough for one skill. Drei has 50+ components across 10 categories — a separate skill ensures proper coverage. Atomic skill principle.

### D-008: Include Audio Skill
**Date:** 2026-03-20
**Decision:** Include `threejs-impl-audio` covering AudioListener, Audio, PositionalAudio.
**Rationale:** Maximum coverage principle. 3D spatial audio is a real use case (games, VR, interactive experiences). Small but self-contained API surface.

### D-009: Include XR/VR Skill
**Date:** 2026-03-20
**Decision:** Include `threejs-impl-xr` covering WebXR session management, VR/AR controllers, hand tracking, hit testing.
**Rationale:** Maximum coverage principle. WebXR is increasingly important and Three.js has built-in support. Developers need guidance on session types, controller events, and teleportation patterns.

### D-010: IFC Skill Scope — Research Determines Depth
**Date:** 2026-03-20
**Decision:** IFC skill covers both web-ifc (MIT) and @thatopen/components (AGPL-3.0) with clear license documentation. Exact depth determined by B2 research.
**Rationale:** This is an open source contribution so AGPL-3.0 is not a blocker, but users MUST be informed about license implications for their own projects. Skill MUST include prominent license warning.

### D-011: Raycasting Scope — Research Determines Placement
**Date:** 2026-03-20
**Decision:** B2 research determines whether raycasting stays in core-scene-graph or becomes a separate skill.
**Rationale:** Raycaster touches scene graph, controls, and physics. Research needed to determine if the API surface justifies a standalone skill.
