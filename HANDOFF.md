# Handoff: Three.js-Claude-Skill-Package

> Gegenereerd: 2026-03-20 vanuit Skill-Package-Workflow-Template

## Status
- **Fase:** B0 Bootstrap DONE — docs compleet, skills/ structuur aangemaakt
- **Skills:** 0/19 geschreven
- **GitHub remote:** Nog niet aangemaakt
- **README:** Ontbreekt

## Wat er is
Alle governance docs zijn klaar: CLAUDE.md, ROADMAP.md, INDEX.md, WAY_OF_WORK.md, DECISIONS.md, LESSONS.md, SOURCES.md, REQUIREMENTS.md, CHANGELOG.md, START-PROMPT.md.
De `skills/source/` directory structuur met 19 lege skill mappen staat klaar.

## Wat er moet gebeuren
1. **Fase B1: Raw Masterplan** — Map het Three.js landschap (r160+)
2. **Fase B2: Deep Research** — 4 research documenten (min. 2000 woorden elk)
3. **Fase B3: Masterplan Refinement** — Skill inventory definitief maken
4. **Fase B4+B5: Skill Creation** — Batches van 3 skills, dependency-aware
5. **Fase B6: Validation** — Quality gates
6. **Fase B7: Publication** — README, LICENSE, GitHub remote, compliance audit

## Batch volgorde
1. core (scene-graph, renderer, math) — 3 skills
2. syntax (geometries, materials, loaders, controls) — 4 skills
3. impl deel 1 (lighting, shadows, post-processing, animation) — 4 skills
4. impl deel 2 (physics, react-three-fiber, webgpu, ifc-viewer) — 4 skills
5. errors (performance, rendering) — 2 skills
6. agents (scene-builder, model-optimizer) — 2 skills

## Hoe te starten
Open deze map in VS Code, Claude Code, typ:
```
Lees START-PROMPT.md en begin fase B1 (Raw Masterplan).
Kopieer threejs-bim skill uit ThatOpenCompany/ als startpunt voor impl/ifc-viewer.
```

## Bijzonderheden
- Three.js r160+ als target (WebGPU renderer stabiel)
- React Three Fiber 8.x als apart impl skill
- IFC viewer skill overlapt met ThatOpen package — duidelijke afbakening nodig
- ThatOpenCompany repo heeft threejs-bim skill als research basis
