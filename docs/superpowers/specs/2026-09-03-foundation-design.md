# De Rosewood Salon — Sub-project 1: Foundation

Status: approved for planning
Date: 2026-09-03

## Context

De Rosewood Salon needs a full Laravel + Blade + Tailwind + jQuery salon
management platform (booking, training, payments, CMS, multi-role admin).
The full scope is too large for one spec, so it is split into sequential
sub-projects. This document covers **sub-project 1: Foundation** — the
database schema for the whole system, role/permission-based authorization,
audit logging, and the admin shell (layout, sidebar, dashboard skeleton,
user + role management).

The repo currently is a stock Laravel 12 + Breeze (Blade + Alpine) install:
`users`/`cache`/`jobs` tables, Breeze auth views/controllers, a Breeze
dashboard, no salon-specific code.

## Goals

- Every future sub-project (CMS, catalog, booking, payments, training,
  public site) builds on this without schema churn to `users`/`roles`.
- Role/permission checks are data-driven (`@can('bookings.view')`
  everywhere), never `if ($user->role === 'admin')`.
- Server-side authorization is the source of truth; hiding a sidebar link
  is a UX nicety, not a security boundary.
- superAdmin can never be locked out or self-demoted by accident; nobody
  can self-elevate.

## Non-goals (later sub-projects)

Services, staff, staff schedules, bookings, payments, training, CMS pages,
gallery, testimonials, offers/coupons, contact messages, reports, real
logo/branding assets. This sub-project only creates the `site_settings`
row structure and seeds the real facts already supplied (salon name,
address, phone, Google rating/review count) — it does not build the CMS
screens to edit them.

## Identity model

One `users` table serves both admin/staff and customers, differentiated by
`role_id`:

- Roles: `superAdmin`, `admin`, `sysUser` (admin-panel roles) and
  `customer` (public role). All four are seeded as system roles.
- `/register` and `/login` (Breeze's existing routes) are the customer
  signup/login path. New registrations always get `role_id` = the
  `customer` role — never user-selectable.
- `/admin/login` is a separate route + view, authenticating against the
  same `users` table/guard. After login, the `admin` middleware checks the
  user holds at least one admin-panel permission; a `customer`-role user
  who somehow reaches `/admin/login` authenticates but is then denied
  entry to `/admin/*`.
- Admin accounts (`superAdmin`/`admin`/`sysUser`) are only created by an
  authorized admin via `/admin/users`, never via public `/register`.

Separately, a `customers` table holds the business/CRM record used by
booking/payment/training (total visits, total spent, notes, guest
contact info) with a nullable `user_id`. This lets guest bookings create a
`customers` row with no `users` account, and lets a registered customer's
`users` row link to their `customers` record. (The booking sub-project
populates and reads this table; Foundation only creates it.)

## Database schema

All new tables use `id` bigint PK, `created_at`/`updated_at` timestamps
unless noted. FKs use `restrict` on delete unless noted otherwise, to
avoid silently orphaning audit trails.

**roles**
- `name` (string, unique) — display name, e.g. "Super Admin"
- `slug` (string, unique) — `superAdmin` | `admin` | `sysUser` | `customer`
- `description` (string, nullable)
- `is_system` (boolean, default false) — true for all four seeded roles;
  blocks deletion and slug edits from the UI
- `is_active` (boolean, default true)

**permissions**
- `name` (string, unique) — dot notation, e.g. `bookings.view`
- `group` (string) — module name for grouped UI display, e.g. `bookings`
- `description` (string, nullable)

**role_permissions**
- `role_id` FK → roles, cascade on delete
- `permission_id` FK → permissions, cascade on delete
- unique (`role_id`, `permission_id`)

**users** (extend existing Breeze migration via a new migration)
- add `role_id` FK → roles, restrict on delete, not null (defaults to
  `customer` role id at application level on registration)
- add `status` enum-like string: `active` | `inactive` | `suspended`,
  default `active`
- add `phone` (string, nullable)
- add `last_login_at` (timestamp, nullable)
- existing Breeze columns (`name`, `email`, `password`, etc.) unchanged

**customers**
- `user_id` FK → users, nullable, set null on delete
- `name`, `phone`, `email` (email nullable — phone is the more reliable
  identifier for walk-ins), `gender` (nullable string), `date_of_birth`
  (nullable date), `address` (nullable string), `notes` (text, nullable)
- `total_visits` (unsigned int, default 0)
- `total_spent` (decimal 10,2, default 0)
- `last_visit_at` (timestamp, nullable)
- soft deletes
- index on `phone`, `email`

**site_settings**
- key/value store: `key` (string, unique), `value` (text, nullable),
  `type` (string, default `string` — for the settings screen a later
  sub-project builds: `string`|`text`|`json`|`bool`|`image`)
- Seeded rows (real facts only): `salon_name`, `phone`, `address`,
  `google_maps_plus_code`, `google_rating`, `google_review_count`,
  `currency` (default `NPR`), `logo` (nullable, placeholder empty)

**audit_logs**
- `user_id` FK → users, nullable, set null on delete (action is still
  logged if the actor is later deleted)
- `action` (string) — e.g. `role.updated`, `user.created`
- `entity_type` (string), `entity_id` (unsigned big int, nullable)
- `old_values` (json, nullable), `new_values` (json, nullable)
- `ip_address` (string, nullable), `user_agent` (string, nullable)
- index on (`entity_type`, `entity_id`), index on `user_id`

## Permission engine

No new package. A `PermissionService` (or a trait on `User`) exposes
`$user->hasPermission(string $name): bool`, backed by an eager-loaded
`role.permissions` relation (cached per-request). `AppServiceProvider`
registers:

```php
Gate::before(function (User $user, string $ability) {
    if ($user->role->slug === 'superAdmin') return true;   // never locked out
    if ($user->hasPermission($ability)) return true;
    return null; // fall through — unknown abilities still resolve normally
});
```

This makes `@can('bookings.view')`, `$this->authorize(...)`, and
`Gate::denies(...)` work uniformly across Blade, controllers, and AJAX
endpoints without per-feature policy classes. Controllers still call
`abort_unless($request->user()->can('...'), 403)` explicitly — Blade
`@can` only hides UI, it is never the enforcement point.

The `admin` middleware = authenticated + `status === 'active'` + holds at
least one permission whose `group` is an admin-panel module (in practice:
checked via `$user->can('dashboard.view')`, which every admin-panel role
gets by default and `customer` never does).

## Safeguards (privilege escalation)

- A role-edit form can never toggle `is_system` roles' slugs, and cannot
  delete a role with `is_system = true` or any role that still has users
  assigned.
- The role-assignment dropdown on `/admin/users` excludes `superAdmin`
  unless the acting user's own role is `superAdmin`.
- A user can never change their own `role_id` (self-service profile forms
  don't expose it; the admin user-edit form blocks editing yourself in
  that field server-side, not just hidden in the UI).
- `suspended`/`inactive` users fail login (`AuthenticatedSessionController`
  / the new admin login controller checks `status` after credential
  verification) and existing sessions are invalidated on suspend
  (`Auth::logoutOtherDevices` equivalent / forget remembered token).
- Permission grants: a user can only assign a permission to a role if they
  themselves currently hold that permission (prevents an `admin` granting
  a `sysUser` role a permission the `admin` lacks).

## Admin shell

- `resources/views/layouts/admin.blade.php`: sidebar + topbar, De Rosewood
  Tailwind palette (rosewood/gold/ivory from the brief) wired into
  `tailwind.config.js`, text serif wordmark reading "De Rosewood" driven
  by `site_settings.logo` (falls back to text when no image is set).
- Sidebar built from a permission-tagged nav config array (module →
  required permission → route); only Foundation's own items
  (Dashboard, Users, Roles, Audit Logs, Settings placeholder) render now,
  but the structure is built to accept later sub-projects' items without
  rework.
- `/admin/dashboard`: "Welcome, {name} — Role: {role}" plus empty
  placeholder cards for modules not yet built (no fabricated numbers).
- `/admin/users`: list (name, email, role, status, last login, created),
  create/edit, role change, enable/disable, delete (blocked for the last
  remaining `superAdmin` and for self-delete).
- `/admin/roles`: list, create/edit, grouped-by-module permission
  checkboxes, activate/deactivate, protected system roles.
- `/admin/audit-logs`: filterable list (actor, action, entity type, date
  range), gated by `audit_logs.view`.
- Every mutating action here (user create/edit/status/role change, role
  create/edit, permission grant/revoke) writes an `audit_logs` row.

## Seeders

- `RoleSeeder`: 4 system roles.
- `PermissionSeeder`: full permission list from the spec (all ~90,
  grouped by module) — seeded now even for unbuilt modules since it's
  inert metadata and avoids touching this seeder again per sub-project.
- `RolePermissionSeeder`: `superAdmin` → all; `admin` → dashboard +
  full CRUD on bookings/customers/services/staff/training + `reports.view`
  + `payments.view` + `cms.manage`; `sysUser` → `dashboard.view`,
  `bookings.view/create/update`, `customers.view`, `services.view`;
  `customer` → none.
- `UserSeeder`: one dev `superAdmin` —
  `admin@derosewood.test` / `Password123!`, clearly commented as
  dev-only and required to change before production.
- `SiteSettingSeeder`: the real supplied facts only.

## Testing

Feature/authorization tests (Pest, matching existing repo convention):

- superAdmin can manage users, roles, permissions.
- admin/sysUser can only reach permitted routes/AJAX endpoints; a direct
  request to an unpermitted admin route or a manually-called AJAX
  endpoint returns 403, not just a hidden button.
- Neither admin nor sysUser can elevate their own role or grant a
  permission they don't hold.
- superAdmin role/last-admin cannot be deleted through normal flows.
- suspended/inactive users cannot authenticate; an active session is
  invalidated when a user is suspended.
- Public `/register` always yields a `customer`-role user, never
  admin-selectable.

## Open items carried to later sub-projects

- Real logo file, favicon, and full brand visual polish (placeholder
  wordmark only here).
- `/admin/settings` screen to edit `site_settings` (table + seed only
  here; CMS sub-project builds the screen).
- `customers` table stays empty/unused until the booking sub-project.
