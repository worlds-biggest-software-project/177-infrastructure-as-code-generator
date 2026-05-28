# Data Model Suggestion 2: Event-Sourced / Audit-First (CQRS)

> Project: Infrastructure as Code Generator · Created: 2026-05-20

## Philosophy

This model treats every action in the system — a prompt submitted, code generated, a security scan completed, a plan approved, a deployment applied — as an immutable event appended to a central event store. The event stream is the single source of truth. Read-optimised materialised views (projections) are derived from the event stream for the various query patterns the application needs (session history, artifact browsing, compliance dashboards, cost reports).

This approach is inspired by how cloud providers themselves model infrastructure changes. AWS CloudTrail, Azure Activity Log, and GCP Cloud Audit Logs all use append-only event streams as the authoritative record of what happened. Terraform's own state model — where every `apply` creates a new state version — is conceptually event-sourced. By adopting this pattern at the application level, the IaC generator gains native support for temporal queries ("what was the generated code at 3pm yesterday?"), complete audit trails without a separate audit system, and the ability to replay events to rebuild any projection.

The CQRS (Command Query Responsibility Segregation) pattern separates write operations (appending events) from read operations (querying projections), allowing each to be optimised independently. Write throughput is high because appending to an event store is a single INSERT. Read performance depends on how projections are maintained — synchronously via triggers or asynchronously via an event processor.

**Best for:** Platforms where full auditability is a non-negotiable requirement, where temporal queries are common, and where the read and write workloads have very different performance characteristics.

**Trade-offs:**
- (+) Complete, immutable audit trail — every state change is recorded forever
- (+) Temporal queries are trivial ("show me the artifact as it existed at time T")
- (+) Event replay enables new projections without data migration
- (+) Natural fit for the IaC domain where change history is the core value
- (+) Write path is extremely fast (single-table INSERT)
- (-) Read path requires maintained projections — stale projections produce stale reads
- (-) Event store grows unboundedly; requires archival/compaction strategy
- (-) More complex application code (event handlers, projection builders)
- (-) Eventual consistency between events and projections may confuse users
- (-) Harder to query across entities without purpose-built projections

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| HCL Specification | Generated HCL content captured in `artifact_generated` event payloads |
| OpenTofu Provider Protocol | Provider schema snapshots stored as `provider_schema_synced` events |
| OPA / Rego | Policy evaluation results captured as `security_scan_completed` events |
| CIS Benchmarks | CIS rule violations stored as structured event data with control IDs |
| OSCAL | Event payloads include OSCAL-compatible control references for compliance evidence |
| OCSF (Open Cybersecurity Schema Framework) | Event taxonomy follows OCSF activity class patterns for security events |
| CloudTrail / Activity Log patterns | Event schema modeled after AWS CloudTrail's `eventName`, `eventSource`, `requestParameters` structure |
| MCP v2025-11-25 | MCP tool invocations recorded as events for session reconstruction |

---

## Event Store (Core)

```sql
-- The immutable event store — single source of truth
CREATE TABLE events (
    event_id        UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    stream_id       UUID NOT NULL,           -- aggregate root ID (session, project, org)
    stream_type     TEXT NOT NULL,            -- 'session', 'project', 'org', 'workspace'
    event_type      TEXT NOT NULL,            -- e.g., 'session.started', 'prompt.submitted', 'artifact.generated'
    event_version   INTEGER NOT NULL,         -- per-stream sequence number for ordering
    payload         JSONB NOT NULL,           -- event-specific data
    metadata        JSONB NOT NULL DEFAULT '{}', -- correlation IDs, user agent, IP
    org_id          UUID NOT NULL,            -- tenant isolation
    user_id         UUID,                     -- who triggered the event (null for system events)
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (stream_id, event_version)
);

-- Primary query pattern: replay a stream in order
CREATE INDEX idx_events_stream ON events(stream_id, event_version);

-- Secondary: query by type across all streams (e.g., all 'artifact.generated' events)
CREATE INDEX idx_events_type ON events(event_type, created_at DESC);

-- Tenant isolation
CREATE INDEX idx_events_org ON events(org_id, created_at DESC);

-- Time-range queries for audit
CREATE INDEX idx_events_created ON events(created_at DESC);

-- Partition by month for performance and archival
-- CREATE TABLE events PARTITION BY RANGE (created_at);
```

### Event Type Taxonomy

```
-- Session lifecycle
session.started              -- user begins a generation session
session.completed            -- user ends or abandons session
session.source_changed       -- session source switched (cli → mcp)

-- Prompt lifecycle
prompt.submitted             -- user sends a prompt
prompt.response_generated    -- LLM returns a response
prompt.tokens_counted        -- token usage recorded

-- Artifact lifecycle
artifact.generated           -- IaC code generated from a prompt
artifact.refined             -- artifact updated via conversational refinement
artifact.format_converted    -- artifact converted (HCL → Pulumi, etc.)
artifact.module_extracted    -- reusable module extracted from artifact

-- Security lifecycle
security.scan_started        -- scan initiated on an artifact
security.rule_evaluated      -- individual rule check result
security.scan_completed      -- scan finished with summary
security.policy_created      -- new OPA/Rego policy added
security.policy_updated      -- policy modified

-- Cost lifecycle
cost.estimate_requested      -- cost estimation initiated
cost.estimate_completed      -- cost estimate returned with breakdown

-- Deployment lifecycle
plan.created                 -- terraform plan initiated
plan.completed               -- plan output captured
plan.approved                -- human approved the plan
plan.rejected                -- human rejected the plan
deployment.started           -- terraform apply started
deployment.succeeded         -- apply completed successfully
deployment.failed            -- apply failed with error

-- Provider lifecycle
provider.schema_synced       -- provider schema cache updated
provider.credential_created  -- new cloud credential registered
provider.credential_rotated  -- credential rotated

-- Organisation lifecycle
org.created                  -- new tenant registered
org.member_added             -- user added to org
org.member_removed           -- user removed from org
org.settings_updated         -- org settings changed
```

### Example Event Payloads

```sql
-- prompt.submitted
-- {
--   "session_id": "uuid",
--   "sequence_num": 3,
--   "content": "Add a read replica in us-west-2 for the RDS instance",
--   "model_id": "claude-opus-4-6"
-- }

-- artifact.generated
-- {
--   "session_id": "uuid",
--   "prompt_id": "uuid",
--   "output_format": "hcl",
--   "content_hash": "sha256:abc123...",
--   "content": "resource \"aws_db_instance\" \"replica\" {\n  ...\n}",
--   "file_path": "modules/database/replica.tf",
--   "resources": [
--     {"type": "aws_db_instance", "name": "replica", "provider": "aws"}
--   ],
--   "version": 2
-- }

-- security.rule_evaluated
-- {
--   "artifact_id": "uuid",
--   "rule_id": "CIS-AWS-2.1.1",
--   "benchmark": "CIS AWS Foundations 3.0",
--   "nist_control": "SC-28",
--   "status": "failed",
--   "resource_address": "aws_db_instance.replica",
--   "message": "RDS instance does not have encryption at rest enabled",
--   "remediation": "Set storage_encrypted = true"
-- }

-- cost.estimate_completed
-- {
--   "artifact_id": "uuid",
--   "total_monthly_usd": 342.50,
--   "line_items": [
--     {"resource": "aws_db_instance.primary", "component": "compute", "monthly_usd": 185.00},
--     {"resource": "aws_db_instance.replica", "component": "compute", "monthly_usd": 157.50}
--   ],
--   "pricing_source": "infracost"
-- }
```

## Identity Tables (Mutable — Not Event-Sourced)

```sql
-- These tables are mutable reference data, not event-sourced.
-- User/org identity changes too frequently and has external auth dependencies.

CREATE TABLE organisations (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            TEXT NOT NULL,
    slug            TEXT NOT NULL UNIQUE,
    plan_tier       TEXT NOT NULL DEFAULT 'free',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE users (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    email           TEXT NOT NULL UNIQUE,
    display_name    TEXT NOT NULL,
    auth_provider   TEXT NOT NULL,
    auth_subject    TEXT NOT NULL,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (auth_provider, auth_subject)
);

CREATE TABLE org_memberships (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id          UUID NOT NULL REFERENCES organisations(id) ON DELETE CASCADE,
    user_id         UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    role            TEXT NOT NULL DEFAULT 'member',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (org_id, user_id)
);
```

## Materialised Projections (Read Models)

```sql
-- Projection: Current session state (rebuilt from session.* and prompt.* events)
CREATE TABLE projection_sessions (
    session_id      UUID PRIMARY KEY,
    project_id      UUID NOT NULL,
    user_id         UUID NOT NULL,
    org_id          UUID NOT NULL,
    workspace_id    UUID,
    title           TEXT,
    status          TEXT NOT NULL,
    source          TEXT NOT NULL,
    model_id        TEXT,
    prompt_count    INTEGER NOT NULL DEFAULT 0,
    artifact_count  INTEGER NOT NULL DEFAULT 0,
    total_tokens    INTEGER NOT NULL DEFAULT 0,
    last_event_version INTEGER NOT NULL,      -- tracks which event was last processed
    created_at      TIMESTAMPTZ NOT NULL,
    updated_at      TIMESTAMPTZ NOT NULL
);

CREATE INDEX idx_proj_sessions_org ON projection_sessions(org_id, created_at DESC);
CREATE INDEX idx_proj_sessions_user ON projection_sessions(user_id, created_at DESC);

-- Projection: Latest artifact per session (rebuilt from artifact.* events)
CREATE TABLE projection_artifacts (
    artifact_id     UUID PRIMARY KEY,
    session_id      UUID NOT NULL,
    org_id          UUID NOT NULL,
    output_format   TEXT NOT NULL,
    content         TEXT NOT NULL,
    content_hash    TEXT NOT NULL,
    file_path       TEXT,
    version         INTEGER NOT NULL,
    resource_count  INTEGER NOT NULL DEFAULT 0,
    scan_status     TEXT DEFAULT 'pending',   -- 'pending', 'passed', 'failed'
    failed_rules    INTEGER DEFAULT 0,
    estimated_monthly_cost_usd NUMERIC(12,2),
    created_at      TIMESTAMPTZ NOT NULL,
    updated_at      TIMESTAMPTZ NOT NULL
);

CREATE INDEX idx_proj_artifacts_session ON projection_artifacts(session_id);
CREATE INDEX idx_proj_artifacts_org ON projection_artifacts(org_id);

-- Projection: Security compliance dashboard
CREATE TABLE projection_compliance_summary (
    org_id          UUID NOT NULL,
    date            DATE NOT NULL,
    framework       TEXT NOT NULL,            -- 'CIS AWS', 'NIST 800-53', etc.
    total_checks    INTEGER NOT NULL DEFAULT 0,
    passed_checks   INTEGER NOT NULL DEFAULT 0,
    failed_checks   INTEGER NOT NULL DEFAULT 0,
    compliance_pct  NUMERIC(5,2) NOT NULL DEFAULT 0,
    PRIMARY KEY (org_id, date, framework)
);

-- Projection: Cost trends per project
CREATE TABLE projection_cost_trends (
    project_id      UUID NOT NULL,
    org_id          UUID NOT NULL,
    date            DATE NOT NULL,
    total_monthly_usd NUMERIC(12,2) NOT NULL,
    resource_count  INTEGER NOT NULL,
    PRIMARY KEY (project_id, date)
);

CREATE INDEX idx_proj_cost_org ON projection_cost_trends(org_id, date DESC);

-- Projection: Deployment history (rebuilt from plan.* and deployment.* events)
CREATE TABLE projection_deployments (
    plan_id         UUID PRIMARY KEY,
    artifact_id     UUID NOT NULL,
    workspace_id    UUID NOT NULL,
    org_id          UUID NOT NULL,
    user_id         UUID NOT NULL,
    status          TEXT NOT NULL,
    resources_add   INTEGER DEFAULT 0,
    resources_change INTEGER DEFAULT 0,
    resources_destroy INTEGER DEFAULT 0,
    approved_by     UUID,
    approved_at     TIMESTAMPTZ,
    started_at      TIMESTAMPTZ,
    completed_at    TIMESTAMPTZ,
    error_message   TEXT,
    created_at      TIMESTAMPTZ NOT NULL
);

CREATE INDEX idx_proj_deployments_workspace ON projection_deployments(workspace_id, created_at DESC);
CREATE INDEX idx_proj_deployments_org ON projection_deployments(org_id, created_at DESC);
```

## Event Processor (Application-Level Pseudocode)

```sql
-- The event processor reads new events and updates projections.
-- This runs as a background worker, not as database triggers.

-- Example: processing an 'artifact.generated' event
-- 1. Read event payload
-- 2. UPSERT into projection_artifacts
-- 3. UPDATE projection_sessions SET artifact_count = artifact_count + 1
-- 4. If auto-scan enabled, emit 'security.scan_started' event

-- Cursor tracking: each projection tracks its position in the event stream
CREATE TABLE projection_cursors (
    projection_name TEXT PRIMARY KEY,         -- 'sessions', 'artifacts', 'compliance', etc.
    last_event_id   UUID NOT NULL,
    last_processed_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

## Temporal Query Examples

```sql
-- "What did the generated artifact look like at 3pm yesterday?"
-- Replay all artifact.* events for the stream up to the target time
SELECT payload
FROM events
WHERE stream_id = '<session_id>'
  AND event_type IN ('artifact.generated', 'artifact.refined')
  AND created_at <= '2026-05-19 15:00:00+00'
ORDER BY event_version DESC
LIMIT 1;

-- "Show all security violations for org X in the last 7 days"
SELECT payload
FROM events
WHERE org_id = '<org_id>'
  AND event_type = 'security.rule_evaluated'
  AND (payload->>'status') = 'failed'
  AND created_at >= now() - INTERVAL '7 days'
ORDER BY created_at DESC;

-- "Reconstruct the full conversation history of a session"
SELECT event_type, payload, created_at
FROM events
WHERE stream_id = '<session_id>'
  AND event_type IN ('prompt.submitted', 'prompt.response_generated', 'artifact.generated', 'artifact.refined')
ORDER BY event_version ASC;

-- "How has monthly cost changed for project X over time?"
SELECT date, total_monthly_usd
FROM projection_cost_trends
WHERE project_id = '<project_id>'
ORDER BY date ASC;
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Event Store | 1 | events (partitioned by month) |
| Identity (Mutable) | 3 | organisations, users, org_memberships |
| Projections — Sessions | 1 | projection_sessions |
| Projections — Artifacts | 1 | projection_artifacts |
| Projections — Compliance | 1 | projection_compliance_summary |
| Projections — Cost | 1 | projection_cost_trends |
| Projections — Deployments | 1 | projection_deployments |
| Infrastructure | 1 | projection_cursors |
| **Total** | **10** | Plus partitions of events table |

---

## Key Design Decisions

1. **Single event store table** — all events across all aggregate types go into one table, partitioned by `created_at`. This simplifies operational management (one table to back up, one to monitor) and enables cross-aggregate temporal queries.

2. **Stream ID + event version for ordering** — each aggregate (session, project, org) has a monotonically increasing version number within its stream. The `UNIQUE (stream_id, event_version)` constraint prevents concurrent writes from corrupting stream ordering (optimistic concurrency control).

3. **Identity tables are NOT event-sourced** — user accounts and org memberships change via external auth providers (GitHub OAuth, SAML). Event-sourcing these would add complexity without benefit. They remain traditional mutable tables.

4. **Projections are disposable** — any projection can be dropped and rebuilt from the event stream. This means new features (e.g., a usage analytics dashboard) can be added by creating a new projection and replaying historical events — no data migration needed.

5. **Event payloads are self-contained** — each event payload contains enough data to be useful without joining to other events. The `artifact.generated` event includes the full content, not just an artifact ID. This makes event replay fast and reduces read-time joins.

6. **OCSF-inspired event taxonomy** — the event type hierarchy (`entity.action`) follows the Open Cybersecurity Schema Framework pattern, making it straightforward to export security events to SIEM systems.

7. **Projection cursors for exactly-once processing** — the `projection_cursors` table tracks which event each projection has processed, enabling idempotent replay and crash recovery of the event processor.

8. **Audit trail is the event store itself** — there is no separate audit log table. The event store IS the audit trail. Any query like "who approved plan X?" is answered by querying `events WHERE event_type = 'plan.approved' AND stream_id = '<plan_stream>'`.

9. **Cost and compliance as projections** — rather than separate transactional tables, cost trends and compliance summaries are materialised projections rebuilt from events. This means historical compliance posture can be recalculated retroactively if rules change.

10. **Monthly partitioning with archival** — events older than the retention period (e.g., 2 years) can be archived to cold storage (S3/GCS) and detached as partitions without affecting live queries. Projections retain summarised data indefinitely.
