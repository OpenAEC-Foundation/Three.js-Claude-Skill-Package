# Three.js Skill Package — LESSONS

## Lessons Learned

_Lessons inherited from Blender, ERPNext, and Draw.io skill packages._

### L-001: Research Before Skills
**Source:** Blender package, ERPNext package
**Lesson:** ALWAYS complete deep research (vooronderzoek) before writing any skills. Skills written without research require major rewrites.

### L-002: Deterministic Language
**Source:** ERPNext package
**Lesson:** ALWAYS use ALWAYS/NEVER language in skills. Avoid "might", "consider", "often", "usually". Deterministic skills produce deterministic code.

### L-003: Description Triggers
**Source:** Blender package (L-006)
**Lesson:** Skill descriptions MUST include trigger keywords. No "Deterministic" prefix. Format: "{What it does}. Triggers: {keywords}".

### L-004: Batch Quality Gates
**Source:** Blender package
**Lesson:** ALWAYS run validation between skill batches. Never start batch N+1 before batch N passes all quality checks.

### L-005: English-Only Skills
**Source:** Both packages
**Lesson:** All skill content MUST be in English. Claude reads English and responds in the user's language automatically.

### L-006: 500-Line Limit
**Source:** ERPNext package
**Lesson:** SKILL.md files exceeding 500 lines lose effectiveness. Split into the main file + references/ for detailed content.

### L-007: Self-Contained Skills
**Source:** Both packages
**Lesson:** Each skill MUST work independently. Cross-references are informational only, never required.

### L-008: Progressive Disclosure (Anthropic Official)
**Source:** Anthropic "Complete Guide to Building Skills for Claude" (2026)
**Lesson:** Skills use a 3-level progressive disclosure system: (1) YAML frontmatter — ALWAYS in system prompt, (2) SKILL.md body — loaded when relevant, (3) Linked files — loaded on demand. Keep SKILL.md focused; move details to references/.

### L-009: Description = Trigger Mechanism
**Source:** Anthropic Official Guide
**Lesson:** The description field is HOW Claude decides to load a skill. Format: `[What it does] + [When to use it] + [Key capabilities]`. MUST include trigger phrases users would actually say. Under 1024 chars. No XML tags.

### L-010: No README.md in Skill Folders
**Source:** Anthropic Official Guide
**Lesson:** NEVER include README.md inside a skill folder. All documentation goes in SKILL.md or references/. Repo-level README is separate and for human visitors.

### L-011: Skill Structure Expanded
**Source:** Anthropic Official Guide
**Lesson:** Official skill structure includes scripts/ (executable code) and assets/ (templates, fonts, icons) alongside references/. Consider using these for validation scripts and scene templates.

### L-012: allowed-tools Field
**Source:** Anthropic Official Guide, Claude Code Plugin Docs
**Lesson:** YAML frontmatter supports an `allowed-tools` field to restrict which tools a skill can use. Format: `"Bash(node:*) Bash(npm:*) WebFetch"`. Use for skills that need specific tool access.

### L-013: Plugin Distribution Model
**Source:** Claude Code Plugin Docs, anthropics/skills repo
**Lesson:** Skills can be distributed as Claude Code plugins with `.claude-plugin/plugin.json` manifest. Install via `/plugin marketplace add`. Consider submitting to official Anthropic marketplace.

### L-014: Research to Core Files Flow
**Source:** User feedback, 2026-03-16
**Lesson:** Research findings MUST ALWAYS be processed into the core files (CLAUDE.md, REQUIREMENTS.md, LESSONS.md, DECISIONS.md, SOURCES.md, WAY_OF_WORK.md). After processing: mark the research document with a processed status. Research is only "done" when it is in the core files.

### L-015: NEVER Interpret Fundamental Questions
**Source:** User feedback, 2026-03-16
**Lesson:** NEVER interpret fundamental questions yourself. ALWAYS ask the user explicitly. Record open questions in OPEN-QUESTIONS.md. This applies to all architecture, scope, approach, and tooling decisions. Interpretations are ONLY allowed for trivial implementation details.
