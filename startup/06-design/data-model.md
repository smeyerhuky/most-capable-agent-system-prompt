---
type: Design Doc
title: "AgentForge Data Model"
description: Postgres database schema for the AgentForge platform. Covers all tables, relationships, indexes, tenant isolation strategy, and migration approach.
tags: [design, data-model, postgres, schema, multi-tenant, migrations]
timestamp: 2026-06-28T00:00:00Z
---

# AgentForge Data Model

All tables use `UUID` primary keys (not auto-increment integers) for global uniqueness and sharding compatibility. Timestamps are `TIMESTAMPTZ` (UTC).

---

## Entity Relationship Overview

```
users (1) ─────────────── (*) pipelines
users (1) ─────────────── (*) runs
users (1) ─────────────── (*) billing_events
users (*) ─── team_members ─── (*) teams
teams (1) ──────────────── (*) team_pipelines (join)
pipelines (1) ──────────── (*) runs
runs (1) ───────────────── (*) run_events
runs (1) ───────────────── (*) billing_events
```

---

## Core Tables

### users

```sql
CREATE TABLE users (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    email VARCHAR(255) UNIQUE NOT NULL,
    name VARCHAR(255),
    password_hash VARCHAR(255),   -- NULL for OAuth-only users
    plan VARCHAR(50) NOT NULL DEFAULT 'hobbyist',
    stripe_customer_id VARCHAR(100),
    stripe_subscription_id VARCHAR(100),
    openai_api_key_encrypted TEXT,   -- user's own OpenAI key, AES-256 encrypted
    preferred_region VARCHAR(20) DEFAULT 'us-east-1',
    created_at TIMESTAMPTZ DEFAULT NOW(),
    updated_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE INDEX idx_users_email ON users(email);
CREATE INDEX idx_users_stripe_customer ON users(stripe_customer_id);
```

### pipelines

```sql
CREATE TABLE pipelines (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    owner_user_id UUID REFERENCES users(id) ON DELETE CASCADE,
    owner_team_id UUID REFERENCES teams(id) ON DELETE CASCADE,
    name VARCHAR(255) NOT NULL,
    source_s3_key TEXT NOT NULL,         -- S3 key for .agent source
    ir_s3_key TEXT,                      -- S3 key for compiled IR (nullable, recompiled on change)
    visibility VARCHAR(20) NOT NULL DEFAULT 'private',  -- private, team, public
    compile_status VARCHAR(20) DEFAULT 'pending',       -- pending, compiled, error
    compile_errors JSONB,
    last_run_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ DEFAULT NOW(),
    updated_at TIMESTAMPTZ DEFAULT NOW(),
    CONSTRAINT owner_check CHECK (
        (owner_user_id IS NOT NULL AND owner_team_id IS NULL) OR
        (owner_user_id IS NULL AND owner_team_id IS NOT NULL)
    )
);

CREATE INDEX idx_pipelines_owner_user ON pipelines(owner_user_id);
CREATE INDEX idx_pipelines_owner_team ON pipelines(owner_team_id);
CREATE INDEX idx_pipelines_visibility ON pipelines(visibility) WHERE visibility = 'public';
```

### runs

```sql
CREATE TABLE runs (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    pipeline_id UUID NOT NULL REFERENCES pipelines(id),
    user_id UUID NOT NULL REFERENCES users(id),
    status VARCHAR(20) NOT NULL DEFAULT 'running',  -- running, paused, completed, failed
    ir_s3_key TEXT NOT NULL,             -- snapshot of IR at run time
    current_step VARCHAR(255),
    current_ip INT DEFAULT 0,
    dry_run BOOLEAN DEFAULT FALSE,
    total_input_tokens BIGINT DEFAULT 0,
    total_output_tokens BIGINT DEFAULT 0,
    total_cost_usd DECIMAL(10, 4) DEFAULT 0,
    error_message TEXT,
    started_at TIMESTAMPTZ DEFAULT NOW(),
    completed_at TIMESTAMPTZ,
    heartbeat_at TIMESTAMPTZ DEFAULT NOW()  -- updated by runtime every 30s; orphan detection
);

CREATE INDEX idx_runs_pipeline ON runs(pipeline_id);
CREATE INDEX idx_runs_user ON runs(user_id);
CREATE INDEX idx_runs_status ON runs(status) WHERE status IN ('running', 'paused');
CREATE INDEX idx_runs_heartbeat ON runs(heartbeat_at) WHERE status = 'running';
```

### run_events

```sql
CREATE TABLE run_events (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    run_id UUID NOT NULL REFERENCES runs(id) ON DELETE CASCADE,
    event_index INT NOT NULL,             -- sequential index within the run
    event_type VARCHAR(100) NOT NULL,
    event_data JSONB NOT NULL,
    emitted_at TIMESTAMPTZ DEFAULT NOW(),
    UNIQUE (run_id, event_index)
);

CREATE INDEX idx_run_events_run ON run_events(run_id, event_index);
```

---

## Teams (Phase 3)

### teams

```sql
CREATE TABLE teams (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name VARCHAR(255) NOT NULL,
    plan VARCHAR(50) NOT NULL DEFAULT 'team',
    stripe_customer_id VARCHAR(100),
    stripe_subscription_id VARCHAR(100),
    created_at TIMESTAMPTZ DEFAULT NOW()
);
```

### team_members

```sql
CREATE TABLE team_members (
    team_id UUID NOT NULL REFERENCES teams(id) ON DELETE CASCADE,
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    role VARCHAR(20) NOT NULL DEFAULT 'member',  -- owner, admin, member
    joined_at TIMESTAMPTZ DEFAULT NOW(),
    PRIMARY KEY (team_id, user_id)
);

CREATE INDEX idx_team_members_user ON team_members(user_id);
```

---

## Auth

### api_tokens

```sql
CREATE TABLE api_tokens (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    token_hash VARCHAR(255) NOT NULL UNIQUE,  -- SHA-256 of the actual token
    name VARCHAR(255),
    last_used_at TIMESTAMPTZ,
    expires_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE INDEX idx_api_tokens_hash ON api_tokens(token_hash);
```

### sso_configurations (Phase 3)

```sql
CREATE TABLE sso_configurations (
    team_id UUID PRIMARY KEY REFERENCES teams(id) ON DELETE CASCADE,
    idp_metadata_url TEXT NOT NULL,
    entity_id TEXT NOT NULL,
    sso_only BOOLEAN DEFAULT FALSE,   -- if true, disable password login for team members
    attribute_mapping JSONB,          -- maps IdP attribute names to AgentForge fields
    created_at TIMESTAMPTZ DEFAULT NOW()
);
```

---

## Marketplace

### marketplace_templates

```sql
CREATE TABLE marketplace_templates (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    pipeline_id UUID NOT NULL REFERENCES pipelines(id),
    author_user_id UUID NOT NULL REFERENCES users(id),
    name VARCHAR(255) NOT NULL,
    description TEXT,
    tags TEXT[] DEFAULT '{}',
    downloads INT DEFAULT 0,
    status VARCHAR(20) DEFAULT 'pending_review',  -- pending_review, published, rejected
    published_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE INDEX idx_marketplace_status ON marketplace_templates(status) WHERE status = 'published';
CREATE INDEX idx_marketplace_downloads ON marketplace_templates(downloads DESC) WHERE status = 'published';
```

---

## Audit Logs (Phase 3)

```sql
CREATE TABLE audit_logs (
    id BIGSERIAL PRIMARY KEY,           -- sequential for tamper detection
    user_id UUID REFERENCES users(id),
    team_id UUID REFERENCES teams(id),
    action VARCHAR(100) NOT NULL,        -- 'run.create', 'pipeline.delete', 'member.invite', etc.
    resource_type VARCHAR(50),
    resource_id UUID,
    ip_address INET,
    user_agent TEXT,
    metadata JSONB,
    prev_checksum VARCHAR(64),          -- SHA-256 of previous row (chain integrity)
    checksum VARCHAR(64),               -- SHA-256 of this row's content
    created_at TIMESTAMPTZ DEFAULT NOW()
);

-- Append-only: no UPDATE or DELETE allowed (enforced via trigger)
CREATE OR REPLACE FUNCTION prevent_audit_modification()
RETURNS TRIGGER AS $$
BEGIN
    RAISE EXCEPTION 'Audit logs are immutable';
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER no_audit_update BEFORE UPDATE ON audit_logs EXECUTE FUNCTION prevent_audit_modification();
CREATE TRIGGER no_audit_delete BEFORE DELETE ON audit_logs EXECUTE FUNCTION prevent_audit_modification();
```

---

## Tenant Isolation

Every query in the API layer must include a `user_id` or `team_id` predicate. Enforced via:

1. **ORM scope:** A custom SQLAlchemy session factory that applies `WHERE user_id = :current_user` to all queries by default. Queries without a `user_id` filter require explicit opt-out (`session.unscoped()`).
2. **Row-level security (Phase 3):** Postgres RLS policies as a defense-in-depth measure for enterprise tier:
   ```sql
   ALTER TABLE pipelines ENABLE ROW LEVEL SECURITY;
   CREATE POLICY pipeline_owner ON pipelines
       USING (owner_user_id = current_setting('app.current_user_id')::uuid
              OR owner_team_id = ANY(
                  SELECT team_id FROM team_members
                  WHERE user_id = current_setting('app.current_user_id')::uuid
              ));
   ```

---

## Migration Strategy

- **Tool:** Alembic (Python) for schema migrations
- **Convention:** Each migration is a numbered file (`001_initial.py`, `002_add_teams.py`, etc.)
- **Deployments:** Migrations run automatically at ECS task startup before the service accepts traffic
- **Backwards compatibility:** All Phase 2 and Phase 3 migrations must be backwards-compatible with Phase 1 code (no column drops or renames in a single deploy — use a two-phase approach: add new column → deploy new code → drop old column in next deploy)
- **Test:** Every migration is tested in CI with a fresh Postgres instance — apply migration, verify schema, apply test data, verify queries

---

## Related

- [Billing Design](billing-design.md) — billing tables (billing_events, billing_monthly_summary, billing_discrepancies)
- [API Spec](../05-specs/api-spec.md) — API endpoints that query these tables
- [TASK-1-19](../02-prd/phase-1-mvp.md) — Phase 1 migration implementation task
