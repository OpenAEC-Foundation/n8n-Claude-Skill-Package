# Approved Sources Registry

All skills in this package MUST be verified against these approved sources. No other sources are authoritative.

## Official Documentation

| Source | URL | Purpose |
|--------|-----|---------|
| n8n Documentation | https://docs.n8n.io/ | Primary reference for all n8n content |
| n8n Node Development | https://docs.n8n.io/integrations/creating-nodes/ | Custom node development guide |
| n8n Workflow Concepts | https://docs.n8n.io/workflows/ | Workflow fundamentals and concepts |
| n8n Code Examples | https://docs.n8n.io/code/ | Code node, expressions, JavaScript/Python |
| n8n Hosting | https://docs.n8n.io/hosting/ | Self-hosting, Docker, configuration |
| n8n API Reference | https://docs.n8n.io/api/ | REST API endpoints and authentication |
| n8n Credentials | https://docs.n8n.io/integrations/creating-nodes/build/reference/credentials/ | Credential type development |

## Source Code

| Source | URL | Purpose |
|--------|-----|---------|
| n8n Core Repository | https://github.com/n8n-io/n8n | Main monorepo (core, nodes, CLI, editor) |
| n8n Nodes Base | https://github.com/n8n-io/n8n/tree/master/packages/nodes-base | Official node implementations |
| n8n Node Dev CLI | https://github.com/n8n-io/n8n/tree/master/packages/cli | CLI tool source |
| n8n Workflow Package | https://github.com/n8n-io/n8n/tree/master/packages/workflow | Core workflow engine |
| n8n Starter Node | https://github.com/n8n-io/n8n-nodes-starter | Community node template |

## Package/Module Docs

| Source | URL | Purpose |
|--------|-----|---------|
| n8n npm packages | https://www.npmjs.com/search?q=%40n8n | Official npm packages |
| n8n-workflow package | https://www.npmjs.com/package/n8n-workflow | Core workflow types and interfaces |
| n8n-core package | https://www.npmjs.com/package/n8n-core | Core execution engine |

## Community (for Anti-Pattern Research)

| Source | URL | Purpose |
|--------|-----|---------|
| n8n Community Forum | https://community.n8n.io/ | Community Q&A, real-world patterns |
| GitHub Issues | https://github.com/n8n-io/n8n/issues | Real-world error patterns |
| GitHub Discussions | https://github.com/n8n-io/n8n/discussions | Architecture discussions, Q&A |

## Claude / Anthropic (Skill Development Platform)

| Source | URL | Purpose |
|--------|-----|---------|
| Agent Skills Standard | https://agentskills.io | Open standard |
| Agent Skills Spec | https://github.com/agentskills/agentskills | Specification |
| Agent Skills Best Practices | https://platform.claude.com/docs/en/agents-and-tools/agent-skills/best-practices | Authoring guide |

## OpenAEC Foundation

| Source | URL | Purpose |
|--------|-----|---------|
| ERPNext Skill Package | https://github.com/OpenAEC-Foundation/ERPNext_Anthropic_Claude_Development_Skill_Package | Methodology template |
| Blender Skill Package | https://github.com/OpenAEC-Foundation/Blender-Bonsai-ifcOpenshell-Sverchok-Claude-Skill-Package | Proven 73-skill package |
| Tauri 2 Skill Package | https://github.com/OpenAEC-Foundation/Tauri-2-Claude-Skill-Package | Proven 27-skill package |

## Source Verification Rules

1. **Primary sources ONLY** — Official docs, official repos, official npm packages.
2. **NEVER trust random blog posts** — Even popular ones may be outdated or wrong for v1.x.
3. **Verify code against official docs** — Every code snippet in a skill MUST match current API.
4. **Note when source was last verified** — Track in the table below.
5. **Cross-reference if docs are sparse** — When official docs lack detail, verify against source code in the monorepo.

## Last Verified

| Technology | Date | Action | Notes |
|------------|------|--------|-------|
| n8n | 2026-03-19 | Initial setup | All URLs pending verification |
