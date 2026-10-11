# Design: workspaces (multi-tenancy)

Status: proposal. Tracking: #178, feature 5, a 2.0 theme. For 1.5.0 this document is
the preparation. Nothing in the schema changes yet.

## Problem

One MATE installation serves one organisation. Projects group agents, but
everything else is shared across the installation:

- users and their roles;
- who is an administrator;
- budgets;
- the audit log;
- settings and provider keys.

An agency hosting agents for several clients, or a company with departments that
must not see each other's agents, conversations and costs, would need separate
installations.

A workspace is an organisation inside one installation. It has its own members,
agents, keys, budgets and audit log. A person can belong to several workspaces.

## What the code does today

| Area | Today | Where |
|---|---|---|
| Users | one global `users` table; `user_id` unique across the installation | `User` |
| Roles | a JSON list on the user (`admin`, `user`, `widget`, custom); global | `User.roles` |
| Administrator | yes or no, for the whole installation. Basic auth, bearer tokens, `AUTH_USERNAME` and the `admin` role are all admins | `server/auth.py` `get_user_role` |
| Dashboard API | every `/dashboard/api/` route is admin-only through `DashboardAuthzMiddleware`, unless allowlisted | `docs/dev/auth-and-access.md` |
| Projects | group agents; no members; `name` unique | `Project` |
| Owned by a project (`project_id`) | agents, memory blocks and their versions, widget keys, channel integrations, triggers, file search stores, response feedback | |
| Agents | `name` unique across the installation, and the same string is the ADK app name, the `/apps/{name}` path, the session key, `token_usage_logs.agent_name` and the `scope_id` of agent budgets | `AgentConfig.name` |
| Agent access | the user's global roles against the agent's `allowed_for_roles` | RBAC callback |
| Logs | `token_usage_logs`, `guardrail_logs`, `agent_responses` and `audit_logs` have no project; usage per project is found through agent → project | |
| Budgets and alerts | scope `user`, `agent`, `project` or global | `RateLimitConfig`, `AlertRule` |
| Installation-wide | model prices, system settings, migrations, provider keys (environment variables), personal access tokens (per user) | |

## The two hard parts

The schema is the easy part. Two things make this a 2.0 feature.

### 1. Agent names are global

An agent's name is also its ADK app name. That name is in URLs, in the session
store's keys, in every token usage row, and in budget scopes. Two workspaces can't
both have an agent called `support`.

- **Keep names unique across the installation (recommended for 2.0).**
  - Creating or importing an agent refuses a name another workspace uses, and says
    so without naming that workspace.
  - Templates already prefix their agent names (`template_prefix`), so imports
    rarely clash.
  - Nothing that uses the name has to change.
- **Give agents an internal name, such as `w12__support`, and show `support`.**
  - This would mean changing the app name everywhere: URLs, sessions, logs, budgets,
    A2A and MCP endpoints, standalone builds, and existing sessions.
  - It's a lot of risk for a cosmetic gain. It can come later if clashes become a
    real problem.

### 2. Every admin route assumes one organisation

There are 44 `_get_is_admin` checks in `dashboard_server.py` alone, and the
middleware makes every dashboard API admin-only. With workspaces:

- "admin" becomes "admin of *this* workspace";
- every query has to filter by the current workspace.

A single query that forgets the filter leaks another organisation's data. The work
is in going through the routes, not in the tables, and it needs a test that catches
a missed filter (see [Testing](#testing)).

## Proposed data model

**New tables:**

```text
workspaces
  id, name (unique), slug (unique), created_at, updated_at

workspace_members
  workspace_id → workspaces.id (cascade)
  user_id      → users.user_id (cascade)
  role         owner | admin | member
  created_at
  unique (workspace_id, user_id)
```

**Changed tables:**

| Table | Change |
|---|---|
| `projects` | `workspace_id` NOT NULL → `workspaces.id`; `name` unique per workspace instead of globally |
| everything with `project_id` | nothing: the workspace comes through the project |
| `token_usage_logs`, `guardrail_logs`, `agent_responses`, `audit_logs` | `workspace_id`, written when the row is written; nullable for installation-level events |
| `rate_limit_config`, `alert_rules` | a new scope, `workspace` |
| `personal_access_tokens` | `workspace_id`: a token acts in one workspace |

**Why copy the workspace onto log rows, instead of joining through the agent?**
- Agents are renamed, deleted and moved between projects. A cost or an audit entry
  must stay where it was recorded.
- Usage queries over large tables stay single-table.

**What stays installation-wide:**
- users (one identity across workspaces);
- migrations, server control and system settings;
- model prices. Per-workspace prices are an open question.

The installation's own administrator becomes the **platform administrator**: the one
who sees every workspace and manages these settings.

**Default workspace.** The migration creates workspace 1, "Default", and:
- moves every project into it;
- makes every user with the `admin` role its `owner`, and every other user a
  `member`;
- sets `workspace_id = 1` on existing log rows.

An installation with one workspace looks the same as today, and the dashboard shows
no workspace switcher until there is a second one.

## Access with workspaces

- **Current workspace.** The dashboard session holds the current workspace, chosen
  in a switcher. A personal access token carries its own. A request for a workspace
  the user isn't a member of is refused.
- **The dashboard API.** The middleware's admin check becomes "owner or admin of the
  current workspace". Routes for installation settings (migrations, server control,
  system settings, model prices) stay platform-admin only.
- **One place builds the filter.** A dependency gives each route its
  `(workspace_id, role)`. Routes take the filter from there, never from a query
  parameter.
- **Agents.** The RBAC callback first checks that the caller is a member of the
  agent's workspace, then `allowed_for_roles`, as today. Widget keys belong to a
  project and so to a workspace. Widget visitors are not members; their access is
  the key, as today.
- **Audit log.** Each workspace sees and exports its own entries. Platform admins
  see all of them. Retention stays installation-wide.

## What 1.5.0 does

Only this document. The schema change touches a dozen tables on three databases,
and without settled answers to the open questions it's easy to add a column we
would later redo.

**One cheap step pays off even without workspaces:** writing `project_id` onto
`token_usage_logs` and `audit_logs` when rows are written. Usage per project would
then not depend on the agent still existing in that project, and the 2.0 backfill
would be exact. It isn't part of 1.5.0; it's a candidate for the first release after.

## Open questions

1. **Provider keys per workspace.**
   - Today `GOOGLE_API_KEY`, `OPENROUTER_API_KEY` and the rest are environment
     variables, so every workspace spends on the same accounts.
   - A hosting setup usually needs each client's own keys, stored encrypted per
     workspace and passed to the model per call.
   - This is the largest piece after scoping, and it decides whether workspaces are
     for departments, where shared keys are fine, or for clients.
2. **Model prices per workspace**, for negotiated rates. They stay installation-wide
   until someone needs this.
3. **Moving a project to another workspace.** Allowed for platform admins? Its logs
   keep the old `workspace_id`, which is the point of copying it.
4. **Sign-up.** Do workspaces get created only by platform admins, or by anyone who
   signs in (self-service SaaS)? The first version is platform admins only.
5. **SSO.** Should an email domain map to a workspace (`OAUTH_ALLOWED_DOMAINS` per
   workspace), so new users land in the right one?

## Phases (2.0)

1. **Schema.** The tables, the default workspace, the backfill, and `workspace_id`
   written on new log rows. No behaviour change.
2. **Scoping.**
   - the workspace context and the switcher;
   - the middleware's per-workspace admin;
   - every dashboard route and the RBAC callback filtered by workspace;
   - the cross-workspace test.
3. **Management.** Workspace and member pages, invitations, and per-workspace
   budgets and alerts.
4. **Provider keys per workspace**, if the answer to open question 1 is "clients".

## Testing

- **A cross-workspace test**, over every route.
  - Fixture: two workspaces, each with a project, an agent, a widget key, usage,
    audit entries, a budget and a member.
  - For every `GET` route under `/dashboard/api/`, list them from the app's router
    so new routes are covered automatically. Call each as workspace A's admin, and
    assert that no id or name from workspace B appears in the response.
  - For every write route, assert that an id from workspace B gets 404.
- **The migration on all three databases:** the backfill puts every existing row in
  the default workspace, and an installation with one workspace behaves as before.
- **The RBAC callback:** a member of A can't run an agent of B, whatever their roles.
