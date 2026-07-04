# Agentic Support Skills

Support skills for agentic workspaces. Each skill helps agents configure, deploy, and maintain their working environment.

## Skills

| Skill | Description |
|---|---|
| [agentic-environment-settings](skills/agentic-environment-settings/) | Discovers, validates, and deploys all configuration (env vars, secrets, files, structured configs) that a set of agents needs to run. |

## Installation

```bash
npx skills add makeyourcrew/agentic-support-skills
```

Or install an individual skill:

```bash
npx skills add makeyourcrew/agentic-support-skills/skills/agentic-environment-settings
```

## Structure

```
agentic-support-skills/
├── skills/
│   └── agentic-environment-settings/
│       ├── SKILL.md                          ← Skill instructions (auto-detected)
│       ├── schemas/
│       │   └── environment.schema.json       ← JSON Schema for environment.json
│       └── references/
│           └── examples.md                   ← Real-world examples
├── README.md
└── LICENSE
```

## Contributing

1. Create a new directory under `skills/<skill-name>/`
2. Add a `SKILL.md` with frontmatter (`name`, `description`) and instructions
3. Add supporting files (`schemas/`, `references/`, etc.)
4. Submit a PR

## License

MIT
