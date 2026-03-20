# Three.js Skill Package — WAY OF WORK

## 7-Phase Research-First Methodology

This methodology is proven across the Blender (73 skills), ERPNext (28 skills), Draw.io (22 skills), and Open-Agents (28 skills) packages.

---

### Phase 1: Raw Masterplan
**Goal:** Define scope and preliminary skill inventory.
- Map the Three.js technology landscape (core, renderers, geometries, materials, loaders, controls, post-processing, physics, R3F)
- Identify all API surfaces, rendering pipelines, and integration points
- Create preliminary skill list with categories
- Define dependencies between skills
- Output: `docs/masterplan/raw-masterplan.md`

### Phase 2: Deep Research (Vooronderzoek)
**Goal:** Build comprehensive knowledge base before writing any skills.
- Minimum 2000 words per research document
- Sources: official docs, source code (MIT license), community resources
- Research documents:
  - `vooronderzoek-threejs-core.md` — Scene graph, renderer, camera, math utilities
  - `vooronderzoek-threejs-geometry-materials.md` — BufferGeometry, materials system, textures, shaders
  - `vooronderzoek-threejs-features.md` — Lighting, shadows, animation, post-processing, physics
  - `vooronderzoek-threejs-ecosystem.md` — React Three Fiber, Drei, WebGPU, loaders, controls
- Output: `docs/research/vooronderzoek-*.md`

### Phase 3: Masterplan Refinement
**Goal:** Finalize skill inventory based on research findings.
- Review research against preliminary skill list
- Merge/split/add skills based on findings
- Define per-skill specification: category, scope, API surface, sources, dependencies, complexity
- Write execution prompts for each skill
- Output: definitive `docs/masterplan/masterplan.md`

### Phase 4: Topic-Specific Research
**Goal:** Deep dive into each skill's specific domain.
- One research document per skill (or per logical group)
- Focus on exact API patterns, class hierarchies, shader programs
- Document anti-patterns and common mistakes
- Output: `docs/research/topic-research/{skill-name}-research.md`

### Phase 5: Skill Creation
**Goal:** Write all skills in quality-gated batches.
- 3 agents per batch, parallel execution
- Batch order follows dependency graph:
  1. Core skills (foundation, no dependencies)
  2. Syntax skills (depends on core)
  3. Basic impl skills (depends on syntax)
  4. Complex impl skills (depends on syntax)
  5. Remaining impl + error skills
  6. Agent skills (depends on all other skills)
- Quality gate between every batch: run validation, review output

### Phase 6: Validation
**Goal:** Verify all skills meet quality requirements.
- Structural: frontmatter, line count, references
- Content: deterministic language, English-only
- Functional: test scene generation using skills
- Cross-reference: verify skill dependencies are correct
- Output: `docs/validation/validation-report.md`

### Phase 7: Publication
**Goal:** Package and publish the skill package.
- Finalize INDEX.md with complete skill catalog
- Write README.md with installation instructions
- Update CHANGELOG.md
- Tag release v1.0.0
- Update ROADMAP.md to 100% complete

---

## Skill Structure

```
skills/source/threejs-{category}/{threejs-{category}-{topic}}/
├── SKILL.md              # Main skill (< 500 lines, YAML frontmatter)
└── references/
    ├── methods.md         # API methods, classes, and properties
    ├── examples.md        # Working code examples
    └── anti-patterns.md   # What NOT to do
```

### SKILL.md Format
```yaml
---
name: threejs-{category}-{topic}
description: >
  Use when [specific trigger scenario -- what the user is doing or asking].
  Prevents the [common mistake / anti-pattern this skill guards against].
  Covers [key topics, API areas, version differences].
  Keywords: [comma-separated technical terms users might type in their prompt].
---

## Quick Reference
[One-paragraph overview]

## Critical Rules
- ALWAYS {do X} — {reason}
- NEVER {do Y} — {reason}

## Decision Tree
[When to use what]

## Essential Patterns
[Core code patterns with examples]

## Common Operations
[Step-by-step workflows]

## Reference Links
[Links to official documentation]
```

---

## Quality Gates

Between every phase:
1. ROADMAP.md updated with current status
2. LESSONS.md updated with new learnings
3. DECISIONS.md updated with new decisions
4. All changes committed to git
