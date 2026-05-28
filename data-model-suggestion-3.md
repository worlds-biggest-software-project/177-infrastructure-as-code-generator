# Data Model Suggestion 3: Hybrid Relational + JSONB

> Project: Infrastructure as Code Generator · Created: 2026-05-20

## Philosophy

This model uses a pragmatic hybrid approach: core structural relationships (users, organisations, projects, sessions) are modeled as traditional relational tables with foreign keys and constraints, while variable, provider-specific, and rapidly evolving data (resource attributes, provider schemas, security rule definitions, cost breakdowns, deployment plan details) is stored in JSONB columns within those tables. The result is a compact schema with fewer tables that still supports efficient querying via PostgreSQL's JSONB operators and GIN indexes.

This approach reflects how real-world IaC platforms handle the inherent variability of cloud infrastructure. An AWS VPC has different attributes than an Azure Virtual Network, and both differ from a GCP VPC Network. Rather than creating separate tables or columns for each provider's resource attributes, the hybrid model stores these as structured JSONB, letting the application layer validate against cached provider schemas. Pulumi's own state checkpoint format uses this exact pattern — a JSON document with typed nested structures whose shape varies by provider and resource type.

The hybrid model is particularly well-suited to an IaC generator because the generator must handle an open-ended set of resource types (3,900+ in the OpenTofu registry) where the attribute schemas are defined externally by providers and change with every provider version. A fully normalized schema would require constant migrations; a fully document-oriented schema would lose referential integrity. The hybrid approach threads this needle.

**Best for:** Rapid MVP development where schema flexibility is needed, multi-cloud platforms where resource attributes vary widely by provider, and teams that want to iterate on the data model without frequent migrations.

**Trade-offs:**
- (+) Far fewer tables (~15) — faster to build, easier to understand
- (+) JSONB columns absorb provider variability without schema changes
- (+) PostgreSQL JSONB operators and GIN indexes provide efficient querying
- (+) Adding new providers or resource types requires zero schema migrations
- (+) Natural fit for storing provider schemas, plan outputs, and cost breakdowns
- (-) No referential integrity within JSONB — application must validate
- (-) JSONB query syntax is less readable than relational JOINs
- (-) Harder to enforce data contracts — bugs can introduce malformed JSON
- (-) JSONB columns can grow large; need careful monitoring of row sizes
- (-) Reporting queries across JSONB fields are slower than indexed relational columns

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| HCL Specification | Generated code stored as text content; HCL version tracked in artifact JSONB metadata |
| OpenTofu Provider Protocol | Full `tofu providers schema -json` output cached as JSONB in provider_schemas |
| JSON Schema (Draft 2020-12) | JSONB columns validated against JSON Schema definitions at the application layer |
| OpenAPI 3.x | Provider schema ingestion from OpenAPI specs stored natively as JSONB |
| OPA / Rego | Policy definitions stored as text; scan results stored as JSONB arrays |
| CIS Benchmarks | CIS control mappings embedded in security scan result JSONB payloads |
| Terraform JSON format | Plan output (`terraform show -json`) stored directly as JSONB in deployment records |
| ISO 3166 | Region metadata within provider JSONB uses ISO 3166 country codes |

---

## Core Identity & Multi-Tenancy

```sql
CREATE TABLE organisations (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            TEXT NOT NULL,
    slug            TEXT NOT NULL UNIQUE,
    plan_tier       TEXT NOT NULL DEFAULT 'free' CHECK (plan_tier IN ('free', 'team', 'enterprise')),
    settings        JSONB NOT NULL DEFAULT '{}',
    -- settings example:
    -- {
    --   "default_provider": "aws",
    --   "default_region": "us-east-1",
    --   "security_scan_on_generate": true,
    --   "cost_estimate_on_generate": true,
    --   "allowed_providers": ["aws", "azurerm", "google"],
    --   "max_monthly_cost_usd": 10000
    -- }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE users (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    email           TEXT NOT NULL UNIQUE,
    display_name    TEXT NOT NULL,
    auth_provider   TEXT NOT NULL,
    auth_subject    TEXT NOT NULL,
    preferences     JSONB NOT NULL DEFAULT '{}',
    -- preferences example:
    -- {
    --   "default_output_format": "hcl",
    --   "theme": "dark",
    --   "editor_tab_size": 2,
    --   "auto_scan": true
    -- }
    last_login_at   TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (auth_provider, auth_subject)
);

CREATE TABLE org_memberships (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id          UUID NOT NULL REFERENCES organisations(id) ON DELETE CASCADE,
    user_id         UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    role            TEXT NOT NULL DEFAULT 'member' CHECK (role IN ('owner', 'admin', 'member', 'viewer')),
    permissions     JSONB NOT NULL DEFAULT '{}',
    -- permissions example:
    -- {
    --   "can_deploy": true,
    --   "can_manage_credentials": false,
    --   "can_approve_plans": true,
    --   "allowed_projects": ["*"]
    -- }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (org_id, user_id)
);

CREATE INDEX idx_org_memberships_user ON org_memberships(user_id);
```

## Projects

```sql
CREATE TABLE projects (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id          UUID NOT NULL REFERENCES organisations(id) ON DELETE CASCADE,
    name            TEXT NOT NULL,
    slug            TEXT NOT NULL,
    description     TEXT,
    config          JSONB NOT NULL DEFAULT '{}',
    -- config example:
    -- {
    --   "default_provider": "aws",
    --   "default_region": "us-east-1",
    --   "output_format": "hcl",
    --   "vcs": {
    --     "repo_url": "https://github.com/org/infra",
    --     "branch": "main",
    --     "base_path": "terraform/"
    --   },
    --   "workspaces": {
    --     "dev":  {"provider": "aws", "region": "us-east-1", "auto_apply": false},
    --     "staging": {"provider": "aws", "region": "us-east-1", "auto_apply": false},
    --     "prod": {"provider": "aws", "region": "us-east-1", "auto_apply": false}
    --   },
    --   "state_backend": {
    --     "type": "s3",
    --     "bucket": "org-terraform-state",
    --     "key_prefix": "project-name/"
    --   }
    -- }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (org_id, slug)
);

CREATE INDEX idx_projects_org ON projects(org_id);
```

## Generation Sessions & Conversation

```sql
CREATE TABLE generation_sessions (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    project_id      UUID NOT NULL REFERENCES projects(id) ON DELETE CASCADE,
    org_id          UUID NOT NULL REFERENCES organisations(id),
    user_id         UUID NOT NULL REFERENCES users(id),
    title           TEXT,
    status          TEXT NOT NULL DEFAULT 'active' CHECK (status IN ('active', 'completed', 'abandoned')),
    source          TEXT NOT NULL DEFAULT 'cli' CHECK (source IN ('cli', 'web', 'mcp', 'api')),
    context         JSONB NOT NULL DEFAULT '{}',
    -- context example:
    -- {
    --   "workspace": "dev",
    --   "model_id": "claude-opus-4-6",
    --   "provider": "aws",
    --   "region": "us-east-1",
    --   "existing_state_summary": {
    --     "resource_count": 42,
    --     "resource_types": ["aws_vpc", "aws_subnet", "aws_instance"]
    --   },
    --   "mcp_client": "claude_code",
    --   "mcp_transport": "streamable_http"
    -- }
    usage           JSONB NOT NULL DEFAULT '{}',
    -- usage example:
    -- {
    --   "total_tokens": 15420,
    --   "prompt_tokens": 8200,
    --   "completion_tokens": 7220,
    --   "total_latency_ms": 34500,
    --   "prompt_count": 5,
    --   "artifact_count": 3
    -- }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_sessions_project ON generation_sessions(project_id);
CREATE INDEX idx_sessions_user ON generation_sessions(user_id);
CREATE INDEX idx_sessions_org ON generation_sessions(org_id, created_at DESC);

-- Conversation messages (prompts and responses)
CREATE TABLE messages (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    session_id      UUID NOT NULL REFERENCES generation_sessions(id) ON DELETE CASCADE,
    sequence_num    INTEGER NOT NULL,
    role            TEXT NOT NULL CHECK (role IN ('user', 'assistant', 'system')),
    content         TEXT NOT NULL,
    metadata        JSONB NOT NULL DEFAULT '{}',
    -- metadata example (for assistant messages):
    -- {
    --   "tokens_used": 3200,
    --   "latency_ms": 8500,
    --   "model_id": "claude-opus-4-6",
    --   "finish_reason": "end_turn",
    --   "artifacts_generated": ["uuid1", "uuid2"]
    -- }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (session_id, sequence_num)
);

CREATE INDEX idx_messages_session ON messages(session_id, sequence_num);
```

## Generated Artifacts

```sql
CREATE TABLE artifacts (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    session_id      UUID NOT NULL REFERENCES generation_sessions(id) ON DELETE CASCADE,
    org_id          UUID NOT NULL REFERENCES organisations(id),
    message_id      UUID NOT NULL REFERENCES messages(id),
    version         INTEGER NOT NULL DEFAULT 1,
    output_format   TEXT NOT NULL CHECK (output_format IN ('hcl', 'pulumi_ts', 'pulumi_py', 'cloudformation', 'helm', 'kubernetes')),
    content         TEXT NOT NULL,
    content_hash    TEXT NOT NULL,
    file_path       TEXT,
    resources       JSONB NOT NULL DEFAULT '[]',
    -- resources example:
    -- [
    --   {
    --     "type": "aws_vpc",
    --     "name": "main",
    --     "provider": "aws",
    --     "address": "module.networking.aws_vpc.main",
    --     "is_module": false,
    --     "attributes_summary": {"cidr_block": "10.0.0.0/16", "enable_dns_support": true}
    --   },
    --   {
    --     "type": "aws_subnet",
    --     "name": "public",
    --     "provider": "aws",
    --     "address": "module.networking.aws_subnet.public",
    --     "is_module": false,
    --     "attributes_summary": {"cidr_block": "10.0.1.0/24", "map_public_ip_on_launch": true}
    --   }
    -- ]
    dependencies    JSONB NOT NULL DEFAULT '[]',
    -- dependencies example:
    -- [
    --   {"artifact_id": "uuid", "type": "module", "name": "networking"},
    --   {"artifact_id": "uuid", "type": "output_ref", "output": "vpc_id"}
    -- ]
    security_scan   JSONB,
    -- security_scan example:
    -- {
    --   "status": "completed",
    --   "scanned_at": "2026-05-20T10:30:00Z",
    --   "scanner": "checkov",
    --   "summary": {"passed": 12, "failed": 2, "skipped": 1},
    --   "results": [
    --     {
    --       "rule_id": "CIS-AWS-2.1.1",
    --       "benchmark": "CIS AWS Foundations 3.0",
    --       "nist_control": "SC-28",
    --       "severity": "high",
    --       "status": "failed",
    --       "resource_address": "aws_db_instance.primary",
    --       "message": "RDS encryption at rest not enabled",
    --       "remediation": "Set storage_encrypted = true"
    --     },
    --     {
    --       "rule_id": "CIS-AWS-4.1",
    --       "benchmark": "CIS AWS Foundations 3.0",
    --       "nist_control": "AC-3",
    --       "severity": "medium",
    --       "status": "failed",
    --       "resource_address": "aws_security_group.app",
    --       "message": "Security group allows unrestricted ingress on port 22",
    --       "remediation": "Restrict SSH access to known CIDR ranges"
    --     }
    --   ]
    -- }
    cost_estimate   JSONB,
    -- cost_estimate example:
    -- {
    --   "total_monthly_usd": 342.50,
    --   "total_hourly_usd": 0.47,
    --   "currency": "USD",
    --   "pricing_source": "infracost",
    --   "estimated_at": "2026-05-20T10:31:00Z",
    --   "line_items": [
    --     {"resource": "aws_db_instance.primary", "component": "compute", "monthly_usd": 185.00, "unit": "hours", "unit_price": 0.253},
    --     {"resource": "aws_db_instance.primary", "component": "storage", "monthly_usd": 23.00, "unit": "GB-month", "unit_price": 0.115},
    --     {"resource": "aws_db_instance.replica", "component": "compute", "monthly_usd": 134.50, "unit": "hours", "unit_price": 0.184}
    --   ]
    -- }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_artifacts_session ON artifacts(session_id);
CREATE INDEX idx_artifacts_org ON artifacts(org_id, created_at DESC);
CREATE INDEX idx_artifacts_hash ON artifacts(content_hash);
CREATE INDEX idx_artifacts_format ON artifacts(output_format);

-- GIN index on resources for querying by resource type
CREATE INDEX idx_artifacts_resources ON artifacts USING GIN (resources jsonb_path_ops);

-- GIN index on security_scan for querying failed scans
CREATE INDEX idx_artifacts_security ON artifacts USING GIN (security_scan jsonb_path_ops);

-- Example query: find all artifacts with failed security scans
-- SELECT id, content, security_scan->'summary'
-- FROM artifacts
-- WHERE security_scan @> '{"status": "completed"}'
--   AND (security_scan->'summary'->>'failed')::int > 0;

-- Example query: find all artifacts containing aws_rds_instance resources
-- SELECT id, content
-- FROM artifacts
-- WHERE resources @> '[{"type": "aws_db_instance"}]';
```

## Provider Schema Cache

```sql
-- Provider schemas cached from `tofu providers schema -json`
CREATE TABLE provider_schemas (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    provider_name   TEXT NOT NULL,           -- 'aws', 'azurerm', 'google'
    provider_version TEXT NOT NULL,
    display_name    TEXT NOT NULL,
    registry_url    TEXT,
    schema          JSONB NOT NULL,
    -- schema stores the full provider schema JSON from `tofu providers schema -json`
    -- This can be large (10-50MB for major providers), so it's stored as a single
    -- JSONB document and queried via jsonb_path_query for specific resource types.
    --
    -- Query example: get schema for aws_instance
    -- SELECT schema->'resource_schemas'->'aws_instance'
    -- FROM provider_schemas
    -- WHERE provider_name = 'aws'
    --   AND provider_version = '5.80.0';
    regions         JSONB NOT NULL DEFAULT '[]',
    -- regions example:
    -- [
    --   {"code": "us-east-1", "name": "US East (N. Virginia)", "country": "US"},
    --   {"code": "eu-west-1", "name": "Europe (Ireland)", "country": "IE"}
    -- ]
    synced_at       TIMESTAMPTZ NOT NULL DEFAULT now(),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (provider_name, provider_version)
);

CREATE INDEX idx_provider_schemas_name ON provider_schemas(provider_name);
```

## Security Policies

```sql
CREATE TABLE security_policies (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id          UUID NOT NULL REFERENCES organisations(id) ON DELETE CASCADE,
    name            TEXT NOT NULL,
    description     TEXT,
    policy_language TEXT NOT NULL DEFAULT 'rego' CHECK (policy_language IN ('rego', 'sentinel', 'checkov')),
    policy_content  TEXT NOT NULL,
    config          JSONB NOT NULL DEFAULT '{}',
    -- config example:
    -- {
    --   "severity": "high",
    --   "enabled": true,
    --   "auto_remediate": false,
    --   "applicable_resource_types": ["aws_s3_bucket", "aws_rds_instance"],
    --   "cis_controls": ["CIS-AWS-2.1.1", "CIS-AWS-2.1.2"],
    --   "nist_controls": ["SC-28", "SC-13"],
    --   "compliance_frameworks": ["SOC 2", "ISO 27001"],
    --   "exclusions": {
    --     "resource_name_patterns": ["*-dev-*", "*-test-*"]
    --   }
    -- }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (org_id, name)
);

CREATE INDEX idx_security_policies_org ON security_policies(org_id);
CREATE INDEX idx_security_policies_config ON security_policies USING GIN (config jsonb_path_ops);
```

## Deployments

```sql
CREATE TABLE deployments (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    artifact_id     UUID NOT NULL REFERENCES artifacts(id),
    org_id          UUID NOT NULL REFERENCES organisations(id),
    user_id         UUID NOT NULL REFERENCES users(id),
    project_id      UUID NOT NULL REFERENCES projects(id),
    workspace       TEXT NOT NULL,            -- workspace name from project config
    status          TEXT NOT NULL DEFAULT 'pending' CHECK (status IN ('pending', 'planning', 'planned', 'approved', 'rejected', 'applying', 'applied', 'failed', 'cancelled')),
    plan_output     JSONB,
    -- plan_output stores the `terraform show -json` output directly
    -- {
    --   "format_version": "1.0",
    --   "resource_changes": [
    --     {
    --       "address": "aws_vpc.main",
    --       "mode": "managed",
    --       "type": "aws_vpc",
    --       "change": {"actions": ["create"], "before": null, "after": {"cidr_block": "10.0.0.0/16"}}
    --     }
    --   ],
    --   "resource_drift": []
    -- }
    summary         JSONB NOT NULL DEFAULT '{}',
    -- summary example:
    -- {
    --   "resources_to_add": 5,
    --   "resources_to_change": 2,
    --   "resources_to_destroy": 0,
    --   "estimated_cost_change_usd": 42.50
    -- }
    approval        JSONB,
    -- approval example:
    -- {
    --   "approved_by": "uuid",
    --   "approved_at": "2026-05-20T11:00:00Z",
    --   "comment": "LGTM, approved for staging"
    -- }
    timing          JSONB NOT NULL DEFAULT '{}',
    -- timing example:
    -- {
    --   "plan_started_at": "2026-05-20T10:55:00Z",
    --   "plan_completed_at": "2026-05-20T10:55:45Z",
    --   "apply_started_at": "2026-05-20T11:01:00Z",
    --   "apply_completed_at": "2026-05-20T11:03:22Z"
    -- }
    error_message   TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_deployments_artifact ON deployments(artifact_id);
CREATE INDEX idx_deployments_org ON deployments(org_id, created_at DESC);
CREATE INDEX idx_deployments_project ON deployments(project_id);
CREATE INDEX idx_deployments_status ON deployments(status);
```

## Provider Credentials

```sql
CREATE TABLE provider_credentials (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id          UUID NOT NULL REFERENCES organisations(id) ON DELETE CASCADE,
    name            TEXT NOT NULL,
    provider_name   TEXT NOT NULL,
    credential_type TEXT NOT NULL CHECK (credential_type IN ('oidc', 'service_account', 'access_key', 'assume_role')),
    encrypted_config BYTEA NOT NULL,
    config_metadata JSONB NOT NULL DEFAULT '{}',
    -- config_metadata (non-sensitive fields only):
    -- {
    --   "account_id": "123456789012",
    --   "region": "us-east-1",
    --   "role_arn": "arn:aws:iam::123456789012:role/terraform",
    --   "expires_at": "2026-12-31T23:59:59Z"
    -- }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (org_id, name)
);

CREATE INDEX idx_provider_creds_org ON provider_credentials(org_id);
```

## Module Registry Cache

```sql
CREATE TABLE module_cache (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    source          TEXT NOT NULL,
    name            TEXT NOT NULL,
    provider_name   TEXT NOT NULL,
    version         TEXT NOT NULL,
    metadata        JSONB NOT NULL DEFAULT '{}',
    -- metadata example:
    -- {
    --   "description": "Terraform module for AWS VPC",
    --   "download_count": 1250000,
    --   "verified": true,
    --   "inputs": [
    --     {"name": "cidr_block", "type": "string", "required": true, "default": null},
    --     {"name": "enable_dns_support", "type": "bool", "required": false, "default": true}
    --   ],
    --   "outputs": [
    --     {"name": "vpc_id", "type": "string"},
    --     {"name": "public_subnet_ids", "type": "list(string)"}
    --   ],
    --   "dependencies": [],
    --   "examples": ["examples/simple-vpc", "examples/complete-vpc"]
    -- }
    synced_at       TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (source, version)
);

CREATE INDEX idx_module_cache_provider ON module_cache(provider_name);
CREATE INDEX idx_module_cache_name ON module_cache(name);
```

## Audit Log

```sql
CREATE TABLE audit_log (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id          UUID NOT NULL,
    user_id         UUID,
    action          TEXT NOT NULL,
    entity_type     TEXT NOT NULL,
    entity_id       UUID NOT NULL,
    details         JSONB NOT NULL DEFAULT '{}',
    -- details example:
    -- {
    --   "ip_address": "203.0.113.42",
    --   "user_agent": "claude-code/1.0",
    --   "changes": {"status": {"from": "planned", "to": "approved"}},
    --   "reason": "Approved after security review"
    -- }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_audit_log_org ON audit_log(org_id, created_at DESC);
CREATE INDEX idx_audit_log_entity ON audit_log(entity_type, entity_id);
CREATE INDEX idx_audit_log_action ON audit_log(action, created_at DESC);
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Identity & Multi-Tenancy | 3 | organisations, users, org_memberships |
| Projects | 1 | projects (workspaces embedded in config JSONB) |
| Generation Sessions | 2 | generation_sessions, messages |
| Generated Artifacts | 1 | artifacts (resources, security_scan, cost_estimate all JSONB) |
| Provider Schema | 1 | provider_schemas (full schema as JSONB) |
| Security Policies | 1 | security_policies (rules, controls in config JSONB) |
| Deployments | 1 | deployments (plan output, approval, timing all JSONB) |
| Credentials | 1 | provider_credentials |
| Module Registry | 1 | module_cache (inputs/outputs as JSONB) |
| Audit | 1 | audit_log |
| **Total** | **13** | |

---

## Key Design Decisions

1. **Workspaces as JSONB within projects** — rather than a separate `workspaces` table, workspace definitions live inside `projects.config`. Workspaces are always accessed in the context of a project, and their structure varies by provider (AWS workspaces need region, Azure needs resource group + location, GCP needs project ID + region). JSONB accommodates this without a sparse relational table.

2. **Security scan results embedded in artifacts** — each artifact carries its own `security_scan` JSONB column containing the full scan result. This eliminates the need for separate `security_scan_results` and `security_rules` tables while keeping scan results co-located with the code they evaluate. GIN indexing on `security_scan` enables efficient filtering (e.g., "show all artifacts with failed CIS checks").

3. **Cost estimates embedded in artifacts** — like security scans, cost estimates are stored directly on the artifact rather than in separate tables. This simplifies queries ("show me the artifact with its cost") to a single-table read instead of a JOIN.

4. **Full provider schema as JSONB** — the `provider_schemas.schema` column stores the entire output of `tofu providers schema -json` as a single JSONB document. For major providers (AWS: ~30MB), this is large but well within PostgreSQL's 1GB JSONB limit. Queries for specific resource type schemas use JSONB path navigation (`schema->'resource_schemas'->'aws_instance'`).

5. **Deployment plan output stored verbatim** — `deployments.plan_output` stores the raw `terraform show -json` output as JSONB. This preserves the full plan detail for audit purposes and avoids remodeling Terraform's plan format into relational tables that would break with every Terraform version.

6. **Permissions as JSONB on org_memberships** — rather than a separate RBAC tables (roles, permissions, role_permissions), permissions are stored as a JSONB object on the membership record. This keeps the schema compact and allows per-org customisation of permission structures without schema changes.

7. **Artifact resources as JSONB array** — the `artifacts.resources` column contains a JSON array of resources extracted from the generated code. This avoids a separate `artifact_resources` junction table while still supporting GIN-indexed queries like `resources @> '[{"type": "aws_vpc"}]'`.

8. **Provider-agnostic credential storage** — `provider_credentials` uses `encrypted_config` (BYTEA) for sensitive data and `config_metadata` (JSONB) for non-sensitive fields like account IDs and role ARNs. The JSONB structure varies by provider without requiring provider-specific columns.

9. **Audit log with JSONB details** — the audit log uses a `details` JSONB column for action-specific context (IP address, changes, reasons) rather than separate columns for each possible audit field. This keeps the audit table generic and extensible.

10. **13 tables total** — compared to 27 tables in the normalized model, this hybrid approach consolidates related data into fewer tables with JSONB columns. The trade-off is that data contracts must be enforced at the application layer (via JSON Schema validation) rather than via database constraints.
