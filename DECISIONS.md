# DECISIONS

Architectural and process decisions with rationale. Each decision is numbered and immutable once recorded. New decisions may supersede old ones but old ones are never deleted.

---

## D-001: English-Only Skills
**Date**: 2026-03-19
**Status**: ACTIVE
**Context**: Team works primarily in Dutch, skills target international audience
**Decision**: ALL skill content in English only
**Rationale**: Skills are instructions for Claude, not end-user documentation. Claude reads English and responds in any language. Bilingual skills double maintenance with zero functional benefit. Proven in ERPNext, Blender, and Tauri projects.
**Reference**: ERPNext LESSONS_LEARNED.md, lesson on English-only skills

## D-002: MIT License
**Date**: 2026-03-19
**Status**: ACTIVE
**Context**: Need to choose open-source license
**Decision**: MIT License
**Rationale**: Most permissive, maximizes adoption. Consistent with OpenAEC Foundation philosophy and all other skill packages.

## D-003: SKILL.md < 500 Lines
**Date**: 2026-03-19
**Status**: ACTIVE
**Context**: Skills need to be concise enough for Claude to load efficiently
**Decision**: SKILL.md must be under 500 lines. Heavy content goes in references/
**Rationale**: Anthropic convention. ERPNext skills ranged 180-427 lines, all well under 500. Keeps the main skill focused on decision trees and quick reference.

## D-004: 7-Phase Research-First Methodology
**Date**: 2026-03-19
**Status**: ACTIVE
**Context**: Need a structured approach to build high-quality skills
**Decision**: Adopt the 7-phase methodology proven in the ERPNext, Blender-Bonsai, and Tauri 2 Skill Packages
**Rationale**: ERPNext produced 28 skills, Blender-Bonsai produced 73 skills, Tauri 2 produced 27 skills — all with this approach. Research-first prevents hallucinated content.
**Reference**: https://github.com/OpenAEC-Foundation/ERPNext_Anthropic_Claude_Development_Skill_Package/blob/main/WAY_OF_WORK.md

## D-005: ROADMAP.md as Single Source of Truth
**Date**: 2026-03-19
**Status**: ACTIVE
**Context**: Need to track project status across multiple sessions and agents
**Decision**: ROADMAP.md is the ONLY place where project status is tracked
**Rationale**: Multiple status locations cause drift and confusion. Single source prevents "which is current?" questions. Proven in all prior skill packages.

## D-006: WebFetch for Research
**Date**: 2026-03-19
**Status**: ACTIVE
**Context**: n8n evolves rapidly with frequent releases and API changes
**Decision**: Use WebFetch tool for all research to ensure latest documentation is consulted
**Rationale**: Claude's training data has a knowledge cutoff. n8n APIs may have changed since cutoff. WebFetch ensures research uses actual latest docs, not stale training data. Critical for API accuracy guarantees.

## D-007: GitHub under OpenAEC Foundation
**Date**: 2026-03-19
**Status**: ACTIVE
**Context**: Need to host the repository publicly
**Decision**: Repository hosted at `OpenAEC-Foundation/n8n-Claude-Skill-Package` on GitHub
**Rationale**: Consistent with all other skill packages under the OpenAEC Foundation organization. Provides discoverability and organizational credibility.

## D-008: Merge core-data-flow into core-architecture
**Date**: 2026-03-19
**Status**: ACTIVE
**Context**: Raw masterplan had separate data-flow and architecture skills
**Decision**: Merge n8n-core-data-flow into n8n-core-architecture
**Rationale**: Item-based data flow IS the core architecture. Splitting forces loading two skills for one concept. Research confirmed data flow is inseparable from execution model.

## D-009: Merge backup-restore into deployment
**Date**: 2026-03-19
**Status**: ACTIVE
**Context**: Raw masterplan had separate backup-restore skill
**Decision**: Merge n8n-impl-backup-restore into n8n-impl-deployment
**Rationale**: CLI export/import is only 3 commands. Backup is a deployment operations concern, too thin for its own skill.

## D-010: Add extension-methods as standalone skill
**Date**: 2026-03-19
**Status**: ACTIVE
**Context**: Research revealed 80+ n8n-specific expression extension methods
**Decision**: Create n8n-syntax-extension-methods as standalone skill
**Rationale**: 17 string + 21 array + 13 number + 12 object + 3 boolean + 25+ DateTime methods. Too substantial for expressions skill references. Needs its own decision trees and reference tables.

## D-011: Merge topic research into skill creation
**Date**: 2026-03-19
**Status**: ACTIVE
**Context**: Phase 4 (topic-specific research) would add a separate step
**Decision**: Merge Phase 4 into Phase 5 — agents read research fragments directly during skill creation
**Rationale**: Research fragments are comprehensive enough (3 documents, covering all areas). Separate topic research would be redundant. Agents can reference fragments directly.
