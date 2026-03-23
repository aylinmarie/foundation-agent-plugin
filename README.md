# agent-toolkit

A stack-agnostic skills directory for AI coding agents. One canonical set of opinionated skill prompts, deployable to Cursor, Claude Code, GitHub Copilot, and Windsurf via framework adapters.

## Installation

**Requirements:** Node.js ≥18, Bash

```bash
git clone https://github.com/your-org/agent-toolkit
cd agent-toolkit
chmod +x install.sh
```

**Install to your project:**

```bash
# Install all adapters
./install.sh all /path/to/your/project

# Install a specific adapter
./install.sh claude /path/to/your/project
./install.sh cursor /path/to/your/project
./install.sh copilot /path/to/your/project
./install.sh windsurf /path/to/your/project
```

Each adapter writes to a different location in your project:

| Adapter | Output |
|---------|--------|
| `claude` | `.claude/CLAUDE.md` |
| `copilot` | `.github/copilot-instructions.md` |
| `cursor` | `.cursor/rules/<category>-<skill>.mdc` (one file per skill) |
| `windsurf` | `.windsurfrules` |

## Directory Map

```
agent-toolkit/
├── skills/                     ← Canonical skill definitions
│   ├── frontend/               ← Client-layer UI and style skills
│   │   ├── accessibility.md
│   │   ├── css-patterns.md
│   │   ├── design-tokens.md
│   │   └── security-audit.md
│   ├── core/                   ← Universal development skills
│   │   ├── codegen.md
│   │   ├── refactor.md
│   │   ├── debug.md
│   │   ├── file-ops.md
│   │   └── search.md
│   ├── git/                    ← Version control workflows
│   │   ├── commit.md
│   │   ├── branch.md
│   │   ├── pr.md
│   │   └── conflict-resolve.md
│   ├── testing/                ← Test authoring skills
│   │   ├── unit.md
│   │   ├── integration.md
│   │   ├── e2e.md
│   │   └── snapshot.md
│   ├── docs/                   ← Documentation skills
│   │   ├── readme.md
│   │   ├── inline-comments.md
│   │   ├── changelog.md
│   │   └── adr.md
│   ├── architecture/           ← System design and review
│   │   ├── plan.md
│   │   ├── review.md
│   │   └── tradeoffs.md
│   ├── security/               ← Full-stack security skills
│   │   ├── audit.md
│   │   ├── secrets.md
│   │   └── dependency-scan.md
│   └── devops/                 ← Infrastructure and delivery
│       ├── ci.md
│       ├── env-setup.md
│       ├── docker.md
│       └── deploy.md
├── adapters/
│   ├── claude/build.js         ← Builds .claude/CLAUDE.md
│   ├── copilot/build.js        ← Builds .github/copilot-instructions.md
│   ├── cursor/                 ← Symlinks (no build script needed)
│   └── windsurf/               ← Concatenation via install.sh
├── registry.json               ← Skill manifest
├── install.sh                  ← Adapter installer
└── README.md
```

## How It Works

### Skill files

Each skill is a Markdown file with YAML frontmatter:

```yaml
---
id: category/slug
name: Human-Readable Name
description: One sentence.
triggers:
  - natural language phrase that activates this skill
agents:
  - claude
  - cursor
  - copilot
  - windsurf
version: 1.0.0
standards:
  - Relevant Standard vX.Y
---
```

The body follows a consistent structure:

1. **Role** — who the agent is acting as
2. **Core Principles** — non-negotiable behaviors (numbered, opinionated)
3. **Patterns** — concrete code examples of correct and incorrect approaches
4. **Process** — the sequence of steps to follow
5. **Output Format** — exact format the agent should produce

### registry.json

The registry is the single source of truth for all skills. Adapters read it to discover skill files, their categories, and their metadata.

Each entry:
```json
{
  "id": "git/commit",
  "path": "skills/git/commit.md",
  "category": "git",
  "name": "Commit",
  "description": "...",
  "triggers": ["write commit", "commit message"],
  "agents": ["claude", "cursor", "copilot", "windsurf"],
  "version": "1.0.0",
  "standards": ["Conventional Commits 1.0"]
}
```

### Adapters

**Claude (`adapters/claude/build.js`):**
Reads all skill files, strips frontmatter, concatenates into `.claude/CLAUDE.md`. Accepts `--filter=category:<name>` to build a subset.

```bash
node adapters/claude/build.js --target=/path/to/project
node adapters/claude/build.js --target=. --filter=category:git
```

**Copilot (`adapters/copilot/build.js`):**
Same logic, outputs to `.github/copilot-instructions.md`.

**Cursor:**
`install.sh cursor` creates one `.mdc` symlink per skill in `.cursor/rules/`. Cursor loads these as per-rule context. Symlinks point back to the source `.md` files — editing the source immediately updates Cursor's context.

**Windsurf:**
`install.sh windsurf` concatenates all skills into `.windsurfrules` (Windsurf's single-file format).

## Adding a Skill

1. **Create the skill file** in the appropriate `skills/<category>/` directory:

```bash
touch skills/devops/monitoring.md
```

2. **Write the frontmatter:**

```yaml
---
id: devops/monitoring
name: Monitoring Setup
description: Configure observability with metrics, logs, and traces for production services.
triggers:
  - set up monitoring
  - add observability
  - configure metrics
agents:
  - claude
  - cursor
  - copilot
  - windsurf
version: 1.0.0
standards:
  - OpenTelemetry
  - SRE (Google)
---
```

3. **Write the body.** Follow the structure: Role → Core Principles → Patterns → Process → Output Format. Be opinionated — state what to do, not "you might consider."

4. **Register the skill** in `registry.json`:

```json
{
  "id": "devops/monitoring",
  "path": "skills/devops/monitoring.md",
  "category": "devops",
  "name": "Monitoring Setup",
  "description": "Configure observability...",
  "triggers": ["set up monitoring"],
  "agents": ["claude", "cursor", "copilot", "windsurf"],
  "version": "1.0.0",
  "standards": ["OpenTelemetry"]
}
```

5. **Re-run the installer** in any projects that have already installed this toolkit:

```bash
./install.sh all /path/to/project
```

## Skill Authoring Contract

### What makes a good skill

- **Opinionated about process.** Don't say "consider your options." Say what to do and in what order.
- **Opinionated about output.** Specify the exact format the agent should produce.
- **Standards-grounded.** Reference a specific standard (OWASP, Conventional Commits, WCAG) rather than vague best practices.
- **Examples over prose.** A before/after code example is worth three paragraphs of explanation.
- **Stack-agnostic.** No framework names in principles. Patterns may show examples in a specific language, but the principle applies to all languages.

### What to avoid

- Wishy-washy language: "you might want to", "it depends", "consider"
- Framework opinions in the Role or Core Principles sections
- Magic numbers without citations
- Output formats with "feel free to adapt" disclaimers
- Skills that duplicate another skill's scope

## Contributing

```bash
# Verify skill frontmatter is valid
node -e "
const fs = require('fs');
const yaml = require('js-yaml');
const files = require('child_process').execSync('find skills -name *.md').toString().trim().split('\n');
files.forEach(f => {
  const content = fs.readFileSync(f, 'utf8');
  const match = content.match(/^---\n([\s\S]*?)\n---/);
  if (!match) { console.error('Missing frontmatter:', f); process.exit(1); }
  const fm = yaml.load(match[1]);
  ['id','name','description','triggers','agents','version'].forEach(key => {
    if (!fm[key]) { console.error('Missing field', key, 'in', f); process.exit(1); }
  });
});
console.log('All frontmatter valid.');
"

# Run the adapters to verify they build cleanly
node adapters/claude/build.js --target=/tmp/toolkit-test
node adapters/copilot/build.js --target=/tmp/toolkit-test
```

## License

MIT
