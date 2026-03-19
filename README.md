# n8n Claude Skill Package

![Claude Code Ready](https://img.shields.io/badge/Claude_Code-Ready-blue?style=flat-square&logo=data:image/svg+xml;base64,PHN2ZyB4bWxucz0iaHR0cDovL3d3dy53My5vcmcvMjAwMC9zdmciIHZpZXdCb3g9IjAgMCAyNCAyNCI+PHBhdGggZD0iTTEyIDJDNi40OCAyIDIgNi40OCAyIDEyczQuNDggMTAgMTAgMTAgMTAtNC40OCAxMC0xMFMxNy41MiAyIDEyIDJ6IiBmaWxsPSIjZmZmIi8+PC9zdmc+)
![n8n v1.x](https://img.shields.io/badge/n8n-v1.x-EA4B71?style=flat-square)
![TypeScript + JSON](https://img.shields.io/badge/TypeScript_+_JSON-Workflow_Automation-3178C6?style=flat-square)
![Phase 1](https://img.shields.io/badge/Phase_1-Infrastructure_Ready-orange?style=flat-square)
![License](https://img.shields.io/badge/License-MIT-yellow?style=flat-square)

**Deterministic Claude AI skills for n8n workflow automation — visual node-based editor for integrating APIs, services, and data.**

Built on the [Agent Skills](https://agentskills.org) open standard.

---

## Why This Exists

Without skills, Claude generates incorrect n8n patterns:

```typescript
// Wrong — incomplete INodeType, missing required properties
export class MyNode {
  execute() {
    return [{ json: { result: 'data' } }];
  }
}
```

With this skill package, Claude produces correct n8n code:

```typescript
// Correct — complete INodeType with description, properties, and proper return type
export class MyNode implements INodeType {
  description: INodeTypeDescription = {
    displayName: 'My Node',
    name: 'myNode',
    group: ['transform'],
    version: 1,
    description: 'Description of the node',
    defaults: { name: 'My Node' },
    inputs: ['main'],
    outputs: ['main'],
    properties: [],
  };

  async execute(this: IExecuteFunctions): Promise<INodeExecutionData[][]> {
    const items = this.getInputData();
    const returnData: INodeExecutionData[] = [];
    for (let i = 0; i < items.length; i++) {
      returnData.push({ json: { result: 'data' } });
    }
    return [returnData];
  }
}
```

---

## Current Progress

**Phase 1 — Infrastructure Ready** (50%)

Core governance files and repository structure established. Next: raw masterplan and deep research.

## Skill Categories

| Category | Purpose |
|----------|---------|
| `n8n-syntax/` | Node APIs, expression syntax, workflow JSON, credential types |
| `n8n-impl/` | Custom node development, deployment, integration workflows |
| `n8n-errors/` | Error diagnosis, execution failures, debugging patterns |
| `n8n-core/` | Architecture, execution model, API overview |
| `n8n-agents/` | Workflow validation, code generation orchestration |

## Tech Coverage (Planned)

| Area | Topics |
|------|--------|
| **Workflow Engine** | Execution model, trigger types, node connections, data flow |
| **Custom Nodes** | INodeType, INodeTypeDescription, execute methods, credential integration |
| **Expressions** | $json, $input, $node, $workflow references, JMESPath |
| **Credentials** | ICredentialType, OAuth2, API key, custom authentication |
| **REST API** | Workflow CRUD, execution management, webhook API |
| **Deployment** | Docker, environment configuration, queue mode, scaling |
| **Error Handling** | continueOnFail, error workflows, retry strategies |

## Installation

### Claude Code

```bash
# Option 1: Clone the full package
git clone https://github.com/OpenAEC-Foundation/n8n-Claude-Skill-Package.git
cp -r n8n-Claude-Skill-Package/skills/source/ ~/.claude/skills/n8n/

# Option 2: Add as git submodule
git submodule add https://github.com/OpenAEC-Foundation/n8n-Claude-Skill-Package.git .claude/skills/n8n
```

### Claude.ai (Web)

Upload individual SKILL.md files as project knowledge.

## Methodology

This package is developed using the **7-phase research-first methodology**, proven across multiple skill packages:

1. **Setup + Raw Masterplan** — Project structure and governance files
2. **Deep Research** — Comprehensive source analysis of n8n documentation, source code, and community resources
3. **Masterplan Refinement** — Skill inventory refinement based on research findings
4. **Topic-Specific Research** — Deep-dive per skill topic
5. **Skill Creation** — Deterministic skill files following Agent Skills standard
6. **Validation** — Correctness, completeness, and consistency checks
7. **Publication** — GitHub release and documentation

## Documentation

| Document | Purpose |
|----------|---------|
| [ROADMAP.md](ROADMAP.md) | Project status (single source of truth) |
| [REQUIREMENTS.md](REQUIREMENTS.md) | Quality guarantees and per-area requirements |
| [DECISIONS.md](DECISIONS.md) | Architectural decisions with rationale |
| [SOURCES.md](SOURCES.md) | Official reference URLs and verification rules |
| [WAY_OF_WORK.md](WAY_OF_WORK.md) | 7-phase development methodology |
| [LESSONS.md](LESSONS.md) | Lessons learned during development |
| [CHANGELOG.md](CHANGELOG.md) | Version history |

## Related Projects

| Project | Description |
|---------|-------------|
| [ERPNext Skill Package](https://github.com/OpenAEC-Foundation/ERPNext_Anthropic_Claude_Development_Skill_Package) | 28 skills for ERPNext/Frappe development |
| [Blender-Bonsai Skill Package](https://github.com/OpenAEC-Foundation/Blender-Bonsai-ifcOpenshell-Sverchok-Claude-Skill-Package) | 73 skills for Blender, Bonsai, IfcOpenShell & Sverchok |
| [Tauri 2 Skill Package](https://github.com/OpenAEC-Foundation/Tauri-2-Claude-Skill-Package) | 27 skills for Tauri 2 desktop application development |
| [OpenAEC Foundation](https://github.com/OpenAEC-Foundation) | Parent organization |

## License

[MIT](LICENSE)

---

Part of the [OpenAEC Foundation](https://github.com/OpenAEC-Foundation) ecosystem.
