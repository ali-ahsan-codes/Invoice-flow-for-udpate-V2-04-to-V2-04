# InvoiceFlow SaaS — Multi-Tenant Design

Status: **design only, no code yet.** This document is the reference for the build phases in section 12.
Date: 2026-10-18.

---

## 1. Decisions (agreed)

| # | Decision | Detail |
|---|----------|--------|
| D1 | **One shared database** for the whole platform | All companies (tenants) live in one MySQL database. There is no per-company database. |
| D2 | **Tenant = company container** | Every company is one tenant. All of its data is tagged with `tenant_id`. Data never crosses a tenant boundary. |
| D3 | **Units = branches and child companies** | A tenant has one or more units. Each unit is a branch, a child company, or the single unit of a single-staff company. |
| D4 | **"Sync" = shared data, governed by role** | Inside a tenant, data is shared. What a user can do (view, add, edit, delete) depends on their role and on the unit scope of that role. |
| D5 | **Single-staff company uses the same model** | One tenant, one unit, one user. No separate code path. |
| D6 | **Scope is configured later, in permissions** | Policy defaults exist from day one. Admins change them per tenant later through the permissions screen. |

---

## 2. Glossary

| Term | Meaning |
|------|---------|
| Platform | The whole SaaS installation: one app, one database. |
| Tenant | One customer company. Owns its units, users, roles and data. |
| Unit | A branch, child company, or the single location of a tenant. Units form a tree under the tenant (`parent_unit_id`). |
| Member | A user account that belongs to a tenant. |
| Role | A named bundle of permissions for a tenant (e.g. "Accountant", "Cashier"). |
| Permission | `resource` × `action` (view, add, edit, delete). |
| Scope | Which units a permission applies to: `own_unit`, `unit_subtree`, or `tenant`. |
| Policy | A tenant-level rule that decides the default scope of a resource (inventory, customers, accounts, roles). |

---

## 3. Isolation model

### 3.1 Tenant boundary (hard rule)
- Every tenant-owned table has `tenant_id NOT NULL`.
- `tenant_id` is **never** taken from request input. The server derives it from the logged-in member's membership.
- Every query on a tenant-owned table goes through one tenant-scoped query helper. A raw query without the helper is a review failure.

### 3.2 Unit boundary (role-driven)
- Rows that belong to a unit carry `unit_id`.
- A permission's scope decides which `unit_id` values are visible:
  - `own_unit`: only the member's unit.
  - `unit_subtree`: the member's unit and every unit below it.
  - `tenant`: every unit in the tenant.
- Scope is applied in the same helper as the tenant filter. The helper is the only place where visibility is decided.

### 3.3 Session
- Login identifies the member. The session stores `member_id`, `tenant_id`, and the active unit.
- The tenant is resolved from the member, not from the URL. Subdomains are optional later.

---

## 4. Data model (target)

All new tables are created with `CREATE TABLE IF NOT EXISTS` in SQL files (see section 9). `upgrade_database.py` is **not** used for these tables, because it runs ALTERs on existing tables.

### 4.1 Platform tables (no tenant_id)

| Table | Key columns | Notes |
|-------|-------------|-------|
| `tenants` | `id`, `name`, `slug` (unique), `status`, `created_at` | One row per company. |
| `accounts` | `id`, `email` (unique), `password_hash`, `display_name`, `status` | Global person login. Needed if one person may work for several tenants later. |

### 4.2 Tenant-owned tables

| Table | Key columns | Notes |
|-------|-------------|-------|
| `units` | `id`, `tenant_id`, `parent_unit_id` (nullable), `name`, `kind` (`branch`, `child_company`, `main`) | Tree under the tenant. |
| `members` | `id`, `tenant_id`, `account_id`, `unit_id` (home unit), `role_id`, `status` | Connects an account to a tenant. |
| `roles` | `id`, `tenant_id`, `name` (unique per tenant) | Tenant-defined roles. |
| `role_permissions` | `id`, `tenant_id`, `role_id`, `resource`, `action`, `scope` | One row per granted permission. |
| `resource_policies` | `id`, `tenant_id`, `resource`, `default_scope` | Tenant-level default scope for inventory, customers, accounts, roles. |
| `invoices`, `invoice_*`, `books`, `batches`, … | `tenant_id`, `unit_id` added | Existing tables, see 4.3. |
| `audit_logs` | `tenant_id`, `member_id`, `unit_id` | Every write is logged with its tenant. |

### 4.3 Changes to existing tables

| Current | Target | Why |
|---------|--------|-----|
| `users` (global username/email) | Moves to `accounts` + `members` | A person can be a member of more than one tenant later. |
| `invoices` `UNIQUE(date, serial_numeric)` (global) | `UNIQUE(tenant_id, date, serial_numeric)` | Serial numbers belong to a company, not to the platform. |
| `invoices` `UNIQUE(batch_id, serial_number)` | Keep, and add `tenant_id` to the index | Batch is still the document series inside a unit. |
| `books` `UNIQUE(batch_id, book_number)` | Keep, add `tenant_id` | Same reasoning. |
| `system_settings.key` (global unique) | `UNIQUE(tenant_id, key)`, with a platform-level row for global keys | Each tenant has its own settings. |
| `review_cases.case_key` (global unique) | `UNIQUE(tenant_id, case_key)` | Cases belong to a tenant. |
| `batches` | Decision pending (see section 11, Q2) | Batch may map to a unit, or stay a separate series inside a unit. |

### 4.4 Indexes
- Every tenant-owned table has an index that starts with `tenant_id`, for example `(tenant_id, unit_id, date)` on `invoices`.
- Foreign keys point to ids that already include `tenant_id` in the row, so a child row can never point to a parent in another tenant. The application enforces this; MySQL FKs alone do not.

---

## 5. Access model

### 5.1 Resources and actions
- Resources (first set): `invoices`, `notes`, `links`, `media`, `books`, `inventory`, `customers`, `accounts`, `roles`, `members`, `reports`, `settings`.
- Actions: `view`, `add`, `edit`, `delete`.
- Each permission is `(resource, action, scope)`.

### 5.2 Scope rules
- A member can act on a row only if: the permission exists for that resource and action, AND the row's `unit_id` is inside the permission's scope.
- Delete is always logged and may need a separate permission from edit.
- Permissions are additive. There is no deny rule in the first version.

### 5.3 Defaults for a new tenant

| Role | Default scope |
|------|---------------|
| Owner / Admin | `tenant` for all resources |
| Branch manager | `unit_subtree` for all resources |
| Staff | `own_unit` for invoices, notes, media, links |

Admins change these per role and per resource. Changes are audited.

### 5.4 Policy (shared data by default)
- `resource_policies` sets the default scope that new roles inherit for each resource.
- Example: `inventory = tenant` (shared across branches), `accounts = own_unit` (hidden from other branches).
- The policy is a default. A role's own permission can be narrower but not wider than the policy.

---

## 6. Enforcement

- A single module (`app/tenancy.py`, planned) provides:
  - `current_tenant_id()`, `current_member()`, `current_unit_ids()`.
  - `tenant_query(model)`: returns a query with `tenant_id` and unit scope applied.
- Services and routes use only this module to read or write tenant data.
- Writes set `tenant_id` from the session and `unit_id` from the target unit, after checking that the target unit is in scope.
- The existing `scope_batch` hook in `app/batches.py` is the pattern to extend. It is not enough alone, because it only knows about batches.
- Admin-only code that must cross tenants (platform support) is a separate module and is audited.

---

## 7. Users, roles, and onboarding

- **Signup:** creates an account, a tenant, a main unit, and an owner member.
- **Invite:** an admin adds a member to a unit with a role. The invitee gets a link.
- **Single-staff company:** the owner is the only member and has the main unit only.
- **Child company / branch:** an admin creates a unit under a parent unit. The new unit inherits the tenant policy.
- **Moving a unit to another parent** is allowed inside the same tenant only. Moving a unit across tenants is not supported.

---

## 8. Migration from the current single-company app

1. Create `tenants`, `accounts`, `units`, `members`, `roles`, `role_permissions`, `resource_policies` (new SQL file).
2. Create one tenant for the existing company and one main unit.
3. Copy `users` into `accounts` and `members` (same ids preserved where possible).
4. Add `tenant_id` and `unit_id` to existing tables with ALTER in a separate, reviewed migration file. Backfill with the default tenant and main unit.
5. Replace the global unique indexes listed in 4.3 with tenant-scoped ones.
6. Switch the app to the tenant helper. Run the existing 6 QA suites and the new isolation tests.

Each step is a separate SQL file with a backup step in the deploy checklist.

---

## 9. MySQL schema rules (repo convention)

- New tables: `CREATE TABLE IF NOT EXISTS` in `sql/<name>.sql`. Safe to re-run.
- Column additions and index changes on existing tables: explicit `ALTER` in its own SQL file, reviewed by a human, run in staging first.
- `upgrade_database.py` is not used for SaaS migrations.
- Every migration file has a backup note and a rollback note.
- Staging run is done by the user; the sandbox has no MySQL.

---

## 10. Security checklist

- `tenant_id` is never read from a request body or query string for tenant data.
- Every list, detail, edit, delete route is tested with a member from another tenant (must return 404).
- Every write endpoint is tested with a unit outside the member's scope (must return 403).
- Audit log includes tenant, member, unit, action, and object id.
- Passwords and API keys stay hashed. API keys are tenant-bound.

---

## 11. Open questions

| # | Question | Default if no answer |
|---|----------|----------------------|
| Q1 | Can one person (account) work for more than one tenant? | Design supports it through `accounts` + `members`. UI first supports one tenant per login. |
| Q2 | Does a batch map to a unit, or stay a series inside a unit? | Batch stays a series inside a unit. Unit is the branch. |
| Q3 | Can a holding group see several tenants? | Not in v1. Would be a separate `groups` table later. |
| Q4 | Is "sync" only access scope, or also offline replication for the mobile app? | Access scope only. Offline replication is a later phase. |
| Q5 | Are roles and permissions editable by every tenant admin, or only platform staff? | Tenant admins. |

---

## 12. Build phases

| Phase | Content | Done when |
|-------|---------|-----------|
| P0 | This document (current step) | Reviewed by the owner |
| P1 | Platform and tenant tables, SQL files, tenant helper, session tenant | Tests pass on SQLite harness; SQL files reviewed |
| P2 | Add `tenant_id` and `unit_id` to existing tables; replace global unique indexes | Existing 6 QA suites pass on the new model |
| P3 | Roles, permissions, resource policies, permissions screen | Scope tests for own_unit, unit_subtree, tenant |
| P4 | Signup, invite, unit creation | End-to-end test with two tenants |
| P5 | Mobile API and Android app use tenant and unit scope | Contract tests updated |
| P6 | Deploy documentation and screenshots | Deploy checklist, new screenshots |

Each phase gets a test report and a new screenshot set. Old screenshots are deleted and replaced when the UI changes.
