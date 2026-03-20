# Three.js Skill Package — OPEN QUESTIONS

## Active Questions

### Q-001: MCP Server for Three.js?
**Status:** OPEN
**Question:** Should the package include a custom MCP server for Three.js scene manipulation, similar to Draw.io's MCP approach?
**Context:** Three.js is code-based (unlike Draw.io's XML), so an MCP server might serve a different purpose — e.g., serving a live preview, running a headless renderer, or providing model analysis tools.

### Q-005: Raycasting Placement
**Status:** ANSWERED (D-011)
**Answer:** Standalone skill `threejs-core-raycaster`. Research shows sufficient API surface: constructor, params, setFromCamera, intersectObject/intersectObjects, InstancedMesh picking, layers filtering, mouse picking patterns.

### Q-006: IFC Depth
**Status:** OPEN — B2 research determines
**Question:** How deep should the IFC skill cover @thatopen/components given AGPL-3.0?
**Context:** D-010 says include both web-ifc (MIT) and @thatopen/components (AGPL-3.0) with license warnings. Research determines exact scope.

---

## Answered Questions

### Q-002: Scope of IFC Viewer Skill
**Status:** ANSWERED (D-010)
**Answer:** Include both web-ifc and @thatopen/components with prominent license warnings. Exact depth from B2 research.

### Q-003: Shader Programming Depth
**Status:** ANSWERED (D-006)
**Answer:** Separate skill `threejs-syntax-shaders` for ShaderMaterial/RawShaderMaterial/GLSL. Materials skill covers PBR only.

### Q-004: R3F + Drei Split
**Status:** ANSWERED (D-007)
**Answer:** Split into two skills: `threejs-impl-react-three-fiber` (core R3F) and `threejs-impl-drei` (Drei helpers).
