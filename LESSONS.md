# Lessons Learned

Observations and findings captured during skill package development.

---

## L-001: Parallel Research Agents Are Highly Effective
**Date**: 2026-03-19
**Context**: Phase 2 deep research
**Lesson**: Splitting n8n research into 3 parallel agents (core architecture, expressions/credentials, deployment/API) covered the entire platform in a single pass. Each agent produced 200+ lines of verified research. Combined vooronderzoek has 20 sections.

## L-002: Merging Topic Research Into Skill Creation Saves a Phase
**Date**: 2026-03-19
**Context**: Phase 4 → merged into Phase 5
**Lesson**: When research fragments are comprehensive enough, separate topic-specific research is redundant. Agents can read research fragments directly during skill creation. This eliminated an entire phase without quality loss.

## L-003: n8n Connection Format Is the #1 Error Source
**Date**: 2026-03-19
**Context**: Phase 2 research on workflow JSON
**Lesson**: IConnections uses 3-level nesting (nodeName → connectionType → outputIndex → targets[]). This is the most common source of errors in workflow JSON. The syntax-workflow-json skill documents this thoroughly.

## L-004: n8n Has 22 Node Property Types
**Date**: 2026-03-19
**Context**: Phase 2 research on INodeProperties
**Lesson**: Beyond basic types (string, number, boolean, options), n8n has specialized types like resourceMapper, assignmentCollection, filter, json, fixedCollection. The node-types skill covers all 22.

## L-005: Code Node Has 5 Critical Restrictions
**Date**: 2026-03-19
**Context**: Phase 2 research on Code node
**Lesson**: No filesystem, no HTTP, no $itemIndex, no $secrets, no $parameter. These restrictions catch many developers off guard. Documented as Critical Warnings in syntax-code-node skill.

## L-006: Single Megacommit Anti-Pattern
**Date**: 2026-03-19
**Context**: Phase 5 skill creation — all 7 batches committed at once
**Lesson**: Committing 96 files in a single commit (research + masterplan + all 21 skills) makes review and rollback nearly impossible. Future packages MUST commit per batch (3 skills) or per phase. Atomic commits preserve the ability to identify when specific issues were introduced.

## L-007: YAML Quoted Strings vs Folded Block Scalar
**Date**: 2026-03-19
**Context**: Phase 6 validation — missed YAML format issue
**Lesson**: The Phase 6 validation checked for YAML frontmatter presence but did not verify the description format (quoted string vs folded block scalar `>`). All 21 skills used quoted strings instead of the required `>` format. Future validation checklists MUST include a specific check for YAML block scalar format.

## L-008: 21 Skills in 7 Batches Is Achievable in One Session
**Date**: 2026-03-19
**Context**: Phase 5 skill creation
**Lesson**: With comprehensive research (3648-line vooronderzoek) and a detailed masterplan with ready-to-execute agent prompts, creating 21 skills in 7 batches of 3 agents is achievable in a single Claude Code session. The key enabler is thorough Phase 2 research that eliminates guesswork during creation.
