# Data Model Suggestion 1: Entity-Centric Normalized Relational

> Project: Infrastructure as Code Generator · Created: 2026-05-20

## Philosophy

This model follows a traditional normalized relational approach where every concept — users, organisations, generation sessions, prompts, generated artifacts, cloud resources, security policies, and cost estimates — has its own dedicated table with explicit foreign key relationships. The schema prioritises referential integrity and query clarity: every relationship is enforced at the database level, and every entity can be independently queried, joined, and indexed.

This approach mirrors how established IaC platforms like Terraform Cloud and Spacelift model their internal state: discrete resource records linked to workspaces, organisations, and policy sets via foreign keys. It is the natural choice for a platform where compliance auditing, multi-tenant isolation, and complex cross-entity queries (e.g., "show all resources generated for org X that failed CIS benchmark checks in the last 30 days") are first-class requirements.

The trade-off is a higher table count and more complex migrations as the schema evolves. Adding a new cloud provider or resource type requires new reference data rows but not schema changes, which is a strength of the normalized approach.

**Best for:** Teams prioritising data integrity, SQL-native querying, and regulatory compliance where every relationship must be explicitly enforced.

**Trade-offs:**
- (+) Strong referential integrity — broken relationships are impossible
- (+) Clear, self-documenting schema — new developers understand the domain from the DDL
- (+) Excellent for complex ad-hoc queries and reporting
- (+) Standard PostgreSQL tooling (pg_dump, logical replication) works out of the box
- (-) Higher table count (~35-40 tables) means more complex migrations
- (-) Schema changes for new resource types or provider attributes require migrations
- (-) JOIN-heavy queries for full artifact retrieval may need careful index tuning
- (-) Less flexible for jurisdiction-specific or provider-specific metadata that varies widely

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| HCL Specification | Generated artifact content stored as text; artifact metadata tracks HCL version and format compliance |
| OpenTofu Provider Protocol | Provider schema cache tables mirror the `tofu providers schema -json` output structure |
| OpenAPI 3.x | Provider resource schemas ingested from OpenAPI specs stored in `provider_resource_types` |
| OPA / Rego | Policy rules stored in `security_policies` table; check results linked to artifacts |
| CIS Benchmarks | Benchmark rules modeled as `security_rules` with CIS control IDs as reference |
| OSCAL | Compliance mappings in `compliance_controls` link generated resources to NIST control families |
| OAuth 2.0 / OIDC | User authentication tokens and provider credentials modeled in `user_credentials` and `provider_credentials` |
| MCP (Model Context Protocol) | MCP session tracking in `mcp_sessions` for IDE-driven generation |
| ISO 3166 | Region codes in `cloud_regions` reference table use ISO 3166-1/2 codes |

---

## Core Identity & Multi-Tenancy

```sql
-- Organisations (tenants)
CREATE TABLE organisations (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            TEXT NOT NULL,
    slug            TEXT NOT NULL UNIQUE,
    plan_tier       TEXT NOT NULL DEFAULT 'free' CHECK (plan_tier IN ('free', 'team', 'enterprise')),
    settings        JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Users
CREATE TABLE users (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    email           TEXT NOT NULL UNIQUE,
    display_name    TEXT NOT NULL,
    avatar_url      TEXT,
    auth_provider   TEXT NOT NULL CHECK (auth_provider IN ('github', 'gitlab', 'google', 'saml', 'oidc')),
    auth_subject    TEXT NOT NULL,
    last_login_at   TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (auth_provider, auth_subject)
);

-- Organisation membership with roles
CREATE TABLE org_memberships (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id          UUID NOT NULL REFERENCES organisations(id) ON DELETE CASCADE,
    user_id         UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    role            TEXT NOT NULL DEFAULT 'member' CHECK (role IN ('owner', 'admin', 'member', 'viewer')),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (org_id, user_id)
);

CREATE INDEX idx_org_memberships_user ON org_memberships(user_id);
CREATE INDEX idx_org_memberships_org ON org_memberships(org_id);

-- Teams within organisations
CREATE TABLE teams (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id          UUID NOT NULL REFERENCES organisations(id) ON DELETE CASCADE,
    name            TEXT NOT NULL,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (org_id, name)
);

CREATE TABLE team_memberships (
    team_id         UUID NOT NULL REFERENCES teams(id) ON DELETE CASCADE,
    user_id         UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    PRIMARY KEY (team_id, user_id)
);
```

## Projects & Workspaces

```sql
-- Projects group related generation work
CREATE TABLE projects (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id          UUID NOT NULL REFERENCES organisations(id) ON DELETE CASCADE,
    name            TEXT NOT NULL,
    slug            TEXT NOT NULL,
    description     TEXT,
    default_provider TEXT NOT NULL DEFAULT 'aws' CHECK (default_provider IN ('aws', 'azure', 'gcp', 'kubernetes')),
    default_region  TEXT,
    vcs_repo_url    TEXT,                    -- linked Git repository
    vcs_branch      TEXT DEFAULT 'main',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (org_id, slug)
);

CREATE INDEX idx_projects_org ON projects(org_id);

-- Workspaces represent deployment targets (dev, staging, prod)
CREATE TABLE workspaces (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    project_id      UUID NOT NULL REFERENCES projects(id) ON DELETE CASCADE,
    name            TEXT NOT NULL,
    environment     TEXT NOT NULL CHECK (environment IN ('development', 'staging', 'production', 'custom')),
    provider        TEXT NOT NULL,
    region          TEXT NOT NULL,
    state_backend   TEXT NOT NULL DEFAULT 'local' CHECK (state_backend IN ('local', 's3', 'gcs', 'azure_blob', 'pulumi_cloud')),
    state_config    JSONB NOT NULL DEFAULT '{}',
    auto_apply      BOOLEAN NOT NULL DEFAULT false,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (project_id, name)
);

CREATE INDEX idx_workspaces_project ON workspaces(project_id);
```

## Generation Sessions & Prompts

```sql
-- A generation session is a conversational interaction
CREATE TABLE generation_sessions (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    project_id      UUID NOT NULL REFERENCES projects(id) ON DELETE CASCADE,
    user_id         UUID NOT NULL REFERENCES users(id),
    workspace_id    UUID REFERENCES workspaces(id),
    title           TEXT,
    status          TEXT NOT NULL DEFAULT 'active' CHECK (status IN ('active', 'completed', 'abandoned')),
    source          TEXT NOT NULL DEFAULT 'cli' CHECK (source IN ('cli', 'web', 'mcp', 'api')),
    model_id        TEXT,                    -- LLM model used (e.g., 'claude-opus-4-6')
    total_tokens    INTEGER DEFAULT 0,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_sessions_project ON generation_sessions(project_id);
CREATE INDEX idx_sessions_user ON generation_sessions(user_id);
CREATE INDEX idx_sessions_created ON generation_sessions(created_at DESC);

-- Individual prompts within a session (conversational refinement)
CREATE TABLE prompts (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    session_id      UUID NOT NULL REFERENCES generation_sessions(id) ON DELETE CASCADE,
    sequence_num    INTEGER NOT NULL,
    role            TEXT NOT NULL CHECK (role IN ('user', 'assistant', 'system')),
    content         TEXT NOT NULL,
    tokens_used     INTEGER,
    latency_ms      INTEGER,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (session_id, sequence_num)
);

CREATE INDEX idx_prompts_session ON prompts(session_id, sequence_num);
```

## Generated Artifacts

```sql
-- Generated IaC code artifacts
CREATE TABLE artifacts (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    session_id      UUID NOT NULL REFERENCES generation_sessions(id) ON DELETE CASCADE,
    prompt_id       UUID NOT NULL REFERENCES prompts(id),
    version         INTEGER NOT NULL DEFAULT 1,
    output_format   TEXT NOT NULL CHECK (output_format IN ('hcl', 'pulumi_ts', 'pulumi_py', 'cloudformation', 'helm', 'kubernetes')),
    content         TEXT NOT NULL,           -- the generated IaC code
    content_hash    TEXT NOT NULL,           -- SHA-256 for deduplication
    file_path       TEXT,                    -- suggested file path (e.g., 'modules/vpc/main.tf')
    is_module       BOOLEAN NOT NULL DEFAULT false,
    module_name     TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_artifacts_session ON artifacts(session_id);
CREATE INDEX idx_artifacts_hash ON artifacts(content_hash);

-- Individual resources within an artifact
CREATE TABLE artifact_resources (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    artifact_id     UUID NOT NULL REFERENCES artifacts(id) ON DELETE CASCADE,
    resource_type   TEXT NOT NULL,           -- e.g., 'aws_vpc', 'azurerm_resource_group'
    resource_name   TEXT NOT NULL,           -- e.g., 'main', 'app_subnet'
    provider        TEXT NOT NULL,           -- e.g., 'aws', 'azurerm', 'google'
    address         TEXT NOT NULL,           -- e.g., 'module.vpc.aws_vpc.main'
    estimated_monthly_cost_usd NUMERIC(12,2),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_artifact_resources_artifact ON artifact_resources(artifact_id);
CREATE INDEX idx_artifact_resources_type ON artifact_resources(resource_type);

-- Artifact dependencies (module references, data source lookups)
CREATE TABLE artifact_dependencies (
    artifact_id     UUID NOT NULL REFERENCES artifacts(id) ON DELETE CASCADE,
    depends_on_id   UUID NOT NULL REFERENCES artifacts(id) ON DELETE CASCADE,
    dependency_type TEXT NOT NULL CHECK (dependency_type IN ('module', 'data_source', 'output_ref')),
    PRIMARY KEY (artifact_id, depends_on_id)
);
```

## Cloud Provider Schema Cache

```sql
-- Cloud providers supported
CREATE TABLE cloud_providers (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            TEXT NOT NULL UNIQUE,     -- 'aws', 'azurerm', 'google', 'kubernetes'
    display_name    TEXT NOT NULL,
    registry_url    TEXT,                     -- e.g., 'https://registry.opentofu.org'
    latest_version  TEXT,
    schema_updated_at TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Cached provider resource type schemas (from `tofu providers schema -json`)
CREATE TABLE provider_resource_types (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    provider_id     UUID NOT NULL REFERENCES cloud_providers(id) ON DELETE CASCADE,
    resource_type   TEXT NOT NULL,            -- e.g., 'aws_instance'
    mode            TEXT NOT NULL DEFAULT 'managed' CHECK (mode IN ('managed', 'data')),
    schema_version  INTEGER NOT NULL DEFAULT 0,
    attributes      JSONB NOT NULL,           -- attribute definitions from provider schema
    block_types     JSONB NOT NULL DEFAULT '{}', -- nested block definitions
    description     TEXT,
    deprecated      BOOLEAN NOT NULL DEFAULT false,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (provider_id, resource_type, mode)
);

CREATE INDEX idx_provider_resources_provider ON provider_resource_types(provider_id);
CREATE INDEX idx_provider_resources_type ON provider_resource_types(resource_type);

-- Cloud regions reference table (ISO 3166 aligned)
CREATE TABLE cloud_regions (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    provider_id     UUID NOT NULL REFERENCES cloud_providers(id),
    region_code     TEXT NOT NULL,            -- e.g., 'us-east-1', 'eastus', 'us-central1'
    display_name    TEXT NOT NULL,
    country_code    CHAR(2) NOT NULL,         -- ISO 3166-1 alpha-2
    continent       TEXT NOT NULL,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (provider_id, region_code)
);
```

## Security & Policy

```sql
-- Security policy sets (OPA/Rego rules)
CREATE TABLE security_policies (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id          UUID NOT NULL REFERENCES organisations(id) ON DELETE CASCADE,
    name            TEXT NOT NULL,
    description     TEXT,
    policy_language TEXT NOT NULL DEFAULT 'rego' CHECK (policy_language IN ('rego', 'sentinel', 'checkov')),
    policy_content  TEXT NOT NULL,            -- the Rego/Sentinel policy code
    severity        TEXT NOT NULL DEFAULT 'medium' CHECK (severity IN ('critical', 'high', 'medium', 'low', 'info')),
    enabled         BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (org_id, name)
);

CREATE INDEX idx_security_policies_org ON security_policies(org_id);

-- Security rules mapped to CIS benchmarks
CREATE TABLE security_rules (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    policy_id       UUID NOT NULL REFERENCES security_policies(id) ON DELETE CASCADE,
    rule_id         TEXT NOT NULL,            -- e.g., 'CIS-AWS-1.1', 'CIS-AZURE-2.3'
    benchmark       TEXT NOT NULL,            -- e.g., 'CIS AWS Foundations 3.0'
    control_id      TEXT,                     -- NIST SP 800-53 control (e.g., 'AC-2')
    description     TEXT NOT NULL,
    resource_types  TEXT[] NOT NULL,          -- which resource types this rule applies to
    auto_remediate  BOOLEAN NOT NULL DEFAULT false,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_security_rules_policy ON security_rules(policy_id);
CREATE INDEX idx_security_rules_resource_types ON security_rules USING GIN (resource_types);

-- Security scan results per artifact
CREATE TABLE security_scan_results (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    artifact_id     UUID NOT NULL REFERENCES artifacts(id) ON DELETE CASCADE,
    rule_id         UUID NOT NULL REFERENCES security_rules(id),
    status          TEXT NOT NULL CHECK (status IN ('passed', 'failed', 'skipped', 'error')),
    resource_address TEXT NOT NULL,           -- which resource in the artifact failed
    message         TEXT,
    remediation     TEXT,                     -- suggested fix
    scanned_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_scan_results_artifact ON security_scan_results(artifact_id);
CREATE INDEX idx_scan_results_status ON security_scan_results(status) WHERE status = 'failed';

-- Compliance control mappings (OSCAL-aligned)
CREATE TABLE compliance_controls (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    framework       TEXT NOT NULL,            -- 'NIST SP 800-53', 'SOC 2', 'FedRAMP', 'ISO 27001'
    control_family  TEXT NOT NULL,            -- e.g., 'AC' (Access Control)
    control_id      TEXT NOT NULL,            -- e.g., 'AC-2(1)'
    title           TEXT NOT NULL,
    description     TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (framework, control_id)
);

CREATE TABLE compliance_resource_mappings (
    control_id      UUID NOT NULL REFERENCES compliance_controls(id),
    resource_type   TEXT NOT NULL,
    required_attributes JSONB NOT NULL,       -- attributes the resource must have to comply
    PRIMARY KEY (control_id, resource_type)
);
```

## Cost Estimation

```sql
-- Cost estimates attached to artifacts
CREATE TABLE cost_estimates (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    artifact_id     UUID NOT NULL REFERENCES artifacts(id) ON DELETE CASCADE,
    total_monthly_usd NUMERIC(12,2) NOT NULL,
    total_hourly_usd  NUMERIC(12,4),
    currency        CHAR(3) NOT NULL DEFAULT 'USD',
    pricing_source  TEXT NOT NULL DEFAULT 'infracost',
    estimated_at    TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_cost_estimates_artifact ON cost_estimates(artifact_id);

-- Per-resource cost breakdown
CREATE TABLE cost_line_items (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    estimate_id     UUID NOT NULL REFERENCES cost_estimates(id) ON DELETE CASCADE,
    resource_id     UUID NOT NULL REFERENCES artifact_resources(id),
    component       TEXT NOT NULL,            -- e.g., 'compute', 'storage', 'data_transfer'
    monthly_cost_usd NUMERIC(12,2) NOT NULL,
    unit            TEXT,                     -- e.g., 'hours', 'GB', 'requests'
    unit_price      NUMERIC(12,6),
    quantity        NUMERIC(12,2),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_cost_line_items_estimate ON cost_line_items(estimate_id);
```

## Deployment & State Tracking

```sql
-- Deployment plans (terraform plan equivalent)
CREATE TABLE deployment_plans (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    artifact_id     UUID NOT NULL REFERENCES artifacts(id),
    workspace_id    UUID NOT NULL REFERENCES workspaces(id),
    user_id         UUID NOT NULL REFERENCES users(id),
    status          TEXT NOT NULL DEFAULT 'pending' CHECK (status IN ('pending', 'planning', 'planned', 'approved', 'rejected', 'applying', 'applied', 'failed', 'cancelled')),
    plan_json       JSONB,                   -- terraform show -json output
    resources_to_add    INTEGER DEFAULT 0,
    resources_to_change INTEGER DEFAULT 0,
    resources_to_destroy INTEGER DEFAULT 0,
    approved_by     UUID REFERENCES users(id),
    approved_at     TIMESTAMPTZ,
    started_at      TIMESTAMPTZ,
    completed_at    TIMESTAMPTZ,
    error_message   TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_deployment_plans_artifact ON deployment_plans(artifact_id);
CREATE INDEX idx_deployment_plans_workspace ON deployment_plans(workspace_id);
CREATE INDEX idx_deployment_plans_status ON deployment_plans(status);

-- Provider credentials (encrypted at rest)
CREATE TABLE provider_credentials (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id          UUID NOT NULL REFERENCES organisations(id) ON DELETE CASCADE,
    provider_id     UUID NOT NULL REFERENCES cloud_providers(id),
    name            TEXT NOT NULL,
    credential_type TEXT NOT NULL CHECK (credential_type IN ('oidc', 'service_account', 'access_key', 'assume_role')),
    encrypted_config BYTEA NOT NULL,         -- AES-256-GCM encrypted credential blob
    expires_at      TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (org_id, name)
);

CREATE INDEX idx_provider_creds_org ON provider_credentials(org_id);
```

## Module Registry

```sql
-- Reusable module recommendations
CREATE TABLE module_registry (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    source          TEXT NOT NULL,            -- e.g., 'registry.opentofu.org/hashicorp/vpc/aws'
    name            TEXT NOT NULL,
    provider        TEXT NOT NULL,
    version         TEXT NOT NULL,
    description     TEXT,
    download_count  INTEGER DEFAULT 0,
    verified        BOOLEAN NOT NULL DEFAULT false,
    inputs          JSONB NOT NULL DEFAULT '[]',
    outputs         JSONB NOT NULL DEFAULT '[]',
    last_synced_at  TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (source, version)
);

CREATE INDEX idx_module_registry_provider ON module_registry(provider);
CREATE INDEX idx_module_registry_name ON module_registry(name);
```

## MCP Session Tracking

```sql
-- MCP (Model Context Protocol) sessions for IDE integration
CREATE TABLE mcp_sessions (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id         UUID NOT NULL REFERENCES users(id),
    session_id      UUID NOT NULL REFERENCES generation_sessions(id),
    client_name     TEXT NOT NULL,            -- 'claude_code', 'cursor', 'copilot'
    transport       TEXT NOT NULL DEFAULT 'streamable_http' CHECK (transport IN ('stdio', 'streamable_http')),
    connected_at    TIMESTAMPTZ NOT NULL DEFAULT now(),
    disconnected_at TIMESTAMPTZ,
    last_activity_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_mcp_sessions_user ON mcp_sessions(user_id);
```

## Audit Log

```sql
-- Immutable audit trail
CREATE TABLE audit_log (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id          UUID NOT NULL REFERENCES organisations(id),
    user_id         UUID REFERENCES users(id),
    action          TEXT NOT NULL,            -- e.g., 'artifact.created', 'plan.approved', 'credential.rotated'
    entity_type     TEXT NOT NULL,            -- e.g., 'artifact', 'deployment_plan', 'security_policy'
    entity_id       UUID NOT NULL,
    metadata        JSONB NOT NULL DEFAULT '{}',
    ip_address      INET,
    user_agent      TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_audit_log_org ON audit_log(org_id, created_at DESC);
CREATE INDEX idx_audit_log_entity ON audit_log(entity_type, entity_id);
CREATE INDEX idx_audit_log_action ON audit_log(action);

-- Partition audit_log by month for performance
-- ALTER TABLE audit_log PARTITION BY RANGE (created_at);
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Identity & Multi-Tenancy | 5 | organisations, users, org_memberships, teams, team_memberships |
| Projects & Workspaces | 2 | projects, workspaces |
| Generation Sessions | 2 | generation_sessions, prompts |
| Generated Artifacts | 3 | artifacts, artifact_resources, artifact_dependencies |
| Cloud Provider Schema | 3 | cloud_providers, provider_resource_types, cloud_regions |
| Security & Policy | 5 | security_policies, security_rules, security_scan_results, compliance_controls, compliance_resource_mappings |
| Cost Estimation | 2 | cost_estimates, cost_line_items |
| Deployment & Credentials | 2 | deployment_plans, provider_credentials |
| Module Registry | 1 | module_registry |
| MCP Integration | 1 | mcp_sessions |
| Audit | 1 | audit_log |
| **Total** | **27** | |

---

## Key Design Decisions

1. **UUID primary keys throughout** — enables distributed ID generation without coordination, essential for a multi-region SaaS deployment where CLI clients may generate IDs offline.

2. **Explicit foreign keys with ON DELETE CASCADE** — ensures that deleting an organisation cascades cleanly through projects, sessions, artifacts, and all child records without orphaned data.

3. **Provider schema cache as relational tables** — rather than storing raw JSON blobs, the `provider_resource_types` table with indexed `resource_type` allows the generator to validate attribute names via SQL queries before returning generated code, eliminating hallucinated provider attributes.

4. **Security rules mapped to CIS controls** — the `security_rules` table directly links OPA/Rego policy rules to CIS benchmark control IDs and NIST SP 800-53 controls, enabling compliance reporting without a separate compliance system.

5. **Separate audit_log table** — append-only, partitionable by month, with no UPDATE or DELETE operations permitted. This satisfies SOC 2 and ISO 27001 audit trail requirements.

6. **Encrypted credential storage** — `provider_credentials.encrypted_config` stores AES-256-GCM encrypted blobs rather than plaintext, with key management delegated to the application layer (AWS KMS, Azure Key Vault, or GCP KMS).

7. **Artifact versioning within sessions** — the `artifacts.version` column tracks iterative refinement within a conversational session, so all versions of generated code are preserved for audit and comparison.

8. **Cost estimation as a first-class entity** — rather than embedding cost data in artifacts, separate `cost_estimates` and `cost_line_items` tables allow re-estimation without regenerating code, and enable cost trend reporting across an organisation.

9. **Module registry for recommendation** — the `module_registry` table caches metadata from the OpenTofu registry, enabling the generator to recommend existing modules before generating from scratch, reducing code duplication.

10. **GIN indexes on array columns** — `security_rules.resource_types` uses a GIN index for efficient containment queries (`WHERE resource_types @> ARRAY['aws_s3_bucket']`), enabling fast lookup of applicable rules during security scanning.
