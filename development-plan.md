# Infrastructure as Code Generator — Phased Development Plan

> Project: 177-infrastructure-as-code-generator · Created: 2026-05-25
> Purpose: Provide sufficient detail for Claude Code (Opus) to implement each phase end-to-end.

---

## Technology Decisions

| Concern | Choice | Rationale |
|---------|--------|-----------|
| Primary language | TypeScript (Node.js 22+) | The IaC domain requires strong JSON/JSONB handling, HCL template manipulation, and async I/O for provider API calls. TypeScript provides type safety for the complex provider schema introspection pipeline, first-class JSON support, and the largest ecosystem of cloud SDK libraries (AWS SDK v3, Azure SDK, GCP client libraries). Pulumi itself uses TypeScript as a primary language, validating domain fit. |
| API framework | Fastify 5 | High-performance HTTP framework with built-in OpenAPI 3.1 schema generation via `@fastify/swagger`, streaming response support for long-running generation operations, and TypeBox schema validation. Preferred over Express for its 2-3x throughput advantage and native TypeScript support. |
| Database | PostgreSQL 16 + Drizzle ORM | PostgreSQL provides JSONB columns with GIN indexes for provider schema caching and variable resource attributes (hybrid model from data-model-suggestion-3), LISTEN/NOTIFY for real-time event propagation, and partitioned tables for audit logs. Drizzle ORM offers type-safe SQL with zero-overhead abstraction and push-based migrations. |
| Task queue | BullMQ (Redis 7) | IaC generation, security scanning, cost estimation, and provider schema syncing are all async workloads requiring job queues with retries, rate limiting, and priority ordering. BullMQ is the standard Node.js queue backed by Redis with dashboard support via Bull Board. |
| LLM integration | Anthropic SDK (`@anthropic-ai/sdk`) + LiteLLM proxy | Claude as the primary generation model; LiteLLM proxy enables future provider-swapping (OpenAI, Bedrock) without code changes. Prompt caching reduces cost for iterative refinement sessions. |
| CLI framework | Commander.js + Ink (React for CLI) | Commander.js for argument parsing; Ink for rich terminal UI showing plan diffs, cost estimates, and security scan results inline. Matches the CLI-first deployment model from the README. |
| Security scanner | Checkov (Python, called via subprocess) | Open-source, 1,000+ built-in policies mapped to CIS/SOC 2/PCI-DSS/HIPAA, SARIF output for CI integration. Called as a subprocess rather than reimplemented — Checkov is the industry standard with Apache 2.0 licence. |
| Cost estimation | Infracost (called via subprocess) | Open-source Terraform cost estimation tool, outputs JSON breakdown per resource. Called via CLI to avoid reimplementing cloud pricing logic. |
| HCL processing | `hcl2-parser` + custom HCL emitter | Parse existing HCL for context; custom emitter generates idiomatic, formatted HCL output from the generation pipeline's AST representation. |
| MCP server | `@modelcontextprotocol/sdk` | Official MCP SDK for TypeScript. Exposes generation, plan preview, and deployment tools via the MCP v2025-11-25 specification for IDE integration. |
| Testing | Vitest + Supertest + Testcontainers | Vitest for unit/integration tests (fast, native ESM, TypeScript-first). Supertest for HTTP endpoint testing. Testcontainers for PostgreSQL and Redis integration tests with real containers. |
| Code quality | ESLint 9 (flat config) + Prettier + `tsc --noEmit` | Standard TypeScript toolchain. ESLint with `@typescript-eslint` rules. Prettier for formatting. TypeScript strict mode for type checking. |
| Containerisation | Docker (multi-stage build) + Docker Compose | Multi-stage build for minimal production image. Docker Compose for local development with PostgreSQL, Redis, and the application. |
| Package manager | pnpm 9 | Strict dependency resolution, efficient disk usage via content-addressable storage, workspace support for monorepo structure. |
| Monorepo structure | pnpm workspaces | Separate packages for CLI, API server, MCP server, core generation engine, and shared types. Enables independent versioning and testing. |

### Project Structure

```
iac-generator/
├── pnpm-workspace.yaml
├── package.json
├── tsconfig.base.json
├── Dockerfile
├── docker-compose.yml
├── .env.example
├── drizzle.config.ts
├── vitest.config.ts
├── packages/
│   ├── core/                          # Shared generation engine
│   │   ├── package.json
│   │   ├── tsconfig.json
│   │   └── src/
│   │       ├── index.ts
│   │       ├── generator/
│   │       │   ├── engine.ts           # Main generation orchestrator
│   │       │   ├── prompt-builder.ts   # LLM prompt construction
│   │       │   ├── hcl-emitter.ts      # HCL code generation from AST
│   │       │   ├── pulumi-emitter.ts   # Pulumi TypeScript/Python output
│   │       │   └── context-loader.ts   # Cloud state + provider schema loader
│   │       ├── providers/
│   │       │   ├── schema-cache.ts     # Provider schema introspection & caching
│   │       │   ├── aws.ts
│   │       │   ├── azure.ts
│   │       │   ├── gcp.ts
│   │       │   └── kubernetes.ts
│   │       ├── security/
│   │       │   ├── scanner.ts          # Checkov integration
│   │       │   ├── policy-engine.ts    # OPA/Rego policy evaluation
│   │       │   ├── cis-defaults.ts     # CIS benchmark default injector
│   │       │   └── types.ts
│   │       ├── cost/
│   │       │   ├── estimator.ts        # Infracost integration
│   │       │   └── types.ts
│   │       ├── modules/
│   │       │   ├── registry-client.ts  # OpenTofu registry API client
│   │       │   └── recommender.ts      # Module recommendation engine
│   │       └── types/
│   │           ├── generation.ts       # Core domain types
│   │           ├── provider.ts         # Provider schema types
│   │           ├── artifact.ts         # Generated artifact types
│   │           └── config.ts           # Configuration types
│   ├── db/                             # Database schema & migrations
│   │   ├── package.json
│   │   ├── tsconfig.json
│   │   └── src/
│   │       ├── index.ts
│   │       ├── schema.ts              # Drizzle schema definitions
│   │       ├── migrate.ts             # Migration runner
│   │       └── migrations/
│   ├── api/                            # Fastify REST API server
│   │   ├── package.json
│   │   ├── tsconfig.json
│   │   └── src/
│   │       ├── index.ts               # Server entry point
│   │       ├── app.ts                 # Fastify app factory
│   │       ├── routes/
│   │       │   ├── sessions.ts        # Generation session endpoints
│   │       │   ├── artifacts.ts       # Artifact retrieval endpoints
│   │       │   ├── projects.ts        # Project management endpoints
│   │       │   ├── deployments.ts     # Plan/apply endpoints
│   │       │   ├── security.ts        # Security policy endpoints
│   │       │   └── health.ts
│   │       ├── middleware/
│   │       │   ├── auth.ts            # JWT/OAuth authentication
│   │       │   ├── tenant.ts          # Multi-tenant org isolation
│   │       │   └── rate-limit.ts
│   │       └── workers/
│   │           ├── generation.worker.ts
│   │           ├── security-scan.worker.ts
│   │           └── cost-estimate.worker.ts
│   ├── cli/                            # CLI application
│   │   ├── package.json
│   │   ├── tsconfig.json
│   │   └── src/
│   │       ├── index.ts               # CLI entry point
│   │       ├── commands/
│   │       │   ├── generate.ts        # Natural language generation
│   │       │   ├── plan.ts            # Plan preview
│   │       │   ├── apply.ts           # Apply deployment
│   │       │   ├── scan.ts            # Security scan
│   │       │   ├── cost.ts            # Cost estimation
│   │       │   ├── init.ts            # Project initialisation
│   │       │   └── config.ts          # Configuration management
│   │       └── ui/
│   │           ├── plan-diff.tsx       # Ink component for plan diff
│   │           ├── cost-table.tsx      # Ink component for cost display
│   │           └── scan-results.tsx    # Ink component for scan results
│   └── mcp/                            # MCP server
│       ├── package.json
│       ├── tsconfig.json
│       └── src/
│           ├── index.ts               # MCP server entry point
│           ├── tools/
│           │   ├── generate.ts        # generate_iac tool
│           │   ├── plan.ts            # preview_plan tool
│           │   ├── scan.ts            # security_scan tool
│           │   └── cost.ts            # estimate_cost tool
│           └── resources/
│               ├── sessions.ts        # Session history resource
│               └── artifacts.ts       # Artifact content resource
├── tests/
│   ├── fixtures/
│   │   ├── prompts/                   # Sample NL prompts
│   │   ├── hcl/                       # Expected HCL output files
│   │   ├── provider-schemas/          # Cached provider schema snapshots
│   │   └── scan-results/             # Sample Checkov SARIF output
│   ├── unit/
│   ├── integration/
│   └── e2e/
└── docs/
    ├── api.md
    └── mcp-tools.md
```

---

## Phase 1: Foundation & Project Scaffolding

### Purpose
Establish the monorepo structure, database schema, configuration system, and development toolchain. After this phase, a developer can clone the repo, run `pnpm install && docker compose up`, and have a working (empty) API server connected to PostgreSQL and Redis with full type checking and test infrastructure.

### Tasks

#### 1.1 — Monorepo Initialisation

**What**: Create the pnpm workspace with all five packages (core, db, api, cli, mcp) and shared TypeScript configuration.

**Design**:

Root `pnpm-workspace.yaml`:
```yaml
packages:
  - 'packages/*'
```

Root `tsconfig.base.json`:
```json
{
  "compilerOptions": {
    "target": "ES2024",
    "module": "NodeNext",
    "moduleResolution": "NodeNext",
    "strict": true,
    "esModuleInterop": true,
    "skipLibCheck": true,
    "forceConsistentCasingInFileNames": true,
    "resolveJsonModule": true,
    "declaration": true,
    "declarationMap": true,
    "sourceMap": true,
    "outDir": "./dist",
    "rootDir": "./src"
  }
}
```

Each package extends the base config:
```json
{
  "extends": "../../tsconfig.base.json",
  "compilerOptions": {
    "outDir": "./dist",
    "rootDir": "./src"
  },
  "include": ["src/**/*"]
}
```

Root `package.json` scripts:
```json
{
  "scripts": {
    "build": "pnpm -r build",
    "test": "vitest run",
    "test:watch": "vitest",
    "lint": "eslint packages/",
    "format": "prettier --write 'packages/**/*.ts'",
    "typecheck": "tsc -b",
    "dev": "docker compose up -d && pnpm -r --parallel dev",
    "db:migrate": "pnpm --filter @iac-gen/db migrate",
    "db:push": "pnpm --filter @iac-gen/db push"
  }
}
```

**Testing**:
- `Unit: pnpm install completes without errors`
- `Unit: tsc --noEmit succeeds across all packages`
- `Unit: eslint runs without configuration errors`
- `Unit: vitest discovers test files in all packages`

#### 1.2 — Database Schema & Migrations

**What**: Implement the hybrid relational + JSONB database schema (based on data-model-suggestion-3) using Drizzle ORM with initial migration.

**Design**:

Drizzle schema definition (`packages/db/src/schema.ts`):

```typescript
import { pgTable, uuid, text, timestamp, jsonb, integer, boolean, numeric, uniqueIndex, index, bytea } from 'drizzle-orm/pg-core';

// --- Identity & Multi-Tenancy ---

export const organisations = pgTable('organisations', {
  id: uuid('id').primaryKey().defaultRandom(),
  name: text('name').notNull(),
  slug: text('slug').notNull().unique(),
  planTier: text('plan_tier').notNull().default('free'),
  settings: jsonb('settings').notNull().default({}),
  createdAt: timestamp('created_at', { withTimezone: true }).notNull().defaultNow(),
  updatedAt: timestamp('updated_at', { withTimezone: true }).notNull().defaultNow(),
});

export const users = pgTable('users', {
  id: uuid('id').primaryKey().defaultRandom(),
  email: text('email').notNull().unique(),
  displayName: text('display_name').notNull(),
  authProvider: text('auth_provider').notNull(),
  authSubject: text('auth_subject').notNull(),
  preferences: jsonb('preferences').notNull().default({}),
  lastLoginAt: timestamp('last_login_at', { withTimezone: true }),
  createdAt: timestamp('created_at', { withTimezone: true }).notNull().defaultNow(),
  updatedAt: timestamp('updated_at', { withTimezone: true }).notNull().defaultNow(),
}, (table) => [
  uniqueIndex('uq_users_auth').on(table.authProvider, table.authSubject),
]);

export const orgMemberships = pgTable('org_memberships', {
  id: uuid('id').primaryKey().defaultRandom(),
  orgId: uuid('org_id').notNull().references(() => organisations.id, { onDelete: 'cascade' }),
  userId: uuid('user_id').notNull().references(() => users.id, { onDelete: 'cascade' }),
  role: text('role').notNull().default('member'),
  permissions: jsonb('permissions').notNull().default({}),
  createdAt: timestamp('created_at', { withTimezone: true }).notNull().defaultNow(),
}, (table) => [
  uniqueIndex('uq_org_memberships').on(table.orgId, table.userId),
  index('idx_org_memberships_user').on(table.userId),
]);

// --- Projects ---

export const projects = pgTable('projects', {
  id: uuid('id').primaryKey().defaultRandom(),
  orgId: uuid('org_id').notNull().references(() => organisations.id, { onDelete: 'cascade' }),
  name: text('name').notNull(),
  slug: text('slug').notNull(),
  description: text('description'),
  config: jsonb('config').notNull().default({}),
  createdAt: timestamp('created_at', { withTimezone: true }).notNull().defaultNow(),
  updatedAt: timestamp('updated_at', { withTimezone: true }).notNull().defaultNow(),
}, (table) => [
  uniqueIndex('uq_projects_org_slug').on(table.orgId, table.slug),
  index('idx_projects_org').on(table.orgId),
]);

// --- Generation Sessions ---

export const generationSessions = pgTable('generation_sessions', {
  id: uuid('id').primaryKey().defaultRandom(),
  projectId: uuid('project_id').notNull().references(() => projects.id, { onDelete: 'cascade' }),
  orgId: uuid('org_id').notNull().references(() => organisations.id),
  userId: uuid('user_id').notNull().references(() => users.id),
  title: text('title'),
  status: text('status').notNull().default('active'),
  source: text('source').notNull().default('cli'),
  context: jsonb('context').notNull().default({}),
  usage: jsonb('usage').notNull().default({}),
  createdAt: timestamp('created_at', { withTimezone: true }).notNull().defaultNow(),
  updatedAt: timestamp('updated_at', { withTimezone: true }).notNull().defaultNow(),
}, (table) => [
  index('idx_sessions_project').on(table.projectId),
  index('idx_sessions_user').on(table.userId),
  index('idx_sessions_org').on(table.orgId, table.createdAt),
]);

export const messages = pgTable('messages', {
  id: uuid('id').primaryKey().defaultRandom(),
  sessionId: uuid('session_id').notNull().references(() => generationSessions.id, { onDelete: 'cascade' }),
  sequenceNum: integer('sequence_num').notNull(),
  role: text('role').notNull(),
  content: text('content').notNull(),
  metadata: jsonb('metadata').notNull().default({}),
  createdAt: timestamp('created_at', { withTimezone: true }).notNull().defaultNow(),
}, (table) => [
  uniqueIndex('uq_messages_session_seq').on(table.sessionId, table.sequenceNum),
  index('idx_messages_session').on(table.sessionId, table.sequenceNum),
]);

// --- Generated Artifacts ---

export const artifacts = pgTable('artifacts', {
  id: uuid('id').primaryKey().defaultRandom(),
  sessionId: uuid('session_id').notNull().references(() => generationSessions.id, { onDelete: 'cascade' }),
  orgId: uuid('org_id').notNull().references(() => organisations.id),
  messageId: uuid('message_id').notNull().references(() => messages.id),
  version: integer('version').notNull().default(1),
  outputFormat: text('output_format').notNull(),
  content: text('content').notNull(),
  contentHash: text('content_hash').notNull(),
  filePath: text('file_path'),
  resources: jsonb('resources').notNull().default([]),
  dependencies: jsonb('dependencies').notNull().default([]),
  securityScan: jsonb('security_scan'),
  costEstimate: jsonb('cost_estimate'),
  createdAt: timestamp('created_at', { withTimezone: true }).notNull().defaultNow(),
}, (table) => [
  index('idx_artifacts_session').on(table.sessionId),
  index('idx_artifacts_org').on(table.orgId, table.createdAt),
  index('idx_artifacts_hash').on(table.contentHash),
]);

// --- Provider Schema Cache ---

export const providerSchemas = pgTable('provider_schemas', {
  id: uuid('id').primaryKey().defaultRandom(),
  providerName: text('provider_name').notNull(),
  providerVersion: text('provider_version').notNull(),
  displayName: text('display_name').notNull(),
  registryUrl: text('registry_url'),
  schema: jsonb('schema').notNull(),
  regions: jsonb('regions').notNull().default([]),
  syncedAt: timestamp('synced_at', { withTimezone: true }).notNull().defaultNow(),
  createdAt: timestamp('created_at', { withTimezone: true }).notNull().defaultNow(),
}, (table) => [
  uniqueIndex('uq_provider_schemas').on(table.providerName, table.providerVersion),
]);

// --- Security Policies ---

export const securityPolicies = pgTable('security_policies', {
  id: uuid('id').primaryKey().defaultRandom(),
  orgId: uuid('org_id').notNull().references(() => organisations.id, { onDelete: 'cascade' }),
  name: text('name').notNull(),
  description: text('description'),
  policyLanguage: text('policy_language').notNull().default('rego'),
  policyContent: text('policy_content').notNull(),
  config: jsonb('config').notNull().default({}),
  createdAt: timestamp('created_at', { withTimezone: true }).notNull().defaultNow(),
  updatedAt: timestamp('updated_at', { withTimezone: true }).notNull().defaultNow(),
}, (table) => [
  uniqueIndex('uq_security_policies_org_name').on(table.orgId, table.name),
  index('idx_security_policies_org').on(table.orgId),
]);

// --- Deployments ---

export const deployments = pgTable('deployments', {
  id: uuid('id').primaryKey().defaultRandom(),
  artifactId: uuid('artifact_id').notNull().references(() => artifacts.id),
  orgId: uuid('org_id').notNull().references(() => organisations.id),
  userId: uuid('user_id').notNull().references(() => users.id),
  projectId: uuid('project_id').notNull().references(() => projects.id),
  workspace: text('workspace').notNull(),
  status: text('status').notNull().default('pending'),
  planOutput: jsonb('plan_output'),
  summary: jsonb('summary').notNull().default({}),
  approval: jsonb('approval'),
  timing: jsonb('timing').notNull().default({}),
  errorMessage: text('error_message'),
  createdAt: timestamp('created_at', { withTimezone: true }).notNull().defaultNow(),
  updatedAt: timestamp('updated_at', { withTimezone: true }).notNull().defaultNow(),
}, (table) => [
  index('idx_deployments_artifact').on(table.artifactId),
  index('idx_deployments_org').on(table.orgId, table.createdAt),
  index('idx_deployments_project').on(table.projectId),
  index('idx_deployments_status').on(table.status),
]);

// --- Provider Credentials ---

export const providerCredentials = pgTable('provider_credentials', {
  id: uuid('id').primaryKey().defaultRandom(),
  orgId: uuid('org_id').notNull().references(() => organisations.id, { onDelete: 'cascade' }),
  name: text('name').notNull(),
  providerName: text('provider_name').notNull(),
  credentialType: text('credential_type').notNull(),
  encryptedConfig: bytea('encrypted_config').notNull(),
  configMetadata: jsonb('config_metadata').notNull().default({}),
  createdAt: timestamp('created_at', { withTimezone: true }).notNull().defaultNow(),
  updatedAt: timestamp('updated_at', { withTimezone: true }).notNull().defaultNow(),
}, (table) => [
  uniqueIndex('uq_provider_creds_org_name').on(table.orgId, table.name),
  index('idx_provider_creds_org').on(table.orgId),
]);

// --- Module Registry Cache ---

export const moduleCache = pgTable('module_cache', {
  id: uuid('id').primaryKey().defaultRandom(),
  source: text('source').notNull(),
  name: text('name').notNull(),
  providerName: text('provider_name').notNull(),
  version: text('version').notNull(),
  metadata: jsonb('metadata').notNull().default({}),
  syncedAt: timestamp('synced_at', { withTimezone: true }).notNull().defaultNow(),
}, (table) => [
  uniqueIndex('uq_module_cache_source_version').on(table.source, table.version),
  index('idx_module_cache_provider').on(table.providerName),
  index('idx_module_cache_name').on(table.name),
]);

// --- Audit Log ---

export const auditLog = pgTable('audit_log', {
  id: uuid('id').primaryKey().defaultRandom(),
  orgId: uuid('org_id').notNull(),
  userId: uuid('user_id'),
  action: text('action').notNull(),
  entityType: text('entity_type').notNull(),
  entityId: uuid('entity_id').notNull(),
  details: jsonb('details').notNull().default({}),
  createdAt: timestamp('created_at', { withTimezone: true }).notNull().defaultNow(),
}, (table) => [
  index('idx_audit_log_org').on(table.orgId, table.createdAt),
  index('idx_audit_log_entity').on(table.entityType, table.entityId),
  index('idx_audit_log_action').on(table.action, table.createdAt),
]);
```

**Testing**:
- `Unit: schema.ts compiles without type errors`
- `Integration (Testcontainers): drizzle-kit push applies schema to a clean PostgreSQL container without errors`
- `Integration (Testcontainers): insert a user, org, membership, project, session, message, artifact — verify cascade deletes remove children`
- `Integration (Testcontainers): insert artifact with JSONB resources array — query with jsonb containment operator returns correct results`
- `Unit: all column defaults produce valid values (UUIDs, timestamps, empty JSONB)`

#### 1.3 — Configuration System

**What**: Define the application configuration schema with environment variable and file-based loading.

**Design**:

Configuration type (`packages/core/src/types/config.ts`):

```typescript
import { z } from 'zod';

export const AppConfigSchema = z.object({
  // Server
  port: z.number().default(3000),
  host: z.string().default('0.0.0.0'),
  logLevel: z.enum(['debug', 'info', 'warn', 'error']).default('info'),

  // Database
  database: z.object({
    url: z.string(),
    maxConnections: z.number().default(20),
    ssl: z.boolean().default(false),
  }),

  // Redis
  redis: z.object({
    url: z.string().default('redis://localhost:6379'),
    maxRetries: z.number().default(3),
  }),

  // LLM
  llm: z.object({
    provider: z.enum(['anthropic', 'openai', 'litellm']).default('anthropic'),
    apiKey: z.string(),
    model: z.string().default('claude-sonnet-4-20250514'),
    maxTokens: z.number().default(8192),
    temperature: z.number().default(0.2),
    baseUrl: z.string().optional(),
  }),

  // Security
  security: z.object({
    checkovEnabled: z.boolean().default(true),
    checkovPath: z.string().default('checkov'),
    defaultScanFrameworks: z.array(z.string()).default(['cis_aws', 'cis_azure', 'cis_gcp']),
    autoScanOnGenerate: z.boolean().default(true),
  }),

  // Cost Estimation
  cost: z.object({
    infracostEnabled: z.boolean().default(true),
    infracostPath: z.string().default('infracost'),
    infracostApiKey: z.string().optional(),
    autoEstimateOnGenerate: z.boolean().default(true),
  }),

  // Auth
  auth: z.object({
    jwtSecret: z.string(),
    jwtExpiresIn: z.string().default('24h'),
    oauth: z.object({
      githubClientId: z.string().optional(),
      githubClientSecret: z.string().optional(),
    }).default({}),
  }),

  // Provider Defaults
  providers: z.object({
    defaultProvider: z.enum(['aws', 'azurerm', 'google', 'kubernetes']).default('aws'),
    defaultRegion: z.string().default('us-east-1'),
    schemaRefreshIntervalHours: z.number().default(24),
  }),

  // MCP Server
  mcp: z.object({
    enabled: z.boolean().default(false),
    port: z.number().default(3001),
    transport: z.enum(['stdio', 'streamable_http']).default('streamable_http'),
  }),
});

export type AppConfig = z.infer<typeof AppConfigSchema>;
```

Config loader (`packages/core/src/config.ts`):
```typescript
export function loadConfig(): AppConfig {
  // 1. Load from .env file (dotenv)
  // 2. Override with environment variables (IAC_GEN_ prefix)
  // 3. Validate with Zod schema
  // 4. Return frozen config object
  // Environment variable mapping: IAC_GEN_DATABASE_URL -> database.url
}
```

**Testing**:
- `Unit: valid complete config object → parsed without errors`
- `Unit: config with missing required field (database.url) → ZodError with path`
- `Unit: config with invalid enum value (logLevel: 'verbose') → ZodError`
- `Unit: environment variable IAC_GEN_PORT=8080 overrides default port`
- `Unit: .env file values are loaded when file exists`
- `Unit: environment variables take precedence over .env file`

#### 1.4 — Docker Compose Development Environment

**What**: Docker Compose configuration for local development with PostgreSQL 16 and Redis 7.

**Design**:

```yaml
# docker-compose.yml
services:
  postgres:
    image: postgres:16-alpine
    environment:
      POSTGRES_DB: iac_generator
      POSTGRES_USER: iac_gen
      POSTGRES_PASSWORD: dev_password
    ports:
      - "5432:5432"
    volumes:
      - pgdata:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U iac_gen"]
      interval: 5s
      timeout: 5s
      retries: 5

  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 5s
      timeout: 5s
      retries: 5

volumes:
  pgdata:
```

Production Dockerfile (multi-stage):
```dockerfile
FROM node:22-alpine AS builder
RUN corepack enable pnpm
WORKDIR /app
COPY pnpm-lock.yaml pnpm-workspace.yaml package.json ./
COPY packages/ ./packages/
RUN pnpm install --frozen-lockfile
RUN pnpm build

FROM node:22-alpine AS runner
RUN corepack enable pnpm
WORKDIR /app
COPY --from=builder /app/dist ./dist
COPY --from=builder /app/node_modules ./node_modules
COPY --from=builder /app/package.json ./
EXPOSE 3000 3001
CMD ["node", "dist/packages/api/src/index.js"]
```

**Testing**:
- `Integration: docker compose up starts PostgreSQL and Redis without errors`
- `Integration: application connects to PostgreSQL via DATABASE_URL`
- `Integration: application connects to Redis via REDIS_URL`
- `Unit: Dockerfile builds successfully with multi-stage build`
- `Integration: health check endpoints return healthy status`

#### 1.5 — Fastify API Server Skeleton

**What**: Empty Fastify application with health check, OpenAPI spec generation, error handling, and request logging.

**Design**:

App factory (`packages/api/src/app.ts`):
```typescript
import Fastify, { FastifyInstance } from 'fastify';
import swagger from '@fastify/swagger';
import swaggerUi from '@fastify/swagger-ui';
import cors from '@fastify/cors';
import { AppConfig } from '@iac-gen/core';

export async function buildApp(config: AppConfig): Promise<FastifyInstance> {
  const app = Fastify({
    logger: {
      level: config.logLevel,
      transport: { target: 'pino-pretty' },
    },
  });

  await app.register(cors, { origin: true });
  await app.register(swagger, {
    openapi: {
      info: {
        title: 'IaC Generator API',
        version: '0.1.0',
        description: 'AI-powered Infrastructure as Code generator',
      },
      servers: [{ url: `http://${config.host}:${config.port}` }],
    },
  });
  await app.register(swaggerUi, { routePrefix: '/docs' });

  // Health check
  app.get('/health', {
    schema: {
      response: {
        200: {
          type: 'object',
          properties: {
            status: { type: 'string', enum: ['ok'] },
            version: { type: 'string' },
            uptime: { type: 'number' },
          },
        },
      },
    },
  }, async () => ({
    status: 'ok' as const,
    version: process.env.npm_package_version ?? '0.0.0',
    uptime: process.uptime(),
  }));

  return app;
}
```

**Testing**:
- `Integration: GET /health returns { status: "ok", version: "...", uptime: ... }`
- `Integration: GET /docs returns Swagger UI HTML`
- `Integration: GET /docs/json returns OpenAPI 3.1 JSON spec`
- `Unit: app factory creates Fastify instance without throwing`
- `Integration: invalid route returns 404 with JSON error body`
- `Integration: server shuts down gracefully on SIGTERM`

---

## Phase 2: Core Generation Engine

### Purpose
Build the heart of the product: the natural language to HCL generation pipeline. After this phase, a user can submit a text prompt and receive valid, modular Terraform/OpenTofu HCL code. This phase deliberately omits security scanning and cost estimation (Phase 4) to keep the core generation loop tight and testable.

### Tasks

#### 2.1 — Provider Schema Introspection & Caching

**What**: Load and cache OpenTofu provider schemas from `tofu providers schema -json` to validate generated resource attributes and eliminate hallucination.

**Design**:

Provider schema types (`packages/core/src/types/provider.ts`):
```typescript
export interface ProviderSchema {
  providerName: string;       // 'aws', 'azurerm', 'google'
  providerVersion: string;    // '5.80.0'
  resourceSchemas: Record<string, ResourceSchema>;
  dataSourceSchemas: Record<string, ResourceSchema>;
}

export interface ResourceSchema {
  version: number;
  block: BlockSchema;
  description?: string;
  deprecated?: boolean;
}

export interface BlockSchema {
  attributes: Record<string, AttributeSchema>;
  blockTypes: Record<string, NestedBlockSchema>;
}

export interface AttributeSchema {
  type: string;              // 'string', 'number', 'bool', 'list(string)', etc.
  description?: string;
  required: boolean;
  optional: boolean;
  computed: boolean;
  sensitive: boolean;
  deprecated?: boolean;
}

export interface NestedBlockSchema {
  block: BlockSchema;
  nestingMode: 'single' | 'list' | 'set' | 'map';
  minItems?: number;
  maxItems?: number;
}
```

Schema cache service (`packages/core/src/providers/schema-cache.ts`):
```typescript
export class ProviderSchemaCache {
  constructor(private db: DrizzleClient, private config: AppConfig) {}

  async getResourceSchema(
    providerName: string,
    resourceType: string,
  ): Promise<ResourceSchema | null>;

  async syncProviderSchema(providerName: string): Promise<void>;
  // Calls `tofu providers schema -json` for the provider,
  // parses the JSON output, and upserts into provider_schemas table.

  async listResourceTypes(providerName: string): Promise<string[]>;

  async getAttributeNames(
    providerName: string,
    resourceType: string,
  ): Promise<string[]>;
  // Returns valid attribute names for the resource type.
  // Used by the generator to validate and correct hallucinated attributes.
}
```

**Testing**:
- `Unit: parseProviderSchema correctly parses AWS provider schema JSON fixture (100+ resource types)`
- `Unit: getResourceSchema("aws", "aws_vpc") returns correct attributes (cidr_block, enable_dns_support, etc.)`
- `Unit: getResourceSchema("aws", "nonexistent_resource") returns null`
- `Unit: getAttributeNames("aws", "aws_instance") includes "ami", "instance_type", excludes fabricated names`
- `Integration (Testcontainers): syncProviderSchema writes schema to database and subsequent reads return cached data`
- `Integration (Testcontainers): re-syncing updates provider_version and schema content`
- `Fixture: tests/fixtures/provider-schemas/aws-partial.json contains a subset of AWS provider schema for offline testing`

#### 2.2 — LLM Prompt Construction

**What**: Build the prompt template system that constructs effective LLM prompts for IaC generation, incorporating provider schema context, existing infrastructure state, and conversation history.

**Design**:

Prompt builder (`packages/core/src/generator/prompt-builder.ts`):
```typescript
export interface GenerationContext {
  userPrompt: string;
  provider: string;                    // 'aws', 'azurerm', 'google'
  region: string;
  outputFormat: 'hcl' | 'pulumi_ts' | 'pulumi_py';
  conversationHistory: ConversationMessage[];
  availableResourceTypes: string[];    // from provider schema cache
  existingResources?: ExistingResource[];  // from cloud state (Phase 5)
  projectModules?: string[];           // existing modules in project
}

export interface ConversationMessage {
  role: 'user' | 'assistant' | 'system';
  content: string;
}

export interface ExistingResource {
  type: string;
  name: string;
  address: string;
  attributes: Record<string, unknown>;
}

export class PromptBuilder {
  buildSystemPrompt(ctx: GenerationContext): string;
  // Constructs system prompt with:
  // 1. Role: "You are an expert IaC engineer..."
  // 2. Output format instructions (HCL syntax rules, module structure)
  // 3. Provider-specific guidelines (naming conventions, region handling)
  // 4. Security defaults to embed (CIS benchmark requirements)
  // 5. Available resource types (from schema cache) to prevent hallucination
  // 6. Existing infrastructure context (if available)

  buildUserPrompt(ctx: GenerationContext): string;
  // Wraps user prompt with structured instructions:
  // 1. The natural language request
  // 2. Output structure requirements (separate files per module)
  // 3. Variable and output extraction instructions
  // 4. JSON-wrapped response format for structured parsing

  buildRefinementPrompt(
    ctx: GenerationContext,
    previousArtifact: string,
    refinementRequest: string,
  ): string;
  // Constructs a refinement prompt that includes:
  // 1. The previous generated code
  // 2. The refinement instruction
  // 3. Instructions to return the complete updated code, not just the diff
}
```

System prompt template (embedded in prompt-builder.ts):
```
You are an expert Infrastructure as Code engineer specializing in {provider}.
You generate production-quality {outputFormat} code following these rules:

OUTPUT FORMAT:
- Generate modular code: separate files for main.tf, variables.tf, outputs.tf, and provider.tf
- Use meaningful resource names that describe purpose, not generic names
- Include inline comments explaining non-obvious configuration choices
- Wrap output in a JSON structure: {"files": [{"path": "...", "content": "..."}]}

SECURITY DEFAULTS (CIS Benchmark aligned):
- All storage resources must have encryption at rest enabled
- All S3 buckets must have public access blocked
- All security groups must have specific CIDR ranges, never 0.0.0.0/0 for ingress
- All RDS instances must have storage_encrypted = true
- All IAM policies must follow least-privilege principle
- Never generate long-lived access keys; use IAM roles and OIDC federation

VALID RESOURCE TYPES for {provider}:
{resourceTypeList}

Do not use resource types or attributes not in this list.

{existingInfrastructureContext}
```

**Testing**:
- `Unit: buildSystemPrompt includes provider name and region in output`
- `Unit: buildSystemPrompt includes CIS security defaults section`
- `Unit: buildSystemPrompt includes resource type list from schema cache`
- `Unit: buildUserPrompt wraps user input in JSON output format instructions`
- `Unit: buildRefinementPrompt includes previous artifact content`
- `Unit: buildSystemPrompt with existingResources includes their addresses and types`
- `Unit: conversation history is correctly serialised into the prompt`
- `Unit: system prompt does not exceed 32K characters (model context budget)`

#### 2.3 — HCL Emitter

**What**: Parse the LLM's JSON response and emit well-formatted, valid HCL files with proper module structure.

**Design**:

HCL emitter types and functions (`packages/core/src/generator/hcl-emitter.ts`):
```typescript
export interface GeneratedFile {
  path: string;        // e.g., 'modules/networking/main.tf'
  content: string;     // raw HCL content
  resourceCount: number;
}

export interface GenerationResult {
  files: GeneratedFile[];
  resources: ExtractedResource[];
  variables: ExtractedVariable[];
  outputs: ExtractedOutput[];
  providers: string[];
  warnings: string[];
}

export interface ExtractedResource {
  type: string;         // 'aws_vpc'
  name: string;         // 'main'
  provider: string;     // 'aws'
  address: string;      // 'module.networking.aws_vpc.main'
  attributes: Record<string, unknown>;
}

export interface ExtractedVariable {
  name: string;
  type: string;
  default?: unknown;
  description?: string;
}

export interface ExtractedOutput {
  name: string;
  value: string;
  description?: string;
  sensitive: boolean;
}

export class HclEmitter {
  constructor(private schemaCache: ProviderSchemaCache) {}

  async parseResponse(llmResponse: string): Promise<GenerationResult>;
  // 1. Extract JSON from LLM response (handle markdown code blocks)
  // 2. Parse the files array
  // 3. Validate each file's HCL syntax (using hcl2-parser)
  // 4. Extract resource declarations, variables, outputs
  // 5. Validate resource types against provider schema cache
  // 6. Validate attribute names against provider schema
  // 7. Format HCL output consistently (2-space indent, aligned = signs)
  // 8. Return result with any warnings (unknown attributes, deprecated resources)

  async validateAttributes(
    resource: ExtractedResource,
  ): Promise<ValidationResult>;
  // Check each attribute against the provider schema.
  // Return errors for invalid attributes, warnings for deprecated ones.

  formatHcl(content: string): string;
  // Apply consistent formatting: 2-space indentation, aligned equals signs,
  // blank lines between resource blocks, sorted meta-arguments first.
}

export interface ValidationResult {
  valid: boolean;
  errors: string[];
  warnings: string[];
}
```

**Testing**:
- `Unit: parseResponse extracts files from valid JSON with code fences`
- `Unit: parseResponse handles LLM response without code fences`
- `Unit: parseResponse with malformed JSON → throws ParseError with descriptive message`
- `Unit: validateAttributes("aws_vpc", {cidr_block: "10.0.0.0/16"}) → valid`
- `Unit: validateAttributes("aws_vpc", {nonexistent_attr: "x"}) → error with attribute name`
- `Unit: formatHcl aligns equals signs within a resource block`
- `Unit: formatHcl sorts meta-arguments (depends_on, lifecycle) before regular attributes`
- `Fixture: tests/fixtures/hcl/valid-vpc.tf contains expected output for a VPC generation prompt`
- `Fixture: tests/fixtures/hcl/multi-module.json contains a multi-file generation response`

#### 2.4 — Generation Orchestrator

**What**: Wire together the prompt builder, LLM client, HCL emitter, and database persistence into a single generation pipeline.

**Design**:

Generation engine (`packages/core/src/generator/engine.ts`):
```typescript
export interface GenerateRequest {
  projectId: string;
  userId: string;
  orgId: string;
  sessionId?: string;           // null for new session, set for refinement
  prompt: string;
  provider?: string;            // default from project config
  region?: string;              // default from project config
  outputFormat?: 'hcl' | 'pulumi_ts' | 'pulumi_py';
}

export interface GenerateResponse {
  sessionId: string;
  artifactId: string;
  files: GeneratedFile[];
  resources: ExtractedResource[];
  variables: ExtractedVariable[];
  outputs: ExtractedOutput[];
  warnings: string[];
  usage: {
    promptTokens: number;
    completionTokens: number;
    totalTokens: number;
    latencyMs: number;
  };
}

export class GenerationEngine {
  constructor(
    private llmClient: LLMClient,
    private promptBuilder: PromptBuilder,
    private hclEmitter: HclEmitter,
    private schemaCache: ProviderSchemaCache,
    private db: DrizzleClient,
  ) {}

  async generate(req: GenerateRequest): Promise<GenerateResponse>;
  // Pipeline:
  // 1. Load or create generation session
  // 2. Load conversation history from messages table
  // 3. Load available resource types from schema cache
  // 4. Build system + user prompts via PromptBuilder
  // 5. Call LLM with prompt caching enabled
  // 6. Parse response via HclEmitter
  // 7. Validate all resources against provider schemas
  // 8. Persist message (user prompt) to messages table
  // 9. Persist message (assistant response) to messages table
  // 10. Persist artifact to artifacts table
  // 11. Return structured response

  async refine(
    sessionId: string,
    refinementPrompt: string,
  ): Promise<GenerateResponse>;
  // Same pipeline but loads existing session context
  // and uses buildRefinementPrompt with previous artifact
}
```

LLM client interface (`packages/core/src/generator/llm-client.ts`):
```typescript
export interface LLMClient {
  complete(messages: LLMMessage[], options: LLMOptions): Promise<LLMResponse>;
}

export interface LLMMessage {
  role: 'system' | 'user' | 'assistant';
  content: string;
  cacheControl?: 'ephemeral';   // Anthropic prompt caching
}

export interface LLMOptions {
  model: string;
  maxTokens: number;
  temperature: number;
}

export interface LLMResponse {
  content: string;
  usage: {
    inputTokens: number;
    outputTokens: number;
    cacheReadTokens?: number;
    cacheWriteTokens?: number;
  };
  stopReason: string;
}

export class AnthropicLLMClient implements LLMClient {
  constructor(private apiKey: string, private baseUrl?: string) {}
  async complete(messages: LLMMessage[], options: LLMOptions): Promise<LLMResponse>;
}
```

**Testing**:
- `Integration (mocked LLM): generate with "Create a VPC with 3 subnets" → returns GenerateResponse with files containing valid HCL`
- `Integration (mocked LLM): generate creates session and message records in database`
- `Integration (mocked LLM): generate persists artifact with content_hash`
- `Integration (mocked LLM): refine loads previous conversation and includes it in prompt`
- `Integration (mocked LLM): refine increments artifact version`
- `Unit: generate with invalid projectId → throws NotFoundError`
- `Unit: LLM returning malformed JSON → retries once, then returns error with original response`
- `Integration (mocked LLM): prompt caching headers sent for system prompt`
- `Unit: usage tracking records correct token counts from LLM response`

#### 2.5 — Generation API Endpoints

**What**: REST API endpoints for creating generation sessions, submitting prompts, refining code, and retrieving artifacts.

**Design**:

Session routes (`packages/api/src/routes/sessions.ts`):
```typescript
// POST /api/v1/sessions
// Create a new generation session and submit the first prompt
// Request: { projectId: string, prompt: string, provider?: string, region?: string, outputFormat?: string }
// Response: GenerateResponse (201 Created)

// POST /api/v1/sessions/:sessionId/messages
// Submit a refinement prompt to an existing session
// Request: { prompt: string }
// Response: GenerateResponse (200 OK)

// GET /api/v1/sessions/:sessionId
// Retrieve session details with conversation history
// Response: { id, projectId, status, source, messages: [...], artifacts: [...], usage: {...} }

// GET /api/v1/sessions
// List sessions for the authenticated user/org
// Query: ?projectId=...&status=...&page=1&limit=20
// Response: { sessions: [...], total: number, page: number }

// DELETE /api/v1/sessions/:sessionId
// Mark session as abandoned
// Response: 204 No Content
```

Artifact routes (`packages/api/src/routes/artifacts.ts`):
```typescript
// GET /api/v1/artifacts/:artifactId
// Retrieve a generated artifact with all metadata
// Response: { id, files: [...], resources: [...], securityScan, costEstimate, version }

// GET /api/v1/artifacts/:artifactId/files
// Download artifact files as a tarball
// Response: application/gzip (tar.gz of all generated files)

// GET /api/v1/artifacts/:artifactId/diff
// Compare two artifact versions within a session
// Query: ?compareWith=<artifactId>
// Response: { additions: number, deletions: number, diff: string }
```

**Testing**:
- `Integration: POST /api/v1/sessions with valid body → 201 with session and artifact`
- `Integration: POST /api/v1/sessions/:id/messages → 200 with refined artifact`
- `Integration: GET /api/v1/sessions/:id returns session with messages and artifacts`
- `Integration: GET /api/v1/sessions with pagination → correct page/total`
- `Integration: DELETE /api/v1/sessions/:id → 204, session marked abandoned`
- `Integration: GET /api/v1/artifacts/:id returns artifact with resources array`
- `Integration: POST /api/v1/sessions without auth → 401`
- `Integration: POST /api/v1/sessions with wrong org → 403`

---

## Phase 3: CLI Application

### Purpose
Build the developer-facing CLI that is the primary interface for the MVP. After this phase, a developer can run `iac-gen generate "Create a VPC with 3 subnets in us-east-1"` and receive generated HCL files written to disk. Conversational refinement is supported via interactive sessions.

### Tasks

#### 3.1 — CLI Framework & Command Structure

**What**: Set up Commander.js with all planned commands, global options, and configuration file loading.

**Design**:

CLI entry point (`packages/cli/src/index.ts`):
```typescript
import { Command } from 'commander';

const program = new Command()
  .name('iac-gen')
  .description('AI-powered Infrastructure as Code generator')
  .version('0.1.0');

// Global options
program
  .option('--config <path>', 'Path to config file', '.iac-gen.yaml')
  .option('--provider <provider>', 'Cloud provider (aws, azurerm, google)', 'aws')
  .option('--region <region>', 'Cloud region')
  .option('--format <format>', 'Output format (hcl, pulumi_ts, pulumi_py)', 'hcl')
  .option('--output <dir>', 'Output directory', '.')
  .option('--json', 'Output as JSON instead of writing files')
  .option('--no-scan', 'Skip security scanning')
  .option('--no-cost', 'Skip cost estimation');

// Commands registered in separate files
program.addCommand(generateCommand);   // iac-gen generate <prompt>
program.addCommand(planCommand);       // iac-gen plan [dir]
program.addCommand(scanCommand);       // iac-gen scan [dir]
program.addCommand(costCommand);       // iac-gen cost [dir]
program.addCommand(initCommand);       // iac-gen init
program.addCommand(configCommand);     // iac-gen config
```

CLI config file (`.iac-gen.yaml`):
```yaml
provider: aws
region: us-east-1
output_format: hcl
output_dir: ./generated
api_url: http://localhost:3000  # or remote API
security:
  auto_scan: true
cost:
  auto_estimate: true
```

**Testing**:
- `E2E: iac-gen --help prints usage with all commands`
- `E2E: iac-gen --version prints version number`
- `E2E: iac-gen generate --help prints generate-specific options`
- `Unit: config file loading merges file config with CLI flags (CLI takes precedence)`
- `Unit: missing config file uses defaults without error`

#### 3.2 — Generate Command with Conversational Refinement

**What**: The core `generate` command that accepts a natural language prompt, calls the generation engine (via API or directly), and writes HCL files to disk with an interactive refinement loop.

**Design**:

Generate command (`packages/cli/src/commands/generate.ts`):
```typescript
// iac-gen generate "Create a VPC with 3 public and 3 private subnets"
// iac-gen generate --interactive   (opens a chat loop)
// iac-gen generate --file prompt.txt

export interface GenerateCommandOptions {
  interactive: boolean;
  file?: string;
  output: string;
  provider: string;
  region: string;
  format: string;
  noScan: boolean;
  noCost: boolean;
  json: boolean;
}
```

Interactive mode flow:
1. User enters prompt
2. CLI calls generation API
3. Display generated file list with resource count
4. Display plan-diff-style summary (N resources to create)
5. Ask: "Refine, save, or cancel? (r/s/c)"
6. If refine: accept refinement prompt, call refine API, loop
7. If save: write files to output directory
8. If cancel: discard session

File output behaviour:
- Create output directory if it doesn't exist
- Write each generated file to its relative path within the output directory
- Display `terraform fmt`-compatible formatting
- Print summary: file count, resource count, estimated cost (if available)

**Testing**:
- `E2E: iac-gen generate "Create an S3 bucket" --output /tmp/test → writes main.tf, variables.tf, outputs.tf to /tmp/test/`
- `E2E: generated main.tf contains resource "aws_s3_bucket" block`
- `E2E: iac-gen generate --file prompt.txt reads prompt from file`
- `E2E: iac-gen generate --json outputs JSON to stdout instead of writing files`
- `Unit: file writer creates nested directories for module paths (modules/vpc/main.tf)`
- `Unit: file writer refuses to overwrite existing files without --force flag`
- `E2E: interactive mode refinement loop sends follow-up prompt and returns updated artifact`

#### 3.3 — Plan Diff Display (Ink Component)

**What**: Rich terminal UI component that displays a terraform-plan-style diff of generated resources.

**Design**:

Ink plan diff component (`packages/cli/src/ui/plan-diff.tsx`):
```typescript
import React from 'react';
import { Box, Text } from 'ink';

interface PlanDiffProps {
  resources: ExtractedResource[];
  previousResources?: ExtractedResource[];
}

// Displays:
// + aws_vpc.main            (create)
// + aws_subnet.public[0]    (create)
// + aws_subnet.public[1]    (create)
// + aws_subnet.public[2]    (create)
// ~ aws_security_group.app  (modify) — when comparing versions
//
// Plan: 4 to add, 1 to change, 0 to destroy.

export const PlanDiff: React.FC<PlanDiffProps>;
```

**Testing**:
- `Unit: PlanDiff with 3 new resources renders green "+" prefix for each`
- `Unit: PlanDiff comparing two versions shows "~" for modified, "+" for added, "-" for removed`
- `Unit: summary line shows correct counts for add/change/destroy`
- `Unit: resources are sorted by address alphabetically`

---

## Phase 4: Security Scanning & Cost Estimation

### Purpose
Integrate security-by-default scanning and cost estimation into the generation pipeline. After this phase, every generated artifact is automatically scanned against CIS benchmarks and priced, with results displayed inline in the CLI and stored on the artifact record.

### Tasks

#### 4.1 — Checkov Integration for Security Scanning

**What**: Call Checkov as a subprocess on generated HCL files, parse SARIF output, and store results on the artifact.

**Design**:

Scanner service (`packages/core/src/security/scanner.ts`):
```typescript
export interface ScanRequest {
  artifactId: string;
  files: GeneratedFile[];
  frameworks?: string[];      // ['cis_aws', 'cis_azure', 'cis_gcp']
}

export interface ScanResult {
  status: 'completed' | 'error';
  scannedAt: string;
  scanner: 'checkov';
  summary: {
    passed: number;
    failed: number;
    skipped: number;
  };
  results: ScanFinding[];
}

export interface ScanFinding {
  ruleId: string;             // 'CIS_AWS_2_1_1'
  benchmark: string;          // 'CIS AWS Foundations 3.0'
  nistControl?: string;       // 'SC-28'
  severity: 'critical' | 'high' | 'medium' | 'low';
  status: 'passed' | 'failed' | 'skipped';
  resourceAddress: string;    // 'aws_s3_bucket.main'
  message: string;
  remediation?: string;
}

export class SecurityScanner {
  constructor(private config: AppConfig) {}

  async scan(req: ScanRequest): Promise<ScanResult>;
  // 1. Write generated files to a temp directory
  // 2. Call: checkov -d <tempDir> --framework <frameworks> -o sarif --quiet
  // 3. Parse SARIF output into ScanResult
  // 4. Map SARIF rule IDs to CIS benchmark control IDs
  // 5. Clean up temp directory
  // 6. Return structured result

  parseSarif(sarifJson: string): ScanFinding[];
  // Parse SARIF 2.1.0 JSON into ScanFinding array
}
```

**Testing**:
- `Integration: scan HCL with public S3 bucket → failed finding with CIS control ID`
- `Integration: scan HCL with encrypted RDS → passed finding`
- `Unit: parseSarif correctly maps SARIF rule/result structure to ScanFinding`
- `Unit: scan with missing checkov binary → error status with descriptive message`
- `Fixture: tests/fixtures/scan-results/s3-public.sarif contains Checkov output for a public bucket`
- `Integration (mocked): scan result is persisted to artifact.securityScan JSONB column`

#### 4.2 — CIS Benchmark Default Injection

**What**: Before passing generated code to Checkov, automatically inject CIS-compliant defaults into resource configurations that are missing security attributes.

**Design**:

CIS defaults injector (`packages/core/src/security/cis-defaults.ts`):
```typescript
export interface CISDefaultRule {
  resourceType: string;           // 'aws_s3_bucket'
  attribute: string;              // 'block_public_acls' (on aws_s3_bucket_public_access_block)
  defaultValue: unknown;
  cisControl: string;             // 'CIS-AWS-2.1.2'
  description: string;
  companion?: CompanionResource;  // companion resource to add if missing
}

export interface CompanionResource {
  type: string;                   // 'aws_s3_bucket_public_access_block'
  nameTemplate: string;           // '{parent_name}'
  attributes: Record<string, unknown>;
  referenceAttribute: string;     // 'bucket' referencing parent
}

// Built-in CIS default rules for AWS, Azure, GCP
export const AWS_CIS_DEFAULTS: CISDefaultRule[] = [
  {
    resourceType: 'aws_s3_bucket',
    attribute: 'block_public_acls',
    defaultValue: true,
    cisControl: 'CIS-AWS-2.1.2',
    description: 'S3 bucket public access block',
    companion: {
      type: 'aws_s3_bucket_public_access_block',
      nameTemplate: '{parent_name}',
      attributes: {
        block_public_acls: true,
        block_public_policy: true,
        ignore_public_acls: true,
        restrict_public_buckets: true,
      },
      referenceAttribute: 'bucket',
    },
  },
  {
    resourceType: 'aws_db_instance',
    attribute: 'storage_encrypted',
    defaultValue: true,
    cisControl: 'CIS-AWS-2.3.1',
    description: 'RDS encryption at rest',
  },
  {
    resourceType: 'aws_db_instance',
    attribute: 'deletion_protection',
    defaultValue: true,
    cisControl: 'CIS-AWS-2.3.2',
    description: 'RDS deletion protection',
  },
  // ... additional rules for each provider
];

export class CISDefaultInjector {
  inject(files: GeneratedFile[], provider: string): GeneratedFile[];
  // 1. Parse each HCL file for resource blocks
  // 2. For each resource, check if required CIS attributes are set
  // 3. If missing, add the attribute with the CIS-compliant default
  // 4. If a companion resource is needed and missing, add it
  // 5. Return modified files with injection comments
}
```

**Testing**:
- `Unit: inject S3 bucket without public access block → adds aws_s3_bucket_public_access_block companion resource`
- `Unit: inject RDS without storage_encrypted → adds storage_encrypted = true`
- `Unit: inject S3 bucket that already has public access block → no modification`
- `Unit: injection adds HCL comment "# CIS-AWS-2.1.2: S3 bucket public access block (auto-injected)"`
- `Unit: inject with provider="azurerm" applies Azure CIS rules, not AWS`

#### 4.3 — Infracost Integration for Cost Estimation

**What**: Call Infracost on generated HCL files to produce per-resource cost estimates.

**Design**:

Cost estimator (`packages/core/src/cost/estimator.ts`):
```typescript
export interface CostEstimate {
  totalMonthlyUsd: number;
  totalHourlyUsd: number;
  currency: 'USD';
  pricingSource: 'infracost';
  estimatedAt: string;
  lineItems: CostLineItem[];
}

export interface CostLineItem {
  resource: string;          // 'aws_db_instance.primary'
  component: string;         // 'compute', 'storage', 'data_transfer'
  monthlyUsd: number;
  unit: string;              // 'hours', 'GB', 'requests'
  unitPrice: number;
  quantity: number;
}

export class CostEstimator {
  constructor(private config: AppConfig) {}

  async estimate(files: GeneratedFile[]): Promise<CostEstimate>;
  // 1. Write generated files to temp directory
  // 2. Call: infracost breakdown --path <tempDir> --format json --no-color
  // 3. Parse JSON output into CostEstimate
  // 4. Clean up temp directory
  // 5. Return structured result
}
```

**Testing**:
- `Integration: estimate HCL with t3.medium EC2 instance → cost estimate with compute line item`
- `Unit: parse infracost JSON output fixture → correct totalMonthlyUsd and line items`
- `Unit: estimate with missing infracost binary → throws with installation instructions`
- `Fixture: tests/fixtures/cost/ec2-estimate.json contains Infracost output for EC2 instance`
- `Integration (mocked): cost estimate is persisted to artifact.costEstimate JSONB column`

#### 4.4 — CLI Scan Results Display (Ink Component)

**What**: Rich terminal display of security scan results and cost estimates after generation.

**Design**:

Scan results component (`packages/cli/src/ui/scan-results.tsx`):
```typescript
// Displays:
// Security Scan: 12 passed, 2 failed, 1 skipped
//
// FAILED:
//   HIGH  CIS-AWS-2.1.1  aws_s3_bucket.data       S3 bucket allows public access
//   MED   CIS-AWS-4.1    aws_security_group.app   SG allows unrestricted SSH ingress
//
// Remediation:
//   aws_s3_bucket.data: Add aws_s3_bucket_public_access_block with block_public_acls = true
//   aws_security_group.app: Restrict ingress CIDR to known IP ranges
```

Cost table component (`packages/cli/src/ui/cost-table.tsx`):
```typescript
// Displays:
// Estimated Monthly Cost: $342.50/mo
//
// Resource                      Component   Monthly
// ─────────────────────────────────────────────────
// aws_db_instance.primary       compute     $185.00
// aws_db_instance.primary       storage      $23.00
// aws_db_instance.replica       compute     $134.50
```

**Testing**:
- `Unit: ScanResults renders FAILED findings in red with severity badge`
- `Unit: ScanResults with zero failures renders green "All checks passed" message`
- `Unit: CostTable renders formatted USD amounts aligned right`
- `Unit: CostTable with no cost data renders "Cost estimation unavailable"`

#### 4.5 — Pipeline Integration: Auto-Scan and Auto-Estimate

**What**: Wire security scanning and cost estimation into the generation pipeline so they run automatically after every generation.

**Design**:

Modify `GenerationEngine.generate()` to add post-generation steps:
```typescript
// After step 10 (persist artifact):
// 11. If config.security.autoScanOnGenerate:
//     a. Inject CIS defaults via CISDefaultInjector
//     b. Run SecurityScanner.scan()
//     c. Update artifact.securityScan with ScanResult
// 12. If config.cost.autoEstimateOnGenerate:
//     a. Run CostEstimator.estimate()
//     b. Update artifact.costEstimate with CostEstimate
// 13. Return GenerateResponse with scan and cost data included
```

Extended `GenerateResponse`:
```typescript
export interface GenerateResponse {
  // ... existing fields ...
  securityScan?: ScanResult;
  costEstimate?: CostEstimate;
}
```

**Testing**:
- `Integration (mocked): generate with autoScan=true → response includes securityScan`
- `Integration (mocked): generate with autoScan=false → response has no securityScan`
- `Integration (mocked): generate with autoCost=true → response includes costEstimate`
- `Integration (mocked): security scan failure does not block generation (warning returned)`
- `Integration (mocked): cost estimation failure does not block generation (warning returned)`

---

## Phase 5: Cloud State Awareness

### Purpose
Enable context-aware generation by reading existing cloud infrastructure state before generating new code. This prevents resource conflicts and allows the generator to reference existing resources. After this phase, the generator can answer prompts like "Add a read replica for the existing RDS instance."

### Tasks

#### 5.1 — Terraform State Reader

**What**: Parse Terraform/OpenTofu state files (local and remote backends) to extract existing resource information.

**Design**:

State reader (`packages/core/src/generator/context-loader.ts`):
```typescript
export interface StateBackendConfig {
  type: 'local' | 's3' | 'gcs' | 'azure_blob' | 'http';
  path?: string;          // for local
  bucket?: string;        // for s3/gcs
  key?: string;           // for s3
  region?: string;        // for s3
  storageAccount?: string; // for azure_blob
  containerName?: string;  // for azure_blob
}

export interface CloudState {
  resources: ExistingResource[];
  outputs: Record<string, { value: unknown; type: string; sensitive: boolean }>;
  serial: number;
  terraformVersion: string;
}

export class StateReader {
  async readState(config: StateBackendConfig): Promise<CloudState>;
  // 1. Load state file from configured backend
  //    - Local: read file from disk
  //    - S3: use AWS SDK to get object
  //    - GCS: use GCP client to get object
  //    - Azure Blob: use Azure SDK to get blob
  // 2. Parse JSON state format
  // 3. Extract resources with their current attributes
  // 4. Return structured CloudState

  async summariseForPrompt(state: CloudState): Promise<string>;
  // Create a concise summary of existing infrastructure for LLM context.
  // Format: list of resource types and names, key attributes (IDs, CIDRs, etc.)
  // Truncate to fit within context budget (~4K characters)
}
```

**Testing**:
- `Unit: readState with local backend → parses tfstate JSON correctly`
- `Unit: summariseForPrompt with 50 resources → output under 4K characters`
- `Unit: summariseForPrompt includes resource type, name, and key attributes`
- `Fixture: tests/fixtures/tfstate/sample.tfstate contains a 10-resource state file`
- `Integration (mocked S3): readState with s3 backend → downloads and parses state`
- `Unit: readState with nonexistent local file → returns empty CloudState`

#### 5.2 — Drift Detection

**What**: Compare generated artifact resources against cloud state to detect drift and signal it in the generation response.

**Design**:

Drift detector (`packages/core/src/generator/drift-detector.ts`):
```typescript
export interface DriftResult {
  hasDrift: boolean;
  drifted: DriftedResource[];
  unmanaged: string[];       // resources in cloud not in state
}

export interface DriftedResource {
  address: string;
  attribute: string;
  expectedValue: unknown;    // from state
  actualValue: unknown;      // from cloud
}

export class DriftDetector {
  async detectDrift(
    stateResources: ExistingResource[],
    provider: string,
    credentials: ProviderCredentials,
  ): Promise<DriftResult>;
  // 1. For each resource in state, query the cloud provider API for current attributes
  // 2. Compare state attributes with live values
  // 3. Report any differences as drifted resources
  // This is resource-type-specific and supports AWS, Azure, GCP core resources
}
```

**Testing**:
- `Integration (mocked AWS API): VPC with modified CIDR tag → drift detected`
- `Integration (mocked AWS API): all resources matching state → no drift`
- `Unit: DriftResult serialises to JSON for inclusion in generation context`
- `Unit: drift detection skips computed-only attributes`

---

## Phase 6: Authentication & Multi-Tenancy

### Purpose
Add JWT-based authentication, OAuth login (GitHub), and organisation-based tenant isolation. After this phase, multiple users can use the platform with proper access controls and data isolation.

### Tasks

#### 6.1 — JWT Authentication Middleware

**What**: Fastify middleware that validates JWT tokens on all API routes except health check and auth routes.

**Design**:

Auth middleware (`packages/api/src/middleware/auth.ts`):
```typescript
export interface AuthenticatedUser {
  userId: string;
  email: string;
  orgId: string;
  role: 'owner' | 'admin' | 'member' | 'viewer';
  permissions: Record<string, boolean>;
}

// Fastify decorator: request.user is set after auth
declare module 'fastify' {
  interface FastifyRequest {
    user: AuthenticatedUser;
  }
}

export async function authPlugin(app: FastifyInstance): Promise<void>;
// 1. Register @fastify/jwt with secret from config
// 2. Add onRequest hook to verify JWT on all routes
// 3. Skip auth for: /health, /auth/*, /docs*
// 4. Extract user claims from JWT
// 5. Set request.user with org context
```

**Testing**:
- `Integration: request without Authorization header → 401`
- `Integration: request with invalid JWT → 401`
- `Integration: request with expired JWT → 401`
- `Integration: request with valid JWT → passes through, request.user populated`
- `Integration: GET /health without auth → 200 (exempt route)`
- `Unit: AuthenticatedUser type matches JWT claims structure`

#### 6.2 — GitHub OAuth Login Flow

**What**: OAuth 2.0 login via GitHub with automatic user and organisation creation.

**Design**:

Auth routes (`packages/api/src/routes/auth.ts`):
```typescript
// GET /auth/github
// Redirect to GitHub OAuth authorization URL
// Query: ?redirect_uri=...

// GET /auth/github/callback
// Handle OAuth callback, create/update user, issue JWT
// Query: ?code=...&state=...
// Response: { token: string, user: { id, email, displayName, orgId } }

// POST /auth/refresh
// Refresh an expiring JWT
// Request: { token: string }
// Response: { token: string }
```

**Testing**:
- `Integration: GET /auth/github redirects to github.com/login/oauth/authorize`
- `Integration (mocked GitHub API): callback with valid code → creates user, returns JWT`
- `Integration (mocked GitHub API): callback for existing user → updates lastLoginAt, returns JWT`
- `Unit: JWT contains userId, email, orgId, role claims`
- `Integration: POST /auth/refresh with valid token → new token with extended expiry`

#### 6.3 — Multi-Tenant Data Isolation

**What**: Middleware that ensures all database queries are scoped to the authenticated user's organisation.

**Design**:

Tenant middleware (`packages/api/src/middleware/tenant.ts`):
```typescript
export async function tenantPlugin(app: FastifyInstance): Promise<void>;
// 1. After auth middleware sets request.user
// 2. Set request.orgId from JWT claims
// 3. All repository methods accept orgId and filter accordingly
// 4. Audit log entries automatically tagged with orgId
```

Repository pattern for tenant isolation:
```typescript
export class ProjectRepository {
  constructor(private db: DrizzleClient) {}

  async findAll(orgId: string, opts: PaginationOpts): Promise<Project[]> {
    return this.db.select().from(projects)
      .where(eq(projects.orgId, orgId))
      .limit(opts.limit)
      .offset(opts.offset);
  }

  async findById(orgId: string, projectId: string): Promise<Project | null> {
    const [result] = await this.db.select().from(projects)
      .where(and(eq(projects.orgId, orgId), eq(projects.id, projectId)));
    return result ?? null;
  }
}
```

**Testing**:
- `Integration: user from org-A cannot access org-B's projects → empty result`
- `Integration: user from org-A cannot access org-B's sessions → 404`
- `Integration: creating a project sets orgId from authenticated user`
- `Integration: audit log entries include orgId from request context`

---

## Phase 7: Git/VCS Integration

### Purpose
Enable PR-based review workflows for generated IaC. After this phase, a user can generate code and have it automatically committed to a branch and opened as a pull request for team review.

### Tasks

#### 7.1 — Git Operations Service

**What**: Programmatic Git operations: clone, branch, commit, push for generated artifact files.

**Design**:

Git service (`packages/core/src/vcs/git-service.ts`):
```typescript
export interface GitConfig {
  repoUrl: string;
  branch: string;
  basePath: string;          // subdirectory in repo for IaC files
  token: string;             // GitHub/GitLab personal access token
}

export class GitService {
  async cloneOrPull(config: GitConfig, workDir: string): Promise<void>;
  async createBranch(workDir: string, branchName: string): Promise<void>;
  async commitFiles(
    workDir: string,
    files: GeneratedFile[],
    message: string,
    author: { name: string; email: string },
  ): Promise<string>;  // returns commit SHA
  async push(workDir: string, branchName: string): Promise<void>;
}
```

**Testing**:
- `Integration (local git): cloneOrPull initialises a local repo`
- `Integration (local git): createBranch creates and checks out a new branch`
- `Integration (local git): commitFiles writes files and creates commit with correct message`
- `Unit: branch name generation from session title sanitises special characters`

#### 7.2 — Pull Request Creation (GitHub API)

**What**: Create pull requests on GitHub with generated code, security scan summary, and cost estimate in the PR body.

**Design**:

PR service (`packages/core/src/vcs/pr-service.ts`):
```typescript
export interface PullRequestConfig {
  owner: string;
  repo: string;
  token: string;
}

export interface CreatePRRequest {
  title: string;
  body: string;
  head: string;            // source branch
  base: string;            // target branch
  labels?: string[];
  reviewers?: string[];
}

export class PullRequestService {
  constructor(private config: PullRequestConfig) {}

  async create(req: CreatePRRequest): Promise<{ url: string; number: number }>;

  buildPRBody(artifact: GenerateResponse): string;
  // Generates markdown body with:
  // - Generated resource summary (table)
  // - Security scan results (pass/fail counts, failed findings)
  // - Cost estimate (monthly cost, line items)
  // - Generation session link
  // - "Generated by IaC Generator" footer
}
```

**Testing**:
- `Integration (mocked GitHub API): create PR → returns URL and PR number`
- `Unit: buildPRBody includes resource table with types and names`
- `Unit: buildPRBody includes security scan summary with failed count`
- `Unit: buildPRBody includes cost estimate with monthly total`
- `Unit: buildPRBody with zero security failures shows green checkmark`

---

## Phase 8: Module Registry Integration

### Purpose
Enable the generator to recommend existing OpenTofu registry modules before generating from scratch, reducing code duplication and improving quality. After this phase, the generator can suggest "Use the terraform-aws-modules/vpc/aws module instead of generating raw resources."

### Tasks

#### 8.1 — OpenTofu Registry Client

**What**: Client for the OpenTofu module registry API to search and retrieve module metadata.

**Design**:

Registry client (`packages/core/src/modules/registry-client.ts`):
```typescript
export interface ModuleSearchResult {
  source: string;           // 'registry.opentofu.org/hashicorp/vpc/aws'
  name: string;             // 'vpc'
  provider: string;         // 'aws'
  version: string;          // '5.16.0'
  description: string;
  downloadCount: number;
  verified: boolean;
  inputs: ModuleInput[];
  outputs: ModuleOutput[];
}

export interface ModuleInput {
  name: string;
  type: string;
  required: boolean;
  default?: unknown;
  description?: string;
}

export interface ModuleOutput {
  name: string;
  type: string;
  description?: string;
}

export class RegistryClient {
  constructor(private baseUrl: string) {}

  async search(query: string, provider: string): Promise<ModuleSearchResult[]>;
  // GET /v1/modules/search?q=<query>&provider=<provider>

  async getModule(namespace: string, name: string, provider: string): Promise<ModuleSearchResult>;
  // GET /v1/modules/<namespace>/<name>/<provider>

  async syncToCache(modules: ModuleSearchResult[]): Promise<void>;
  // Upsert module metadata to module_cache table
}
```

**Testing**:
- `Integration (mocked API): search("vpc", "aws") → returns terraform-aws-modules/vpc/aws`
- `Integration (mocked API): getModule returns inputs and outputs`
- `Integration (Testcontainers): syncToCache persists modules to database`
- `Unit: search with empty results → returns empty array`

#### 8.2 — Module Recommender

**What**: Given a generation prompt, recommend relevant registry modules that could replace raw resource generation.

**Design**:

Module recommender (`packages/core/src/modules/recommender.ts`):
```typescript
export interface ModuleRecommendation {
  module: ModuleSearchResult;
  relevanceScore: number;     // 0.0 - 1.0
  reason: string;             // "This module creates VPC with subnets, which matches your prompt"
  suggestedInputs: Record<string, unknown>;  // pre-filled input values from prompt context
}

export class ModuleRecommender {
  constructor(
    private registryClient: RegistryClient,
    private db: DrizzleClient,
  ) {}

  async recommend(
    prompt: string,
    provider: string,
  ): Promise<ModuleRecommendation[]>;
  // 1. Extract infrastructure concepts from prompt (vpc, subnet, rds, etc.)
  // 2. Search module cache for matching modules
  // 3. Score relevance based on concept match and download count
  // 4. Return top 3 recommendations with suggested input values
}
```

**Testing**:
- `Unit: recommend("Create a VPC with subnets", "aws") → includes vpc module recommendation`
- `Unit: recommend returns max 3 recommendations sorted by relevance`
- `Unit: recommendations include pre-filled input values derived from prompt`
- `Unit: recommend for a niche resource type → empty recommendations (fall back to generation)`

---

## Phase 9: MCP Server

### Purpose
Expose the generation engine as an MCP server so any MCP-compatible AI assistant (Claude Code, Cursor, GitHub Copilot) can drive infrastructure generation without a dedicated UI. This follows the MCP v2025-11-25 specification.

### Tasks

#### 9.1 — MCP Server with Generation Tools

**What**: MCP server exposing tools for IaC generation, plan preview, security scanning, and cost estimation.

**Design**:

MCP server entry point (`packages/mcp/src/index.ts`):
```typescript
import { McpServer } from '@modelcontextprotocol/sdk/server/mcp.js';
import { StreamableHTTPServerTransport } from '@modelcontextprotocol/sdk/server/streamableHttp.js';

const server = new McpServer({
  name: 'iac-generator',
  version: '0.1.0',
});
```

MCP Tools:

```typescript
// Tool: generate_iac
// Input: { prompt: string, provider?: string, region?: string, sessionId?: string }
// Output: { files: [...], resources: [...], securityScan: {...}, costEstimate: {...} }

// Tool: preview_plan
// Input: { artifactId: string, workspace: string }
// Output: { resourceChanges: [...], summary: { add, change, destroy } }

// Tool: security_scan
// Input: { artifactId: string, frameworks?: string[] }
// Output: ScanResult

// Tool: estimate_cost
// Input: { artifactId: string }
// Output: CostEstimate

// Tool: list_sessions
// Input: { projectId?: string, limit?: number }
// Output: { sessions: [...] }
```

MCP Resources:

```typescript
// Resource: session://{sessionId}
// Returns full conversation history and artifacts for a session

// Resource: artifact://{artifactId}
// Returns artifact content, resources, scan results, and cost estimate

// Resource: project://{projectId}
// Returns project configuration and workspace details
```

**Testing**:
- `Integration: MCP generate_iac tool returns valid generation response`
- `Integration: MCP security_scan tool returns ScanResult`
- `Integration: MCP estimate_cost tool returns CostEstimate`
- `Integration: MCP session resource returns conversation history`
- `Integration: MCP server responds to initialize request with capabilities`
- `Unit: MCP tools have correct JSON Schema input definitions`
- `E2E: connect MCP client to server via streamable HTTP transport → call generate_iac tool → receive files`

#### 9.2 — MCP Session Management

**What**: Track MCP client connections and map them to generation sessions.

**Design**:

Extend generation sessions to track MCP context:
```typescript
// When source is 'mcp', the session.context JSONB includes:
// {
//   "mcp_client": "claude_code",
//   "mcp_transport": "streamable_http",
//   "mcp_session_id": "...",
//   "connected_at": "2026-05-25T10:00:00Z"
// }
```

**Testing**:
- `Integration: MCP tool call creates session with source='mcp' and mcp_client in context`
- `Integration: subsequent MCP tool calls on same session maintain conversation history`
- `Unit: MCP disconnect cleans up session tracking metadata`

---

## Phase 10: Deployment Pipeline (Plan/Apply)

### Purpose
Enable the full plan/apply workflow for generated IaC. After this phase, users can preview what Terraform will do, get approval, and apply changes to real cloud infrastructure through the platform.

### Tasks

#### 10.1 — Terraform Plan Execution

**What**: Execute `tofu plan` on generated artifacts within a workspace and capture the plan output.

**Design**:

Plan executor (`packages/core/src/deployment/plan-executor.ts`):
```typescript
export interface PlanRequest {
  artifactId: string;
  workspace: string;
  projectId: string;
  userId: string;
  orgId: string;
}

export interface PlanResult {
  planId: string;
  status: 'planned' | 'failed';
  planJson: Record<string, unknown>;  // terraform show -json output
  summary: {
    resourcesToAdd: number;
    resourcesToChange: number;
    resourcesToDestroy: number;
  };
  output: string;                      // human-readable plan output
  errorMessage?: string;
}

export class PlanExecutor {
  constructor(
    private db: DrizzleClient,
    private credentialManager: CredentialManager,
  ) {}

  async plan(req: PlanRequest): Promise<PlanResult>;
  // 1. Load artifact files from database
  // 2. Decrypt provider credentials for the workspace
  // 3. Write files to secure temp directory
  // 4. Run: tofu init -backend-config=...
  // 5. Run: tofu plan -out=plan.bin -json
  // 6. Run: tofu show -json plan.bin → capture plan JSON
  // 7. Parse resource changes from plan JSON
  // 8. Create deployment record with status='planned'
  // 9. Clean up temp directory
  // 10. Return structured PlanResult
}
```

**Testing**:
- `Integration (mocked tofu): plan with valid HCL → PlanResult with summary`
- `Integration (mocked tofu): plan with invalid HCL → PlanResult with status='failed' and errorMessage`
- `Integration (Testcontainers): deployment record created with plan_output JSONB`
- `Unit: plan JSON parser extracts correct resource change counts`

#### 10.2 — Plan Approval Workflow

**What**: Human-in-the-loop approval gate before applying infrastructure changes.

**Design**:

Approval routes:
```typescript
// POST /api/v1/deployments/:planId/approve
// Request: { comment?: string }
// Response: { status: 'approved', approvedBy: string, approvedAt: string }

// POST /api/v1/deployments/:planId/reject
// Request: { reason: string }
// Response: { status: 'rejected' }
```

Approval rules:
- Only `owner` and `admin` roles can approve plans
- Self-approval allowed for non-production workspaces
- Production workspaces require approval from a different user than the creator
- Approval records stored in deployment.approval JSONB

**Testing**:
- `Integration: approve with admin role → deployment status changes to 'approved'`
- `Integration: approve with member role → 403 Forbidden`
- `Integration: approve production plan by same creator → 403 (requires different approver)`
- `Integration: reject plan → deployment status changes to 'rejected' with reason`

#### 10.3 — Terraform Apply Execution

**What**: Execute `tofu apply` on approved plans and track the deployment lifecycle.

**Design**:

Apply executor:
```typescript
export class ApplyExecutor {
  async apply(planId: string): Promise<ApplyResult>;
  // 1. Verify deployment status is 'approved'
  // 2. Load plan binary from storage
  // 3. Run: tofu apply plan.bin
  // 4. Stream output to audit log
  // 5. Update deployment status: 'applying' → 'applied' or 'failed'
  // 6. Update deployment timing JSONB
  // 7. Record new state serial
}
```

**Testing**:
- `Integration (mocked tofu): apply approved plan → status changes to 'applied'`
- `Integration (mocked tofu): apply unapproved plan → error "Plan not approved"`
- `Integration (mocked tofu): apply failure → status='failed' with error message`
- `Integration: apply records timing in deployment.timing JSONB`

---

## Phase 11: Pulumi Output Support

### Purpose
Add Pulumi TypeScript and Python as alternative output formats alongside HCL. After this phase, users can generate the same infrastructure in their preferred IaC language.

### Tasks

#### 11.1 — Pulumi TypeScript Emitter

**What**: Generate Pulumi TypeScript code from the same generation pipeline, as an alternative to HCL output.

**Design**:

Pulumi emitter (`packages/core/src/generator/pulumi-emitter.ts`):
```typescript
export class PulumiEmitter {
  async parseResponse(
    llmResponse: string,
    targetLanguage: 'typescript' | 'python',
  ): Promise<GenerationResult>;
  // Same contract as HclEmitter.parseResponse but for Pulumi output
  // Validates import statements against Pulumi provider packages
  // Generates index.ts (or __main__.py), Pulumi.yaml, and package.json (or requirements.txt)

  generateProjectFiles(
    provider: string,
    projectName: string,
    language: 'typescript' | 'python',
  ): GeneratedFile[];
  // Generate Pulumi.yaml, tsconfig.json/requirements.txt, package.json
}
```

The prompt builder is extended with a Pulumi-specific system prompt:
```
OUTPUT FORMAT (Pulumi TypeScript):
- Generate a Pulumi program with index.ts as the entry point
- Use @pulumi/aws (or @pulumi/azure-native, @pulumi/gcp) package imports
- Include Pulumi.yaml with project name and runtime: nodejs
- Include package.json with correct dependencies
- Use Pulumi Config for parameterised values
- Wrap output in: {"files": [{"path": "...", "content": "..."}]}
```

**Testing**:
- `Unit: parseResponse extracts Pulumi TypeScript files correctly`
- `Unit: generated index.ts contains valid import statements for @pulumi/aws`
- `Unit: Pulumi.yaml includes correct runtime and project name`
- `Unit: package.json includes correct @pulumi/* dependencies`
- `Integration (mocked LLM): generate with outputFormat='pulumi_ts' → Pulumi TypeScript files`

#### 11.2 — Pulumi Python Emitter

**What**: Generate Pulumi Python code as an output format.

**Design**:

Extends PulumiEmitter with Python-specific prompt template and file generation:
```
OUTPUT FORMAT (Pulumi Python):
- Generate a Pulumi program with __main__.py as the entry point
- Use pulumi_aws (or pulumi_azure_native, pulumi_gcp) package imports
- Include Pulumi.yaml with runtime: python
- Include requirements.txt with correct dependencies
- Use pulumi.Config for parameterised values
```

**Testing**:
- `Unit: generated __main__.py contains valid import statements for pulumi_aws`
- `Unit: requirements.txt includes correct pulumi_* packages`
- `Integration (mocked LLM): generate with outputFormat='pulumi_py' → Pulumi Python files`

---

## Phase 12: OPA Policy Engine & Audit Trail

### Purpose
Add custom organisational policy enforcement via OPA/Rego and comprehensive audit logging. After this phase, organisations can define custom policies beyond CIS benchmarks, and all actions are logged for compliance.

### Tasks

#### 12.1 — OPA/Rego Policy Engine

**What**: Evaluate generated artifacts against organisation-defined OPA/Rego policies.

**Design**:

Policy engine (`packages/core/src/security/policy-engine.ts`):
```typescript
export interface PolicyEvalRequest {
  artifactId: string;
  orgId: string;
  policies?: string[];        // specific policy IDs, or all org policies if empty
}

export interface PolicyEvalResult {
  totalPolicies: number;
  passed: number;
  failed: number;
  violations: PolicyViolation[];
}

export interface PolicyViolation {
  policyId: string;
  policyName: string;
  severity: 'critical' | 'high' | 'medium' | 'low';
  resourceAddress: string;
  message: string;
  remediation?: string;
}

export class PolicyEngine {
  constructor(private db: DrizzleClient) {}

  async evaluate(req: PolicyEvalRequest): Promise<PolicyEvalResult>;
  // 1. Load org's security policies from database
  // 2. Load artifact content
  // 3. Convert HCL to JSON (for OPA input)
  // 4. Evaluate each Rego policy against the artifact
  // 5. Collect violations
  // 6. Return structured result

  async createPolicy(
    orgId: string,
    name: string,
    regoContent: string,
    config: Record<string, unknown>,
  ): Promise<string>;
  // Validate Rego syntax before saving
}
```

**Testing**:
- `Integration: evaluate with "no public subnets" policy on artifact with public subnet → violation`
- `Integration: evaluate with passing artifact → zero violations`
- `Unit: createPolicy with invalid Rego syntax → validation error`
- `Unit: PolicyViolation includes policy name, severity, and resource address`
- `Integration: policy evaluation results merged into artifact.securityScan`

#### 12.2 — Comprehensive Audit Logging

**What**: Log all significant actions to the audit_log table with structured details.

**Design**:

Audit service (`packages/core/src/audit/audit-service.ts`):
```typescript
export interface AuditEntry {
  orgId: string;
  userId?: string;
  action: string;              // 'session.created', 'artifact.generated', 'plan.approved'
  entityType: string;          // 'session', 'artifact', 'deployment', 'policy'
  entityId: string;
  details: Record<string, unknown>;
}

export class AuditService {
  constructor(private db: DrizzleClient) {}

  async log(entry: AuditEntry): Promise<void>;
  // Insert into audit_log table. Never throws — logs errors to stderr.

  async query(
    orgId: string,
    filters: AuditFilters,
  ): Promise<{ entries: AuditLogRecord[]; total: number }>;
}

export interface AuditFilters {
  action?: string;
  entityType?: string;
  entityId?: string;
  userId?: string;
  from?: Date;
  to?: Date;
  page?: number;
  limit?: number;
}
```

Audited actions:
- `session.created` / `session.completed` / `session.abandoned`
- `artifact.generated` / `artifact.refined`
- `scan.completed` (with pass/fail summary)
- `cost.estimated` (with total)
- `plan.created` / `plan.approved` / `plan.rejected`
- `deployment.applied` / `deployment.failed`
- `policy.created` / `policy.updated` / `policy.deleted`
- `credential.created` / `credential.rotated`
- `user.login` / `user.logout`

**Testing**:
- `Integration: generating an artifact creates audit entry with action='artifact.generated'`
- `Integration: approving a plan creates audit entry with approver details`
- `Integration: query with action filter returns only matching entries`
- `Integration: query with date range returns entries within range`
- `Unit: audit log never throws — errors logged to stderr`
- `Integration: audit entries include IP address and user agent from request context`

---

## Phase Summary & Dependencies

```
Phase 1: Foundation & Project Scaffolding ─── required by everything
    │
Phase 2: Core Generation Engine          ─── requires Phase 1
    │
    ├── Phase 3: CLI Application          ─── requires Phase 2
    │       │
    │       └── Phase 4: Security Scanning & Cost Estimation ─── requires Phase 2, enhances Phase 3
    │
    ├── Phase 5: Cloud State Awareness    ─── requires Phase 2, can parallel with Phase 3
    │
    └── Phase 6: Authentication & Multi-Tenancy ─── requires Phase 1, can parallel with Phases 3-5
         │
         ├── Phase 7: Git/VCS Integration ─── requires Phase 2 + Phase 6
         │
         ├── Phase 8: Module Registry Integration ─── requires Phase 2, can parallel with Phases 6-7
         │
         └── Phase 9: MCP Server          ─── requires Phase 2 + Phase 6
              │
              Phase 10: Deployment Pipeline (Plan/Apply) ─── requires Phase 5 + Phase 6
              │
              Phase 11: Pulumi Output Support ─── requires Phase 2, can parallel with Phases 6-10
              │
              Phase 12: OPA Policy Engine & Audit Trail ─── requires Phase 4 + Phase 6
```

### Parallelism Opportunities

- **Phases 3, 5, 6** can be developed concurrently after Phase 2
- **Phase 4** can start as soon as Phase 2 is complete (independent of Phase 3, but enhances it)
- **Phases 8 and 11** are largely independent and can be developed in parallel with most other phases after Phase 2
- **Phase 7** requires both Phase 2 and Phase 6 but is otherwise independent
- **Phase 9** and **Phase 10** can be developed in parallel after Phase 6

---

## Definition of Done (per phase)

1. All tasks implemented with code matching the design specifications.
2. All unit tests pass (`vitest run`).
3. All integration tests pass (Testcontainers for database, mocked external services).
4. ESLint passes with zero errors (`pnpm lint`).
5. Prettier formatting applied (`pnpm format --check`).
6. TypeScript strict mode type checking passes (`tsc --noEmit`).
7. Docker build succeeds (`docker build .`).
8. Feature works end-to-end (manual or E2E test validation).
9. New configuration options documented in `.env.example` with comments.
10. New API endpoints appear in auto-generated OpenAPI spec (`/docs/json`).
11. Database migrations created and tested (Drizzle push/migrate).
12. Audit log entries emitted for all state-changing operations (from Phase 12 onward).
13. No secrets committed to source (`.env` in `.gitignore`).
