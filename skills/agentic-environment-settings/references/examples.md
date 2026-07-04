# Environment Configuration Examples

This document shows real-world examples of `environment.json` manifests for different workspace scenarios.

---

## Example 1: Minimal Single-Agent Workspace

A single agent that reads from an API and writes results to a file.

```json
{
  "version": "1.0",
  "workspace": {
    "name": "Simple Scraper",
    "description": "Scrapes product data from a supplier API and writes JSON output.",
    "environment": "development"
  },
  "requirements": [
    {
      "key": "SUPPLIER_API_KEY",
      "description": "API key for the supplier's product catalog API. Request access from the procurement team or generate a key at https://supplier.example.com/settings/api",
      "type": "secret",
      "required": true,
      "target": "envfile",
      "format": "string",
      "sensitive": true,
      "source": "manual",
      "example": "sup_key_abc123def456",
      "validation": {
        "pattern": "^sup_key_[a-zA-Z0-9]+$"
      },
      "agentScope": ["scraper"]
    },
    {
      "key": "OUTPUT_DIR",
      "description": "Directory where scraped product JSON files are written. Defaults to ./output if not set.",
      "type": "env",
      "required": false,
      "target": "env",
      "defaultValue": "./output",
      "format": "string",
      "source": "manual",
      "example": "/data/scraped-products"
    }
  ]
}
```

---

## Example 2: Multi-Agent Production Workspace with Vault

A production workspace with a chat agent, a background runner, and shared database access.

```json
{
  "version": "1.0",
  "workspace": {
    "name": "MakeYourCrew Production",
    "description": "Production workspace for the MakeYourCrew agent platform. Xerraire handles chat, agent-runner executes long tasks, and both share a PostgreSQL database and GitHub integration.",
    "environment": "production"
  },
  "requirements": [
    {
      "key": "DEEPSEEK_API_KEY",
      "description": "API key for the DeepSeek LLM service. Used by the Xerraire chat agent to process user messages and route tasks to the agent-runner. Without this key, the chat agent cannot function. Generate at https://platform.deepseek.com/api-keys",
      "type": "secret",
      "required": true,
      "target": "vault:agent-secrets",
      "format": "string",
      "sensitive": true,
      "source": "manual",
      "example": "sk-xxxxxxxxxxxxxxxxxxxxxxxx",
      "validation": {
        "pattern": "^sk-[a-zA-Z0-9]+$",
        "minLength": 20
      },
      "agentScope": ["xerraire"]
    },
    {
      "key": "PG_CONNECTION_STRING",
      "description": "PostgreSQL connection string for the main application database. Format: postgres://USER:PASSWORD@HOST:PORT/DBNAME. Used by both agents for session persistence and conversation storage. The database must already exist and the user must have read/write permissions.",
      "type": "secret",
      "required": true,
      "target": "vault:agent-secrets",
      "format": "string",
      "sensitive": true,
      "source": "manual",
      "example": "postgres://studio:password@155.133.27.1:5432/studio_agents"
    },
    {
      "key": "GITHUB_APP_PRIVATE_KEY",
      "description": "PEM-encoded private key for the GitHub App used to clone agent repositories and manage data repos. Generate from the GitHub App settings page (https://github.com/organizations/makeyourcrew/settings/apps). Must be the full PEM file contents including BEGIN/END markers.",
      "type": "file",
      "required": true,
      "target": "/app/secrets/github-app.pem",
      "format": "text",
      "sensitive": true,
      "source": "manual",
      "dependsOn": ["GITHUB_APP_ID", "GITHUB_CLIENT_ID", "GITHUB_CLIENT_SECRET"],
      "agentScope": ["agent-runner"]
    },
    {
      "key": "GITHUB_APP_ID",
      "description": "Numeric App ID for the GitHub App. Found at the top of the GitHub App settings page.",
      "type": "env",
      "required": true,
      "target": "vault:agent-secrets",
      "format": "string",
      "sensitive": true,
      "source": "manual",
      "example": "1234567"
    },
    {
      "key": "FIREBASE_SERVICE_ACCOUNT",
      "description": "Firebase service account JSON for authenticating API requests. Download from Firebase Console > Project Settings > Service Accounts > Generate New Private Key. The entire JSON object is stored as a single secret.",
      "type": "config",
      "required": true,
      "target": "/app/config/firebase-service-account.json",
      "format": "json",
      "sensitive": true,
      "source": "manual",
      "example": "{\"type\":\"service_account\",\"project_id\":\"makeyourcrew\",...}"
    },
    {
      "key": "NODE_ENV",
      "description": "Node.js environment flag. Set to 'production' to enable production optimizations and disable verbose logging.",
      "type": "env",
      "required": true,
      "target": "env",
      "defaultValue": "production",
      "format": "string",
      "source": "manual",
      "validation": {
        "enum": ["development", "staging", "production"]
      }
    }
  ]
}
```

---

## Example 3: Workspace with Generated Secrets

A workspace that needs a self-signed TLS certificate and a random session secret.

```json
{
  "version": "1.0",
  "workspace": {
    "name": "Secure API Gateway",
    "description": "API gateway that terminates TLS and manages session tokens for downstream agents.",
    "environment": "production"
  },
  "requirements": [
    {
      "key": "SESSION_SECRET",
      "description": "Random secret used to sign session JWTs. Must be at least 32 characters. The system will generate a cryptographically secure random string if no value is provided.",
      "type": "secret",
      "required": true,
      "target": "vault:agent-secrets",
      "format": "string",
      "sensitive": true,
      "source": "generated",
      "validation": {
        "minLength": 32
      }
    },
    {
      "key": "TLS_CERT",
      "description": "Self-signed TLS certificate for HTTPS. The system generates a 2048-bit RSA certificate valid for 365 days if no value is provided. For production with a real domain, replace this with a certificate from Let's Encrypt or your CA.",
      "type": "file",
      "required": true,
      "target": "/app/secrets/tls.crt",
      "format": "text",
      "sensitive": true,
      "source": "generated",
      "dependsOn": ["TLS_KEY"]
    },
    {
      "key": "TLS_KEY",
      "description": "Private key matching the TLS certificate. Generated alongside the certificate.",
      "type": "file",
      "required": true,
      "target": "/app/secrets/tls.key",
      "format": "text",
      "sensitive": true,
      "source": "generated"
    },
    {
      "key": "ALLOWED_ORIGINS",
      "description": "Comma-separated list of allowed CORS origins. Include the scheme and port, e.g. 'https://app.example.com,https://admin.example.com'.",
      "type": "env",
      "required": true,
      "target": "env",
      "format": "string",
      "source": "manual",
      "example": "https://app.example.com,https://admin.example.com"
    }
  ]
}
```

---

## Example 4: Workspace with External Cloud Secrets

A workspace that pulls secrets from AWS Secrets Manager and Google Cloud Secret Manager.

```json
{
  "version": "1.0",
  "workspace": {
    "name": "Multi-Cloud Agent",
    "description": "Agent that accesses both AWS and GCP resources. Secrets are managed centrally in cloud secret managers rather than stored locally.",
    "environment": "production"
  },
  "requirements": [
    {
      "key": "STRIPE_SECRET_KEY",
      "description": "Stripe API secret key for processing payments. Stored in AWS Secrets Manager under prod/stripe-secret-key. The agent fetches it at startup — never hardcode this value.",
      "type": "secret",
      "required": true,
      "target": "vault:agent-secrets",
      "format": "string",
      "sensitive": true,
      "source": "external",
      "externalRef": "aws-secrets-manager:prod/stripe-secret-key",
      "example": "sk_live_xxxxxxxx"
    },
    {
      "key": "GCP_PROJECT_ID",
      "description": "Google Cloud project ID where the agent's resources live. Used to construct API endpoints and storage paths.",
      "type": "env",
      "required": true,
      "target": "env",
      "format": "string",
      "source": "external",
      "externalRef": "gcp-secret:projects/myproject/secrets/project-id",
      "example": "my-production-project"
    },
    {
      "key": "AWS_REGION",
      "description": "AWS region for all AWS SDK calls. Must match the region where the Secrets Manager secrets are stored.",
      "type": "env",
      "required": true,
      "target": "env",
      "defaultValue": "eu-west-1",
      "format": "string",
      "source": "manual",
      "validation": {
        "enum": ["eu-west-1", "eu-central-1", "us-east-1", "us-west-2"]
      }
    }
  ]
}
```

---

## Example 5: Environment Variants

Base manifest with overrides for different environments.

### `environment.json` (base, committed)

```json
{
  "version": "1.0",
  "workspace": {
    "name": "Data Pipeline",
    "description": "ETL pipeline that syncs data between sources.",
    "environment": "production"
  },
  "requirements": [
    {
      "key": "DATABASE_URL",
      "description": "Primary database connection string. Different per environment: dev uses localhost, prod uses the managed instance.",
      "type": "secret",
      "required": true,
      "target": "env",
      "format": "string",
      "sensitive": true,
      "source": "manual"
    },
    {
      "key": "LOG_LEVEL",
      "description": "Logging verbosity. 'debug' for development, 'info' or 'warn' for production.",
      "type": "env",
      "required": false,
      "target": "env",
      "defaultValue": "info",
      "format": "string",
      "validation": {
        "enum": ["debug", "info", "warn", "error"]
      }
    },
    {
      "key": "BATCH_SIZE",
      "description": "Number of records to process per batch. Lower in development to avoid long runs.",
      "type": "env",
      "required": false,
      "target": "env",
      "defaultValue": 1000,
      "format": "number"
    }
  ]
}
```

### `environment.values.dev.json` (not committed)

```json
{
  "DATABASE_URL": "postgres://dev:devpass@localhost:5432/devdb",
  "LOG_LEVEL": "debug",
  "BATCH_SIZE": 10
}
```

### `environment.values.prod.json` (not committed)

```json
{
  "DATABASE_URL": "postgres://pipeline:strongpass@db.internal:5432/pipeline_prod",
  "LOG_LEVEL": "warn",
  "BATCH_SIZE": 5000
}
```

---

## Validation Report Example

After running validation against a manifest, the output looks like:

```
Environment Validation Report
═══════════════════════════════════════════

Workspace: MakeYourCrew Production (production)
Manifest:  environment.json (6 requirements)

Summary:
  ✓ Satisfied:  4   ████████████░░░░░░  67%
  ✗ Missing:    1   ████░░░░░░░░░░░░░░  17%
  ! Invalid:    1   ████░░░░░░░░░░░░░░  17%
  ○ Optional:   0

Details:

  ✓ DEEPSEEK_API_KEY
     Type: secret | Target: vault:agent-secrets
     Value found in vault, matches pattern ^sk-[a-zA-Z0-9]+$

  ✓ PG_CONNECTION_STRING
     Type: secret | Target: vault:agent-secrets
     Value found in vault

  ✗ GITHUB_APP_PRIVATE_KEY
     Type: file | Target: /app/secrets/github-app.pem
     MISSING: File does not exist at target path.
     Action: User must provide the PEM key contents.

  ! FIREBASE_SERVICE_ACCOUNT
     Type: config | Target: /app/config/firebase-service-account.json
     INVALID: File exists but is not valid JSON.
     Error: Unexpected token } at position 245
     Action: Fix the JSON syntax or re-download from Firebase Console.

  ✓ NODE_ENV
     Type: env | Target: env
     Value: "production" (matches enum)

Blocked: 1 missing, 1 invalid. Resolve before deployment.
```
