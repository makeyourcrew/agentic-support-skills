---
name: agentic-environment-settings
description: >
  Identifies, validates, and configures all environment variables, secrets, files,
  and structured configuration that a set of agents needs to run in a workspace.
  Use when setting up a new agent workspace, preparing a deployment, debugging
  missing configuration, generating an environment.json manifest, or validating
  that all required configuration is present before launch. Triggers on
  "environment setup", "configure agents", "missing variables", "secrets",
  "deployment config", "environment.json", "workspace setup", "agent configuration",
  "prepare environment", or "what does this workspace need".
---

# Agentic Environment Settings

## What This Skill Does

This skill manages the full lifecycle of configuration for agent workspaces:

1. **Discover** — Scans the workspace to find every piece of configuration the agents need.
2. **Manifest** — Creates or updates an `environment.json` file that declares all requirements.
3. **Validate** — Checks that every requirement has a value and that values match expected formats.
4. **Prepare** — Generates templates, prompts the user for missing values, and resolves external secrets.
5. **Deploy** — Writes the final configuration to the correct targets (env vars, .env files, vault entries, config files).

## Core Concept: `environment.json`

The `environment.json` file is the single source of truth for what a workspace needs. It lives at the root of the workspace (or at a path the user specifies). Its structure follows the JSON Schema in `schemas/environment.schema.json`.

```json
{
  "version": "1.0",
  "workspace": {
    "name": "SEO Team Workspace",
    "description": "Technical SEO audits and content optimization",
    "environment": "production"
  },
  "requirements": [
    {
      "key": "DEEPSEEK_API_KEY",
      "description": "API key for the DeepSeek LLM service. Used by the Xerraire chat agent to process user messages and route tasks. Obtain from https://platform.deepseek.com/api-keys",
      "type": "secret",
      "required": true,
      "target": "vault:agent-secrets",
      "format": "string",
      "sensitive": true,
      "source": "manual",
      "example": "sk-xxxxxxxxxxxxxxxx",
      "validation": {
        "pattern": "^sk-[a-zA-Z0-9]+$",
        "minLength": 20
      },
      "agentScope": ["xerraire"]
    },
    {
      "key": "agents-config",
      "description": "Main agent configuration file. Defines which agents are available, their system prompts, and tool permissions. Must be valid JSON.",
      "type": "config",
      "required": true,
      "target": "/app/config/agents.json",
      "format": "json",
      "source": "manual",
      "example": "{\"agents\": [\"jan\", \"seo-auditor\"]}",
      "dependsOn": ["DEEPSEEK_API_KEY"]
    }
  ]
}
```

---

## Workflow

### Phase 1: Discovery

The goal is to find every configuration dependency in the workspace.

**What to search for:**

| Pattern | What it means | How to find it |
|---|---|---|
| `process.env.XXX` | Environment variable dependency | Grep source code for `process.env.` |
| `.env` / `.env.local` / `.env.production` | Environment file usage | Look for dotenv imports and `.env*` files |
| Vault / secrets manager calls | Secret dependencies | Grep for `vault`, `getSecret`, `SecretManager`, `secrets.get` |
| Config file reads | File-based configuration | Grep for `readFile`, `fs.readFile`, `loadConfig`, `require('./config')`, `import.*config` |
| Cloud SDK initializations | Cloud provider credentials | Grep for `AWS_`, `GCP_`, `GOOGLE_APPLICATION_CREDENTIALS`, `AZURE_` |
| Database connection strings | Database config | Grep for `postgres://`, `mongodb://`, `DATABASE_URL`, `DB_HOST` |
| API client constructors | External API dependencies | Grep for `new Client`, `apiKey`, `api_key`, `bearerToken` |
| Docker / Kubernetes config | Container config | Read `Dockerfile`, `docker-compose.yml`, `k8s/*.yaml` for `env:`, `envFrom:`, `secretKeyRef:` |
| CI/CD config | Build/deploy-time config | Read `.github/workflows/`, `.gitlab-ci.yml`, `Jenkinsfile` |

**Output of discovery:** A list of findings, each with:
- The key/variable name
- Where it was found (file:line)
- What it appears to control (based on context)
- Whether it looks sensitive (contains "key", "secret", "password", "token", "credential")

### Phase 2: Manifest Creation

Transform discovery findings into a structured `environment.json`.

For each finding, determine:

1. **type**: Is it a plain env var, a secret, a file path, or structured config?
   - Contains "KEY", "TOKEN", "SECRET", "PASSWORD", "CREDENTIAL" → `secret`
   - Used as a file path argument → `file`
   - Loaded as JSON/YAML/TOML → `config`
   - Everything else → `env`

2. **target**: Where should the value live at runtime?
   - If it's a secret in production → `vault:<vault-name>`
   - If it's a simple env var → `env`
   - If it's a file → the actual path from the code

3. **source**: How should the value be obtained?
   - User must provide it → `manual`
   - System can generate it (certificates, random tokens) → `generated`
   - External system has it → `external` (with `externalRef`)

4. **required**: Is the code path guarded with a default or a conditional?
   - If the code has `process.env.X ?? 'default'` or `if (process.env.X)` → `required: false`
   - If the code crashes without it → `required: true`

5. **agentScope**: Which agents reference this in their code? If only one agent's directory references it, scope it to that agent.

**Rules:**
- Write the `description` field as if explaining to someone who has never seen this codebase. Include: what the variable controls, which service/agent uses it, and where to obtain it (URL, dashboard, team, etc.).
- Never put real secret values in the manifest. Use `example` with clearly fake values.
- Set `sensitive: true` for anything that could be a credential, even if it's typed as `env`.

### Phase 3: Validation

Before deployment, verify the manifest is complete and correct.

**Validation checks:**

1. **Schema validation**: Every requirement has `key`, `description`, `type`, and `required`. If `type` is `file` or `config`, `target` must be set.

2. **Value presence**: For every `required: true` requirement, verify a value exists in:
   - The deployment environment (env vars already set)
   - A `.env` file
   - The vault
   - A user-provided overrides file (`environment.values.json`)

3. **Format validation**: If `validation.pattern` is set, test the value against the regex. If `validation.enum` is set, check the value is in the list. If `format` is `json`, try parsing it.

4. **Dependency order**: For every requirement with `dependsOn`, verify those dependencies appear earlier in the requirements array.

5. **Conflict detection**: No two requirements should have the same `key`. No two `file` requirements should target the same path unless they are in different workspaces.

6. **Secret safety**: Any requirement with `sensitive: true` must NOT appear in any committed file (`.env`, config files in git). If it does, report it as a **critical security issue**.

**Output:** A validation report with:
- Total requirements
- Satisfied (value present and valid)
- Missing (required but no value found)
- Invalid (value present but fails validation)
- Warnings (optional requirements without values, sensitive values in committed files)

### Phase 4: Preparation

Resolve all missing values.

**For `source: "manual"` requirements:**
- Present the `description`, `example`, and `format` to the user
- Accept the value (or let the user skip if `required: false`)
- If `validation.pattern` is set, re-prompt on mismatch

**For `source: "generated"` requirements:**
- Generate the value (random secret, self-signed cert, UUID, etc.)
- Show what was generated and where it will be stored

**For `source: "external"` requirements:**
- Use `externalRef` to fetch the value from the external system
- If the external system is unavailable, fall back to asking the user

**For `source: "inherited"` requirements:**
- Look up the parent workspace's manifest and copy the value

Store all resolved values in `environment.values.json` (this file must be in `.gitignore`).

### Phase 5: Deployment

Write the resolved values to their targets.

| Target type | Action |
|---|---|
| `env` | Set as an environment variable in the process or container |
| `envfile` | Append `KEY=VALUE` to the specified `.env` file (create if missing) |
| `vault:<name>` | Store the value in the named vault using the vault API |
| `<file-path>` | Write the value to the file at that path (create parent directories if needed) |

**Post-deployment verification:**
- Re-run Phase 3 (Validation) to confirm all values are now in place
- For `file` targets, verify the file exists and is readable
- For `vault` targets, verify the secret was stored by reading it back
- Print a summary: "X/X required configurations deployed. Y optional configurations set."

---

## Conflict Resolution

If two agents in the same workspace need different values for the same key:

1. Check if `agentScope` is set. If so, deploy different values to different agents.
2. If `agentScope` is not set and the values differ, report a conflict and ask the user to resolve it.
3. Never silently overwrite one agent's configuration with another's.

## Environment Variants

A workspace can have multiple environments. Store variant-specific values in:

```
environment.json              ← base manifest (shared across environments)
environment.values.dev.json   ← development overrides (in .gitignore)
environment.values.prod.json  ← production overrides (in .gitignore)
```

When deploying, load the base manifest, then overlay the values file for the target environment.

## Schema Reference

The full JSON Schema for `environment.json` is at:
`schemas/environment.schema.json`

Key fields at a glance:

| Field | Required | Description |
|---|---|---|
| `key` | Yes | Unique identifier (UPPER_SNAKE_CASE for env/secret) |
| `description` | Yes | Human-readable explanation (10+ chars) |
| `type` | Yes | `env`, `secret`, `file`, or `config` |
| `required` | Yes | Whether the agents can run without it |
| `target` | For file/config | Where to deploy the value |
| `defaultValue` | No | Fallback when no value is provided |
| `example` | No | Non-sensitive example for documentation |
| `format` | No | `string`, `number`, `boolean`, `json`, `yaml`, `toml`, `text`, `binary` |
| `sensitive` | No | If true, never log or print the value (auto-true for secrets) |
| `dependsOn` | No | Keys that must be configured first |
| `validation` | No | `pattern` (regex), `minLength`, `maxLength`, `enum` |
| `source` | No | `manual`, `generated`, `external`, `inherited` |
| `externalRef` | No | When source=external, where to find the value |
| `agentScope` | No | Which agents need this (omit = all agents) |
| `notes` | No | Free-form additional instructions |

## Examples

See `references/examples.md` for:
- A minimal single-agent workspace
- A multi-agent production workspace with vault secrets
- A workspace with generated certificates
- A workspace with external cloud secrets
