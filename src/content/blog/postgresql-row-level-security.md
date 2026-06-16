---
title: "Row Level Security in PostgreSQL: Multi-Tenant Data Isolation Done Right"
description: "How to implement per-tenant data isolation at the database layer using PostgreSQL RLS — without leaking data across tenants or killing performance."
category: "backend"
tags: ["postgresql", "multi-tenancy", "rls", "saas", "security"]
publishedAt: 2025-04-10
slug: "postgresql-row-level-security"
---

Multi-tenancy is one of those things that looks straightforward in architecture diagrams and turns into a minefield the moment you go to production. Row Level Security in PostgreSQL is the cleanest solution I've found.

## Why RLS Over Application-Layer Filtering

The alternative — filtering every query with a `WHERE tenant_id = $1` clause — has a catastrophic failure mode: one forgotten `WHERE` clause, and tenant A sees tenant B's data. This has happened in production at several high-profile SaaS companies.

RLS moves isolation to the database itself. Even if your application code has a bug and omits the tenant filter, Postgres enforces the policy.

## Setting Up the Policy

```sql
-- Enable RLS on the table
ALTER TABLE cases ENABLE ROW LEVEL SECURITY;

-- Create policy: users only see rows matching their tenant
CREATE POLICY tenant_isolation ON cases
  USING (tenant_id = current_setting('app.tenant_id')::uuid);
```

Your application sets the tenant context at the start of each request:

```sql
SET LOCAL app.tenant_id = '550e8400-e29b-41d4-a716-446655440000';
```

## Performance Considerations

RLS policies are evaluated per row. For tables with millions of rows across hundreds of tenants, ensure `tenant_id` is indexed:

```sql
CREATE INDEX CONCURRENTLY idx_cases_tenant_id ON cases(tenant_id);
```

With this index, Postgres can use index scans to filter to the tenant's rows before applying any other predicates.

## Testing Isolation

Never trust that it works — test it explicitly:

```sql
-- Set tenant A context
SET LOCAL app.tenant_id = 'tenant-a-uuid';
SELECT count(*) FROM cases; -- should return only tenant A rows

-- Attempt cross-tenant read (should return 0 rows, not error)
SET LOCAL app.tenant_id = 'tenant-b-uuid';
SELECT * FROM cases WHERE tenant_id = 'tenant-a-uuid'; -- returns empty
```

The policy silently filters — no errors, just zero rows. This is the expected behaviour.

## Caveats

- Superuser roles bypass RLS by default. Use `SET ROLE` in your connection pool to assume a non-superuser role.
- RLS adds a small per-query overhead. On write-heavy workloads, profile before deploying.
- Schema migrations need careful handling — adding columns to RLS-protected tables requires elevated privileges.
