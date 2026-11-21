# Admin API and admin UI: detailed design

Detail step 10 of the Källa design. The rules of the weakness review and the simplification, performance and
consistency passes are stated here (reasoning: [`../archive/passes/simplification-pass.md`](../archive/passes/simplification-pass.md) §1, §2, §5,
§7–§10, §13, §15). Details [`../archive/area-plans/admin.md`](../archive/area-plans/admin.md) (archived: history, not binding; this design states every rule, the plan's answered questions in §14) and the Decisions table
in [`../decisions.md`](../decisions.md). Consumes, without redesigning: the BFF, claim mapping,
`RagPrincipal`, internal callers, keys and security events of [`../identity/design.md`](../identity/design.md) (§1–§4,
§6, §10–§12); tables, grants, the outbox, encryption and retention of
[`../platform/data-model.md`](../platform/data-model.md); templates, parameters, preview, publish, rollback and site
profiles of [`../ingestion-worker/design-engine.md`](../ingestion-worker/design-engine.md); source settings, preflight,
crawl report, uploads, effect tags and the deletion guard of
[`../ingestion-worker/design-connectors.md`](../ingestion-worker/design-connectors.md) (§4–§5, §9, §12.3, §13); PII
flags and evaluation sets of [`../ingestion-worker/design-processing.md`](../ingestion-worker/design-processing.md)
(§9, §13–§14); `RetrievalSettings` and the test-panel scope of [`../knowledge-api/design.md`](../knowledge-api/design.md)
(§4.2, §4.5, §12.1); residency, not-found texts, output filter, retention and feedback of
[`../chat-api/design.md`](../chat-api/design.md) (§8.2, §9.4, §10, §13–§14); skills, tool servers, approvals, grants,
drift, policy, agent limits and `GatewayModels` of [`../tools/design.md`](../tools/design.md) (§3–§6, §9); projects,
build, images, networks, CI and observability of [`../platform/repo-and-deployment.md`](../platform/repo-and-deployment.md).
Where the code exists (`src/Kalla.AdminApi`, `src/admin-ui`), the text states what it does. Changes needed
elsewhere: §13; questions: §14; unverified facts: §15; deferred or removed: §16.

Gateway citations are `path:line` in `~/git/llmgateway` at `e25b491` (v0.8.5); `AdminApi/` = `src/Fors.AdminApi/`,
`ui/` = `src/admin-ui/src/`. "Copy" = copied with names changed; "adapt" = copied and changed as stated; "new" = no
gateway counterpart.

**Sources read for this step** (all read only, never executed):

| Source | Version | Used for |
|---|---|---|
| Gateway `~/git/llmgateway` | `e25b491` (v0.8.5) | admin API conventions (endpoints, validation, problem details, audit, pagination, CSP), admin UI (stack, shell, router, client, forms, tables, charts), Data API catalogue, E2E and accessibility tests, `docs/admin-ui.md`, `docs/admin-api.md`, `docs/accessibility-statement.md` |
| Open WebUI `~/git/rag-platforms/open-webui` | `f7ac7ee` (v0.12.5) | fixed feedback reason keys (`src/lib/components/chat/Messages/RateComment.svelte:17-35`, `:186-212`) |

**Gateway facts this design relies on** (checked in the source):

- The gateway admin UI's **texts are English**: `index.html:2` `lang="en"`, labels in `ui/utils/labels.ts` and
  components (nav labels in `ui/router/nav.ts:43-150`); only numbers use `sv-SE` (`ui/utils/format.ts:2-6`), dates
  `en-GB` (`:7-10`), time zone `Europe/Stockholm` (`:1`); Swedish names are marked `lang="sv"`
  (`ui/views/OrganisationView.vue:150`). The RAG admin UI follows the gateway: **English** texts and `lang="en"`, every
  string in a labels file (Decisions, "Admin UI in English").
- No general ETag/If-Match handling: concurrency tokens only on keys (`src/Fors.Infrastructure/Persistence/
  GatewayDbContext.cs:71-72`); any `DbUpdateConcurrencyException`, unique (23505) or FK (23503) violation becomes a 409
  (`AdminApi/Program.cs:20-27`). The client has `GET/POST/PUT/DELETE` only (`ui/api/client.ts:8`).
- The UI shows the server's `detail`/`title` verbatim (`ui/utils/problem.ts:7-8`); field errors are English
  (`docs/admin-api.md:5-6`, `AdminApi/KeyEndpoints.cs:31-55`).
- Pagination `page`/`pageSize` ≤ 200, `PageDto(items, total)` (`AdminApi/ReportEndpoints.cs:268-271`, `:160-169`); the
  Data API uses keyset feeds `after`/`limit` (`src/Fors.DataApi/DataEndpoints.cs:67-80`).
- The Data API catalogue has departments, teams, keys, providers (with `Residency`), models (name, kind, provider,
  residency), prices, budgets (`DataEndpoints.cs:139-160`) — **no routes/aliases**, no `Features`, no `ContextWindow`.
- Accessibility target WCAG 2.2 AA incl. 24 px targets (`docs/accessibility-statement.md:9`), axe A/AA on every routed
  page and on drawers and dialogs in three themes (`tests/e2e/specs/accessibility.spec.ts:6-9`, `:24-70`), target-size
  check (`tests/e2e/helpers.ts:160-172`); **no screen-reader check, "ej utfört och inte planerat"**
  ("not done and not planned"; `docs/accessibility-statement.md:22`, `:31`).

## 1. Summary of decisions

Moved to [`../archive/design-history/admin.md`](../archive/design-history/admin.md) on 2026-11-13 (M-14): history, not binding. A citation of §1 or a section under it resolves there.

## 2. Placement and interfaces

### 2.1 Projects

| Project | Owns (admin) | Notes |
|---|---|---|
| `Kalla.AdminApi` | BFF (identity §1, adapted from `AdminApi/AdminAuthentication.cs`), endpoint groups (§4.5), `AdminContext` (scope, version lock, audit, save), `ConfigRules` (§4.6), `JobEnqueuer` (§4.7), `ContentReader`/`ChatReader` (§4.8), `TestPanelService` (§4.9), `AssistantVersions` (draft, publish, restore, diff; §4.5), `AssistantTestRunner` (§7.4), `DepartmentMap` (§5.5.6), `AccessExplainer`, `OrganisationSync`, `GatewayModelSync` (§4.10), `RetentionService` (data-model §8), `ConversationAccessService` (§8); P2 `AnswerEvaluationService` (run kind `answers`, processing §14.8), `DefinitionDriftMonitor` (tools §4.4), `SkillGateRunner` (tools §3.7) | serves `src/admin-ui/dist` as `wwwroot` (`PublishAdminUi=true`) |
| `Kalla.Pipelines` | `PipelineValidator` (parameters against the template's schema, template version supported by a running worker) | shared with the worker |
| `Kalla.Retrieval` | retrieval core | test panel |
| **`Kalla.Answering`** | `KnowledgeService` core (access within a given scope, hydrate, sections, citations), chat answer core (`RagAnswerStrategy`, `PromptBuilder`, `TokenBudget`, `Condenser`, `AnswerFilter`, `CitationEmitter`, `LanguageGuess`), P2 `AgentAnswerStrategy` | the knowledge and chat roles keep HTTP, SSE, writer and importer; the test panel shows the real prompt, budget and filter (chat §9.4) |
| `Kalla.Tools` (P2) | in-process hub and MCP client (tools §2.1) | discovery, connection test, drift leader, test-panel tool calls |
| `Kalla.Domain` | `RagRole`, `AdminPolicy`, `PermissionMatrix` (data, §3.4), `ValidationCode`, `IAccessPolicy` (pure; used by explain access) | pure |
| `src/admin-ui` | Vue 3.5 + Vite 8 + Tailwind 4 + Pinia + vue-router (gateway `package.json`), copied shell, design system, client, composables (§5.1) | port 5174 |

### 2.2 Hosting, slots and background work

- One slot `kalla-admin` in the single-host pilot (one-slot deployment with a short announced maintenance window,
  simplification pass §1); `kalla-admin-b` with the a/b slots of P2. Control zone; sessions in Postgres
  (`config."AdminSessions"`, owned by `kalla-admin`; identity §1.1), so either slot serves any user; no Redis in P1
  (simplification pass §14).
- Networks: `edge` (proxy, inbound only), `data` (Postgres), `llm` (the default route, so also the OIDC authority and,
  from P2, Graph for feature C; repo-and-deployment §6.1; gateway: test panel, model validation, answer-level evaluation, P2 skill gate, the organisation and model syncs), P2 `mcp` (hosted MCP servers:
  discovery, connection test, drift polling, test-panel and skill-gate tool calls; the worker is not on it).
- Background services:

| Service | Interval | Leader | Writes |
|---|---|---|---|
| `OrganisationSync` | 5 min + on demand | a config write (version lock, data-model §5, since the mirror tables carry outbox triggers), then `pg_try_advisory_xact_lock` per run, as `AdminApi/ModelListSync.cs:148-153`; an unchanged run writes nothing and doesn't bump the version | `Departments`, `Teams` (changed rows only) |
| `GatewayModelSync` | 5 min + on demand | same pattern | `GatewayModels` (changed rows only) |
| `RetentionService` | nightly, 03:00 Europe/Stockholm (batches of 5,000 rows; both constants, only `Admin:Retention` is a knob) | pilot: the one slot runs it; from P2 each step batch is idempotent under `pg_try_advisory_xact_lock(<step>)` on both slots (reasoning: [`../archive/passes/performance-pass.md`](../archive/passes/performance-pass.md) §3.5); `Admin:Retention` (default `true`) stays an off switch | chat/audit/config deletions, expired admin sessions; **bucket-key destruction only when the shredding setting is on, which stays off until A1 is answered** (simplification pass §5) |
| `DefinitionDriftMonitor` (P2) | 15 min | advisory lock (the one drift leader) | `Tools.State`, repins from pre-registrations (tools §4.4) |
| `ConfigVersionMonitor` | `rag_config` + 2 s poll | every instance | none (keeps the admin's own snapshot for validators and the test panel) |
| `AnswerEvaluationService` (P2, weakness review §1) | nightly | `pg_try_advisory_xact_lock` per assistant run | `EvaluationRuns` of kind `answers` (faithfulness, citation precision) through `Kalla.Answering` and the judge at the assistant's ceiling (processing §14.8) |
| `AssistantTestRunner` (P2, weakness review §1) | on assistant publish/restore and **Run test questions** | `pg_try_advisory_xact_lock` on (assistant, version); an unfinished run is restarted after a restart | `EvaluationRuns` of kind `assistant-tests` and their `EvaluationResults` (§7.4) |
| `SkillGateRunner` (P2) | on publish and **Run again** | `pg_try_advisory_xact_lock` on the version id; an unfinished run is restarted after a restart | `EvaluationRuns` with `Trigger = skill-publish` (tools §3.7) |

- Tick leaders (`OrganisationSync`, `GatewayModelSync`, P2 drift monitor) keep `pg_try_advisory_xact_lock` and no
  in-memory state the next run needs (the in-memory sync status is display only, §4.10); in P2, with two slots, the
  config singletons (`Admin:Retention`) run as lock-guarded idempotent batches.
- Request body: 1 MB like the gateway (`AdminApi/Program.cs:20`), except the upload file route (50 MB + multipart
  overhead, `IHttpMaxRequestBodySizeFeature` per endpoint) with its own rate-limit policy; the proxy's
  `client_max_body_size` for that path is 60 MB.

### 2.3 BFF, CSP and how the UI is served

- BFF exactly as identity §1, against RAG's own registration `kalla-admin`: cookie names `__Host-kalla-admin`,
  `__Host-kalla-csrf`, readable `RAG-XSRF-TOKEN` (so `ui/api/client.ts:3` changes only the constant),
  `/bff/login|logout|user|session/extend`, 401 for `/api` without a session, 403 for missing rights,
  6002/6003/6005/6006 events.
- Pipeline order copied from `AdminApi/Program.cs:67-98`: security headers with the same CSP string
  (`default-src 'self'; script-src 'self'; style-src 'self'; img-src 'self' data:; font-src 'self'; connect-src 'self';
  object-src 'none'; base-uri 'self'; form-action 'self'; frame-ancestors 'none'`, `:68-73`), `no-store` on `/api` and
  `/bff`, Scalar with nonce for platform admins (`:74`, `:94-95`), `AdminActivityMiddleware`, authn/z, per-user rate
  limit (`Admin:RequestsPerMinute` 120), antiforgery, `/api` group with `AdminValidationFilter` (`:84-85`),
  `UseDefaultFiles`/`UseStaticFiles`/`MapFallbackToFile("index.html")` (`:96-98`). Authorization runs before the
  antiforgery check, as in the gateway, so an anonymous unsafe call is challenged before the CSRF check. OpenAPI and
  Scalar are the P1a set-up surface (§12).
- CSP holds for everything new: fonts self-hosted (fontsource), no inline script. Preview outputs, CSV exports and
  conversation views are fetched from `/api` (never blob URLs from another origin).
- Cross-links to the gateway admin (separate app, plan answered 1): the context's `admin_url` (gateway phase 1), else
  `Admin:GatewayAdminUrl`; member groups → `{url}/organisation`, a team's budget → `{url}/budgets`
  (`docs/admin-ui.md:97-99`), an assistant's or knowledge base's gateway key and its budget → the gateway's key page.

### 2.4 Interfaces to other components

| To | How | Used for |
|---|---|---|
| `config` | EF `RagDbContext`, role `kalla_admin` S I U D (data-model §1.2) | all configuration |
| `content` | S, plus I on `Jobs`, I U on `EvaluationRuns`, `EvaluationResults` (`answers` and `assistant-tests` runs, P2 skill gate) | documents, runs, reports, evaluation, previews, site profiles, test panel retrieval |
| `chat` | S, plus **U on `Feedback.ReviewState`** (§13) | feedback page, comment reveal, conversation review |
| `audit` | S I on `AuditLog`, `ConversationAccess`, `AuthFailures`; S on tool tables (P2 S I on `ToolDefinitionObservations`) | audit list, access log |
| Blobs | `kalla-blobs` rw (uploads; preview outputs read); `internal` blobs unwrapped with the level's master key, `public` blobs are not envelope-encrypted (simplification pass §4) | uploads, preview outputs, PII highlight previews |
| `IKeyProvider` | chat master keys (file provider in the pilot), once A1 allows stored content (weakness review §4) | comment and conversation decryption (§8) |
| Gateway | the admin's credential key `Gateway__Key`, acting as an assistant's or knowledge base's assigned key from phase 2 (identity §4.7); from phase 3 the Integration API with the same key's `usage.read`; the same key's `organisation.read` for the organisation sync (§4.10) | test panel embedding + answer, `/v1/models` validation (§4.6 V8, V39), answer-level evaluation and assistant test questions with the judge, P2 skill gate, cost and budget status, §4.10 |
| Hosted MCP servers (P2) | in-process `Kalla.Tools`, `client-credentials` with the `kalla-tools` certificate (tools §4.6) | discovery, connection test, drift polling, test-panel and skill-gate tool calls |

## 3. Roles and permissions

### 3.1 Roles and policies

Identity §1.3; policies stay `RequireRole(any of …)` as in the gateway (`AdminApi/AdminAuthentication.cs:151-158`), so
roles add up.

| Role | Phase | Policies | Scope |
|---|---|---|---|
| `rag-platform-admin` | P0 | `Admin`, `Manage`, `EditPipelines`, `EditSkills`, `Edit`, `Read` | every department; platform objects |
| `rag-department-admin` | P0 | `Manage`, `EditPipelines`, `EditSkills`, `Edit`, `Read`, **`FeedbackComments`** | `departmentCodes` + feature C |
| `rag-pipeline-editor` | P0 | `EditPipelines`, `Read` | same |
| **`rag-reviewer`** | P1 (defined; nothing to read until A1 allows stored content, weakness review §4) | **`Review`**, **`FeedbackComments`**, `Read` | same |
| `rag-skill-editor` | P2 | `EditSkills`, `Read` | same |
| `rag-editor` | P2 | `Edit`, `Read` | same |
| `rag-viewer` | P0 | `Read` | same |
| no `rag-*` role | – | none: every `/api` call 403, UI shows the no-access page | – |

`Review` and `FeedbackComments` are **not** part of `Admin`. A platform admin who must review is also given
`rag-reviewer` (visible in the audit actor roles). `rag-reviewer` is granted per department: in P1 as the realm role `rag-reviewer`
in the shared Keycloak, given to the account or a Keycloak group per department (identity §1.6, §11.2, 2026-10-31);
from P2 by assigning a group per department (Entra app role assignment on `kalla-admin` or
*Settings → Role groups*, `Settings.Identity.RoleGroups`). The UI mirrors the policies
with an implied-roles map as the gateway does (`ui/stores/auth.ts:10-15`, `:60-69`), but the API decides.

### 3.2 Department scoping and sharing

- `AdminContext.Departments()` copied (`AdminApi/AdminContext.cs:81-90`) plus feature C (claims only in P1, identity
  §2.3); every query of a department-owned entity (`Collections`, `Sources`, `Pipelines`, `Assistants`, `Skills`, `Keys`
  via teams, evaluation and run data via their collection/source) filters by it. An id outside the scope is **404**,
  never 403.
- Platform objects (templates, site profiles, embedding profiles, clients, `GatewayModels`, tool servers, settings,
  data-protection policies) are readable where a form needs them (pickers show name, kind, level, residency only; client rows also carry `portalUrl` for the "Open in <client>" link, §5.4 step 13) and
  writable by `Admin` only.
- **Sharing** (Decisions "Organisation: department-owned, municipality-shared knowledge bases with topics (user, 2026-11-02)"): the owning department's `Manage` sets a collection's sharing level: *only our
  department* (default), *selected departments* (`CollectionShares`) or *whole municipality* (every active department in
  the gateway catalogue). Skills (P2) are shared with named departments (`SkillShares`, `EditSkills`). An assistant links
  collections of its own department and those shared with it, and skills of its own department, shared with it or
  platform skills (V11, V12). Sharing gives read use through assistants only: another department sees a shared
  collection's name, owning department, level, contact and topics (§4.5), never its sources, documents, settings or
  evaluation, and cannot edit or publish it. Narrowing the sharing is refused while another department's published
  assistant links the collection (V35).
- **Changing the owner department** (Decisions "Moving to another department"; `Admin` only, #811): *Change
  department* on a knowledge base or an assistant moves it, its sources, documents, runs, versions and test questions
  with it. First `…/move/impact` says what the move does, then `…/move` does it with the impact's token (a hash of the
  object's id, version, the request and the shares to add, so a change in between answers `move_not_confirmed`).
  - **Links kept, only those that work today** (#821 review: a draft may link a knowledge base it cannot reach, V11 being
    a warning there). A knowledge base moving out: a department whose assistants (published version or draft, archived
    ones too) link it gets a share when its link works today (it is the old owner, or the sharing reaches it now), the
    new owner's sharing would not reach it, and it is active; level *only our department* becomes *selected
    departments* with just those shares, after dropping the stale rows kept at that level. The new owner's own share row
    goes, and a *selected* list left empty falls back to *only our department*. An assistant moving: each knowledge base
    its published version or draft links, owned by another department, linkable for its department today and not
    reaching the new one, gets a share for the new one. Links that don't work today stay broken and the impact lists
    them (`brokenLinks`). The shares are listed in the impact and audited `share` on each knowledge base, at its level,
    with the reason `move`.
  - **Owner team and key**: the request names an owner team of the new department, one of its active teams; without
    one the move takes the department's only active team, none when it has no team, and `invalid_value` on
    `ownerTeamId` when it has several (the impact answers the team it takes). The gateway key is one of the new
    department's (or none); V39 checks the key as on save, and a published assistant without a key is refused once
    the gateway acts as keys (`gateway_key_required`).
  - **Pipelines**: a knowledge base with a live source on a pipeline of the old department is refused
    (`move_pipeline_department`, listing the sources).
  - **What stays**: cost and usage rows (`AssistantDailyUsage`, `AssistantDailyTotals`, `CollectionDailyUsage`) keep
    the department of their calls, by day: the row of the move day is one per object and step, so an assistant's keeps
    the department of its first call that day and a knowledge base's takes the department of its last; audit entries
    keep theirs. Queued jobs keep their fairness department until they run. Department parse rules follow the
    knowledge base's department from the next parse. An assistant's conversations and feedback follow the assistant:
    the new department's reviewers see them, as they see everything else about it.
  - **Client keys**: a live key of another department scoped to the object keeps its scope and keeps working (the
    run-time scope is by id); V23 no longer holds for it, so its next scope save is refused until the object is taken
    out, and only its own department's admins see it. The impact lists these keys (`scopedKeys`); the move changes no
    key.
  - **Audit**: `move` on the object under the new department, its `departmentId`, `ownerTeamId` and `gatewayKeyId`
    before and after, and the same entry under the old department, so both departments' admins see it.
- **Topics**: the tags the sources' path rules put on chunks (processing §7). A link from an assistant to a collection
  may name topics; retrieval then keeps only that collection's chunks carrying one of them (knowledge §4.2, §6.2).
- **Ownership of shared content**: content used across the municipality is owned by the department that runs it (umea.se:
  Communications; DoIT until Communications has admins). The platform team owns no content.
  Knowledge reaches an assistant only through its own collection links; skills carry know-how and tools, never
  collections (simplification pass §15).
- **Owner-team scoping (P2, simplification pass §15)**: assistants, collections (with their sources, pipelines and
  test questions) and skills keep an owner team (`OwnerTeamId`). A department may switch on
  `DepartmentSettings.EditOwnerTeamOnly` (`Manage`, audited `departmentsettings.update`); while it is on, an `Edit`,
  `EditPipelines` or `EditSkills` write to such an object needs membership of its owner team (identity §1.3) **or**
  `Manage` in the department, so department and platform admins are unaffected. Reads are unchanged. A refused write
  is 403 `owner_team_only` (6006); creating or copying sets the owner team to one of the writer's teams; changing an
  object's owner team needs `Manage` while the setting is on. `AdminContext.CanWrite(entity)` applies it after the
  department filter, and the generated role tests cover it (§11.2).
- A source belongs to exactly one collection; keys only for teams of the department (identity §6.4).
- Users with several departments get a department filter in the page header (URL query `dept=`), defaulting to all.

### 3.3 Pages per role

R read, W write, – hidden (route guard → `/forbidden`, copied `ui/router/index.ts:87-98`). PA platform admin, DA
department admin, PE pipeline editor, RV reviewer, SE skill editor (P2), ED editor (P2), VW viewer. "Generic" = generic
schema-driven form (§5.8).

| Page (UI label, §5.7) | Phase | PA | DA | PE | RV | SE | ED | VW |
|---|---|---|---|---|---|---|---|---|
| Home | P1 | R (all + platform health) | R | R | R | R | R | R |
| Map (department map) | P1b, table first (§5.5.6) | R | R | R | R | R | R | R |
| Knowledge bases (incl. Documents, Test questions and quality, Search settings, Topics, Sharing (`Manage`)) | P1 | W | W | R | R | R | W (no sharing) | R |
| Sources, runs, files, deletions (incl. template parameters) | P1 | W | W | W (template parameters only) | R | R | W (no deletion decisions) | R |
| Publication (plan, report, keep/restore) | P1 | W | W | W | R | – | W | R |
| Assistants (incl. Test, Versions; Test questions P2, weakness review §1) | P1 | W | W | R | R | R (+ test) | W | R |
| Audience (assistant audiences) | P1 | W | W | – | R | – | R | R |
| Feedback | P1 (comments when A1 allows stored content, weakness review §4) | R | R + comments | – | R + comments | – | – | R (no comments) |
| Conversation review, Access log | when A1 allows stored content (weakness review §4; limited first, then full) | access log R | access log R (own depts) | – | W | – | – | – |
| Keys (generic) | P1 | W | W (own teams) | – | – | – | – | – |
| Settings, Data protection, Clients, Templates, Assistant templates (P2, weakness review §1), Site profiles, Embedding profiles, Gateway models, Organisation (generic) | P1 | W | R (data-protection matrix, models, clients) | – | – | – | R (models) | – |
| Department settings (owner-team scoping, generic) | P2 | W | W (own departments) | – | – | – | R | R |
| Audit log (list) | P1 | R (all) | R (own departments' entries) | – | – | – | – | – |
| Costs (per department, §8.6; the cost cards on the assistant and knowledge base pages follow those pages' rows) | P1 (cost attribution stage 1) | R (all) | R (own departments) | – | – | – | – | – |
| Parse-profile proposals | P2 | W (approve, reject) | R | R | – | – | R | R |
| Skills | P2 | W (platform skills) | W | – | R | W | R (link only) | R |
| Tool servers | P2 | W (register, approve, grant) | R | – | – | R | – | – |

A system owner of a tool server (any `rag-*` role, tools §4.5) sees that server's approval and pre-registration page
regardless of the matrix; the API checks `ToolServers.SystemOwnerSubjects` (`oid`). Departments ask for a tool server
or a grant through the platform team's normal intake; there is no request flow in the admin.

### 3.4 API actions per policy

The matrix is data (`PermissionMatrix` in `Kalla.Domain`), used for endpoint authorisation and generated tests (§11.2).

| Policy | Actions (department-scoped unless noted) |
|---|---|
| `Read` | list/detail of everything in scope; runs, reports, documents (metadata), evaluation reports, feedback metadata, the cost of an assistant or own knowledge base (§8.6) |
| `Edit` | collections (create, edit, retrieval settings, test retrieval, run evaluation), sources (create, edit, pause/resume, sync now, retry failed, upload, delete files), assistants (create, edit the draft, publish, restore a version, archive/unarchive, test panel; P2: create from a template, copy, assistant test questions and runs), human test questions, generate test questions, link served skills (P2); P2 limited to the owner team when the department says so (§3.2) |
| `EditPipelines` | template parameters of the department's sources (preview, plan, publish, roll back), pipeline test retrieval |
| `EditSkills` (P2) | skills of own departments: draft, publish, serve, roll back, share list, re-run the gate; skill test panel |
| `Manage` | collection publish, collection sharing level and departments, delete collections/sources, deletion confirm/reject, release holds, PII clear (a decision on the document), assistant audiences, retention and history mode within policy, keys of own teams, document reprocess, explain access for own department's assistants and clients, audit log (own departments, §9.1), the cost report per department with its CSV (§8.6), P2 department settings (owner-team scoping) and owner-team changes while it is on |
| `FeedbackComments` | reveal one feedback comment (`public`/`internal`) |
| `Review` | open a conversation with a reason; P2: request/approve `sensitive` access, search conversations |
| `Admin` (platform) | templates, assistant templates (P2), site profiles (edit, roll back; approve P2), embedding profiles, clients, `GatewayModels` manual rows, settings, data protection, raised agent limits, tool servers, approvals (platform side, incl. pre-registrations), grants, platform skills, parse-profile proposals (P2), audit log (all), organisation sync now, explain access |

## 4. Admin API

### 4.1 Conventions

| Topic | Rule | Source |
|---|---|---|
| Shape | Minimal API groups per area under `/api`, `MapGroup(...).RequireAuthorization(policy).WithTags(...)`, `.WithName()`/`.WithSummary()`; thin endpoints, logic in services; JSON camelCase, enums as strings, ISO-8601 with offset, money in USD fields `…Usd` | `AdminApi/KeyEndpoints.cs:59-80`, `docs/admin-api.md:3-6` (copy) |
| Requests | sealed records deriving `AdminRequest` with DataAnnotations + `IValidatableObject` for field rules; these records are also the source of the generic forms' schemas (§5.8) | `KeyEndpoints.cs:11-55`, `AdminContext.cs:216-258` (copy) |
| Errors | RFC 9457 problem details via `ApiFaultException(status, detail, extensions)`; validation = 400 with `errors`; 23505/23503/concurrency → 409 | `ServiceDefaults/ApiFaultException.cs:12-33`, `Program.cs:20-27` (copy) |
| Error codes | every non-validation fault has extension `code`; validation faults add `errorCodes: { field: [{ code, params }] }` (§4.2) | new |
| CHECK violations | 23514 → 400 `code: platform_rule` with the constraint name | new |
| Concurrency | `version` in DTOs and update/delete bodies (§4.3) | new (gateway: keys only) |
| Audit | snapshot records `XxxAudit.Of(entity)` with own columns only, never secrets; `Action` verb + `EntityType`; actor `oid` (fallback `sub`); staged with the change | `AuditSnapshots.cs:11-80`, `AdminContext.cs:141-169` (adapt: §4.4) |
| Propagation | no Redis invalidation bus: the outbox is in the transaction (data-model §5) | replaces `AdminContext.cs:177-191` |
| Pagination | small lists `page`/`pageSize` (≤ 200, default 50) → `PageDto(items, total)`; large lists (run items, documents, feedback, audit, access log) keyset `after`/`limit` → `{ items, next }` | `ReportEndpoints.cs:268-271`; `DataEndpoints.cs:67-80` |
| Search | `q` on list endpoints; `GET /api/search?q=` for the command palette (collections, sources, assistants, skills, documents by title; ≤ 20 hits, scope-filtered) | new |
| Rate limit | per user 120/min (`Admin:RequestsPerMinute`); upload files and job polling have own policies (600/min each) | `Program.cs:53-64` (copy + 2 policies) |
| OpenAPI | `/openapi/v1.json` + Scalar for `Admin` (in P1 also the action UI for platform objects without a dedicated page); the error-code list is exported as `contracts/admin/v1/error-codes.json` (CI checks the UI covers every code) | `Program.cs:94-95` (copy) |
| Data reads | 6007 `rag.admin.data.read` on audit log, keys, feedback lists, access log, conversation views (not explain access: nothing is accessed, identity §4.9) | identity §10 |

### 4.2 Error codes and UI texts

The gateway UI prints the server's message (`ui/utils/problem.ts:7-8`). The API's `detail` is written for logs and
Scalar, not for department admins, so `problem.ts` is adapted: `problemMessage` looks up `problem.code` in
`ui/utils/messages.ts` (templates with `{params}`, English now, the place a Swedish translation goes later), then a
status text, and shows the server's `detail` only as a last resort. `mapFieldErrors` (`:26-36`) prefers `errorCodes[field]`.

```json
{ "type": "https://tools.ietf.org/html/rfc9110#section-15.5.1", "title": "Check the highlighted fields.", "status": 400,
  "errors": { "classification": ["The assistant's level (public) is below the collection 'bygglov' (internal)."] },
  "errorCodes": { "classification": [ { "code": "assistant_below_collection",
                                         "params": { "collection": "Bygglov", "level": "internal" } } ] } }
```

```ts
// ui/utils/messages.ts (excerpt)
assistant_below_collection: "The assistant's level is below the knowledge base {collection} ({level}). Raise the assistant's level or remove the knowledge base.",
concurrent_change: 'Someone else changed this while you were working. Reload the page; your changes are still in the form.',
```

### 4.3 Concurrency

- Every editable aggregate root maps Postgres `xmin` as a row-version token (`uint`, Npgsql EF; *to verify*, §15).
  Child rows (`AssistantAudiences`, `CollectionShares`, `SkillVersionLinks`, …) are saved through the root, and every
  root save sets `UpdatedAt`/`UpdatedBy`, so `xmin` changes whenever any part of the aggregate does.
- DTOs carry `version` (string); `PUT`/`DELETE`/state actions require it; mismatch → 409 `concurrent_change` with the
  current `version`. The form keeps the user's input and offers "Reload and compare".
- An assistant's draft (its `AssistantVersions` row with `State = draft`) carries its own `version`, so editing the
  draft and editing the assistant's own fields (audience, level, retention) do not conflict; publish and restore
  require both.
- Singletons (`Settings`, `DataProtectionPolicies` per level) use the same rule; `Settings` has one `version` (the
  singleton's `xmin`) for all its sections (§4.5).
- Job-backed actions are idempotent by `DedupKey` (§4.7), not by version.

### 4.4 Write path (one transaction, serialised)

```
BEGIN; SET LOCAL lock_timeout = '5s'
  1. SELECT … FROM config."ConfigVersion" FOR UPDATE      ← every admin config transaction, before anything else
  2. load aggregate (+ version check)
  3. apply request; ConfigRules (§4.6) against state no other admin write can change until COMMIT
  4. SaveChanges → row triggers write ConfigOutbox and bump ConfigVersion (data-model §5)
  5. SELECT "Version" FROM config."ConfigVersion" → v
  6. INSERT audit."AuditLog" (ActorOid…, Action, EntityType, EntityId, Details {before, after}, DepartmentId,
     ConfigVersion = v, Reason, Classification, Service = 'kalla-admin', the container and OTEL service name)
  7. INSERT content."Jobs" (§4.7) when the action enqueues work + pg_notify('rag_jobs')
COMMIT  → rag_config and rag_jobs notifications delivered
```

- **Why the lock first** (simplification pass §9): cross-rules read several aggregates (an assistant's level, its
  collections' levels, its clients' ceilings); two concurrent saves could each pass them and jointly break one (write
  skew). Taking the `ConfigVersion` row lock before reading makes admin config writes serial. Admin writes are a few
  per minute, so the cost is nil; a lock wait over 5 s answers 503 `config_busy` (retry) and is measured
  (`rag.admin.config_lock.wait`).
- **No gateway call under the lock** (#696): the lock is cheap only while nothing slow runs under it. The gateway's
  model lists that V8 and V39 check (`GatewayModelCheck`) are memoised per request by (capped ceiling, key) and fetched
  in parallel **before** step 1 for the pairs the write will check (client, assistant and collection writes);
  failures are memoised too and refused under the lock as before. A pair the locked code needs that was not fetched is
  still asked there, so correctness never depends on the pre-fetch. The pre-fetch calls as a key only when it is a
  usable key of the object's department. Each model-list call has its own 5 s timeout (`gateway_unavailable`). Settings
  V32 is the exception: it lists models under the lock, with the memo and the timeout, as settings saves are rare.
- Adapted from `AdminContext.SaveAsync` (`AdminContext.cs:158-169`): the audit entry is staged with the change; the
  actor stamp (`AdminActor.Stamp`, `:38-49`) and the 6001 event after save (`:50-58`) are copied.
- Audit-only actions (reveal a key, read a comment, read the audit log) use `AuditAsync` without the lock: the audit
  entry is saved (with the current `ConfigVersion`) before a revealed secret is returned (#696).
- Writes to `chat."Feedback".ReviewState` and `audit."ConversationAccess"` are two transactions; the access event is
  written **first**, so a crash never leaves an unaudited read (§8.2).

### 4.5 Resources per area

Policy in brackets; department scope applies. Bodies and DTOs are the data-model columns unless noted. Two policies
separated by a comma (`Edit, EditPipelines`) mean either suffices (a combined policy in `PermissionMatrix`).

**Organisation, models, access**

| Method and path | Policy | Notes |
|---|---|---|
| `GET /api/organisation/departments` | Read | the mirror: departments in scope (all for a platform admin), active first, each with `SyncedAt`, its teams nested (name, code, active), its member groups and a sharing summary (`sharing`: its live knowledge bases per sharing level, and how many other departments its `selected` ones are shared with); there is no separate teams route |
| `GET /api/organisation/departments/names` | Read | every active department's id and name, outside the caller's scope too (the *Selected departments* picker of the Sharing tab); names only, as pickers read platform objects (§3.2) |
| `GET /api/organisation/sync`, `POST /api/organisation/sync` | Admin | last runs and outcome (held in memory per instance, §4.10); run now |
| `GET /api/gateway/models` `?level=` | Read | `GatewayModels` with `modelKind` (`chat`/`embedding`/`rerank`), residencies, `SupportsTools`, context window, prices, `Source`, and **`available`** (`true`/`false`/`null`): listed by the gateway's `/v1/models` called with the admin's credential key at the level's ceiling (`level` = an assistant level; absent: the platform ceiling) and, with `gatewayKeyId` and from gateway phase 2, as that key (`x-fors-on-behalf-of`), cached 5 minutes per ceiling and key; a `gatewayKeyId` that is not a `GatewayKeys` row of a department in the caller's scope is refused (`404`) before any gateway call, as `GET /api/gateway/keys`; `null` when the gateway could not be asked (#447) |
| `POST /api/gateway/sync`; `PUT /api/gateway/models/{name}` | Admin | run the model sync now; manual row (aliases, `SupportsTools` until the gateway sends `features`; not built yet); on any row, `contextWindow` and `maxOutputTokens` (null clears; integers 1–10,000,000, `maxOutputTokens` ≤ `contextWindow` when both are set), kept by the sync until the gateway sends them on `/v1/models` (§4.10); from then on the sync's values replace them and the fields are read-only |
| `GET /api/gateway/keys?departmentId=` | Read (the department in scope) | the key picker: the department's rows of `GatewayKeys` (§4.10), each with its read-only settings (models, residencies, PII policy, content guard, rate limits, output cap, `hasSystemPrompt`), status and expiry, `usable` (V39's status part and the gateway's own `usable`; an unusable key is shown, not selectable, with its reason) and the gateway link (`admin_url`); never a secret |
| `GET /api/gateway/keys/{id}/budget` | Read (the key's department in scope) | from gateway phase 3: the key's active budgets from `/integration/v1/budgets?keyId=` (gateway-changes §5.6), each `scope` (`key`, `team`, `department`, from the gateway's `VirtualKey`, `Team`, `Department`), `unit` (`usd`, `tokens`), `limit`, `spent`, `percentUsed`, `exhausted`, `residency`, `period` with `periodStart`/`periodEnd`, cached 5 minutes per key; `null` when `Gateway:IntegrationApiUrl` is unset or the gateway cannot be asked (`401`/`403`/`429`: the *Gateway* page shows the reason) |
| `GET /api/gateway/status` | Admin | per service key: one `x-fors-max-residency` embedding call; the context's `version` and `features`, and once they contain `act-as-key` one acted call's echo (`GET /v1/models` as the first usable mirrored key: no cost); each profile's canary: recorded, last check time and similarity |
| `GET /api/gateway/canary-check` | Admin | each active profile's last canary check from the database (`{ profileId, profile, recorded, similarity, passed, checkedAt }`, `passed` null when never checked); makes **no** gateway call, so opening the page costs nothing |
| `POST /api/gateway/canary-check` | Admin | the embedding canary on demand (processing §11): for each active profile with a recorded canary, one embedding call at the profile's ceiling; stores `CanaryCheckedAt` and `CanarySimilarity`, audited `check-canary` on the profile; returns per profile `{ profileId, profile, similarity, passed, checkedAt, error, detail }` (`error` null when the check ran, else why it could not, e.g. `gateway_not_configured`; nothing is stored then). Nothing calls it on a timer. The platform's gateway models page has the button (§5.8) |
| `GET /api/organisation/departments/{id}/map` | Read | the Map (§5.5.6) as rows: department → collection (own, and shared in with its owning department) → assistant (the department's own assistants: published links; draft-only links flagged; the link's topics) → audience (clients), each with its contact; an own collection that none of its own assistants links gets a row without an assistant (other departments' use is in the collection's `usedBy[]`); a platform admin may ask for any department; from the admin's snapshot; no content |
| `GET/PUT /api/organisation/departments/{id}/settings` (P2) | Read / Manage | `DepartmentSettings` (`EditOwnerTeamOnly`, `version`); audited `departmentsettings.update` (§3.2) |
| `POST /api/access/explain` `{ clientId, subject, assistantId, action? }` | Admin; Manage (own department's assistants and clients) | runs the pure `IAccessPolicy` steps (identity §4.3) against the admin's snapshot for that client, subject (`oid`, key id or Open WebUI user) and assistant; returns `{ allowed, step, reason, status, effectiveLevel, maxResidency, configVersion }` (`step` = the stable step id, `reason` = the stable deny code, identity §4.9); no content; no security event |

**Collections and retrieval**

| Method and path | Policy | Notes |
|---|---|---|
| `POST /api/collections/{id}/move/impact`, `POST /api/collections/{id}/move` | Admin | §3.2 *Changing the owner department*: body `{ departmentId, ownerTeamId, gatewayKeyId, version }` (+ `impactToken` on the move); the impact answers `{ shares: [{ collectionId, collection, departmentId, department }], sourcesOnDepartmentPipelines: [...], ownerTeamId, brokenLinks: [{ assistantId, assistant, department, collectionId, collection }], scopedKeys: [{ keyId, key, department }], impactToken }`; the move answers the collection's detail |
| `GET/POST /api/collections`, `GET/PUT/DELETE /api/collections/{id}` | Read / Edit / Manage (delete) | create: name, slug (from name, ASCII), department, owner team, classification, contact, optional `gatewayKeyId` (V39); profile = default; an empty owner team becomes the creator's first active team in the department by name (from the `teamCodes` claim), none when the creator has no team there; the owner's detail (`shared: false`) carries `sharedWith`, `sharedWithDepartmentIds` (with `selected`) and `usedBy[]` (assistants whose published version or draft links it, with `published`/`draft`), which the save drawer shows as "Affects the assistants: …"; delete (Decisions "Deleting knowledge bases, sources and assistants"; a soft delete, the slug free at once, the documents out of the index at once) refused while a published version or a draft of an assistant that is not deleted links it, an archived one too (409 `collection_in_use` with the list; superseded versions do not block, a restore of one fails V34); also refused while it has sources that are not deleted (same code, `sources` in the list), so the worker never writes into a deleted collection: delete the sources first. A department the collection is shared with gets the shared summary of the list row (`shared: true`) from `GET …/{id}`, the read-only Overview of §5.5.2; every other route of the collection answers it 404 |
| `GET /api/collections?view=mine\|shared\|all` | Read | `mine` (default): the caller's departments' collections; `shared`: collections of other departments shared with one of the caller's departments or with the municipality, as the shared summary (name, owning department, level, contact, document count, topics; no sources); `all`: both. The assistant's collection picker reads `all` |
| `PUT /api/collections/{id}/sharing` `{ sharedWith, departmentIds, version }` | Manage | `sharedWith` = `department`/`selected`/`municipality`; `departmentIds` only with `selected`, at least one, active departments other than the owner (otherwise 400 `invalid_value`); at the other levels the share rows are kept but unused; narrowing is refused (V35) with the linking assistants and their departments; audited `collection.share`; the detail's `usedBy[]` lists other departments' assistants too |
| `GET /api/collections/{id}/topics` | Read (also a department the collection is shared with) | the tags on the collection's live chunks with document counts; for the owning department also the path rules that set each one (`sourceId`, `sourceName`, `on`, `match`), `rules: null` for a department it is only shared with (path rules are source settings, §3.2); read from `content`, no text |
| `PUT /api/collections/{id}/retrieval-settings` | Edit | saved directly within the V16 ranges; enqueues an advisory evaluation run (§4.7) whose report is compared with the previous run (none when the collection has no evaluation set) |
| `POST /api/collections/{id}/test-retrieval` | Edit, EditPipelines | retrieval-only test panel (§4.9), incl. unpublished |
| `POST /api/collections/{id}/publish` | Manage | sets `PublishedAt`; the latest evaluation report (if any) is shown before confirm; never blocked |

**Sources, runs, uploads**

| Method and path | Policy | Notes |
|---|---|---|
| `GET /api/collections/{id}/sources` | Read | the collection's sources, each with its live document count (`documents`, §5.5.2) and `lastRun` (id, state, start, end) |
| `POST /api/collections/{id}/sources`, `GET/PUT/DELETE /api/sources/{id}` | Edit / Manage (delete) | settings per `contracts/sources/v1/{type}.schema.json`; template + parameters (§6.1); schedule default per type; `ContainsThirdPartyText` (default true) writable by `Admin` only (tools §5.4, V13); delete: a soft delete at any time, a running run is cancelled, and the worker soft-deletes the source's documents at once (data-model §8), so nothing of it is found from then on |
| `POST /api/sources/preflight` | Edit | unsaved settings → `preview` `preflight` job (§4.7) |
| `POST /api/sources/{id}/impact` | Edit | effect tags of a changed settings object, the source's live document count (a `scope` change removes those that fall outside at the next fetch; the exact number would need the worker to check the new scope against the frontier) and the collection's `usedBy[]` ("Affects the assistants: …"); returns `impactToken` that the `PUT` must echo when the tag is `scope`/`identity` (V25). The token is a hash that proves the impact was asked for this exact change from this exact version; it is not an authorization |
| `POST /api/sources/{id}/sync`, `/retry-failed`, `/pause`, `/resume` | Edit | sync now answers 200 with the existing run when one is queued/running (dedup); sync now and retry failed are audited as actions on the source (`sync`, `retry-failed`) and move no config version |
| `GET /api/sources/{id}/runs` (keyset), `GET /api/runs/{id}`, `GET /api/runs/{id}/items` (keyset; `outcome`, `reason`, `q`) | Read | crawl report from `Runs.Report` (incl. the site-profile version used, extraction quality, the "no complete traversal for N runs" alert, and `malwareScan: "off"` for a run whose files were not scanned, connectors §11) |
| `POST /api/runs/{id}/cancel` | Edit | `command` `cancel-run` |
| `POST /api/runs/{id}/continue` | Edit | a `sync` job of kind `continue` (`continuesRunId` = the run, trigger `manual`, priority 10, dedup `sync:{sourceId}`, as sync now; a sync already queued or running is answered instead); only for the source's latest crawl run, ended and not awaiting a deletion decision, that left an open frontier row or a failed page (`canContinue` on `GET /api/runs/{id}`), else 409 `run_not_continuable`; audited as `continue` on the run (connectors §5.4) |
| `GET /api/runs/{id}/deletions` (+ `?format=csv`), `POST /api/runs/{id}/deletions/confirm`, `/reject` | Read / Manage | connectors §13.3–§13.5; body carries `proposedCount` seen by the user; 409 `deletion_superseded` if a newer run took over; the `GET` carries `guard` (the configured deletion-guard limits, for the card's text, so its readers need no Settings route; #487) |
| `GET /api/sources/{id}/held`, `POST /api/sources/{id}/held/release` | Read / Manage | `command` `release-hold` |
| `POST /api/sources/{id}/upload-batches` → `{ batchId }`; `PUT /api/upload-batches/{batchId}/files/{n}` (raw body, `X-File-Name`); `POST /api/upload-batches/{batchId}/commit`; `DELETE /api/upload-batches/{batchId}` | Edit | per-file progress and retry; `X-File-Name` is a percent-encoded UTF-8 **relative path** (§5.5.7): `\` read as `/`, a leading `/` dropped, 1–10 segments of 1–255 characters, no `.` or `..` segment, no control characters, at most 300 characters in all; it is stored as the upload's file name, which is the document's display name, so path rules `on: path` match its folders; extension pre-check 415 on the last segment; SHA-256 while streaming; commit writes the manifest and the `sync` job; uncommitted batches swept after 24 h. Identity is (source, upload slot), so replacing a file keeps the document id |
| `GET /api/sources/{id}/files` (`deleted`; `q` = case-insensitive substring of the path; `state` = `processing`, `indexed` or `failed` (§5.5.7); `sort` = `name` (default, path order, so a folder's files are together) or `uploaded` (newest first); opaque `after` cursor, `limit`), `GET /api/sources/{id}/files/summary` → `{ total, processing, indexed, failed, bytes, lastUploadAt }` (deleted files not counted), `POST /api/sources/{id}/files/restore` | Read / Edit (restore: platform admin, #489) | restore brings a deleted uploaded file back within its 14-day grace, else 409 `file_not_restorable`; crawled documents have no restore route; connectors §9.3; a delete is an upload batch whose commit carries `deletes` (upload slot ids; the same manifest, `sync` job (dedup `sync-upload:{batchId}`) and audit entry as an upload, so there is one write path); each file carries `scannedAt`, null for a file that was not scanned for malware (connectors §9.1) |
| `GET /api/sources/{id}/quarantine` | Read | metadata only |
| `GET /api/sources/{id}/pii-report` | Read | the web editors' personal-data report, a list per source: it applies only to a `piiMode: report` source in a `public` collection (any other answers `applies: false`); held documents first, then live documents whose live set is flagged, each with the non-zero counts per type, the number of fired rules and the fired types (numbers and type names only, never values) and the last `pii.flagged` run item; at most 500 documents (`truncated`) |

**Site profiles** (simplification pass §13; `content."SiteProfiles"`/`"SiteProfileVersions"` per (level, host), written by the worker, so changes are `command` jobs; connectors §5.6). In P1 every profile is `manual` (the hand-checked Sitevision preset for umea.se); `learned` versions, proposals and approve/reject are **P2** (weakness review §1)

| Method and path | Policy | Notes |
|---|---|---|
| `GET /api/site-profiles` (`host`, `level`) | Admin (P2, user 2026-11-07: also Read for the hosts of the caller's sources on the source page; matching sources to hosts needs the worker's canonical hosts) | level, host, CMS preset, applied version, origin (`learned`/`manual`/`preset`), last re-learn, sources using it, open proposal |
| `GET /api/site-profiles/{id}/versions`, `GET …/versions/{v}` | Admin | versioned JSON, check results of a proposal (words lost per sample page, empty pages, boilerplate share), sample before/after texts |
| `POST /api/site-profiles/{id}/versions/{v}/approve`, `/reject` (P2) | Admin | a `proposed` version (re-learned profile that failed the auto-apply check); `command` `approve-site-profile`/`reject-site-profile`; audited |
| `POST /api/site-profiles` `{ host, level, presetId?, profile }` | Admin | a host's first profile as a `manual` version; `command` `edit-site-profile` (dedup `site-profile:{level}:{host}`); audited |
| `POST /api/site-profiles/{id}/rollback` `{ toVersion }` | Admin | one click; `command` `rollback-site-profile`; audited |
| `PUT /api/site-profiles/{id}` `{ profile, version }` | Admin | a platform admin's edit as a new `manual` version (generic JSON form, §5.8); `command` `edit-site-profile`; audited. A profile belongs to no department, so its commands are queued (`Jobs.DepartmentId`, fairness only) for the department of the oldest live source whose settings name the host, else the first department (409 `no_department` before the organisation sync) |

**Documents**

| Method and path | Policy | Notes |
|---|---|---|
| `GET /api/documents` (keyset; `collectionId`, `sourceId`, `q`, `pii`, `outcome`) | Read | `pii` is `flagged` (a held flagged set or a flagged live set) or `cleared` |
| `GET /api/documents/{id}` | Read | its source, chunk sets (pipeline version, site-profile version), run items 90 days, PII counts and state of the live chunk set |
| `GET /api/documents/{id}/chunks` | Edit, EditPipelines | chunk text with heading path (breadcrumb prefix), tokens |
| `POST /api/documents/{id}/pii-clear` | Manage | reason ≥ 10 (V26); `command` `clear-pii-flag` |
| `POST /api/documents/{id}/reprocess` | Manage | `command` `reprocess-document` |

**Pipelines (template + parameters)**

| Method and path | Policy | Notes |
|---|---|---|
| `GET /api/templates`, `GET /api/templates/{id}/versions/{v}` | Read | parameter schema (`paramsSchema` with `x-labels`, `x-help`, `x-unit`), defaults, effect tags per parameter, worker support |
| `GET /api/pipelines/{id}`, `GET /api/pipelines/{id}/versions` | Read | a source's pipeline: template version + parameters per version |
| `GET /api/pipelines?collectionId=&sourceType=` | Read | the pipelines a new source of the type can use in the collection, the default (the one a source created without `pipelineId` gets) first; the add drawer's preview of an unsaved source runs on it |
| `POST /api/pipelines/{id}/preview` `{ collectionId, sourceId?, documentIds \| urls }` | EditPipelines, Edit | `preview` `stage-preview` on the pipeline's saved draft version, or the published one without a draft (§6.4); unsaved parameters are previewed by saving them as the draft first (the publication plan, §6.6) |
| `POST /api/pipelines/{id}/publications` `{ templateVersionId, parameters, version }` | EditPipelines | V24; enqueues `publish-plan`; returns `publicationId` |
| `GET /api/publications/{id}`, `POST /api/publications/{id}/confirm` `{ version }` | Read / EditPipelines | §6.6; confirm refused (409) if the pipeline changed since the plan; there is no cancel: an unconfirmed plan changes nothing, and the next plan replaces the draft |
| `POST /api/pipelines/{id}/rollback` `{ versionId }` | EditPipelines | one click; within 14 days the previous chunk sets are swapped back without re-parsing (§6.7) |
| `PUT /api/templates/{id}/draft`, `POST /api/templates/{id}/publish` | Admin | generic form (§5.8); a template version never re-processes sources by itself |

**Jobs and previews**

| Method and path | Policy | Notes |
|---|---|---|
| `GET /api/jobs/{id}` | Read (job's department) | `State`, `Result` (secrets stripped), queue position and age |
| `GET /api/previews/{jobId}/documents/{document}/stages/{stage}/output` | Edit, EditPipelines | decrypted preview blob, ≤ 2 MB, `text/markdown` or JSON; `{document}` is the document id or its index in the result (a URL or upload preview has no document id), `{stage}` a stage name or `before`/`after` (a crawled page's extraction without and with the site profile, §6.4) |
| `POST /api/documents/{id}/pii-preview` | Manage | `preview` job up to the personal-data stage; spans from preview blobs (24 h); audited `pii.view` |

**Evaluation**

| Method and path | Policy | Notes |
|---|---|---|
| `GET /api/collections/{id}/evaluation` | Read | set size per layer, stale/retired counts, latest reports |
| `GET /api/collections/{id}/evaluation/questions` (keyset; `layer`, `state`, `q`) | Read | question, expected answer, expected documents (titles), last rank |
| `POST /api/collections/{id}/evaluation/generate` | Edit | `eval` `generate-set`/`top-up-set` with cost forecast (≈ 400 prompt and 120 output tokens per candidate, around twice the mean eligible chunk; a top-up's forecast is a lower bound, since the stale questions its re-check retires are known only in the worker); P1b, with the evaluation items (P1a's evaluation is the human gold set plus on-demand runs; generated sets are made and judged at the collection's ceiling, Decisions "Evaluation models at the collection's ceiling") |
| `POST/PUT/DELETE /api/collections/{id}/evaluation/questions[/{qid}]` | Edit | human layer via `command` `edit-eval-question`; expected documents picked by search |
| `POST /api/collections/{id}/evaluation/run` | Edit | on demand (§4.7); a refused `candidateSettings` value is reported as `settings.<key>`, as the retrieval-settings save does |
| `GET /api/evaluation-runs/{id}`, `GET /api/evaluation-runs/{id}/results` | Read | recall@10 and MRR with counts, the previous run's values, per-question rank changes (§7.2) |

**Assistants**

| Method and path | Policy | Notes |
|---|---|---|
| `POST /api/assistants/{id}/move/impact`, `POST /api/assistants/{id}/move` | Admin | §3.2 *Changing the owner department*, as for a collection; the move answers the assistant's detail |
| `GET/POST /api/assistants`, `GET/PUT/DELETE /api/assistants/{id}` | Read / Edit / Manage (delete) | the assistant's own fields, saved directly under the V-rules: name, slug, owner team, `gatewayKeyId` (V39), classification (a change to `public` with general knowledge on takes `confirmGeneralKnowledge`, V36), **description** (≤ 500 characters) and **contact** (`ContactName`, `ContactEmail`); create takes an optional `templateId` (P2, weakness review §1) and starts with only a draft prefilled from the template (or `Settings.Chat` defaults), so `PublishedVersionId` is null and no client lists it; the detail carries `state` (`active`/`archived`), `publishedVersion`, `draft { number, updatedAt, baseChanged }`, `usesTools` (derived, never written: a linked skill's served version links a tool not excluded) and `lastUsedAt`. **`DELETE`** (Decisions "Deleting knowledge bases, sources and assistants"): from any state, `DeletedAt` set and the draft discarded; no client lists or reaches it from then on (as archived, identity §4.3 step 5), every route of it answers 404, its slug is free at once, and it no longer blocks the delete of a knowledge base. Its conversations, feedback and statistics are not touched (data-model §8). Audit `assistant.delete` with the last values |
| `GET/PUT/DELETE /api/assistants/{id}/draft` | Read / Edit / Edit | the version fields: instructions, model, collection links (`collectionLinks: [{collectionId, tags[]}]`; one link per collection, the first wins; topics trimmed and de-duplicated, otherwise kept as typed, so a bad topic stays in the draft as a V11 `topic_invalid` warning), skills (with excluded tools), query steps, P2 agent limits, not-found texts and links. One draft per assistant (`DraftVersionId`), created on the first edit as a copy of the published version with `Number` + 1; saving applies field rules and returns the assistant V-rules as `warnings[]` (never refused, so work in progress can be saved); `DELETE` discards it |
| `POST /api/assistants/{id}/publish` `{ draftVersion, version, confirmBaseChanged?, confirmGeneralKnowledge? }` | Edit | V34, then every assistant rule (V3–V17, V39) as refusals and V8's `/v1/models` call; one transaction (§4.4) marks the draft `published` and the previous one `superseded`, sets `PublishedVersionId` and clears `DraftVersionId` (data-model §2.1); starts the test run (§7.4; P2). A draft created before the current published version (a restore happened since; its `Number` is lower) answers 409 `draft_base_changed` (with the diff from P2) until confirmed |
| `GET /api/assistants/{id}/versions`, `GET …/versions/{n}`, `GET …/versions/{n}/diff?against={m}` (`m` = a number or `draft`; **diff P2**, weakness review §1) | Read | number, state, who and when, restored-from, test-run summary (P2); the diff is structured: changed settings by name, collections and skills added or removed, and a line diff of instructions and not-found texts (in-house Myers diff over ≤ 20,000 characters, no package) |
| `POST /api/assistants/{id}/versions/{n}/restore` `{ version, confirmGeneralKnowledge? }` | Edit | one click: a **new** version with version n's settings (`RestoredFromVersionId`), published at once exactly as publish; an open draft is kept (its base is now older); refused with the list when a link no longer resolves (V34); restoring the draft or the published version is refused (409 `version_not_restorable`) |
| `POST /api/assistants/{id}/copy` `{ name, slug, departmentId, ownerTeamId }` (P2, weakness review §1) | Read (source) + Edit (target department) | new assistant with only a draft = the source's published version (its draft if never published); test questions copied; audiences, keys, versions and statistics not copied; V11/V12 for the target department as warnings; audited `assistant.copy` |
| `POST /api/assistants/{id}/archive`, `/unarchive` | Edit | `State = archived` hides the assistant from every client (identity §4.3 step 5) and keeps versions, audiences and statistics; unarchive returns `State = active` with the same published version |
| `GET/POST/PUT/DELETE /api/assistants/{id}/test-questions[/{qid}]` (P2, weakness review §1) | Read / Edit | V33; the expected document is picked by title search in the draft's collections |
| `POST /api/assistants/{id}/test-runs` `{ target: draft \| published }`; `GET /api/assistants/{id}/test-runs` (P2, weakness review §1) | Edit / Read | §7.4: on demand (the publish run starts by itself); one running run per assistant (409 `test_run_running`); results through `GET /api/evaluation-runs/{id}/results` |
| `PUT /api/assistants/{id}/audiences` | Manage | P1 kind `client` only; V7–V9; not versioned |
| `PUT /api/assistants/{id}/retention` | Manage | `HistoryMode`, `RetentionDays` within the level's policy (V10); not versioned |
| `POST /api/assistants/{id}/test` `{ target: draft \| published, … }` | Edit, EditSkills | §4.9; never stored |
| `GET /api/assistants/{id}/stats?from=&to=` | Read | `AssistantDailyTotals` |
| `PUT /api/assistants/{id}/limits` (P2) | Edit (≤ defaults) / Admin (≤ hard caps) | writes the draft's `QuerySteps.agent`; V14 |
| `GET/POST/PUT/DELETE /api/assistant-templates[/{id}]` (P2, weakness review §1) | Read / Admin | platform-owned **assistant templates** through the generic form (§5.8): name, description, instructions, not-found texts sv/en with links, query steps; copied into a new assistant's draft at creation, never linked afterwards; audited `assistanttemplate.*`; **seeded** insert-if-absent by name (`AssistantTemplateSeeder` in `Kalla.MigrationService`, beside `PipelineTemplateSeeder`) with the three presets of the architecture plan's item 3: *Allmän rådgivning* (citizen service; its instructions are the text of `Settings.Chat.defaultSystemPrompt` at seeding, the code default when unset), *Intern policy* (internal policy questions from staff: answers only from the linked documents, names the document and its owner, says when a rule is unclear) and *Handläggningsstöd* (case-handling support: step-by-step guidance from the routines, never decides or predicts a case's outcome); a deleted seed stays deleted (seeded only into an empty table). Creating an assistant without a template (the template field emptied) keeps P1's start from `Settings.Chat.defaultSystemPrompt` |

**Skills and tool servers (P2)** — rules as tools design §3.5–§3.7, §4.4–§4.5 and §9.2: `/api/skills` (draft, validate,
publish → skill gate (run in the admin, tools §3.7), gate report, serve, rollback, retire, share list, re-run the gate), `/api/tool-servers`
(register, update, connection test, discover), `/api/tool-definitions/{id}/approvals` (`platform-admin` by `Admin`,
`system-owner` by a listed subject; two different people for new and unexpected definitions),
`/api/tool-servers/{id}/pre-registrations` (golden `tools/list` upload by a listed system owner; one `Admin` approval),
`/api/tools/{id}/grants`, `/api/tool-observations`.

**Clients, keys, settings, data protection** (generic forms in P1, §5.8)

| Method and path | Policy | Notes |
|---|---|---|
| `GET/POST/PUT/DELETE /api/clients[/{id}]` | Read (pickers) / Admin | `CK_Clients_*` as field rules (V21); changing the ceiling (either way) or `StoresHistory` re-checks every assistant offered to the client with the updated client (V7, V9; for a published assistant its rule set and V8 at the gateway, V36 excepted), refusals as field errors (`classificationCeiling`, `storesHistory`) naming the assistant (#700) |
| `GET /api/clients/{id}/health` | Admin | missing `X-Rag-Owui-Message-Id` warning (chat §11.4), feedback import state (§8.1) |
| `/api/keys…` | Manage (own teams) / Admin | copied from the gateway (`KeyEndpoints.cs:61-133`): create (secret once), update, rotate (immediate or 24 h grace), reveal/copy (Admin, audited), revoke; `kalla-sk-`/`-pk-` (identity §6.4) |
| `GET/PUT /api/settings/{section}` (`network`, `retention`, `deletion-guard`, `crawl`, `evaluation`, `retrieval`, `chat`, `pii`, `worker`, `identity`; `tools` from P2) | Admin | sections of `config."Settings"`; `chat` carries `maxAnswerTokens` (optional integer 256–1,000,000, absent = no platform ceiling; shown as **Longest answer, all assistants** with the hint "Caps every assistant's answer length, overrides included; leave empty to use each model's limit", chat §6.3, Decisions "Answer length from the model") and `prompts` (V38): the Settings page's **Prompt texts** panel shows the whole frame in order, each editable text as a field prefilled with its current value and a **Reset to default** link, the locked lines read-only between them, and a note that a save changes every assistant's answers at once (Decisions "Chat prompt texts configurable"); `network` holds `TrustedProxies` (data-model §2.1), as on the gateway's settings page; one `version` (the singleton's `xmin`) shared by all sections, so a save in one section makes a concurrent save in another answer `concurrent_change` |
| `GET/PUT /api/data-protection/{level}` | Read / Admin | V29; key service status read-only; allowed history modes `none` only and the shredding switch off until A1 (weakness review §4) |
| `GET /api/embedding-profiles[/{id}]` | Read | every profile, with `inUse` (true when a `ChunkSets` row of any state names it) and the canary state (recorded, last check). No profile is seeded: a platform admin creates the first one (next row). Until an active default profile exists, `GET /api/overview` answers `needsEmbeddingProfile: true` to a platform admin and Home shows a card that leads to the Embedding profiles panel; a new collection needs the profile (`no_embedding_profile`) |
| `POST /api/embedding-profiles` `{ gatewayModel, name?, requestDimensions, dimensions?, maxInputTokens?, maxResidency? }` | Admin | Decisions "Deploy configuration". `gatewayModel` is one of the gateway mirror's embedding models served at a residency within the platform ceiling, else 400 with `settings_model_residency` on `gatewayModel` (`setting` = "the embedding profile", `kind` = `embedding`). Each member left out comes from the model: `maxResidency` the highest residency it runs at within the ceiling, `maxInputTokens` its context window (`invalid_value` when the mirror has none), `name` the model's name. `dimensions` is required with `requestDimensions` (the size the model is asked for); otherwise the length of the test vector is the profile's `Dimensions`. The create first embeds the canary text with these values, as **Test** does; if that fails, 422 `profile_test_failed` with the outcome. The test's vector is stored as `CanaryEmbedding`. A taken name answers 409 `already_exists`. The first profile becomes the default (`IsDefault`), which new collections use. The worker creates the profile's partial HNSW index before its first chunk (data-model §3.2). Audited `create` on `embedding-profile`; 201 with the profile |
| `PUT /api/embedding-profiles/{id}` `{ gatewayModel, dimensions, requestDimensions, maxInputTokens, maxResidency, version }` | Admin | Decisions "Editable embedding profiles before use". While `inUse` is false, every field may change. Once it is true, only `maxResidency` may; any other change is refused with 409 `profile_in_use`, because it is the P3 profile migration (R3). `maxResidency` is at most the platform ceiling (`Eu` until M2). The save first embeds the canary text with the new values, as **Test** does. If that fails, the save is refused with 422 `profile_test_failed` and that outcome. The write transaction locks the profile row (`FOR UPDATE`) before the in-use check, so a worker inserting the profile's first chunk set either finishes first (the save then answers 409) or finds the new values and inserts nothing (processing §11 *Profile guard*). On an unused profile the save stores the new canary (`CanaryEmbedding`, clearing `CanaryCheckedAt` and `CanarySimilarity`) and resets the profile with `content.rag_reset_profile` (owner's rights): its `EmbeddingCache` rows are deleted, and when `dimensions` changed its partial HNSW index is dropped and `content.rag_ensure_profile_index` runs again. On a profile in use (only `maxResidency` changes) the recorded canary stays, and the test's vector must match it (cosine ≥ 0.999), else 422 `profile_test_failed` with `error: canary_mismatch`. Audited `update` on `embedding-profile`; the config trigger reloads the services |
| `POST /api/embedding-profiles/{id}/test` `{ gatewayModel, dimensions, requestDimensions, maxResidency }` (each optional; the stored value otherwise) | Admin | one embedding of the canary text with the admin key; nothing is stored. Returns `{ ok, dimensions, residency, error, detail }`. `error` is the gateway's code (`model_not_allowed`, `model_not_found`, `max_residency_not_met`, …), `gateway_not_configured`, `gateway_unavailable`, `bad_vector` with the dimension count that came back, `gateway_key_missing`, `residency_refused`, `canary_mismatch` or `error`. The form shows each in plain words, with the fix on the gateway side (texts in `messages.ts`) |
| `GET /openapi/v1.json` | Admin | the generic forms read each operation's request schema from the OpenAPI document; the resources, list columns and actions are a registry in the UI (§5.8) |

**Feedback, conversations, audit, overview** — §8, §9; `GET /api/overview` returns the attention items of §5.5.1 for
the caller's scope (P2; user 2026-11-07), and from P1 `needsEmbeddingProfile` for a platform admin (the embedding profile row above).

**Cost** — §8.6: `GET /api/assistants/{id}/costs`, `GET /api/collections/{id}/costs` (Read) and `GET /api/costs`
with its CSV (Manage).

### 4.6 Cross-rules (V1–V39; V20 and V27 retired)

Run by `ConfigRules` under the `ConfigVersion` lock (§4.4) on save of the entity, on every change that can break a
dependant (column *Re-checked when*), and again inside the publish transaction. A failing dependant lists every
affected object by name (problem extension `affected[]`). For an assistant, rules on version fields (instructions,
model, collections, skills, query steps, not-found texts) answer as `warnings[]` on draft save and refuse on publish and
restore; dependants are re-checked against the published version, and a draft is checked again when it is published.
Levels order `public < internal < sensitive`; residency
ladder `public → External`, `internal → Eu`, `sensitive → OnPrem`, ordered `OnPrem < Eu < External`. Retired numbers are
kept so references stay stable.

Each rule: what it requires, the problem code(s) it answers with, when it is re-checked, and its source.

#### V1 Source level ≤ its collection's level

- **Code:** `source_above_collection`
- **Re-checked when:** source save; collection level lowered
- **Source:** data-model §2.1

#### V2 Lowering a collection's level

- **Rule:** lowering a collection's level below a source or below a linking assistant's need is refused; raising it is a config change the worker applies (engine §15; no membership recompute)
- **Code:** `collection_level_conflict`
- **Re-checked when:** collection save
- **Source:** engine §15

#### V3 Assistant level covers its collections

- **Rule:** assistant level ≥ every linked collection (own or shared; collections are linked directly only)
- **Code:** `assistant_below_collection`
- **Re-checked when:** assistant save; publish; collection level raised
- **Source:** data-model §2.1, knowledge §4.3

#### V4 Assistant level ≥ every linked skill; skill level ≥ its tools' **result level**

- **Code:** `assistant_below_skill`, `skill_below_link`
- **Re-checked when:** skill draft/serve; tool server result level
- **Source:** tools §3.10

#### V5 Tool server clearance

- **Rule:** for every reachable tool server: max(assistant level, ceilings of history-resending audience clients) ≤ **clearance**
- **Code:** `tool_clearance`
- **Re-checked when:** publish; level or audience save; skill serve; server edit
- **Source:** tools §5.3

#### V6 Every reachable tool server's result level ≤ assistant level

- **Code:** `tool_result_level`
- **Re-checked when:** as V5
- **Source:** tools §5.3

#### V7 An audience client's ceiling ≥ assistant level (else the assistant is invisible to it)

- **Code:** `audience_above_ceiling`
- **Re-checked when:** audience save; client ceiling lowered; assistant level raised
- **Source:** identity §4.1

#### V8 Residency

- **Rule:** the `GatewayModel` servable at the ladder of max(assistant level, ceiling of every history-resending audience client): `min(GatewayModels.Residencies) ≤ ladder`; unknown model → refused; **on publish and on a level or audience save, the model is listed by the gateway's `/v1/models` called with `x-fors-max-residency` = the level's ceiling**, from gateway phase 2 as the assistant's key (V39) (gateway unreachable → refused, retry; a model whose in-ceiling targets are all drained is not listed either, gateway-changes design §3.4, so the `model_not_listed` text suggests retrying later)
- **Code:** `model_residency`, `model_not_listed` (409), `gateway_unavailable` (503), `gateway_key_missing` (503: the ceiling is above what this service may send, or until gateway phase 1 it holds no key of that level, identity §4.7)
- **Re-checked when:** publish; level save; audience save; model sync changed residency (warning on Overview; chat checks per call)
- **Source:** chat §8.2, simplification pass §7

#### V9 Sensitive assistants only for clients without history

- **Rule:** `sensitive` assistants only for clients with `StoresHistory = false` when `DataProtectionPolicies.ClientsWithoutHistoryOnly`
- **Code:** `client_stores_history`
- **Re-checked when:** audience save; policy change
- **Source:** platform §8a

#### V10 History mode and retention per level

- **Rule:** `HistoryMode` ∈ allowed modes, `RetentionDays` ≤ level max (default ≤ max ≤ 90)
- **Code:** `retention_outside_policy`
- **Re-checked when:** assistant save; policy tightened (lists assistants)
- **Source:** data-model §2.1

#### V11 Collection links and topics per assistant

- **Rule:** ≤ 20 collection links per assistant; each collection of the assistant's department or shared with it (`SharedWith`, `CollectionShares`, §3.2); ≤ 20 topics per link, each in the chunk tag format (processing §7)
- **Code:** `too_many_collections`, `collection_not_shared`, `topic_invalid`
- **Re-checked when:** publish; sharing narrowed
- **Source:** knowledge §4.2, §3.2 here

#### V12 Skills per assistant

- **Rule:** ≤ 20 skills per assistant, each of its department, shared with it, or a platform skill; for an assistant without tools, skill bodies ≤ 6,000 estimated tokens in total
- **Code:** `too_many_skills`, `skill_not_shared`, `skill_bodies_too_long`
- **Re-checked when:** publish; skill serve; share list narrowed
- **Source:** tools §3.4, §3.9

#### V13 Lethal trifecta false — a hard refusal, no acceptance in P2

- **Code:** `lethal_trifecta`
- **Re-checked when:** publish; skill serve; tool approval/grant; `ContainsThirdPartyText` changed
- **Source:** tools §5.4

#### V14 Assistants that use tools

- **Rule:** assistant that uses tools (a linked skill's served version links a tool): `SupportsTools`, synced price; limits ≤ `AgentDefaults` unless `Admin`, always ≤ `AgentHardCaps`
- **Code:** `model_no_tools`, `price_unknown`, `limit_above_default`, `limit_above_cap`
- **Re-checked when:** publish; skill serve (it can switch an assistant to tools); settings change
- **Source:** tools §6.4–§6.5

#### V15 Not-found texts and links

- **Rule:** not-found texts ≤ 1,000 characters each, ≤ 5 links, `https`, host matches `Settings.Chat.allowedLinkHosts` (an empty or missing list refuses every link)
- **Code:** `link_host_not_allowed`
- **Re-checked when:** publish; allow-list narrowed
- **Source:** chat §9.4, §10.1

#### V16 Retrieval settings ranges

- **Rule:** `RetrievalSettings` ranges: `vectorWeight`, `keywordWeight` 0–5, `candidates` 20–200 (the range retrieval accepts, `RetrievalSettings.Validated()`; there is no `efSearch`, knowledge §12.1), `maxChunksPerDocument` 1–10, `coverageMaxDistance` 0.05–1.0, `coverageMinKeywordScore` null or > 0 (weakness review §2), `recency` null or `halfLifeDays` 30–3,650, `weight` 0–0.5, `kinds` non-empty subset of the document types (weakness review §3), `expandMaxTokens` 0–4,000; saved directly (an advisory evaluation run follows)
- **Code:** `retrieval_settings_invalid`
- **Re-checked when:** collection settings save
- **Source:** knowledge §12.1

#### V17 Query step ranges

- **Rule:** `QuerySteps`: `retrieval.topK` 1–50, `history.maxMessages` 0–20 and `history.maxTokens` ≤ 8,000, `answer.maxOutputTokens` absent (the model's limit, chat §6.3) or 64 – the lowest of the model's `MaxOutputTokens`, the assistant's gateway key's output cap and `Settings.Chat.MaxAnswerTokens` (1,000,000 when neither is known), `answer.temperature` 0–1, `answer.language` `auto`/`sv`, `answer.generalKnowledge` `off`/`supplement`/`fallback`/`open`, `retrieval.rerank` `off`/`chat`/`gateway` (absent = `off`; a reranker the platform has no model for is allowed, knowledge §6.7); `answer.contextTokens` absent (the model's `ContextWindow`, else 16,384) or 4,096 – the model's `ContextWindow` (1,048,576 when unknown), `condense.enabled` a boolean; `topK`, `maxMessages`, `maxTokens`, `maxOutputTokens` and `contextTokens` are **integers** (V17 accepts exactly what the runtime reads, `AssistantChatSettings.Parse`: a fraction, a string or `10.0` is refused, never silently replaced by the default); the model's limits are those of the version's `GatewayModel` row and are rechecked when the sync or an admin lowers them (an override above the new limit is a warning on the draft and is lowered to the limit at run time, never a failed chat; the same when a platform admin lowers `Settings.Chat.MaxAnswerTokens` below it); not built yet: the condense model within the V32 ceiling
- **Code:** `query_steps_invalid`
- **Re-checked when:** publish; model sync; model limits edited; chat settings saved
- **Source:** chat §5–§6, §10.3

#### V18 Skill versions and serving

- **Rule:** skill version label > every existing; **serve** only after the version's gate run finished, and only if V4–V6, V12–V14 and V19 hold for every assistant whose published version links the skill
- **Code:** `version_not_increasing`, `gate_not_finished`, `serve_breaks_assistants`
- **Re-checked when:** skill publish/serve/rollback
- **Source:** tools §3.5

#### V19 Skill tool grants

- **Rule:** skill tool links need a `ToolDepartmentGrant` for the skill's department; platform skills only `AllDepartments` tools; sharing never grants: a linking assistant of another department sees a shared skill's tool only with its own grant (warning lists the tools)
- **Code:** `tool_grant_missing`
- **Re-checked when:** skill draft; grant revoked (lists skills); share list widened
- **Source:** tools §3.6

#### V20 *retired* (skill MCP exposure vs audience ceilings: per-skill endpoints moved to P3)

- **Code:** –
- **Re-checked when:** –
- **Source:** –

#### V21 Client ceilings by kind

- **Rule:** clients: `StoresHistory` ⇒ ceiling ≠ `sensitive`; `mcp-client` ⇒ ceiling ≤ `internal`; `ToolPolicy` per kind default
- **Code:** `client_ceiling_rule`
- **Re-checked when:** client save
- **Source:** data-model `CK_Clients_*`, tools §5.1

#### V22 Tool server registration

- **Rule:** tool servers: `CK_ToolServers_Clearance`; `Hosting = hosted` (P2); approvers differ for new and unexpected definitions; system-owner approval and pre-registration only by a listed subject; `write` tools refused while `WriteToolsEnabled = false`
- **Code:** `toolserver_rule`, `same_approver`
- **Re-checked when:** server save; approval; pre-registration
- **Source:** tools §4.4–§4.5, §5.3

#### V23 Keys and their scopes

- **Rule:** keys: team in the caller's departments; scopes only assistants/collections of the key team's department; `pk` keys only `public` assistants, exact origins
- **Code:** `key_scope_invalid`
- **Re-checked when:** key save; assistant level raised (lists `pk` keys)
- **Source:** identity §6.4–§6.5

#### V24 Pipeline parameters and template

- **Rule:** pipeline: parameters valid against the template version's `paramsSchema`; template version supported by a running worker; the template matches the source type
- **Code:** `pipeline_params_invalid`, `template_unsupported`
- **Re-checked when:** parameters save; publish; template version change
- **Source:** engine (templates and parameters)

#### V25 Source settings

- **Rule:** source settings: URL list ≤ 5,000, exact duplicates refused (canonicalisation is the worker's: `UrlCanonicalizer` runs when it lists the URLs); crawl `include`/`exclude`/`demote` compile (`re:` with `NonBacktracking`, no match timeout, connectors §5.2), `demote` factor in (0, 1), default 0.5 (weakness review §3); `maxPages` ≤ the sources schema's maximum (there is no `Settings.Crawl` cap key); `maxFileSizeMb` ≤ 50; cron 5 fields; `collectionId` immutable; `scope`/`identity` changes need the `impactToken`
- **Code:** `source_settings_invalid`, `impact_not_confirmed`
- **Re-checked when:** source save
- **Source:** connectors §4–§5, §9, §12.3

#### V26 Reasons for PII clears and conversation access

- **Rule:** reasons: PII clear ≥ 10 and the document's source in the caller's departments; conversation access ≥ 20 (comment reveal: the fixed one-click text)
- **Code:** `reason_too_short`, `not_in_scope`
- **Re-checked when:** the action
- **Source:** processing §9.5

#### V27 *retired* (collection floor before publish: evaluation is advisory)

- **Code:** –
- **Re-checked when:** –
- **Source:** –

#### V28 Deletion decisions

- **Rule:** deletion decisions only by `Manage` of the source's department, while the run is `awaiting-deletion-confirmation` and the proposal is the current one
- **Code:** `deletion_superseded`
- **Re-checked when:** confirm/reject
- **Source:** connectors §13.3

#### V29 Data protection and history modes

- **Rule:** data protection: history modes other than `none` refused until stored chat content is built after A1 (weakness review §4); `internal`/`sensitive` ⇒ chat encryption required; default ≤ max ≤ 90; changes apply to messages written afterwards (their expiry-month bucket); bucket-key destruction cannot be switched on before A1 is recorded as answered
- **Code:** `policy_rule`
- **Re-checked when:** policy save
- **Source:** data-model `CK_DataProtectionPolicies_*`, simplification pass §5

#### V30 Settings ranges

- **Rule:** `Settings` ranges as the `CK_Settings_*` checks (guard 1–100 %, min/max counts, sample sizes, …)
- **Code:** `setting_out_of_range`
- **Re-checked when:** settings save
- **Source:** data-model §2.1

#### V31 Embedding profile per collection

- **Rule:** embedding profile fixed per collection (`platform-default`); other profiles only `Admin` (P3)
- **Code:** `profile_admin_only`
- **Re-checked when:** collection save
- **Source:** R3

#### V32 Platform model settings

- **Rule:** platform model settings `Settings.EvaluationModel`, `Settings.EvaluationJudgeModel`, `Settings.Chat.CondenseModel`, `Settings.Tools.SkillGateModel` and `Settings.Retrieval.rerankModel` are `chat` rows, `Settings.Retrieval.gatewayRerankModel` a `rerank` row (empty = unset, knowledge §6.7), of `GatewayModels` servable at most at the platform ceiling: `Eu` until M2 (on-prem models once the on-prem hardware exists, register M2) (`SkillGateModel` also `SupportsTools`); on save each set model is also listed by the gateway's `/v1/models` at the platform ceiling, as V8 (#447)
- **Code:** `settings_model_residency`, `model_not_listed` (409), `gateway_unavailable` (503)
- **Re-checked when:** settings save; model sync changed residency (warning on Overview, P2 with the attention items)
- **Source:** processing §13.1, chat §5, tools §3.7

#### V33 Assistant test questions

- **Rule:** assistant test questions: ≤ 30 per assistant; question 1–2,000 characters; expected behaviour 1–2,000; expected phrase ≤ 200; an expected document must belong to a collection the draft or published version links (unlinking it later marks the question in the report, never refuses)
- **Code:** `test_question_invalid`, `test_question_document_unlinked`
- **Re-checked when:** test-question save
- **Source:** §7.4

#### V34 Publish and restore resolve

- **Rule:** publish and restore: the definition resolves (collections not deleted, skills not retired and usable by the department, the model a `chat` row of `GatewayModels`); description and contact are set; a draft created before the current published version (lower `Number`) is published only when confirmed
- **Code:** `version_link_missing`, `assistant_contact_missing`, `draft_base_changed`
- **Re-checked when:** publish; restore
- **Source:** §4.5, simplification pass §15

#### V35 Narrowing a collection's sharing

- **Rule:** narrowing a collection's sharing (to a lower level, or removing a department from `selected`) is refused while a published assistant of a department that loses access links it; the refusal lists the assistants and their departments
- **Code:** `collection_shared_in_use`
- **Re-checked when:** sharing save
- **Source:** §3.2, §4.5

#### V36 General knowledge on a public assistant

- **Rule:** an assistant of level `public` with `answer.generalKnowledge` other than `off` needs an explicit confirmation (Decisions "General knowledge on a public assistant", #445): a `warnings[]` entry on draft save; publish, restore and a level change to `public` answer 409 until the request carries `confirmGeneralKnowledge: true`, which the audit event records as `generalKnowledgeConfirmed`
- **Code:** `general_knowledge_public_unconfirmed` (409)
- **Re-checked when:** draft save (warning); publish; restore; level change
- **Source:** chat §7.3, §10.4

#### V37 Host politeness profiles

- **Rule:** a host's politeness profile in `Settings.Crawl.HostOverrides` (connectors §6.6): members within their ranges and the deployment's ceilings (`Crawl:MaxHostRate`, `Crawl:MaxHostConnections`); `robots: ignore` needs a non-empty `reason`; `userAgent` 1–256 printable ASCII; the host is a canonical host without a port
- **Code:** `crawl_politeness_invalid`
- **Re-checked when:** settings save
- **Source:** connectors §6.6

#### V38 Chat prompt texts

- **Rule:** `Settings.Chat.Prompts` (chat §7.3, Decisions "Chat prompt texts configurable"): each member optional (absent or empty = the built-in default); `identity`, `sourcesLabel`, the rule lines, `conversational`, `degraded` ≤ 1,000 characters each, `condense`, `followUps` and `escalation` ≤ 2,000, each caveat ≤ 500 per language; no `<sources`, `</sources`, `<source`, `</source`, `<skill_` or `<tool_result` (any case, spaces allowed, as chat §7.2's neutralisation); no C0/C1 controls except line breaks; `sourcesLabel` one line; placeholders: `{assistant}` only in `identity`, `{N}` only in `followUps`, any other `{…}` refused; unknown members refused
- **Code:** `prompt_text_invalid` (field = the member)
- **Re-checked when:** settings save
- **Source:** chat §5, §7.2–§7.3, §9.8, §9.9, §10.2, §10.4

#### V39 Gateway key

- **Rule:** (cost-attribution §4.2; data-model V39): a set `gatewayKeyId` is a `GatewayKeys` row with `Status` `Active` and `OwnerActive`, of the object's own department; its `AllowedResidencies` reach the object's ceiling; it allows every model the object uses (an assistant: the answer model, the condense model, each linked collection's embedding profile model, the rerank model; a collection: its embedding profile model and the evaluation and judge models). Once the context's `features` contain `act-as-key` the models are checked authoritatively with `GET /v1/models` as the key (`x-fors-on-behalf-of`) at the ceiling: the gateway lists the key's models within its residencies, the admin credential key's and the ceiling (gateway-changes §5.3, §5.5), and the refusal names each missing model; the answer's echo is checked (5011 → `gateway_unavailable`), and a refusal of the key (`403 delegation_not_allowed`, `403 key_disabled`, `403 key_revoked`/`key_expired`, gateway-changes §5.7) is `gateway_key_invalid`. An object without a key gets a warning ("charged to the platform's owner team"); once the context's `features` contain `act-as-key`, an assistant's publish and restore refuse it, and so does a save that clears the key of a published assistant (`gateway_key_required`), since it would then run on kalla-api's credential key outside its department's budget. A key with `HasSystemPrompt` is a warning: its team's system prompt would change the assistant's prompts
- **Code:** `gateway_key_invalid` (unknown, not active, another department), `gateway_key_residency`, `gateway_key_model` (`affected[]` = the models the key's list and the residency cases don't explain), `gateway_key_model_residency` and `gateway_key_embedding_residency` (#720: the key allows the model, but within the ceiling it runs only at residencies the key doesn't allow; params `key`, `model`, `modelResidencies`, `keyResidencies`; the embedding variant names the knowledge base's embedding model, recognised by the mirror's `ModelKind`), `gateway_key_required` (409), `gateway_key_system_prompt` (warning only); `gateway_unavailable` (503)
- **Re-checked when:** object save (key, level, audience); draft save (warnings on the version's models and links); publish; restore; key sync (a key turned unusable or narrowed: an attention item, §5.5.1)
- **Source:** cost-attribution §4.2, data-model §2.1


### 4.7 Jobs for the worker

The admin holds `INSERT` on `content."Jobs"`; payloads are ids and parameters only (≤ 4 KB). Enqueue = `INSERT … ON
CONFLICT DO NOTHING` against the partial unique index on `DedupKey WHERE State IN ('queued','running')` (simplification
pass §9) and `pg_notify('rag_jobs', class)` in the same transaction as any config change it belongs to. `DepartmentId`
= the subject's department (fairness), `TeamId` = owner team (cost).

| Admin action | Class / kind | Dedup key | Priority | Result read from |
|---|---|---|---|---|
| Preview (sample pages) | `preview` `stage-preview` (≤ 5 `documentIds`, `urls` or `uploadIds`; `collectionId`; optional `embed`); for a crawl host without a profile first `preview` `site-profile` (learns it; before/after; **P2**, weakness review §1: in P1 the host's manual profile or the Sitevision preset is applied) | – | – | `Jobs.Result` + preview blobs |
| Preflight (unsaved source) | `preview` `preflight` (URL lists: first 20 URLs; the unsaved settings as an object, checked first against the sources schema, V25) | – | – | `Jobs.Result`: worst state, one `<area>.<code>` code per check (no Swedish text; the UI maps codes to text in `messages.ts`), `estimatedPages`, `cms`, and per rendered start URL (at most 3) the raw and rendered main-text word counts, `render.suggested` when the raw count is under half, else `render.compared` (connectors §8.1; needs the renderer) |
| PII highlights | `preview` `stage-preview` until the personal-data stage, `piiSpans: true` | `pii-preview:{documentId}` | – | preview blobs (24 h) |
| Publish plan | `preview` `publish-plan` (P1: every document of the collection; from effect tags in P2, weakness review §1) | `plan:{pipelineId}` | – | `Jobs.Result` |
| Evaluation run (retrieval settings saved, **Run evaluation**) | `eval` `on-demand` (`variants` in one job; optional `candidateSettings`, runs marked `candidate`) | `eval:{collectionId}` | – | `EvaluationRuns` |
| Generate / top up test questions | `eval` `generate-set` / `top-up-set` | `eval-generate:{collectionId}` | – | `EvaluationSets`, questions |
| Sync now / retry failed / continue | `sync` (`Trigger = manual`, `kind` `sync`/`retry-failed`/`continue`) | `sync:{sourceId}` | 10 | `Runs` |
| Upload commit | `sync` (`Trigger = upload`) | `sync-upload:{batchId}` | – | `Runs`, `Uploads` |
| Publish confirm / rollback | `sync` `reprocess` per source (`Trigger = publish`/`rollback`) | `sync:{sourceId}` (a running sync gets a follow-up) | – | `Runs`; the post-publish evaluation run (`eval` `publish-report`, dedup `eval-report:{runId}`) is enqueued by the worker when re-processing completes (§6.6) |
| Deletion decision | `command` `confirm-deletion` / `reject-deletion` (with `decidedBy`) | `deletion:{runId}` | – | `Runs.State`, `DeletionProposals` |
| Release held references | `command` `release-hold` | `release-hold:{sourceId}` | – | `Documents.HeldAt` |
| PII clearance (a decision on the document) | `command` `clear-pii-flag` (with `decidedBy`; the reason stays in the audit event) | `pii-clear:{documentId}` | – | `Documents.PiiClearedAt`/`PiiClearedCounts`; the document's `held` set released (processing §9.5) |
| Human question edit | `command` `edit-eval-question` | – | – | `EvaluationQuestions` |
| Reprocess one document | `command` `reprocess-document` (`fromStage` default `parse`) | `reprocess:{documentId}` | – | run items |
| Cancel a run | `command` `cancel-run` | `cancel:{runId}` | – | `Runs.State` |
| Site profile approve / reject (P2) / roll back / edit | `command` `approve-site-profile` / `reject-site-profile` / `rollback-site-profile` / `edit-site-profile` | `site-profile:{siteProfileId}`; an `edit-site-profile` that creates a host's first profile: `site-profile:{level}:{host}` | – | `SiteProfileVersions`, `SiteProfiles.AppliedVersionId` |
| Approve / reject a parse-profile proposal (P2) | `command` `approve-profile` / `reject-profile` | `profile:{id}` | – | `ParseProfiles` |

Payload schemas are in `contracts/jobs/v1/` (engine design §9.1):
- The preview kind is `stage-preview`.
- The worker enqueues two kinds the table does not show: `eval` `nightly` (dedup `eval-nightly:{collectionId}`) and
  `publish-report`. `eval-nightly:` is kept apart from `eval:{collectionId}`, so a queued on-demand run and the
  nightly run never drop each other.
- Commands carry `decidedBy` (the admin's `oid`), because `Jobs` has no actor column.
- `edit-site-profile` carries `baseVersion` (`site_profile.version_conflict` on mismatch).
- Only P1 kinds exist; the P2 rows above have no schema yet.

What the commands do (engine design §9.1, §12):
- `reprocess-document` re-runs the document from stored content in a `reprocess` run whose id is the job id;
  `fromStage: fetch` is refused (`reprocess.fetch_unsupported`). It bypasses the stage caches from `fromStage` (default `parse`; `embed` re-runs through the caches), and a `sync` `reprocess` with
  `force` is accepted (default `fromStage` `parse`, `fetch` refused the same way; engine §8).
- `release-hold` clears deletion holds only; a PII-held set is released by `clear-pii-flag`, without re-processing.
- `edit-site-profile` and `rollback-site-profile` take an optional `decidedBy` (absent = `system:worker`) and enqueue
  a `sync` `reprocess` (`Trigger = site-profile`, dedup `sync:{sourceId}`) per source with live HTML on the host.
  When the version changes `excludedTypes`, `identityRule` or `ignoreQueryParams`, each crawl source with live
  documents on the host gets a `sync` of kind `sync` instead (connectors §5.6).
- `confirm-deletion` sets the run back to `running` for the finalizer; `reject-deletion` sets `degraded` without a
  stored reason.
- The worker enqueues `publish-report` for `reprocess` runs with `Trigger = publish` or `rollback` only, whatever
  state they finished in. The rollback is driven as for a publish: flip the version states and
  `Pipelines.PublishedVersionId`, then enqueue a `reprocess` with `Trigger = rollback`.

Assistant publish, restore and copy enqueue nothing: they are config changes only. Assistant test runs (§7.4) are not
worker jobs either: like answer-level evaluation and the skill gate they run in `kalla-admin` (`AssistantTestRunner`,
§2.2), which needs `Kalla.Answering` and the gateway keys of every level; the `EvaluationRuns` row is their durable
state.

The UI polls `GET /api/jobs/{id}` (or the run/evaluation it created) every 2 s, every 5 s after 30 s, every 15 s after 5
minutes, and not while the tab is hidden; there is no push channel. A `dead` or `failed` job shows its `LastError` code
through the §4.2 table.

### 4.8 Reading `content`, `chat` and `audit`

- **Scope first**: every content query starts from config ids in the caller's scope (collections, sources,
  assistants), so a row of another department is never selected. No SQL join between `config` and `content`/`chat`
  tables: config names are resolved from the admin's snapshot.
- **Counts**: run counters are derived; the overview uses `Runs` and `Jobs` indexes only, never a `RunItems` scan.
- **Display names**: a document belongs to one source, so its upload file name (`DisplayName`) is shown only to scopes
  that see the source.
- **Chat**: metadata columns only, except the two audited decryption paths of §8. **Audit**: §9.

### 4.9 Test panel

Runs `Kalla.Answering` in the admin process (identity §4.8): principal `admin-console` with the admin's `oid`, access
by the admin's role scope (assistant's department ∈ caller's departments), assistant audiences ignored, **collections
not yet published included**, every call as the assistant's gateway key (`GatewayKeyId`, from gateway phase 2;
cost-attribution §4.1; knowledge §4.2, §4.5), residency ceiling at the assistant's level (with test history: chat §8.2's rule if the assistant has a history-resending client in its
audience). It runs the assistant's **draft** by default (the published version when there is no draft, or on request
with `target = published`), so a change is tried before **Publish**; the draft's V-rule warnings are shown above the
answer. Nothing is stored but the **usage**: every model call of a test turn (condense, long-question condense, query
embedding, rerank, answer, follow-ups) is counted as chat counts it (chat §8.3: tokens, cached input, the gateway's cost
or an estimate from the mirror's prices, kept apart), in `chat."AssistantDailyUsage"` under the assistant and the
**`admin-console` client row** (identity §4.1's pilot row, so test traffic stays apart from real users' clients;
data-model §2.3), upserted by `kalla-admin` after the answer with the same per-step deltas and key order as the chat
writer (chat §12.2), never in `AssistantDailyTotals` (test turns are not chat statistics; cost-attribution §5 C2). A
failed write is logged and dropped; the test answer is unaffected. Query embeddings use the in-process L1 cache only, keyed with a per-process random HMAC
key. **Release coupling**: `kalla-admin` and `kalla-api` are built and deployed from one release (`release.yml`), and the
panel ships in the same P1 release as the chat role, so it never runs a different `Kalla.Answering` than chat does.

| Input | Output (JSON, non-streamed through the chat client; ≤ 60 s by its timeouts) |
|---|---|
| `question` (≤ 2,000 characters); optional `history[]` (≤ 10 messages of ≤ 4,000 characters, to test condense); assistant id and `target` (`draft` default, `published`); optional `generalKnowledge` (`off`/`supplement`/`fallback`/`open`, #726): the stricter of it and the target version's level, as a chat request's `kalla.general_knowledge` (chat §3.8); the panel offers the levels up to the version's own, and the response's top-level `generalKnowledge` names the effective one | `target` and `versionNumber`; `retrieval`: searched collections (with a not-published flag), coverage `{status, bestDistance, threshold, collection}`, `degraded`, candidates `{rank, documentId, title, url or file name, headingPath, text (600 characters, expandable), truncated, distance, keywordRank, vectorRank, fused, cited}`, timings; `condense` `{used, outcome}`; `answer` `{raw, delivered, filtered (counts per kind), citations [n] → document, outcome, language}`; `gateway` `{model, residency, latencyMs, inputTokens, outputTokens, maxTokens, maxTokensSource, finishReason, costUsd, costEstimated}` (`costUsd` the sum over the turn's calls, `costEstimated` true when any part of it is an estimate; shown as "0,0012 USD", with "estimated" when so) (`maxTokens` = the `max_tokens` sent, null when left out; `maxTokensSource` `override`/`model`/`ceiling`/`key`/`provider`; `finishReason` from the gateway, the panel saying "The answer stopped at the length limit" for `length`; Decisions "Answer length from the model"); `durationMs` (`AssistantTestEndpoints.cs`). **P2** (user, 2026-11-07, M-3): a `collectionIds` subset, the condense step's rewritten question and why it was skipped, the prompt token budget per part, `filtered[]` as (kind, text) and `ttftMs`; `usesTools`, `toolCalls[]` `{tool, state, durationMs, arguments (admin view), resultExcerpt}` and skill inclusion with tools |

- Rate: 20 test questions per minute per user on the full panel (the retrieval-only variant has no limit), metric
  `rag.admin.test_panel{outcome}`; no audit entry, no 6007 (the content is the caller's own department's).
- The answer is shown as **plain text** with preserved line breaks and `[n]` chips (no Markdown renderer).
- P2: an assistant that uses tools (its draft, when testing the draft) calls hosted MCP servers through the admin's in-process hub as `admin-console` (needs
  `Clients.ToolPolicy = read` for that client, §13), with the same limits; write tools stay impossible.
- Retrieval-only variant `POST /api/collections/{id}/test-retrieval` for pipeline editors and collection editors.

### 4.10 Organisation and model catalogue sync

Every 5 minutes and on demand. The gateway's discovery calls (gateway-changes §5.5) use the admin's credential key,
send `If-None-Match` with the last `ETag`, and write nothing on a `304`.

| Sync | Source | Rule |
|---|---|---|
| Gateway context | `GET /v1/fors/context` (gateway integration phase 1) | `gateway.features` and `version` held in memory and shown on the *Gateway* page (`act-as-key` switches on the admin's acting as keys and V39's publish rule; the api and the worker read their own context for it, identity §4.7; `stream-cost` the stream cost comment, chat §8.3); `gateway.admin_url` for the links (§2.3); `assigned_keys` → `GatewayKeys` (data-model §2.1, `usable` included) upserted by id, only changed rows written, a key no longer listed (revoked) deleted. A key that turns unusable (not `Active`, owner inactive, gone) or loses a model or residency an object relies on raises the attention item of §5.5.1 for the objects that use it |
| Departments | the context's `departments` (same call; the admin's credential key holds `organisation.read`, gateway-changes §5.5) | taken only when `gateway.features` contains `organisation` and `key.integration_scopes` contains `organisation.read`; otherwise departments and teams are left as they are and the *Gateway* page and health show `degraded` with the reason (without the scope the context lists only the assigned keys' departments, which would deactivate the rest). Upsert by id, `is_active` copied; only changed rows written; a department missing from the context → `IsActive = false`; member groups replace the stored array from `member_groups` when `features` contains `member-groups` (gateway v0.10.0; identity §2.1); they are never edited in Källa |
| Teams | the context's `teams` (`code` included) | as departments, under the same condition |
| Models | `GET /v1/models` at the platform ceiling | `GatewayModels` row per chat/embedding/rerank model: `Kind = model`, `ModelKind` from `fors.kind` (`chat`/`embedding`/`rerank`), `Residencies` from `fors.residency`, prices from `fors.pricing`, `ContextWindow`, `MaxOutputTokens` and `SupportsTools` from `fors.context_window`, `fors.max_output_tokens` and `fors.features` (`tools`); while the gateway sends a null limit the manual value is kept (Decisions "Answer length from the model"), and the manual fields go once it sends them; the rows of models no longer listed are deleted (`GatewayModels` has no "disabled" column), so an assistant that still names one is refused by V34/V8 on its next publish. Källa's assumption: the admin's credential key has no model allow-list, so the list is the gateway's whole catalogue within the ceiling |
| Route aliases | the aliases on `GET /v1/models` (`fors.residencies`) | `Kind = route`, `Residencies` = its targets' residencies, `Source = catalog`; servable at a ceiling when any target is within it. Until phase 1 they are manual rows (`Source = manual`). Multi-target model names are routes |

The mirror feeds pickers and V8's first check; V8's `/v1/models` call at save is the authoritative check, because it
asks the gateway what it will serve at that ceiling. The mirror is the gateway's whole catalogue, not what an
object's key may use, so every model picker greys out a model whose `available` is `false` ("Not available to this
key") and cannot choose it; with `null` every mirror model stays choosable with "Availability unknown"
(Decisions "Model pickers show what the key may use", #447). The Settings page's model fields (evaluation and judge,
condense, chat rerank: `chat`; gateway rerank: `rerank`) are such pickers over `GET /api/gateway/models` without
`level` (the V32 ceiling): an empty choice unsets the model, all-unknown availability says that saving checks it, a
current value no longer offered stays selectable and marked, and if the list cannot be read the form shows the error
and does not open (#466).
Failures keep the last mirror; after 3 failed runs the Overview
shows "Synkningen mot AI-gatewayen fungerar inte sedan {tid}" to platform admins, metric `rag.admin.sync{kind,outcome}`.
A stale mirror narrows or keeps sign-in scope, never widens it (identity §2.6). The sync status is held in memory per
instance (`GET /api/organisation/sync`); the run history and the 3-failures banner come with `Settings` and the
Overview page (P1).

## 5. UI information architecture

### 5.1 Stack and what is copied

- Copied unchanged: toolchain and scripts, `.npmrc`, lint (incl. `eslint-plugin-vuejs-accessibility`), design system
  (`styles/*`, `components/ui/*`, `components/layout/*`, icons subset), `design-system.lock` drift report. Shared
  components with built-in English text (`AsyncState`, `DataTable`, `ConfirmDialog`, `Pagination`, `UiDrawer`, the
  layout components) are **adapted** to take their text from `labels.ts` and marked `adapted` in the lock;
  gateway-only components (`KeyStatusBadge`, `ResidencyLabel`, `AddressListField`, `SecretDialog`) arrive with keys in
  P1. `package.json` pins the gateway's resolved versions exactly and the lock file is the gateway's, so CI and
  developers use `npm ci` only (npm's `allow-remote=none` refuses to re-resolve); `vite.config.ts` allows
  `contracts/admin` for the error-code spec;
  `api/client.ts` (constant `RAG-XSRF-TOKEN`), `composables/useFormErrors.ts:8-44`, `useTableView.ts:9-54`,
  `stores/loadable.ts`, `test/*` helpers; charts `components/usage/TimeSeriesChart.vue` for evaluation trends
  when the chart view comes (P2, §7.1).
- Adapted: `utils/problem.ts` (§4.2), `stores/auth.ts` (RAG roles), `router/nav.ts` + `router/index.ts` (pages below;
  same `meta.roles` guard and focus handling; title suffix "· Knowledge platform", product name N1 open); `utils/format.ts`
  and `index.html` (`lang="en"`) as the gateway's.
- New: `utils/labels.ts`, `utils/messages.ts`, `composables/useJobPoller.ts`, `components/rag/*` (classification badge,
  run state badge, evaluation report card, setup checklist, preview viewer with before/after, test panel),
  `platform/*` (the generic forms of §5.8: resource registry, schema mapper, `GenericForm`, list and form views),
  `views/*`.

### 5.2 Navigation per role

Grouped by the job people come to do, as the gateway's side menu (`docs/admin-ui.md:5-13`); items filtered by role.
Ctrl K search copied (`AppShell.vue:62`), extended with `GET /api/search`. Phone bottom bar: Home, Knowledge bases,
Assistants, Feedback, Menu.

```
┌──────────────────────────┬──────────────────────────────────────────────────────────────────────────────────────┐
│ ◧ Knowledge platform     │  Search (Ctrl K)                                  Department: All ▾   ☾   Anna A. ⎋  │
│   Umeå kommun            ├──────────────────────────────────────────────────────────────────────────────────────┤
│ OVERVIEW                 │                                                                                      │
│   Home              (3)  │   <main id="main">                                                                   │
│   Map                    │                                                                                      │
│ CONTENT                  │                                                                                      │
│   Knowledge bases        │                                                                                      │
│   Assistants             │                                                                                      │
│   Skills             P2  │                                                                                      │
│   Tool servers       P2  │                                                                                      │
│ FOLLOW-UP                │                                                                                      │
│   Feedback               │                                                                                      │
│   Costs                  │                                                                                      │
│   Conversation review    │                                                                                      │
│ PLATFORM                 │                                                                                      │
│   Organisation           │                                                                                      │
│   Platform objects (forms): Settings · Data protection · Clients · Keys · Templates                               │
│     Assistant templates · Site profiles · Embedding profiles · Gateway models                                     │
│   Audit log                                                                                                       │
│ ↗ API (Scalar) · ↗ AI gateway admin                                                                               │
└──────────────────────────┴──────────────────────────────────────────────────────────────────────────────────────┘
```

The Platform group of the drawing is replaced by §5.10's (Apps and keys, Models, Settings, Organisation, Audit log).

**Home** shows the selected department's setup checklists, attention items and a one-line summary with its contacts;
**Knowledge bases** has the tabs *Ours*, *Shared with us* and *All* (`view=`, §4.5). The department picker in the
header filters every list; a user with one department never sees it.

Start page: Home (a reviewer-only user starts on Feedback). Badges: Home counts attention items (gateway
`badge` pattern, `nav.ts:25`). No e-mail: awaiting deletions, worse evaluation reports, site-profile proposals, P2
drift and pending approvals appear as attention cards and badges, plus SIEM events for the security-relevant ones.

### 5.3 Pages by phase

| Page | P1 | P2 | P3 |
|---|---|---|---|
| Home | setup checklists (folded once published), sync health per department, queue age, org/model sync, a Live card per published assistant (§5.5.1); attention items P2 | drift, approvals, parse-profile proposals | trends |
| Map | P1b: the Map as a table (§5.5.6) | the drawing, skills and workspaces in the map | – |
| Knowledge bases | list; detail tabs Overview (checklist, sources, counts, state, contact), Sources, Documents (incl. personal data), Topics, Test questions and quality, Search settings, Sharing; the list's *Shared with us* and *All* tabs | History tab, coverage-threshold histogram and proposal | analytics |
| Source | URL list, crawl, upload forms with template parameters; preflight; preview with site-profile before/after; impact; runs; crawl report; files; quarantine; deletions; held | SharePoint with preflight, crawl hierarchy tree, parameter diff | more connectors |
| Publication | plan, re-process progress, evaluation report, keep/roll back | – | – |
| Assistants | create, draft form with description and contact, test panel on the draft, Publish, Versions (restore), audiences (clients), retention, stats, archive | create from a template, copy, version diff, Test questions with the test report, latest answer evaluation (weakness review §1); skills (with **Uses tools** when one carries tools), limits, widget audiences | risk acceptance |
| Skills, Tool servers | – | tools §3, §4 pages | import/export |
| Feedback | ratings, import health; comment reveal when A1 allows stored content (weakness review §4) | rewrite into test question | trends |
| Costs | per department, assistant, knowledge base and step, by day or month, CSV; cost cards on the assistant and knowledge base pages (§8.6) | the gateway's figures and budget status beside them (cost-attribution §6 B7) | – |
| Conversation review | – (when A1 allows stored content, weakness review §4: conversation behind a feedback item, access log) | search by assistant and period, `sensitive` two-person flow | user notification |
| Platform objects | generic forms (§5.8) | dedicated pages where generic forms prove too thin; parse-profile proposals | OBO configuration, profile changes |
| Audit log | list with names linked to their objects, events in plain words, pickers (§9.1) | field-level diff viewer | – |

### 5.4 The pilot department admin's journey (answered 2)

The platform team pairs with the department's admins for the first collection; the pages must work without them
afterwards. Every step is reachable from the **setup checklist** card ("Get started"), computed from state (no stored
progress flags), which disappears when the collection has a published assistant in Open WebUI. P1 has one client kind,
so "in Open WebUI" is checked as "a published assistant that links the collection has a client audience". The start
page's **Create knowledge base** opens the Knowledge bases page, where the drawer is.

**Without Open WebUI** (the deployment has no `OpenWebUi:Enabled`; `/bff/user` says `openWebUi.enabled: false`, for
every role) the admin UI names nothing that only holds with it: the checklist's last step reads *Connect an app with a
key under Apps and keys* and opens the assistant's Audience tab (the same done rule, a client audience); the feedback
page has no import card and its empty text names `POST /v1/feedback`; Add app offers no Open WebUI card; Check access
offers no Open WebUI user. Texts that describe apps in general name no product. A managed Open WebUI client that
exists from before stays listed as it is.

| # | Step (UI) | Page | Defaults and guidance | Guard |
|---|---|---|---|---|
| 1 | Sign in | Home | empty state "Here you build knowledge bases and assistants for your department." + **Create knowledge base** | role and department from sign-in |
| 2 | Create knowledge base | drawer | name → slug; contact (name, e-mail; the creator by default); no team field (the owner team is the creator's first team in the department, changeable under ⋯); **information class** as three explained options, none preselected ("Who may read the content?") | V1–V2 later |
| 3 | Add source | source drawer, type cards *Address list*, *Website*, *Files* | crawl: start URL; include `host/**`, default excludes, 10,000 pages, depth 10, incremental by sitemap `lastmod` once the host's site profile marks it reliable (umea.se: weekly full traversals until the 500-URL `lastmod` sample passes, weakness review §3), weekly; optional **Demote** patterns for archive sections (factor 0.5); the first `website` source of a collection turns on age decay for PDFs (730 days, weight 0.15) unless the collection already has `recency` set, shown in the drawer and editable in Search settings (weakness review §3); URL list nightly; template chosen by type, its parameters under "More settings" | V25 |
| 4 | Check | same drawer | preflight: reachability, robots status, sitemap found, estimated pages, egress, "the same website is already fetched by X" (shared crawl frontier, P2; weakness review §1), CMS recognised ("Sitevision") | fail blocks save; warn explains |
| 5 | Preview | preview panel | up to 5 URLs or uploaded files; for a crawl source the site profile the worker would apply (P1: the manual Sitevision preset; learned from P2), shown as **Before / After** text per sample page (boilerplate removed, breadcrumb, date); each stage's output (text, sections, chunks with token counts) | preview isolated |
| 6 | Save and fetch | source page → run page | saves, enqueues the first sync; live counters, the run's documents with their outcome and reason in plain English (filter: failed only) | – |
| 7 | Test questions | Test questions and quality | generated after the first runs, cost shown before (at the collection's ceiling, processing §13.1); own questions with expected documents | – |
| 8 | Publish the knowledge base | collection page | latest evaluation report shown ("Accuracy 0,78 · 86 questions"); **Publish** | – |
| 9 | Create assistant | assistant drawer → assistant page | pick an **assistant template** ("Allmän rådgivning" (general advice) preselected; P2, weakness review §1: in P1 the draft starts from `Settings.Chat` defaults: its instructions are `Settings.Chat.defaultSystemPrompt`, a
citizen-service text seeded with the first `Settings` row and edited on the Settings platform page; a missing key gives
the code's text, an emptied one an empty draft; on the Draft tab the model and the query settings sit in a collapsed
*Advanced* section, collapsed even before a model is chosen, since the missing model shows as a `version_link_missing`
warning above the form); name, description, contact person; collections (the new one preselected; unpublished ones work in Test only), model picker showing only `chat` models (`GatewayModels.ModelKind`; never embedding models) servable for the level, not-found texts and contact link from the template; saved as a **draft** | warnings only (V3, V8, V15 at publish) |
| 10 | Test | Test tab | runs the draft; sample questions from the evaluation set as one-click prompts; sources, coverage, raw vs delivered answer; **Save as test question** (P2) turns a tried question into a test question with the expected behaviour typed in | V33 |
| 11 | Publish | assistant page | **Publish**; the model is checked against the gateway (V8); from P2 the test-question report follows on the Test questions tab (advisory, §7.4) | V3–V17, V34 |
| 12 | Make available | Audience | pick client "Open WebUI (pilot)"; residency check explained if refused ("The model runs outside the EU; choose a model in the EU or on own servers") | V7–V9 |
| 13 | Open in Open WebUI | assistant page | link from the client's `PortalUrl`; the model appears under its slug | API lists it by audience |

### 5.5 Key pages (wireframe level)

#### 5.5.1 Home

```
Home                                                                         Department: Stadsbyggnad ▾
┌ Needs attention (3) ──────────────────────────────────────────────────────────────────────────────────────┐
│ ⚠ Removal waiting for a decision: Website umea.se/bygga – 212 of 840 documents (25 %)       [Review]     │
│ ⚠ The evaluation got worse after publishing: Website umea.se/bygga (−6 points)              [Show]       │
│ ● 4 documents with personal data cannot be shown in public knowledge bases                   [Show]       │
└───────────────────────────────────────────────────────────────────────────────────────────────────────────┘
┌ Fetches, last 24 hours ────────────────┐ ┌ Queue ────────────────────┐ ┌ Assistants, 7 days ───────────────┐
│ 12 done · 1 with remarks · 0 failed    │ │ 0 waiting, oldest –       │ │ 1 240 questions · 6 % not found   │
│ [Show runs]                             │ │ Processing: 3, 2 min     │ │ 👍 41  👎 9   [Feedback]           │
└────────────────────────────────────────┘ └───────────────────────────┘ └───────────────────────────────────┘
```

**P1's Home** (user, 2026-11-15, admin UI review UI1 B; `proposals/admin-ui-review.md`), top to bottom:

1. **Live**: one card per published, active assistant in the caller's departments (platform admins: all), over the
   last 7 days (Europe/Stockholm, today included): questions, not found, thumbs up and down, cost (the gateway's cost
   plus the estimate, as §8.6), each linking to the assistant's Statistics tab; "No questions in the last 7 days" when
   all are zero. Ordered by questions, most first. Absent when no assistant is published.
2. **Health**: per department, the runs started in the last 24 hours by outcome (done, with remarks, failed, running;
   linked to the knowledge base list); the queue (waiting jobs and the oldest one's age); organisation sync and model
   sync (last success, and consecutive failures when any, linked to Organisation and Models).
3. **Get started**: the setup checklists, as now. A checklist whose knowledge base is published is folded to one line
   ("Personalhandbok: 5 of 8 steps done"), and each checklist has **Hide**, remembered per browser (local storage; a
   reset brings them back). The embedding-profile first-run step stays first while it applies.
4. **Your access**, as now.

`GET /api/overview` adds, beside `hasCollections`, `setup` and `needsEmbeddingProfile`:
`live: [{ assistantId, name, departmentId, publishedVersion, questions, notFound, thumbsUp, thumbsDown, costUsd }]`
(`questions` = `AssistantDailyTotals.Messages`, `notFound` = `NotFound`, thumbs from the same rows, `costUsd` =
`AssistantDailyUsage` cost plus estimate, all summed over the 7 days and every client) and `health: { departments:
[{ departmentId, departmentName, runs: { succeeded, degraded, failed, running } }], queue: { waiting,
oldestQueuedAt }, organisationSync: { lastSuccessAt, consecutiveFailures }, modelSync: { lastSuccessAt,
consecutiveFailures } }`. `setup` items gain `published` already; the UI folds on it. As built (#724): Health lists
only departments with runs in the window; a run awaiting the deletion decision counts as *with remarks* and cancelled
runs are not counted; the queue counts `queued` jobs within the caller's scope (platform admins: all); the sync
status (a time and a count, held per instance) shows to every role; Live keeps its cards when every count is zero,
since cost may come from the test panel.

**Deployment notices** (Decisions "Deployment values across releases"; user, 2026-11-16): for a platform admin,
`GET /api/overview` returns `deploymentNotices`, a list of `{ code, names }`, shown at the top of Home after the secret
reminders (§5.11) and hideable per browser for that code and those names. The codes: `retired_variables`, the
deployment variables the stack still sets and this version no longer reads, which kalla-admin gets from the bootstrap as
`Deployment:RetiredVariables` (a comma-separated list; only names of capital letters, digits and `_` are kept, at most
50). The notice names them, says they have no effect and that the person who runs the deployment removes them, with
`kalla-bootstrap`'s log and the upgrade guide saying what replaced each. A setting that a person must choose after an
upgrade adds a code here (or a card as the embedding profile's). They are not the attention items below.

**Attention items are P2** (user, 2026-11-07, M-3): P1's Home shows the setup checklists and sync health; the
items below, and V32's Overview warning, come with `GET /api/overview`'s attention list in P2. Until then each signal is
seen on its own page (run page, source page, feedback import state, gateway status). Attention item sources: runs
`awaiting-deletion-confirmation`; evaluation reports ≥ 5 points worse than the previous run
(after a publish or nightly); a nightly recall@10 5 points below its 28-day median (processing §14.8); an assistant
test run with fewer questions as expected than the previous version's (§7.4); PII-flagged documents in `public`
collections; degraded runs; sources with ≥ 3 failed
runs; sources with N (default 3) runs without a complete traversal (simplification pass §2); feedback import failing ≥ 4
runs; org or model sync failing; an assistant or knowledge base whose gateway key is unusable (disabled, revoked,
expired, its team or department inactive), has lost a model or residency the object uses, or whose budget is spent,
with the reason and a link to the key in the gateway (§4.10, V39; cost-attribution §4.2). The admin derives these from
the mirror (§4.10) and, from gateway phase 3, the key's budgets (`exhausted`, §4.5); it stores no refusal of its own. A
`402` or key refusal seen by chat or the worker shows in the turn outcomes (`unavailable`) and the run's reason
(`gateway.budget_exhausted`, `gateway.key_unavailable`, engine §7.3); template versions without a supporting worker; from P2 (weakness review §1): site-profile proposals (platform
admins) and published assistants without a message in 90 days (`AssistantDailyTotals`; published ≥ 90 days ago), with
**Archive** and **Keep** (dismissed for 90 days) (simplification pass §15); a test report on an assistant's new
version with more questions not as expected than the previous version's (P2); P2 drift and pending approvals.

#### 5.5.2 Knowledge base

```
Knowledge bases › Bygglov                                      [Internal] [Not published]   [Publish] [⋯]
Overview  Sources (3)  Documents (1 204)  Topics (4)  Test questions and quality  Search settings  Sharing
┌ Get started ────────────────────────────────────────┐ ┌ State ────────────────────────────────────────────┐
│ ✓ Source added            ✓ First fetch done        │ │ Not published: only visible in Test.              │
│ ✓ Test questions created (86) ◐ Evaluation running  │ │ Last fetched 30 Oct 02:14 · 1 204 documents        │
│ ○ Publish   ○ Create assistant   ○ Make available    │ │ Used by: Bygglovsassistenten (draft)               │
└─────────────────────────────────────────────────────┘ └───────────────────────────────────────────────────┘
Sources                                                                                           [Add source]
│ Name                  │ Type       │ Last run               │ Documents│ Next        │                      │
│ umea.se/bygga         │ Website    │ ✓ 30 Oct 01:12         │ 840      │ Sat 01:00   │ [Fetch now] [⋯]      │
│ Blanketter            │ Files      │ ⚠ 2 failed             │ 41       │ –           │ [Upload] [⋯]         │
```

Sharing and Topics tabs (English UI):

```
Knowledge bases › umea.se (Communications)                 [Public] [Published] [Edit] [Publish] [⋯]
Overview  Sources (1)  Documents (2 323)  Topics (9)  Test questions and quality  Search settings  Sharing
┌ Sharing ──────────────────────────────────────────────────────────────────────────────────────────┐
│ Who may use this knowledge base in their assistants?                                              │
│ ( ) Only Communications   ( ) Selected departments [ … ]   (•) Whole municipality                 │
│ Shared use is read-only: other departments use the content in their assistants, not the sources.  │
│ Used by 4 assistants in 3 departments                                                             │
│ │ Assistant            │ Department              │ Topics                 │ Status     │          │
│ │ Permits assistant    │ Building & Environment  │ Building and housing   │ Published  │          │
│ │ Customer service     │ Contact centre          │ all                    │ Draft      │          │
│                                                                                  [Save sharing]   │
└───────────────────────────────────────────────────────────────────────────────────────────────────┘
Topics (set by path rules on the sources, Processing › Topics)
│ Topic                │ Rule                               │ Documents │
│ building-and-housing │ www.umea.se/byggaboochmiljo/**     │ 412       │
│ politics             │ www.umea.se/kommunochpolitik/**    │ 655       │
```

A department the knowledge base is shared with opens it read-only: Overview (owner, contact, level, document count)
and Topics, nothing else.

The document count (the list, the Overview and the Sources table's *Documents*) is one number: **live** documents, not
deleted and with a live chunk set, so a source in its first fetch shows 0 until its documents are indexed. The
*Documents* tab lists every document that is not deleted, marked *Searchable* or *Not searchable*. *Used by* lists
archived assistants too, marked *Archived* (they still block deletion and can be unarchived, #374); a deleted
assistant is not listed and does not block.

**Delete in the UI** (Decisions "Deleting knowledge bases, sources and assistants"). *Delete* is in the ⋯ menu of a
knowledge base, of a source and of an assistant, for `Manage`. Each opens a dialog that says what goes and cannot be
restored, and enables its button when the object's name is typed:
- a knowledge base: its documents count; when it is in use, the dialog shows `collection_in_use`'s list instead
  ("Used by: …; delete or unlink them first", "Delete its sources first") and has no delete button;
- a source: its documents count, and that they leave the search at once;
- an assistant: the apps it is offered to, its questions in the last 7 days, and that its name is free afterwards;
  its conversations and statistics stay their time.

**The knowledge base list** (user, 2026-11-15, UI2 B) has the columns Name, Department, Information class, Sharing,
Documents, **Last fetch** (the latest run of any of its sources: its start time and outcome, *Done*, *Done with
remarks*, *Failed* or *Running*, linked to the run; "Never" before the first) and **Quality** (the latest finished
evaluation's recall@10 with its date; "–" before the first), then State. `GET /api/collections` items add `lastRun: {
id, sourceId, state, startedAt, finishedAt } | null` and `latestEvaluation: { recallAtK, finishedAt } | null`, read in
grouped queries for the page (no per-row round trips). On a phone the card shows the same fields. A shared row
(§4.5) has neither (shown "–"); Quality takes the hybrid run first, as the Overview does.

**Lists open as links** (every list page): the first cell of a clickable row is a real link (`<a href>` through the
router) to the row's page, so a row opens in a new tab, its address can be copied, and keyboard users get a link;
clicking elsewhere on the row still opens it. The *Overview* knowledge base's **Used by** names link to the
assistants, and an empty **Contact** offers **Add contact** (it opens the Edit drawer).

**Edit** (a header button next to **Publish**, as on the assistant page, until the admin UI has a menu component for
`⋯`) opens a drawer with the collection's own fields: name, contact, **level** (`edit` on the
collection; Decisions "Level change in the edit drawer", #446) and the **gateway key** (the key picker of §5.5.4: the
assigned keys of the knowledge base's department; indexing, re-embedding and evaluation run as it). The Overview's
*State* card shows the key's name, its read-only settings, its status (an unusable key: "Fetching is paused: the
gateway key is disabled", with the reason) and from gateway phase 3 its budget status, with a link to the key in the
gateway; without a key it says "Charged to the platform: choose a gateway key". Below it the Overview has the *Cost*
card of §8.6 (indexing and evaluation, last 30 days), for the owning department only. A level change is saved with the rest; a refusal
(`collection_level_conflict`: sources above the new level; `assistant_below_collection`: published assistants below
it) is shown in the drawer after **Save** with the affected objects by name.

#### 5.5.3 Source (website) and the deletion decision

```
Website: umea.se/bygga                                                     [Fetch now] [Pause] [⋯]
Settings  Runs  Crawl report  Deletions (1)  Kept (0)
┌ Start addresses ─────────────────────────────────────────────────────────────────────────────────────────┐
│ https://www.umea.se/bygga                                                                       [Remove]  │
│ Only include  www.umea.se/bygga/**               Skip  **/sok?*  **?*print=*  [+]                          │
│ Demote  www.umea.se/bygga/**/arkiv/**  ×0,5  [+]                                                           │
│ At most 10 000 pages · depth 10 · changed pages by the sitemap · Saturdays 01:00                          │
│ Site profile: Sitevision, version 3 (learned 30 Oct)                             [Preview]                 │
│ ▸ More settings (template parameters: passage length, overlap, …)                                          │
└───────────────────────────────────────────────────────────────────────────────────────────────────────────┘
  ⓘ The scope changes. Of the source's 840 searchable documents, those outside are removed at the next fetch.
    Affects the assistants: Bygglovsassistenten, Kundtjänst bygg (draft).             [Cancel] [Save anyway]

Removal waiting for a decision · run 30 Oct 01:12
  212 documents were not found in the run and have not been removed. The limit is 20 % (at least 5).
  Examples │ https://www.umea.se/bygga/…/altan │ Reason: 404 · last seen 23 Oct  [Show more] [Download all (CSV)]
  [Keep the documents]  [Remove 212 documents]
```

**JavaScript pages** (connectors design §8; Decisions "JavaScript rendering"): the URL-list and crawl forms have *Fetch
pages with* *HTTP* (default) / *A browser (JavaScript)* (`fetchMode`) and, for the browser, an optional *Content is
ready when this element has text* (`readySelector`, CSS). The preflight shows the raw and rendered word counts per start
URL and, when the raw page holds under half the text, "This site seems to need JavaScript" with a button that switches
the setting. The add drawer's *Preview* fetches over HTTP whatever the unsaved fetch mode (the stage preview's payload
carries no fetch settings). The crawl report counts rendered pages, render failures and pages kept from their last good
content.

#### 5.5.4 Assistant with test panel

The wireframe shows the P2 page. In P1 ([weakness review §1](../archive/passes/weakness-review.md#1-p1-builds-a-platform-before-anyone-sees-one-good-answer)) the tab Test questions, **Compare** in Versions, **Copy**,
**Save as test question**, the test summaries and the answer evaluation on Statistics are absent; **Publish** shows
the warnings only.

```
Assistants › Bygglovsassistenten  (model name: bygglov)   [Internal] [Published: version 3] [Draft changed 30 Oct]
Svarar på frågor om bygglov, anmälan och avgifter. Contact: Anna Andersson, bygglov@umea.se  [Edit]
Uses tools (P2)                                                               [Publish]  [⋯ Copy, Archive]
Draft  Test  Test questions (8)  Versions (3)  Audience  Storage  Statistics
┌ Draft (changed since version 3) ─────────────────┐ ┌ Test (runs the draft ▾) ───────────────────────────┐
│ Instructions  [textarea, template filled in]     │ │ Question [Behöver jag bygglov för en altan?] [Test]│
│ Knowledge bases [Bygglov ×] [Detaljplaner ×] [+] │ │ Suggestions: · Vad kostar ett bygglov? · …         │
│ Skills (P2) [Bygglov ×] [+]                      │ │ Sources in the search (coverage: ok, 0,31 ≤ 0,50)  │
│ Model  [fors/chat-eu ▾] EU · 128k                 │ │ 1 Altaner och uteplatser – umea.se/…  0,31  cited  │
│ Not found text (sv) [Jag hittar inget …]         │ │ Answer the user sees                  | Raw text   │
│ Not found text (en) [I cannot find …]            │ │ Nej, om altanen är lägre än 1,8 m … [1]            │
│ Contact links [Kontaktcenter 090-16 10 00]       │ │ Model fors/chat-eu · EU · 2,1 s · 0,0012 USD        │
│ ▸ Advanced: follow-up questions, history, length │ │ [Save as test question]                            │
│ ⚠ 1 warning (shown at Publish)                   │ │                                                    │
│ [Save draft]  [Discard draft]                    │ │                                                    │
└──────────────────────────────────────────────────┘ └────────────────────────────────────────────────────┘
```

- **Knowledge** (Knowledge bases in the draft): each link shows the knowledge base, its owning department when it is
  shared ("umea.se (Communications, shared)"), and its topics ("all topics", or chips with **+ topic**); **Add
  knowledge base** opens a picker with *Ours*, *Shared with us* and *All* (relative to the assistant's department, not
  the user's), showing level, document count and topics. The topic field, labelled "Limit to topics (empty: all
  topics)", is free text with the knowledge base's current
  topics as suggestions, so a topic a path rule is about to set can be linked; V11 `topic_invalid` checks the format.
- **General knowledge** (`answer.generalKnowledge`, in the Draft tab, not under Advanced, since it changes what
  users get): a radio group *Off: only the knowledge bases* (`off`, default), *Supplement: knowledge bases first,
  general knowledge for explanations* (`supplement`), *Fallback: as Supplement, and answer from general knowledge when
  nothing is found* (`fallback`), *Open: knowledge bases and general knowledge* (`open`), each with one line of help
  that says what sets it apart (UI4 B, 2026-11-15): *Off* "Answers only from the knowledge bases; when they have
  nothing, it says so and shows the contact links"; *Supplement* "May use general knowledge to explain terms in an
  answer from the knowledge bases; when they have nothing, it says so and shows the contact links"; *Fallback* "As
  Supplement, and when the knowledge bases have nothing it answers from general knowledge, marked as such, with the
  contact links"; *Open* "Uses the knowledge bases and general knowledge freely; answers from general knowledge are
  marked, with the contact links". For a `public` assistant all four stay enabled; any choice but `off` shows the
  warning "Not recommended for a public assistant: citizens may get general answers that are wrong for Umeå kommun"
  under the group, and **Publish** then asks for one extra confirmation (V36, #445). A
  published version with a level other than `off`
  version shows the badge `[General knowledge]` on the assistant page and in the assistant list. The test panel runs
  the chosen mode, so a weak question shows the caveat line.
- **Answer length** (under Advanced, Decisions "Answer length from the model"): *Longest answer* and *Context size* each
  have a checkbox **Use the model's limit** (checked by default; checked stores no value, `answer.maxOutputTokens` and
  `answer.contextTokens` absent). Next to it the model's limit is shown ("Model limit: 128,000 tokens", or "Model limit
  unknown: the provider's default applies; a platform admin can enter it under Gateway models"), and, when
  `Settings.Chat.MaxAnswerTokens` is set and lower, "Platform ceiling: 8,000 tokens" next to it. Unchecked, a number
  field opens with the model's limit as its upper bound (V17). Changing the model keeps the checkboxes, so a version
  that uses the model's limit follows the new model.
- **Gateway key** (in the *Edit* drawer, an assistant's own field, saved directly under V39): a picker of the assigned
  keys of the assistant's department (`GET /api/gateway/keys`), each with its name, team, status and models; an
  unusable key is shown but cannot be chosen. The page header shows the chosen key's name with its settings
  read-only (models, residencies, PII policy, content guard, rate limits, output cap), its expiry, from gateway phase 3
  its budget status (`GET /api/gateway/keys/{id}/budget`: per budget its scope, spent of limit with the percentage,
  the period's end, and *Used up* when exhausted), and a link to the key in the gateway (`admin_url`); nothing about the key
  is edited here. A key with a system prompt shows V39's warning. The model pickers mark models the key may not use
  as unavailable ("Not available to this key"), and the answer-length limits show the key's output cap when it is
  lower than the model's. An assistant whose key is unusable, or whose budget is spent, shows the badge
  `[Unavailable]` with the reason ("The gateway key is disabled", "The department's budget is used up") and raises
  the attention item (§5.5.1); chat answers it as unavailable (chat §3.6). Without a key the header says "Charged to
  the platform: choose a gateway key", and once the gateway acts as keys, **Publish** asks for one first (V39).
- **Who can use it** (Audience) says in plain words what an audience means: "Open WebUI (pilot): everyone who can sign
  in to it"; department and group audiences follow the W3 spike and E1. The tab's **Check access** runs explain access
  (§4.5, identity §4.9) for a client and a subject; a key subject is entered by its key id (no key picker), and the
  residency ceiling shows as the API returns it (the header form, e.g. `eu`).
- **Wording** (UI4 B, 2026-11-15): the header's button that opens the drawer of the assistant's own fields is **Edit
  details**; the Draft tab's lead says what saving does: before the first publish "Saving keeps a draft; publish it to
  make the assistant available", afterwards "Saving starts a draft from version n; version n stays live until you
  publish", and with a draft "Draft changed since version n".
- **The assistant list** (UI2 B) has Name (with the model name), Information class, **Knowledge bases** (names, from the
  published version, else the draft), **Model** (the same version's), **Published** (version and date; "Draft only"
  before the first; *Archived*) and **Questions, 7 days**. `GET /api/assistants` items add `collections: [{ id, name }]`,
  `model: string | null`, `publishedVersion: number | null`, `publishedAt` and `questions7d` (`AssistantDailyTotals.
  Messages`), in grouped queries. "Model name" here is the slug.
- **Badges**: `[Draft]` alone before the first publish; `[Published: version n]` afterwards, plus `[Draft changed …]`
  while a draft exists; `[Archived]` when archived. **Uses tools** is a derived label (P2), never a
  setting: it appears when a linked skill's served version carries tools, and the Skills field explains that such
  a skill switches the assistant to the agent loop. Description, contact (name and e-mail) and **level** (in the
  *Edit* drawer, #446; the published version is re-checked, V3/V7/V8/V9/V36, refusals shown in the drawer, and a
  change to `public` with general knowledge on asks the V36 confirmation) are the assistant's own
  fields, saved directly, like Audience and Storage; everything in the Draft tab is versioned.
- **Publish** opens a dialog with the diff against the published version (P2), the remaining warnings (refusals
  block) and the cost of the test run (P2); after publish (P2) the Test questions tab shows the
  report as it arrives.
- **Versions**: a list (number, published by, when, "restored from version n", test summary "7 of 8 as
  expected", P2); **Compare** (P2) shows the structured diff (settings by name, collections and skills added or removed, a line
  diff of the instructions with additions and removals as text marked `<ins>`/`<del>`, not by colour alone);
  **Restore** on any earlier version publishes it again as a new version after one confirmation ("Version 2
  is published again as a new version. Your draft is kept.").
- **Test questions** (P2): the set (question, expected behaviour, expected document or phrase), **Add**, **Run test
  questions** (on the draft or the published version), and the latest report as the §7.2 card (§7.4).
- **Copy** (in ⋯, P2) asks for name, slug, department and owner team and opens the copy's draft.

The *Statistics* tab shows the date of last use, the *Cost* card of §8.6 (by step and client, gateway and estimated
cost apart) and, from P2 (weakness review §1), the **latest answer evaluation**: faithfulness and citation precision of the last
`answers` run with question counts, compared with the previous run, next to the live `uncited_answer` rate
(processing §14.8, `AnswerEvaluationService` §2.2).

#### 5.5.5 Feedback

```
Feedback                                      Period: 7 days ▾  Assistant: All ▾  Rating: 👎 ▾  [Only with comment]
│ Time           │ Assistant           │ Rating│ Reason                │ Answer       │ Cited documents    │          │
│ 30 Oct 10:42   │ Bygglovsassistenten │ 👎    │ Incorrect             │ Answered     │ Altaner och …      │ [Read comment]  │
│ 30 Oct 09:15   │ Bygglovsassistenten │ 👎    │ Other                 │ Found nothing│ –                  │ [Read comment]  │
   Read comment → dialog: "You are reading a user's comment. The reading is logged with the reason: Feedback review."
   [Cancel] [Read comment]   (reviewers also see [Read the conversation], which needs its own reason)
Import from Open WebUI: Last run 10:45 · 2 could not be matched to an answer in the last 24 hours
```

#### 5.5.6 Map (department map; P1b as a table)

P1b builds the **table**; the drawing below follows later. The table has one row per department → knowledge base →
assistant → audience path, with the responsible contact of each, and filters for department and object. Archived
assistants are left out (the map shows who uses what now, #374):

```
Map                                          Department: All ▾   Show: Everything ▾   [Table] [Drawing]
│ Department      │ Knowledge base     │ Shared       │ Assistant          │ Channel            │ Responsible  │
│ Communications  │ umea.se            │ municipality │ –                  │ –                  │ K. Lund      │
│ Building & Env. │ umea.se ↗ (shared) │ –            │ Permits assistant  │ Open WebUI (pilot) │ A. Andersson │
│ Building & Env. │ Permits forms      │ –            │ Permits assistant  │ Open WebUI (pilot) │ A. Andersson │
```

The drawing:

```
Map · Stadsbyggnad                                               Show: [Everything ▾]  [Table]
Sources                   Knowledge bases              Assistants                        Audiences
umea.se/bygga  ─────────▶ Bygglov (Internal) ────────▶ Bygglovsassistenten (v3) ───────▶ Open WebUI (pilot)
Blanketter     ─────────▶                    └───────▶ Kundtjänst bygg (draft)           –
umea.se/plan   ─────────▶ Detaljplaner ─────────────▶ Bygglovsassistenten (v3)
                          Avfall (shared from Teknik) ▶ Kundtjänst bygg (draft)
```

One column per stage, each item a link to its page; selecting an item highlights what it feeds and what feeds it. The
lines are drawn as plain SVG next to lists that carry the same relations as text ("feeds: …", "used by: …"), and
**Table** shows one row per source → collection → assistant → audience path, so nothing depends on the drawing (no
graph library, §10.2). Shared collections appear with their owning department; assistants show their published
version, draft-only links are marked "(draft)". In the table, *Responsible* is the assistant's contact when the row
has an assistant, else the knowledge base's; *Shared* is the own knowledge base's sharing level, "–" for one shared
in. Source: `GET /api/organisation/departments/{id}/map`.

#### 5.5.7 Files: a knowledge base of uploaded files (user, 2026-11-09)

Many knowledge bases hold only files someone uploads (a department's guidelines, a support team's instructions). The
pages treat that as a first-class case, not as a crawl source without a crawl:

1. **One step to start.** The *Create knowledge base* drawer asks what goes in: *Files I upload* or *A website or a
   list of addresses*. *Files* creates the knowledge base and one upload source (named *Files*, the collection's
   classification, default template) in the same submit, then opens the knowledge base on its **Files** tab. The other
   choice keeps today's flow (the source drawer opens next).
2. **A Files tab on the knowledge base.** Shown when the knowledge base has an upload source; with several, a picker
   above it names the source. It holds the drop zone (files and folders), the list and the details below, so nobody
   has to find the source page. The source page keeps the same card. A knowledge base without an upload source gets
   *Add files* on its Sources tab, which creates the source and opens the tab.
3. **Live progress.** A summary bar from `files/summary`: *120 files · 34 processing · 84 indexed · 2 failed*, with a
   progress bar. While anything is processing, the summary and the visible rows refresh every 3 s; polling stops when
   nothing is processing or the tab is hidden, and starts again after an upload commit.
4. **A list for hundreds of files.** Search by name or path; state filter chips with their counts (*All*,
   *Processing*, *Indexed*, *Failed*); sort by name (folders shown as group headers) or newest; select all loaded
   rows; delete the selection; **Retry failed** (the source's `retry-failed`) when anything failed; *Load more*.
   States: *processing* is `received`, `clean` and `unscanned`; *indexed*; *failed* is `refused` and `malware`, each
   with its reason in plain language (`messages.ts`). A file that is `unscanned`, or `indexed` with no `scannedAt`,
   carries the note *Not scanned for malware* in its row and in its details (the deployment runs with the scan off,
   connectors §11). A run whose report says `malwareScan: "off"` shows *Files in this run were not scanned for
   malware* in its summary, on a crawl source's page as well.
5. **Folders are kept.** A dropped or picked folder sends each file's relative path (`HR/Policys/semester.pdf`) in
   `X-File-Name` (§4.5). The list groups by folder, and the tab says that a topic rule on the path
   (`HR/**`, Topics tab) tags a folder's documents. Files picked one by one have no folder.
6. **File details.** A row opens a panel: name and path, size, type, uploaded by and when, replaced when, state and
   reason, a download link, and for an indexed file the document's chunks (`/api/documents/{id}`, `/chunks`) and the
   assistants that use the knowledge base. *Replace* and *Delete* are there too.

The main session owns this feature's code (user, 2026-11-09), on `main-session/` branches, in stream b's paths
(`src/admin-ui/**`, `src/Kalla.AdminApi/**`).

### 5.6 Empty and error states

| Situation | Text and action |
|---|---|
| No collection | "Here you build knowledge bases and assistants for your department. Start by creating a knowledge base." [Create knowledge base] |
| No source | "The knowledge base has no content yet. Add a website, a list of addresses or files." [Add source] |
| No run yet | "Nothing has been fetched yet. The first fetch starts when you save the source." |
| Not published | "Not published: the knowledge base is only visible in Test until you publish it." |
| No test questions | "Test questions are created automatically when the first fetch is done (about 10–20 minutes)." |
| Assistant without test questions (P2) | "The assistant has no test questions yet. Add a few questions it should handle, and they run every time you publish." [Add test question] |
| No feedback | "No feedback yet. It arrives when users give a thumbs up or down in Open WebUI." |
| No role | copied no-access page: "Your role does not give access to this page. Contact the platform team if you need it." |
| Load failure | copied `AsyncState` with "Try again" (`components/ui/AsyncState.vue:14`) |

### 5.7 Wording conventions

- Plain English for non-specialists: "you", short sentences, verbs on buttons ("Publish", "Fetch now", "Keep the
  documents"), sentence case; technical terms (`recall@10`, `OCR`, `robots.txt`) are explained once. Swedish names and
  content (departments, collections, page titles, chunks) are shown as written and marked `lang="sv"`.
- Every error says what happened and what to do; destructive confirmations name the object, the count and the
  consequence, and the button repeats the action ("Remove 212 documents").
- Dates, numbers and times as the gateway's `utils/format.ts`, time zone Europe/Stockholm; money stays USD.
- Every UI string lives in `utils/labels.ts` or `utils/messages.ts`, so a Swedish translation later (Decisions, "Admin UI in
  English") touches only those files. The page, button and step names quoted in this document are the English UI
  terms; the glossary's Swedish column below is that translation.
- One glossary, enforced in review (`utils/labels.ts` is the single source of these words). The English UI term is the
  concept in plain words, except: collection → "knowledge base", chunk → "passage", pipeline → "processing", sync now
  → "fetch now", PII → "personal data". The Swedish column is the later translation:

| Concept (code) | Swedish term | Concept (code) | Swedish term |
|---|---|---|---|
| collection | kunskapsbank | evaluation set / question | testfrågor / testfråga |
| source: `url-list`, `crawl`, `upload`, `sharepoint` | källa: adresslista, webbplats, filer, SharePoint | evaluation report | utvärdering |
| document / chunk | dokument / textavsnitt | recall@10 | träffsäkerhet (recall@10) |
| pipeline / template / parameters | bearbetning / mall / mallinställningar | coverage threshold | täckningsgräns |
| draft / published | utkast / publicerad | not published | ej publicerad |
| run / sync now | körning / hämta nu | deletion guard / held | borttagningsspärr / behållna |
| preview / preflight | förhandsgranska / kontrollera | PII / clear | personuppgifter / godkänn för offentlig |
| site profile | webbplatsprofil | share list | delning |
| assistant / audience | assistent / målgrupp | skill / tool server / grant | färdighet / verktygsserver / behörighet |
| feedback / comment / reason | återkoppling / kommentar / skäl | conversation / reviewer | samtal / granskare |
| `public` / `internal` / `sensitive` | offentlig / intern / känslig | `OnPrem` / `Eu` / `External` | kommunens servrar / EU / utanför EU |
| department / team | förvaltning / team | audit log | granskningslogg |
| assistant version / published version / draft | version / publicerad version / utkast | restore / copy / archive | återställ / kopiera / arkivera |
| assistant test question / expected behaviour | testfråga / förväntat beteende | uses tools (derived) | använder verktyg |
| assistant template / department map | assistentmall / karta | owner team | ägarteam |
| topic | ämne | sharing level: only our department / selected departments / whole municipality | delning: bara vår förvaltning / valda förvaltningar / hela kommunen |
| responsible (contact) | ansvarig | shared with us | delade med oss |
| `demote` pattern (weakness review §3) | nedprioritera | age decay `recency` (weakness review §3) | åldersavtagning |

### 5.8 Generic forms for platform objects (P1)

Platform-admin objects get no dedicated pages in P1 (reasoning: simplification pass §10). In P1a they are set up
through Scalar only; the generic forms below come in P1b (Decisions "P1a plan answers"): settings sections, data protection,
clients, keys, templates, assistant templates (simplification pass §15; P2, weakness review §1), site profiles, embedding profiles (read),
gateway models, organisation mirror (read); P2 department settings (owner-team scoping, also for department admins).

- One **Platform objects** page (`/plattform`, platform admins) lists the resources; each opens a list and a form in a
  drawer. A registry in the UI (`platform/resources.ts`) names each resource's list, create, update and action endpoints
  and its list columns; the form's fields come from that operation's request schema in the OpenAPI document
  (`/openapi/v1.json`, generated from the same DataAnnotations the API validates with, so the form and the server agree;
  Admin-only, the audience of these forms). Controls follow the table of §6.3; `jsonb` members (e.g. a site profile,
  `Settings.Crawl`) are a JSON text area validated by the server, except the crawl section's **host politeness**
  (connectors §6.6), which is a table (host, preset, the members that differ, the robots reason) with a drawer per host,
  since a mistake there changes how hard we hit someone's site. A new, rotated or revealed key's secret is shown once
  with a copy button.
- Labels come from `PLATFORM.fields` (`labels.ts`), else the schema `title` (the .NET generator writes none), else the
  property name; help is the schema `description`. `minimum`, `maximum` and `maxLength` in the document are not
  checked in the form; the server validates (§6.3).
- A field whose meaning the schema cannot carry gets a hint under it from `PLATFORM.fieldHints` (`labels.ts`). Keys'
  `requestsPerMinute`: "Chat messages per minute for apps that use this key as one shared user (integration and
  internal clients). Empty: the platform's per-user limit. Open WebUI users always get the per-user limit." (chat design
  §4.1 step 4)
- Actions that are not CRUD (rotate a key, run a sync, approve or roll back a site profile) are buttons from a small
  per-resource action list; anything rarer is done in Scalar (`/openapi/v1.json`).
- Site profiles: the list shows host, preset, version, state (applied/proposal), sources using it; the detail shows the
  version JSON, the proposal's check results and the preview viewer's before/after texts, with **Approve**,
  **Reject** and **Restore to version …**. In P1 (weakness review §1) every version is a manual, hand-checked JSON
  edit (the umea.se Sitevision preset first): no proposals, so only editing and **Restore**.
- Gateway models: a **Check embedding canary** button (`POST /api/gateway/canary-check`, processing §11) shows each active profile's similarity, pass or fail and time; on opening, the page shows the last check from `GET /api/gateway/canary-check` and calls nothing that reaches the gateway. It is the only way the canary runs.
- The same concurrency (`version`), error codes, audit and accessibility rules apply.
- Clients and keys left the generic forms (§5.9, user 2026-11-13). Every generic form shows the row's `id` with a copy
  button, a schema `enum` as a select, and a `date-time` field as a date and time input.
- §5.10 puts each platform page in the side menu, drops this index page and gives Settings a page of
  its own; §5.9.1 removes the generic `keys` resource.

### 5.9 Clients and keys (user, 2026-11-13)

Connecting an app took two generic forms, free-text kinds and levels, a client GUID nobody could see and a JSON scope
object (session 34 review). Clients get a page of their own, and keys are made from their client.

**Page `/plattform/clients`** (platform admins; it replaces the generic resource `clients` and keeps its place in the
Platform index). The list shows name, kind (its label), information class, enabled and the number of active keys.
**Add client** opens a drawer in three steps:

1. **Kind**, as cards with one sentence each (`PLATFORM_CLIENTS.kinds` in `labels.ts`). `open-webui`, `integration`
   and `internal` can be picked; `widget`, `own-client`, `teams`, `mcp-client` and `entra-app` show "Later" and are
   disabled (P2).
2. **Fields for that kind** only:

   | Field | Control | Shown for |
   |---|---|---|
   | Name | text | all |
   | Information class (`classificationCeiling`) | select `public`/`internal`/`sensitive`; default `internal` for `open-webui`, `sensitive` for `internal`, `public` for `integration` | all |
   | Stores chat history | switch; disabled with the reason at `sensitive` (`StoresHistory ⇒ ceiling ≠ sensitive`); default on for `open-webui` | `open-webui`, `integration` |
   | Feedback kept (days) (`clientRetentionDays`) | number, default 90 | `open-webui` |
   | OAuth client ID | text, with the hint "the `azp` of the app's tokens" | `internal`, `integration` |
   | Portal URL | URL | `open-webui` |
   | Enabled | switch, default on | all |

   `toolPolicy` is not shown (P2): the server's default stays. `allowedOrigins` waits for `widget`.
3. **First key**: "Create a key now" (on by default) with the key's name (default "<client name> key"). Created with
   the client in one transaction. The result shows the secret once with a copy button and the client's id with a copy
   button; for `open-webui` also the two stack lines `KALLA_OPENWEBUI_API_KEY=<secret>` and
   `KALLA_OPENWEBUI_CLIENT_ID=<id>` with one copy button, and the sentence that Open WebUI picks them up when the
   stack is updated.

**A client's drawer** has the facts (id with copy, kind, created/updated), the edit form of step 2 (the kind is fixed
after creation), **Keys** (this client's keys: name, prefix, status, expiry; **Rotate**, **Reveal**, **Revoke** and
**New key** in place) and **Used by** (the assistants whose audience includes it, linked to their pages; "No assistant
yet: add this client under an assistant's *Who can use it*" when there is none).

**Key form** (`KeyForm.vue`, in the client's drawer and on the Keys page): name, description, team (a picker of the
organisation's teams, optional), **Limit to assistants** and **Limit to knowledge bases** (multi-selects; hint "Empty:
everything this client's audience allows"), expiry (a date; empty never expires), allowed IP ranges (one per line),
requests per minute (with the hint of §5.8). On the Keys page a new key first picks its client from a select; the list
gains a client column.

**API** (contract for the page):
- `ClientRequest`'s `kind`, `classificationCeiling` and `toolPolicy` carry their allowed values as `enum` in the OpenAPI
  document.
- A client with `storesHistory` at `sensitive` answers `400` with a field error on `storesHistory` ("A client that keeps
  chat history cannot be sensitive.") before the database check.
- `POST /api/clients` takes an optional `firstKey: { name, teamId? }` and answers `201` with
  `ClientCreatedDto { client: ClientDto, key: KeyCreatedDto | null }`; client, key and both audit rows are one
  transaction.
- `GET /api/clients` (`ClientSummary`) adds `activeKeys` (int). `GET /api/clients/{id}` (`ClientDto`) adds
  `assistants: [{ id, name, departmentId, published }]` (the assistants with a `client` audience row for it).
- `GET /api/keys` keeps its `clientId` filter; `KeyDto` adds `clientName`.

#### 5.9.1 Second pass: connect an app in one go (user, 2026-11-13; M-16, built in 0.6.0)

After §5.9's first pass a new client still cannot use anything (access is given on each assistant's page, one at a
time), an integration's result is a secret and a GUID with no address or model name, the key form shows every assistant
and knowledge base of the platform, and keys have a second page. This pass changes §5.9 as follows; where it differs
from the text above, this wins once built.

**Name.** The page is **Apps and keys** in the UI (`labels.ts`); the API, the tables and the code keep `client`.

**Add app** has four steps:

1. **Kind** (unchanged).
2. **Fields** (unchanged).
3. **What it may use**: the assistants this app offers, as a searchable checklist (name, department, *Published* or
   *Draft*, information class) from `GET /api/assistants`. An assistant above the app's information class is disabled
   with "Above this app's information class" (V7); a `sensitive` assistant is disabled for an app that keeps history
   when the policy says so (V9). Nothing ticked is allowed and says "This app can't use any assistant yet". Opened from
   an assistant's *Who can use it* tab (**Connect a new app**), that assistant is ticked.
4. **First key** (unchanged).

Client, audience rows, key and their audit rows are one transaction.

**The result is "How to connect"**, built from what was just made, each value with a copy button:

| Kind | Shown |
|---|---|
| `integration`, `internal` | the API address (`<apiBaseUrl>/v1`), the key (once), the model names (the slugs of the assistants picked in step 3, as `GET /v1/models` lists them), and one `curl` call of `POST /v1/chat/completions` with the first model, copied as a block |
| `open-webui` | a sentence: the deployment connects Open WebUI by itself (identity design §5.2, M-17 C4). The kind is not offered in Add app ("Set up by the deployment"); the deployment's own client is marked "Managed by the deployment" in the list and the drawer, which hide New key, Rotate, Show key and Revoke for it (the API refuses them with `client_managed_by_deployment`); its settings and assistants are edited as any app's |

The client id is under "Details", not in the main path. Without
`apiBaseUrl` the address row says "Ask the platform team for kalla-api's address".

**An app's drawer** gains **How to connect** (the same block, the key shown through *Show key*) and replaces *Used by*
with the checklist of step 3 and a **Save** of its own. The list and the drawer show when each key was **last used**
("Never used" until the app's first call), so the person connecting an app sees that it worked.

**Key form.** A new key asks for its name only. Description, team, **Limit to assistants**, **Limit to knowledge
bases**, expiry, allowed IP ranges and requests per minute are under a collapsed **Restrictions**, open when any of
them is set (or one of them is refused). A new key has no Enabled switch; an edit has it beside the name. **Limit to
assistants** lists the assistants the app offers, and any the key still names that it no longer offers, marked so they
can be unticked. **Limit to knowledge bases** lists every knowledge base: it also limits the knowledge routes, which an
`internal` app calls without an assistant (`Knowledge:OpenToClientKinds`), and an assistant whose published knowledge
bases are all outside it (the access policy's key scope).

**Keys page.** The generic resource `keys` and `/plattform/keys` go. The Apps and keys page has two tabs: **Apps** and
**All keys** (every key with its app, status, expiry and last use; filter by status; a row opens its app's drawer at
the key). Department admins' keys of own teams (§3.3) have no page today and get none here.

**Other generic forms.** A field that takes another object's id (a GUID) is a select of that object's rows, by name;
no form asks for a typed GUID.

**API** (contract):
- `POST /api/clients` takes an optional `assistantIds: [guid]`; each gets a `client` audience row for the new client,
  checked as `PUT /api/assistants/{id}/audiences` checks it (V7, V8, V9). A refusal is a field error on `assistantIds`
  that names the assistant.
- `PUT /api/clients/{id}/assistants` with `{ assistantIds, version }` (policy `Admin`) makes the set of assistants
  that have a `client` audience row for this client exactly `assistantIds`; same checks; one `audiences` audit row per
  assistant that changed. Answers `ClientDto`.
- `ClientDto.assistants` items add `slug` and `classification`.
- `GET /api/clients/connection` answers `{ apiBaseUrl: string | null }` from the deployment knob `Admin:ApiPublicUrl`.
  The deployment templates give kalla-admin the value kalla-api already gets as `Knowledge:PublicBaseUrl`
  (`KALLA_API_PUBLIC_URL`, or its default from the host once the proposal's C1 is in): no new variable to fill in.
- `KeyDto` adds `lastUsedAt` (from `KeyLastUse`, data-model §2.1); `ClientSummary` adds `lastUsedAt`, the latest of its
  keys.

### 5.10 Platform menu and the Settings page (user, 2026-11-13; M-16, built in 0.6.0)

Today the side menu's Platform group has one item, **Platform**, that opens an index of seven cards (§5.8), so every
platform object is two clicks away; Organisation is both a menu item and a card; and Settings is a table of nine
sections that each open a drawer with a generated form, several members as JSON text. Once built, this section
replaces §5.2's Platform group and §5.8's index page.

**Menu.** The Platform group lists its pages directly, filtered by role as before:

| Item | Route | Holds | Roles |
|---|---|---|---|
| Apps and keys | `/plattform/clients` | §5.9, §5.9.1 | platform admin |
| Models | `/plattform/gateway-models` | gateway models with their limits, the gateway's context, embedding profiles and the canary (the page as it is) | platform admin |
| Settings | `/plattform/settings/:section?` | below | platform admin |
| Organisation | `/organisation` | the page as it is, plus the member groups, the sharing summary and the gateway link of the generic resource `organisation`, which goes | every role |
| Audit log | `/audit` | unchanged | as today |

The index `/plattform` and `PlatformIndexView` go; search (Ctrl K) lists each page and each settings section. On
phones the items are under Menu as today.

**Settings** is one page: the sections in a list on the left (a select above the form on phones), the chosen
section's form on the page, not in a drawer. The address names the section (`/plattform/settings/chat`); without one
the first section opens. Sections, grouped by what they affect:

| Group | Sections |
|---|---|
| Chat and answers | Chat (with the answer ceiling and the condense model), Prompt texts (the panel of chat design §7.3), Search (`retrieval`) |
| Content | Website fetching (`crawl`, with the host politeness table), Site profiles, Processing (`worker`), Personal data (`pii`) |
| Data protection | Data protection (the three levels), Retention, Deletion guard |
| Quality | Evaluation |
| Platform | Network, Role groups (`identity`) |

Site profiles and Data protection are lists (a row opens its form in a drawer, as today); they move here from the
index. **Role groups** (`GET/PUT /api/settings/identity`, M-17 C10) has one list field per role (one group per line):
platform admin, department admin, pipeline editor, reviewer, editor, viewer. A group must be a `rag-` name or an Entra
object id; groups are trimmed and de-duplicated case-insensitively, and a role without groups isn't stored. The hints say
that a role in the sign-in token counts too, that the deployment's break-glass group (`KALLA_OIDC_GROUPS_PLATFORM_ADMIN`)
always applies, and that a change applies at each user's next sign-in (identity §2.5, Decisions "Role groups as a setting").

A section's form is built in two steps:

1. **The shell:** the forms as they are today (generated from the OpenAPI schema, with the dedicated ones for crawl
   and prompt texts), on the page. Each save is one section, with its `version`, as today.
2. **Typed members:** a member that is JSON today gets controls of its own (number, switch, select, a small table),
   with a label, a hint and the default the code uses when it is unset, in `labels.ts`. Order: Chat, Search,
   Processing, Personal data. JSON stays only under **Advanced (JSON)** for members with no control yet. The server's
   validation is unchanged.

### 5.11 The setup wizard (user, 2026-11-15; Decisions "Setup wizard"; `work/main.md` M-20)

`/setup`, outside the signed-in shell, before step 1 of §5.4. The rules it shows are identity §19. Texts are in
`labels.ts` and `messages.ts`. While setup is closed the page says so and links to sign-in.

1. **Setup code.** One field; where to find the code (the kalla-admin container's output, with the command per deploy
   form). A wrong code and the attempt limit have their own messages.
2. **Addresses.** The redirect URIs to register at the provider (sign-in and sign-out callback, as kalla-admin builds
   them for the address the wizard is opened at), each with a copy button, and what kalla-admin's own configuration says
   of the deployment (the gateway's address, the public API address, the break-glass platform-admin groups). kalla-admin
   has no list of the other host names, so there is no per-host test here.
3. **Sign-in.** Provider address, client id, client secret, an optional expiry date of the secret; *Advanced* holds
   the token-shape values with a preset for Keycloak and for Entra ID. A value set by the deployment is locked with
   *Set by the deployment*. **Test sign-in** opens the provider in a new window; the result shows who signed in, the
   roles, groups and departments the token carried, and whether that person is a platform admin, with the claim that
   was read. A failed step says which (address, discovery, secret, a refused claim check).
   When neither the deployment nor the wizard sets the authority, the provider address field is filled in with the
   gateway's address (identity §19), with the hint *From the AI gateway, which signs in at the same provider. You can
   change it.*; the filled-in value is sent, tested and compared like a typed one. When the gateway has the feature but
   no address yet and the field is not locked, a note says the gateway's sign-in is not set up yet, with a link to
   `<gateway admin>/setup` when the admin address is known (new tab); with an older gateway the step is unchanged.
4. **Open WebUI** (only when the deployment runs it): Open WebUI's redirect URI with a copy button; provider address
   (empty follows the sign-in provider), client id, client secret, optional expiry date, provider name. **Check** asks
   the provider, without a sign-in: the discovery document found, and the client *accepted*, *refused* or *could not
   be verified* (saving is still allowed, with a warning). **Save** needs a check that wasn't refused, unchanged
   values, and step 3's passed test; the bottom button reads **Skip for now** until a save. Built in #773.
5. **Summary.** The values to save with their sources (set by the deployment, set here, default), where the secret
   comes from, the same deployment facts as step 2, and the tested person. **Finish** is enabled when step 3's test
   passed for a platform admin and no value changed since, and leads to "Sign-in is set up" with a **Sign in** button
   (no automatic navigation), then the *Get started* checklist (§5.5.1).

The sign-in page offers **Set up sign-in** while setup is open (and no *Sign in* while sign-in isn't configured), and
says to restart kalla-admin when setup has ended without a sign-in. The setup window's end shows from step 2 on. Built
in M-20 S4 (#769); the Open WebUI step follows S5.

Home shows platform admins a notice 30 days before a client secret's expiry date, with the re-open steps; each notice can be hidden per browser (#773).

## 6. Pipelines

### 6.1 Template parameters on the source

- A pipeline is a template version + a parameter set (versioned JSON); each source type has one linear pipeline
  (simplification pass §2). The source form shows the template's parameters under "More settings", generated from the
  template's `paramsSchema` (§6.3), with the template's defaults; most departments never open it.
- On a **new** source the pipeline is published with the source (no documents: plan empty, no evaluation yet). On an
  existing source a changed parameter opens the publish dialog (§6.6); the source page explains "The change splits 840
  documents into passages again (about 25 minutes, 0,00 USD)", from the effect tags of the changed parameters (P2,
  weakness review §1; in P1 publish re-processes every document of the collection and the estimate counts them all, with
  unchanged stages free from the per-stage caches).
- `rag-pipeline-editor` may change only the parameters; other source settings need `Edit`.

### 6.2 Node editor

Removed (simplification pass): pipelines are template + parameters; there are no user-added nodes, ports or locks to
edit (§16).

### 6.3 Forms generated from schemas

The generic forms (§5.8) map the OpenAPI request schemas in the UI (`platform/schema.ts`) with the table below. The
template-parameter form (**Edit processing** on the source page, §6.6) maps the template's `paramsSchema` from
`GET /api/templates/…` in the UI too (`components/sources/PipelineCard.vue`): `enum` → `SelectField`, `integer` → a
numeric `TextField` with its range as the hint, anything else a JSON text area; the server validates the values
against the same schema (`PipelineValidator`, V24). Both render with the copied `components/ui` fields. No JSON Schema
form library and no browser-side validator: the server validates.

| Schema | Control |
|---|---|
| `boolean` | `UiSwitch` |
| `enum` (≤ 5) / (> 5) | `SegmentedControl` / `SelectField`, labels from `x-labels.en` (Decisions "Admin UI in English") |
| `integer`/`number` with `minimum`/`maximum` | `TextField` numeric with unit from `x-unit` |
| `string` (+ `pattern`, `maxLength`) | `TextField` |
| `array` of strings | `AddressListField`-style list editor (gateway component) |
| `object` with known properties | `FieldGroup` |
| `array` of objects | repeating `FieldGroup` rows with add/remove/move buttons (no drag) |
| anything else | a JSON text area for that member, with a note |

A template parameter's label is the UI's own text when `labels.ts` has one (`PIPELINE.params`, §5.7), else the
template's `x-labels` (`en`, then `sv`), else the parameter name; templates set `x-labels` where the name says too
little (`pathRules` is labelled Topics / Ämnen, tested in `Kalla.Pipelines.Tests`).

### 6.4 Preview

Preview runs on the form's values saved as the pipeline's draft version: the `stage-preview` payload names a
`pipelineVersionId` (engine design §9.1), and draft pipeline versions come with pipeline publish (§6.6). The result view: documents as tabs, each with
a stage strip (state, duration, cache hit, cost) and the selected stage's output (parsed Markdown with page markers,
sections, chunks with heading path and token count, PII counts). For a crawl source the first tab is **Before / After**:
the raw page text next to the extracted main text under the site profile the worker learned for the host or would apply
(preset or existing host profile), with the profile's version, CMS preset, breadcrumb and date found; a new source on a
host without a profile shows the profile learned from the preview sample (learning P2, weakness review §1; in P1 the manual
profile or the Sitevision preset). Changing a parameter re-runs from the first
affected stage.

### 6.5 Diff

Removed (simplification pass): no version diff; a parameter diff is a P2 option if needed (§16).

### 6.6 Publish flow

```
form change ──Publish…────▶ plan (dialog) ──Confirm───▶ re-process (new chunk sets next to the live ones,
                               │                         swapped per document)
                            Cancel                               │
                                                                 ▼
                                    evaluation report (full collection, vs the previous run)
                                                                 │
                                            Keep (nothing to do)   │   Restore (one click, ≤ 14 days)
```

1. `POST /api/pipelines/{id}/publications`: V24; enqueue `publish-plan`; the publication's id is the plan job's id
   (`GET /api/publications/{id}` reads the job's result with the draft and the pipeline version it was planned
   against). Nothing is frozen; the confirm carries the pipeline's `version` and is refused (409) if it changed since
   the plan. **Copy on write:** platform pipelines (`DepartmentId` null) are published by `rag-platform-admin` only (403
   `admin_only` otherwise). A department user's first publication on a platform pipeline creates a department pipeline
   whose version 1 is the platform pipeline's published version, re-points the collection's sources of that template to
   it, and publishes the draft there, in one config transaction; later publications of that collection go to the
   department pipeline.
2. Dialog: **Plan** ("1 240 documents: 1 180 from the Passages step (no new reading, ~25 min), 60 from Read (~900 pages,
   ~1 h); estimated cost 0,00 USD"); **Affected sources** with their document counts and links (the plan's result
   names sources, not assistants; the source page's impact line names the assistants).
   The per-stage split comes from effect tags and is **P2** (weakness review §1); in P1 the plan lists every document
   of the collection as re-processed, with unchanged stages free from the per-stage caches.
3. **Confirm**: one config transaction (§4.4) publishes the version, audits `pipeline.publish`, and enqueues one
   `reprocess` sync per source.
4. Progress is shown on the publish screen; live answers keep using the old chunk sets until each document's swap.
   When re-processing completes, the worker runs the `eval` `publish-report` on the full collection; its **report** (§7.2)
   appears on the publish screen and the collection page.
5. **Keep** dismisses the card; **Restore** rolls back (§6.7). Nothing blocks either choice.
6. Templates: platform admins edit and publish template versions through the generic form; a new template version
   never re-processes sources by itself. A source moves to the latest template version when its parameters are next
   published; the plan names the template change.

### 6.7 Rollback

**Restore** on the report or on any earlier version: the confirmation names the version and the re-processing
("within 14 days: the earlier passages are taken back, no new reading"); never blocked; an evaluation run follows
as after a publish.

### 6.8 Canvas

Removed (simplification pass): the Vue Flow canvas is dropped from the roadmap; reconsidered only if editors
demonstrably need custom flows (§16).

## 7. Evaluation pages

### 7.1 Test questions and quality (per collection)

- Set: size per layer (`generated`, `human`, P2 `skill`/`feedback`), stale and retired counts, last generation (model,
  cost), **Create more** with the forecast (processing §13.1). Generated sets run at the collection's ceiling (processing
  §13.1); the `human` gold set stays the headline metric.
- Questions table (keyset): question, layer, expected documents (a count; the titles show in the editor), last rank,
  state (a `stale` human question says "Its expected document was deleted", processing §13.3); editors add/edit/retire
  human questions (expected documents picked by title search, at most 10); a generated question is changed by saving it as a new human question. `edit-eval-question` changes only the `human` layer
  (`question_not_human` otherwise), so a generated question is not retired by hand.
- Reports: latest runs (after publish, nightly, after a retrieval-settings change, on demand **Run evaluation**), each
  as the §7.2 card; the trend of recall@10 and MRR is a table of the latest 10 runs (the `TimeSeriesChart` view is
  P2); per-question drill-down (rank per run).
- **Headline metric** ([weakness review §2](../archive/passes/weakness-review.md#2-nothing-proves-the-answers-are-good-and-not-found-is-effectively-off-at-launch)): the card and the trend lead with the `human` layer (the
  pilot gold set: recall@10 on answerable questions, false not-found, unanswerable questions answered "not found");
  the generated layer follows as a regression signal only.
- **Coverage-threshold proposal** (P1, weakness review §2): the run proposes `coverageMaxDistance` and
  `coverageMinKeywordScore` as the values that best separate the `human` answerable and unanswerable questions
  (processing §13.6, knowledge design §7), shown as numbers on the card; the editor saves them in Search settings. The
  histogram stays P2 (§16). Until `coverageMinKeywordScore` is set, not-found is decided by the vector side alone.
- Retrieval settings form (Search settings tab): all values within V16 ranges saved directly; the following
  evaluation report shows the effect. A run does not record the values it replaced; the audit log keeps them
  (`retrieval-settings` before/after, §9.1). The field hints show the code defaults (`RetrievalSettings.Default`),
  since department admins can't read `Settings.Retrieval`. It includes **Age decay**
  (`recency`: on/off, half-life in days, weight, document types; weakness review §3), whose effect the gold set's
  temporal questions show.

### 7.2 Evaluation report card (publish, collection, assistant; P2 skill)

```
Evaluation · Website umea.se/bygga, publication 30 Oct 11:02 (compared with 29 Oct)
  Test questions from your organisation (human) · 140 questions
    Accuracy (recall@10)  0,81 → 0,79   Wrongly "found nothing" 6 %   Unrelated questions answered "found nothing" 84 %
  Generated questions (regression)
    Accuracy (recall@10)  0,74 → 0,68   (−6 points)    MRR 0,61 → 0,55
  86 questions · 9 worse, 3 better · the whole knowledge base
  ▸ Questions that got worse (9)   ▸ Questions that got better (3)   ▸ About the evaluation
  [Keep]  [Restore the previous version]
```

"About the evaluation" explains: the report is advisory and stops nothing; differences of a few points can be chance;
below 20 questions it says "Too few questions to compare.". The skill gate's report (tools §3.7) uses the same card
per model with tool use and restraint lines and **Serve** instead of Keep/Restore. The assistant test report
(§7.4) uses it with the counts of §7.4.

### 7.3 Overrides

Removed (simplification pass): no gate, so no overrides (§16).

### 7.4 Test questions per assistant (simplification pass §15)

**Later (P2):** per-assistant test questions (weakness review §1).

- **Set**: `AssistantTestQuestions` per assistant (V33, ordered by `Ordinal`): question, expected behaviour as free text ("Ska hänvisa till
  kontaktcenter och inte ange någon avgift"), optional expected document (picked by title in the linked collections)
  and optional expected phrase. Added on the Test questions tab, from **Save as test question** in Test, or from a feedback
  row (P2, "rewrite into test question"). The set belongs to the assistant, not to a version, so every version is
  measured against the same questions; copying an assistant copies its set.
- **When**: after every publish and restore (on the new published version), and on demand with **Run test questions** on
  the draft or the published version. One run per assistant at a time; an assistant without questions gets no run.
- **How**: `AssistantTestRunner` in `kalla-admin` (§2.2) runs each question once through `Kalla.Answering` exactly as
  the test panel does (principal `admin-console`, the target version's definition, residency at the assistant's level,
  temperature as configured, `x-fors-step: assistant-test`), then:
  - **expected document**: cited or not (deterministic, from the answer's citations);
  - **expected phrase**: present in the delivered answer or not (case-insensitive, after Unicode normalisation);
  - **expected behaviour**: the answer judge (`Settings.EvaluationJudgeModel`, V32, called at the assistant's level; the prompt family of the
    `answers` run, processing §14.8) gets the question, the expected behaviour, the delivered answer and the cited
    titles, and returns `pass`, `fail` or `unsure` with a short note.
  A question passes ("as expected") when every check it has passes. Stored (data-model §2.2): an `EvaluationRuns`
  row (`Kind = assistant-tests`, `SubjectId` = assistant, `SubjectVersion` = the version number, a draft's included,
  `ComparedRunId` = the previous published version's last run; the published version's `TestRunId` points at it) and
  one `EvaluationResults` row per question (`Verdict`, `Hit` for the expected document and phrase, the question as
  asked, the answer at the assistant's level, `JudgeNote`). The report shows question, answer and note; **Test**
  re-runs one question in the panel.
- **Report**: the §7.2 card: "8 test questions · 7 as expected (previous version 8) · 1 worse, 0 better" with the
  questions that changed and the judge's reasons; after a publish it carries **Keep** and **Restore the previous
  version** (a restore, §4.5); a worse result than the previous version also becomes a Home attention item.
  "About the test questions" says the judge can be wrong and that nothing is blocked.
- **Cost**: questions × (one answer + one judge call), shown before **Publish** and **Run test questions**; at most 30
  questions (V33), metric `rag.admin.assistant_tests{outcome}`. Old runs: the last 20 per assistant are kept (data-model
  §8).

## 8. Feedback, conversation review, access audit and cost

**P1: ratings only** ([weakness review §4](../archive/passes/weakness-review.md#4-data-protection-guards-the-secondary-copy-while-the-primary-stays-unprotected)). The pilot assistant runs history mode `none` until A1 is
answered, so our store holds no comment and no conversation content: §8.1 shows ratings, reason categories and import
health; the API takes the "has comment" filter but the page has no control for it (nor a comment column) until A1,
and comments stay in Open WebUI. §8.2–§8.5 (comment reveal, conversation
review, `sensitive` flow, access log) are built when A1 allows stored content.

### 8.1 Feedback (P1)

- Rows from `chat."Feedback"` of the caller's assistants (keyset by `CreatedAt`), joined in memory with message
  metadata (`Outcome`, `Coverage`, `Language`, the assistant version that answered from `Messages.AssistantVersionId`),
  cited document titles (`MessageSources.DocumentId` → `Documents.Title`), assistant and client names. **No content**: no question, answer, title or comment.
- **Reason category** in plaintext: `Feedback.ReasonCategory` filled by the importer and `/v1/feedback` only from the
  fixed Open WebUI keys (`RateComment.svelte:17-35`) and the API reasons of chat design §3.5 (`incorrect`, `incomplete`,
  `wrong_sources`, `not_relevant`, `other`), else `other` (the column has no CHECK, so a reason keeps its own category); the free text stays in the ciphertext (in P1 it is not stored, weakness review §4). English
  labels in `labels.ts`.
- `ReviewState` (`open`/`reviewed`/`dismissed`) set by `FeedbackComments` holders ("Mark as handled"). Filters:
  period, assistant, rating, outcome, has comment, state. Totals from `AssistantDailyTotals`.
- Import health per Open WebUI client from `FeedbackImportState` (last run, outcome, unmapped) (chat §14.2–§14.3).

### 8.2 Comment reveal (`FeedbackComments`; when A1 allows stored content)

1. "Read comment" → dialog with the fixed reason "Feedback review" (one click, chat answered 7).
2. Server: feedback in scope; level `public`/`internal` (`sensitive` → 403 `review_flow_required`); write
   `ConversationAccess` events `requested` and `decrypted` (`Scope = comment`, `FeedbackId`, `AssistantId`,
   `DepartmentId`, reason) and event **6004** — before decrypting.
3. Unwrap the row's bucket key (`DataKeyId`: level and expiry month, simplification pass §5) with the level's master
   key, decrypt `CommentCiphertext`, return `{ reason, comment }` only; keys stay ≤ 5 minutes in memory; response
   `no-store`; the UI drops it when the dialog closes. One comment per request; no bulk reveal.

### 8.3 Conversation review (`Review`)

- First (when A1 allows stored content, weakness review §4): from a feedback row, **Read the conversation** (reviewer only). Then also by assistant and period (metadata list).
- Request: free-text reason ≥ 20 characters (V26); `public`/`internal` → events `requested` and `decrypted`; access
  window **30 minutes** per request id; each fetch within the window checks scope again; one `decrypted` event per
  request id, 6004 per fetch.
- View: messages in order with times, assistant, outcome, coverage; cited snippets under each answer; feedback; P2 tool
  calls. Read-only, no export, no copy button; a banner names the reviewer and the logged reason.
- A conversation in history mode `none` shows metadata only; a deleted one "The conversation has been deleted".

### 8.4 `sensitive` (P2; no `sensitive` data while file keys are used, platform §8a)

**Later (P2):** request (reason) → a second `rag-reviewer` with the department in scope approves (`approved`, approver ≠ requester)
within 24 h → the requester decrypts within 30 minutes. Comments of `sensitive` conversations only through this flow
(chat §14.4). The pending queue is a tab of Conversation review.

### 8.5 Access log (access audit)

`audit."ConversationAccess"` as a list: time, reviewer, scope (comment/conversation), assistant, level, reason, events
(requested → approved → decrypted). Visible to platform admins (all), department admins (their departments) and
reviewers (their own). Reading the list emits 6007. Export CSV (`;`, as `ReportEndpoints.cs:152-158`).

### 8.6 Cost (cost attribution stage 1)

What assistants and knowledge bases cost, finer than the gateway shows (cost-attribution §5 C4, C5): read from
`chat."AssistantDailyUsage"` (every model call of chat, chat §8.3, and of the test panel, §4.9) and
`content."CollectionDailyUsage"` (the worker's indexing and evaluation calls, engine §10.6), data-model §2.2, §2.3.
**Reporting only**: no budget, limit or stop is set here; a per-assistant budget is its gateway key's (cost-attribution
§4.2), and the gateway is the record for charging (it also counts fallbacks and retries Källa does not see). The
gateway's cost and the estimated cost (calls without a gateway cost, priced from the model mirror) are always shown apart.

| Method and path | Policy | Returns |
|---|---|---|
| `GET /api/assistants/{id}/costs` `?from=&to=&granularity=day\|month` | Read | per period, step and client: `calls`, `inputTokens`, `cachedInputTokens`, `outputTokens`, `costUsd`, `estimatedCostUsd`, `estimatedCalls`, and the totals |
| `GET /api/collections/{id}/costs` (same query) | Read, own department only (a department the knowledge base is shared with sees no cost, §3.2) | per period and step, the same fields |
| `GET /api/costs` `?from=&to=&granularity=&departmentId=&kind=assistant\|collection` (+ `&format=csv`) | Manage: department admins their departments, platform admins all | per period, department, object (assistant or knowledge base) and step, the same fields; the CSV has one row each |

- **Periods**: Europe/Stockholm days (the tables' `Day`); `month` is the calendar month (`date_trunc` in SQL);
  `from` and `to` are inclusive dates, default the last 30 days by day; at most 400 days by day and 36 months by month
  (the tables keep 3 years, data-model §8), else 400 `period_too_long` (a `messages.ts` text, §4.2).
- **Department of a row** is the row's `DepartmentId` (the object's department when the call was made), so a moved
  object's earlier cost stays with the department that had it; the scope filter is `DepartmentId` ∈ the caller's
  departments, as the feedback totals (§8.1). Names come from the admin's snapshot (§4.8); a deleted object shows its id
  and "(deleted)".
- **Who pays which call** follows cost-attribution §4.1: an assistant's query embeddings and rerank count under the
  assistant, also on another department's shared knowledge base; a knowledge base's cost is its indexing, preview and
  evaluation. Test-panel calls are the assistant's, under the client `admin-console` (§4.9), so they show apart from
  real use.
- **CSV**: `;` as `ReportEndpoints.cs:152-158`, English headers `period;department;kind;object;step;client;calls;`
  `input_tokens;cached_input_tokens;output_tokens;cost_usd;estimated_cost_usd;estimated_calls` (client empty for
  knowledge bases). Reads and exports are not audited: they hold no content.
- **Beside the gateway** (cost-attribution §6 B7): from gateway phase 3, while the gateway's context names its Integration API (`gateway.integration_api_url`, M-17
  C11; sent to only over `https`), the
  assistant's and knowledge base's *Cost* cards show the gateway's figures for the object's key next to these:
  `GET /api/gateway/keys/{id}/usage` `?from=&to=&granularity=day|month` (Read, the key's department in scope) reads
  `/integration/v1/usage?groupBy=step&keyId=` with `kalla-admin`'s credential key (`usage.read`; gateway-changes §5.6)
  and returns per period and step `requests`, `successfulRequests`, `inputTokens`, `cachedInputTokens`,
  `outputTokens`, `costUsd`, cached 5 minutes per key and query (the gateway allows 60 requests a minute per key), at
  most 366 days (else 400 `period_too_long`); and the budget status (`GET /api/gateway/keys/{id}/budget`, §4.5). The
  gateway's figures are the key's: when several objects share a key the card says so. Källa's stored figures are
  never overwritten by the gateway's. Without the read-back each view links to the gateway's usage report
  (`admin_url`). How to read the two side by side is in `docs/runbook.md` (cost-attribution §5 C6).

**UI** (C5; every label in `labels.ts`, every sentence in `messages.ts`, Decisions "Admin UI in English"):

- **Assistant page, Statistics tab** (§5.5.4): a *Cost* card with a period picker (last 30 days, by day or month), the
  total with its gateway and estimated parts, and a table by step (*Answer*, *Condense*, *Long-question condense*,
  *Follow-up suggestions*, *Rerank*, *Query embedding*; P2 agent steps and test runs as their own rows) and by client
  (*Test panel* for `admin-console`).
- **Knowledge base Overview** (§5.5.2): a *Cost* card for the last 30 days, split into indexing (`embed`) and
  evaluation (`eval-generate`, `eval-judge`, `eval-dedup`, `query-embed`, `rerank`), with a link to the Costs page
  filtered to the knowledge base's department for those who see that page.
- **Costs** (FOLLOW-UP in the navigation, §5.2; department and platform admins): department picker (the header's
  department filter), period and granularity, one row per assistant and knowledge base with its cost, sorted by cost,
  expandable by step, department totals, and **Download CSV**. A note under the table: "Estimated: {n} calls had no
  cost from the gateway and are priced from the model list." and "The gateway's figures are the record for internal
  charging; they also count retries and fallbacks."

## 9. Audit log and security-relevant views

### 9.1 Audit log

- A list page in P1: platform admins every `AuditLog` entry (filters: period, actor, action, entity type, entity id,
  department, service incl. `system:retention` and worker entries), keyset; the detail shows the stored before/after
  snapshot JSON as text. Policy `Manage` with department scope: department admins see their departments' entries
  through `AuditLog.DepartmentId` (§13). The API takes the department filter; the page (its own **Audit log** menu
  item, §5.2) offers the others, since the API already scopes department admins. The
  field-level diff viewer and **History** tabs on entity pages are P2.
- **Readable entries** (user, 2026-11-15, UI3 A): `GET /api/audit` items add `entityName`: the object's current name
  for the types with one (collection, source, assistant, client, key, embedding profile, site profile, gateway model),
  read in one grouped query per type for the page; else the `Name` in the entry's snapshot; else null; and
  `entityExists` (whether the lookup found it), so a deleted object's name shows without a link. A site profile's
  name is its host; a settings row shows its section's label; no entry carries a publication id, so none links to a
  publication page. The page shows
  the name (the id only when there is no name), linked to the object's page where it has one (collection, source,
  assistant, run, publication; settings sections to *Settings → section*); a deleted object's name is shown
  without a link. Events show in plain words, one label per §9.2 action in `labels.ts` (`source.sync` "Fetch
  started", `collection.publish` "Knowledge base published"), the code as the label's title; an unknown action shows
  its code. **Event** and **Type of object** are pickers built from those labels; each row's object has **Only this
  object**, which sets the object filter, and the object-id field moves under *More filters*.
- Reading the audit log emits 6007 (identity §10).

### 9.2 Audit actions

`EntityType` + `Action` columns as the gateway (`AuditSnapshots.cs:11-51`); dotted names are `entity.action` shorthand.
Admin entries: `collection.create|update|delete|publish|share|retrieval_settings`,
`source.create|update|delete|pause|resume|sync|retry-failed`, `upload.batch|delete|restore`, `deletion.confirm|reject`,
`hold.release`, `pii.view|clear`, `pipeline.publish|cancel|rollback`, `template.*`,
`siteprofile.update|approve|reject|rollback`, `evaluation.generate|question|run`,
`assistant.create|update|delete|draft|discard_draft|publish|restore|copy|archive|unarchive|audience|retention|limits`,
`assistanttest.create|update|delete|run`, `assistanttemplate.*`, `departmentsettings.update` (P2), `client.*`,
`key.create|update|rotate|reveal|copy|revoke`, `gatewaymodel.update`, `settings.update`, `dataprotection.update`,
`organisation.sync`, `feedback.review_state`, plus tools design §9.2 and the conversation access events of §8 (in
`ConversationAccess`). The actor is the `oid`.

### 9.3 Security-relevant views

| View | Contents | Who |
|---|---|---|
| Keys | status, last use (`KeyLastUse`), grace, keys used after rotation | Manage (own teams), Admin |
| Sign-in failures | `audit."AuthFailures"` aggregated by credential kind and client (never the presented value) | Admin |
| Personal data | the per-source report (`GET /api/sources/{id}/pii-report`, §4.5): held and live flagged documents of a `piiMode: report` source in a `public` collection, non-zero counts per type, the fired-rule count and the fired types (no values); per document the highlight preview and clearance with reason | Read (report); Manage (preview, clearance) |
| Quarantine | quarantined files: name (own sources), type, signature name, date; no download | Read |
| Tools (P2) | drift observations, pending approvals and pre-registrations, grants, definition diff | Admin, system owners |
| Access log | §8.5 | §8.5 |

Security events themselves are not stored (ADR 015 pattern): the SIEM is where they are read.

## 10. Accessibility, performance, observability

### 10.1 Accessibility

- **Target**: WCAG 2.2 AA, the gateway's design target (`docs/accessibility-statement.md:9`), which contains the legal
  baseline (DOS-lagen 2018:1937 → EN 301 549 → WCAG 2.1 AA). An internal admin app is a digital public service under
  DOS-lagen, so it gets a tillgänglighetsredogörelse in the gateway's format.
- **Copied patterns**: skip link (`AppShell.vue:91-93`), focus to the page heading after navigation
  (`ui/router/index.ts:46-74`), first invalid field focused (`useFormErrors.ts`), `role="status"` loading and
  `aria-live="polite"` toasts, session expiry dialog with explicit extension (`AppShell.vue:249`), three themes, 24 px
  targets, charts with a table view.
- **New patterns**: job progress announced in one polite live region per page (state changes only); publish dialog
  sections are headings; evaluation reports as text first, chart second; before/after previews as two labelled regions;
  drag never required (rows move with buttons, files by a native file input); test panel announces "The answer is ready"
  once; status never by colour alone; long tables with captions and sortable headers (`aria-sort`); generic forms use
  the same field components, so they inherit labels and error focus.
- **Verification**: copied axe spec (`tests/e2e/specs/accessibility.spec.ts:6-19`) over every routed page and every
  drawer/dialog in three themes; target size (`helpers.ts:160-172`); reflow at 320 px and 200 % text; **keyboard-only
  run of the pilot journey** (§11.4); Vitest component axe checks; one NVDA + Firefox pass of the pilot journey before
  the pilot (answered 4).

### 10.2 Performance budgets

| Item | Budget | How measured |
|---|---|---|
| Initial JS (shell + Home), gzip | ≤ 200 KB | CI build size report; fails above budget |
| Each lazy route chunk, gzip | ≤ 100 KB | same |
| API p95 (server, pilot data) | detail ≤ 200 ms, lists ≤ 400 ms, writes ≤ 500 ms (incl. the version lock), job enqueue ≤ 200 ms, overview ≤ 500 ms | `http.server.request.duration` by route; integration perf test with 50 sources, 100k run items, 10k documents |
| Test panel | retrieval as knowledge §11; answer as chat §16 plus ≤ 100 ms admin overhead | `rag.admin.test_panel.duration` |
| Preview | first document's first stages ≤ 30 s for HTML (worker-bound), deadline 2 min | E2E timing, `rag.jobs.age{class=preview}` |
| Interaction | visible response ≤ 100 ms (spinner/optimistic state) | manual + Playwright |

No new runtime dependency: no Markdown renderer, no chart library (gateway SVG charts), no JSON-schema form library
(§6.3), no graph library, no i18n library, no date library (`Intl`).

### 10.3 Observability

- Copied: `AdminActivityMiddleware` (6006 throttled), 6001–6007, log scopes incl. `ConfigVersion`, `RequestId`; no
  content in telemetry (test-panel questions, comments, conversation text, URLs of `sensitive` sources never logged).
- Metrics (meter `Källa`, low cardinality): `rag.admin.validation.refusals{code}`, `rag.admin.jobs.enqueued{class,
  kind}`, `rag.admin.sync{kind,outcome}`, `rag.admin.test_panel{outcome}` + `.duration`,
  `rag.admin.conversation_access{scope,level}`, `rag.admin.uploads{outcome}` + `.bytes`, `rag.admin.publish{kind,
  outcome}`, `rag.admin.config_lock.wait` (histogram, s; one per config write attempt, the two syncs included), `rag.admin.access_explain{allowed}`, `rag.admin.assistant_publish{kind}` (`publish`/`restore`),
  `rag.admin.assistant_tests{outcome}`.
- Dashboard *Config & security* gains admin panels (refusals by code, access events, sync health, lock waits).

## 11. Tests

### 11.1 Unit (`Kalla.AdminApi.Tests`, `Kalla.Domain.Tests`, `Kalla.Pipelines.Tests`)

- `ConfigRules`: one table-driven test per active V-rule (pass, fail, boundary; the re-check triggers of §4.6).
- `PermissionMatrix` invariants: `Admin` never implies `Review`/`FeedbackComments`; every endpoint has a policy.
- The UI's schema mapper: every generic-form operation maps to fields (vitest, against a cut of the OpenAPI
  document); the template-parameter form edits the parameters and plans a publication (vitest).
- `AccessExplainer`: one case per `IAccessPolicy` step (identity §4.3) returns that step and code.
- Assistant versions: the structured diff (P2) (settings, links, line diff of instructions incl. Swedish characters and
  empty texts); `usesTools` derivation (no skills, skills without tools, a tool excluded by `ExcludedToolIds`, a served
  version adding a tool); the test verdict rule (every check passes) and phrase matching.
- Audit snapshots never contain secret columns (reflection over `*Audit` records vs a deny list: `EncryptedSecret`,
  `KeyHash`, `EncryptedKey`).
- Error-code coverage: every `ValidationCode` has an English message and is in `contracts/admin/v1/error-codes.json`.

### 11.2 API integration (WebApplicationFactory; Testcontainers: the stock pgvector image, Keycloak; FakeGateway)

- **Role matrix**: generated from `PermissionMatrix`: every endpoint × every Keycloak test user → expected
  200/403/404; ids of another department → 404; shared collections usable for links but not editable.
- **Write path**: lock + change + outbox row + version bump + audit row with that `ConfigVersion` + job row in one
  transaction (kill the connection before commit → none of them); concurrent edits → 409 `concurrent_change`; 23514 →
  400 `platform_rule`.
- **Write skew**: two concurrent saves that each pass V3 alone but break it together (raise an assistant's level vs
  lower a linked collection's) → the second sees the first's state and is refused; lock wait > 5 s → 503
  `config_busy`.
- **Jobs**: dedup under parallel "Fetch now" (partial unique index); payload ≤ 4 KB; `pg_notify` on commit only.
- **Publish**: plan, confirm with a changed pipeline → 409; rollback within 14 days enqueues reprocess; nothing blocks.
- **V8**: FakeGateway `/v1/models` at a ceiling without the model → `model_not_listed`; gateway down →
  `gateway_unavailable`.
- **Feedback and review**: comment reveal writes `ConversationAccess` before decrypting (fault injection after the
  insert → no plaintext returned, events present); department admin can reveal comments but gets 403 on conversations;
  platform admin without `rag-reviewer` gets 403 on both; `sensitive` → `review_flow_required`; access window expiry.
- **Sync**: org and model sync write nothing when unchanged; alias residency rule; failure keeps the mirror.
- **Cost** (§8.6, §4.9): a test-panel turn on the FakeGateway adds its steps to `AssistantDailyUsage` under
  `admin-console` and nothing to `AssistantDailyTotals`; the cost routes sum seeded usage rows by day and by month,
  gateway and estimated cost apart; a department admin sees only rows of their departments (another department's
  assistant → 404, `GET /api/costs` without their rows), a platform admin all; the CSV has the documented headers;
  a period over the limit → 400 `period_too_long`.
- **Assistant versions**: a draft save with a broken rule is stored with `warnings[]` and bumps no config version;
  publish refuses it; publish moves `PublishedVersionId` and the api's view `PublishedAssistantVersions` shows only that
  version; restore creates a new version and keeps the draft; `draft_base_changed` after a restore; audiences, level
  and retention unchanged by publish and restore; copy carries no audiences or keys; deleting a collection a draft
  links is refused.
- **Assistant tests** (P2): a publish starts one run (FakeGateway answers and a scripted judge); a restart mid-run restarts
  it; the report compares with the previous version's run; results are readable only within the assistant's scope.
- **Owner-team scoping (P2)**: with `EditOwnerTeamOnly` an editor outside the owner team gets 403 `owner_team_only`
  on assistant, collection, source and skill writes and still reads them; a department admin is unaffected.

### 11.3 Worker in the loop (`Kalla.IntegrationTests`)

Admin-enqueued preview (with site-profile before/after on the umea.se fixture pages), publish plan, re-process and
post-publish evaluation, deletion confirm, PII clear, site-profile rollback and upload commit consumed by a real worker;
the admin reads the results through its own endpoints.

### 11.4 Browser (Playwright TypeScript, gateway `tests/e2e` setup; nightly and release)

- **Pilot journey** (`pilot.spec.ts`, department admin of department 1000): §5.4 steps 1–13 against a static fixture site
  (nginx container with saved Swedish pages incl. the umea.se fixture set, and 3 PDFs), FakeGateway, real worker and
  parser; ends by following the assistant page's **Open in** link (the dev client's `PortalUrl`) and calling
  `/v1/models` with the Open WebUI client's key and finding the assistant. Step 7 adds its own test question (not
  generated ones), and the knowledge base's *Largest distance* is set to 1 before publishing, since FakeGateway's
  hash embeddings put every distance at about 0.7–0.9, so the spec depends on neither model output. Run twice: mouse,
  and **keyboard only**.
- Role journeys: viewer read-only, pipeline editor (template parameters, preview, publish, roll back), platform admin
  (generic forms for settings, clients, a site-profile edit and rollback; proposal approve from P2), reviewer (comment reveal,
  conversation view, access log), no-role user sees the no-access page, deletion confirmation and rejection, upload with
  a refused file type; assistant change → Test on the draft → Publish → Restore from Versions; sharing a knowledge base with the
  municipality and linking it with a topic from another department's assistant; the Map table; from P2 the test
  report and copying an assistant.
- Accessibility spec copied and extended (§10.1); CSP violations collected with `page.on('console')` and
  `securitypolicyviolation` listeners must be zero.

### 11.5 Unit UI (Vitest)

Stores, `messages.ts` mapping, form generation (template and generic), checklist computation, polling back-off,
component axe checks.

## 12. Milestones

Moved to [`../archive/design-history/admin.md`](../archive/design-history/admin.md) on 2026-11-13 (M-14): history, not binding. A citation of §12 or a section under it resolves there.

## 13. Changes to other documents

Moved to [`../archive/design-history/admin.md`](../archive/design-history/admin.md) on 2026-11-13 (M-14): history, not binding. A citation of §13 or a section under it resolves there.

## 14. Questions

Moved to [`../archive/design-history/admin.md`](../archive/design-history/admin.md) on 2026-11-13 (M-14): history, not binding. A citation of §14 or a section under it resolves there.

## 15. To verify

- **Npgsql EF Core 10.0.3**: mapping `xmin` as a `uint` row version (`IsRowVersion`) on every aggregate root, and its
  behaviour in `ExecuteUpdate` paths.
- **Version lock** (new): `SELECT … FOR UPDATE` on `config."ConfigVersion"` before `SaveChanges`, then reading the
  bumped value after the outbox trigger in the same transaction; no deadlock with the trigger's own update of that row;
  lock waits under the admin's real write rate (integration test + `rag.admin.config_lock.wait`).
- **Gateway**: whether release 2's `memberGroups`/`code` fields land, and where the gateway publishes member groups for
  integrations; that release 1's `/v1/models` test at a ceiling passes, and from the integration phase 2 also as an
  assigned key (gateway-changes design §3.6, §5.5; V8 and V39 rely on it); residency semantics of a route alias under
  the header.
- **Generic forms**: the .NET 10 OpenAPI generator emits DataAnnotations ranges, required members and enums as JSON
  Schema keywords the UI's mapper renders (checked against the live document: every create, update and action of the
  generic-form resources and all nine settings sections map); `jsonb` members round-trip.
- **Site profiles** (new, P2 with learning): the worker's preview can return before/after text for a host without a profile within the
  preview deadline (2 min).
- **Open WebUI v0.12.5**: the feedback `data.reason` values are always one of the fixed keys and never free text from
  the API; behaviour for older imported feedback.
- **Kestrel/proxy**: per-endpoint 60 MB body limit on the upload route while the rest keeps 1 MB; nginx streaming
  (`proxy_request_buffering off`) for that route.
- **DOS-lagen scope** for an intranet admin application and the required content of its tillgänglighetsredogörelse
  (municipal web/accessibility coordinator).
- **Budgets**: API p95 numbers and the overview query at 100k run items in the R1 test.
- **Assistant tests** (new, P2): that the judge's verdicts on free-text expected behaviour agree with a person's on
  the pilot assistant's first 20 questions (spot check before the report is shown to departments).
- **Open WebUI v0.12.5** (new): whether a description returned with an external connection's model (`/v1/models`)
  is shown to users at all, or must be set in Open WebUI's own model settings (chat design owns the field).
- **Team membership** (new, P2): the municipality has no team attribute, as it had no department attribute; whether
  owner-team scoping can use `teamCodes` or needs member groups on teams (identity §1.3).

## 16. Deferred or removed (simplification pass)

Moved to [`../archive/design-history/admin.md`](../archive/design-history/admin.md) on 2026-11-13 (M-14): history, not binding. A citation of §16 or a section under it resolves there.

## 17. Deferred or removed (weakness review)

Moved to [`../archive/design-history/admin.md`](../archive/design-history/admin.md) on 2026-11-13 (M-14): history, not binding. A citation of §17 or a section under it resolves there.

