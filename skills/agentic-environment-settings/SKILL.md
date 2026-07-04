---
name: agentic-environment-settings
description: >
  Analyzes all agents, tools, MCP servers, and services in a workspace and generates
  the environment.json manifest that declares every piece of configuration the workspace
  needs to run. The skill IS the tool that produces the file — it scans code, inspects
  agents and their tools, reads Dockerfiles and deployment manifests, and writes the
  complete environment.json to the repository root. Use when setting up a new agent
  workspace, preparing a deployment, debugging missing configuration, generating or
  updating an environment.json manifest, or auditing what a workspace needs.
  Triggers on "environment setup", "configure agents", "missing variables", "secrets",
  "deployment config", "environment.json", "workspace setup", "agent configuration",
  "prepare environment", "what does this workspace need", or "scan workspace".
---

# Agentic Environment Settings

## Responsibility

**This skill is the active agent that produces `environment.json`.** It does not just
give instructions — it scans, analyzes, and writes the output file.

When invoked, the skill must:

1. **Scan every agent definition** (`.opencode/agents/*.md`, agent config files) to understand what each agent does, which tools it uses, and which external services it talks to.
2. **Scan every tool and MCP server** configuration (`.opencode/tool/`, `.opencode/mcp.json`, `opencode.json`) to find tool-level configuration needs (API keys, connection strings, permissions).
3. **Scan all source code** for `process.env.*`, `os.Getenv()`, config file reads, vault calls, cloud SDK initializations, database connections, and API client constructors.
4. **Scan deployment manifests** (Dockerfiles, docker-compose, Kubernetes YAML, CI/CD pipelines) for `env:`, `secretKeyRef:`, `envFrom:`, build args.
5. **Cross-reference all findings** to produce a single deduplicated `environment.json` at the repository root that covers every agent, tool, and service.

The output `environment.json` is the authoritative manifest. No other file should be needed to understand what configuration the workspace requires.

## Lifecycle

1. **Discover** — Scan the workspace to find every configuration dependency (agents, tools, code, deployments).
2. **Manifest** — Generate the `environment.json` file at the repository root declaring all requirements.
3. **Validate** — Check that every requirement has a value and that values match expected formats.
4. **Prepare** — Generate templates, prompt for missing values, resolve external secrets.
5. **Deploy** — Write the final configuration to the correct targets (env vars, .env files, vault entries, config files).

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

The goal is to find **every** configuration dependency in the workspace by scanning
agents, tools, source code, and deployment manifests.

#### 1a. Agent Analysis

Scan every agent definition to understand what configuration each agent needs:

| Source | What to look for |
|---|---|
| `.opencode/agents/*.md` | System prompts that reference external services, API keys, tool requirements. Agent frontmatter (`mode`, `tools`, `permission`). Mentions of databases, APIs, external integrations. |
| Agent config / tools section | Declared tool permissions that imply config needs (e.g. `github` tool → needs GitHub token, `database` tool → needs connection string). |
| Agent instructions referencing files | If an agent references reading/writing specific paths, those paths may need to exist and be writable. |
| Agent instructions referencing external services | If an agent mentions "call the API", "query the database", "authenticate with", extract the implied service and add a requirement. |

For each agent, produce a summary:
```
Agent: <name>
  File: <path>
  Mode: <primary|subagent|all>
  Tools: <list>
  External services referenced: <list>
  Env vars referenced: <list>
  Files referenced: <list>
```

#### 1b. Tool & MCP Server Analysis

Scan every tool and MCP server configuration:

| Source | What to look for |
|---|---|
| `.opencode/mcp.json` / `opencode.json` | MCP server definitions with `env`, `command`, `args`. Each env entry is a configuration requirement. |
| `.opencode/tool/` | Tool definitions that reference env vars or need credentials. |
| `package.json` / `go.mod` dependencies | SDKs that imply config needs (e.g. `@google-cloud/firestore` → needs service account, `pg` → needs connection string). |
| `.skills/` directories | Skills that reference env vars or external services in their instructions. |

#### 1c. Source Code Analysis

Grep all source files for configuration patterns:

| Pattern | What it means | How to find it |
|---|---|---|
| `process.env.XXX` (JS/TS) | Environment variable dependency | `grep -rn "process\.env\." --include="*.js" --include="*.ts"` |
| `os.Getenv("XXX")` / `os.LookupEnv` (Go) | Go env var | `grep -rn "os\.Getenv\|os\.LookupEnv"` |
| `os.environ.get` / `os.environ["XXX"]` (Python) | Python env var | `grep -rn "os\.environ"` |
| `.env` / `.env.local` / `.env.production` | Environment file usage | Look for dotenv imports and `.env*` files |
| Vault / secrets manager calls | Secret dependencies | Grep for `vault`, `getSecret`, `SecretManager`, `secrets.get` |
| Config file reads | File-based configuration | Grep for `readFile`, `fs.readFile`, `loadConfig`, `require('./config')`, `import.*config`, `os.ReadFile` |
| Cloud SDK initializations | Cloud provider credentials | Grep for `AWS_`, `GCP_`, `GOOGLE_APPLICATION_CREDENTIALS`, `AZURE_`, `firebase`, `admin.initializeApp` |
| Database connection strings | Database config | Grep for `postgres://`, `mongodb://`, `DATABASE_URL`, `DB_HOST`, `sql.Open` |
| API client constructors | External API dependencies | Grep for `new Client`, `apiKey`, `api_key`, `bearerToken`, `Authorization` |
| TLS / crypto material | Certificate needs | Grep for `tls.Config`, `cert`, `key.pem`, `ca.crt` |

#### 1d. Deployment Manifest Analysis

| Source | What to look for |
|---|---|
| `Dockerfile` | `ENV`, `ARG`, `COPY` of config/secrets, `VOLUME` mounts |
| `docker-compose.yml` | `environment:`, `env_file:`, `secrets:`, `volumes:` for config files |
| `k8s/*.yaml` | `env:`, `envFrom:`, `secretKeyRef:`, `configMapRef:`, `volumeMounts:` for config |
| `.github/workflows/` | `env:` and `secrets.` references in CI steps |

#### 1e. Skills Analysis

| Source | What to look for |
|---|---|
| `.skills/*/SKILL.md` | Skill instructions that reference env vars, external services, or require credentials |
| `skills/*/SKILL.md` | Same as above (different common location) |

**Output of discovery:** A deduplicated list of findings, each with:
- The key/variable name
- Where it was found (file:line, agent name, tool name)
- What it appears to control (based on context)
- Whether it looks sensitive (contains "key", "secret", "password", "token", "credential")
- Which agent(s) or service(s) use it

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
