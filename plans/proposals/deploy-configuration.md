# Deploy configuration: from environment variables to settings and automatic setup

**Status:** **agreed** (user, 2026-11-13: D1–D10 each as recommended, D9 nginx with the operator's certificates in
Källa's template for all five hosts). The binding rules are in Decisions ("Deploy configuration", "Residency per
level", "Role groups as a setting"). **Built** (M-17, released in `v0.6.0`; C13 came with `v0.8.0`'s
`init-deployment.sh`; the item is in `archive/work/to-v0.9.0.md`); the gateway changes are thekim1/fors
#87 (G1), #88 (G2), #89 (G4), #90 (G5). G3 is not asked (the embedding dimensions are measured). Written by the main
session, session 34.

**Goal:** a deploy, and the Portainer stack most of all, needs only what a process can't start without. Everything
else becomes a run-time setting (`Settings.<Section>.<Key>`, edited in the admin UI), is set up automatically, or
comes from the gateway.

**Rules this plan keeps:** Decisions "Settings rule" (deployment knobs `Section:Key`, run-time values in the database);
Decisions "Gateway keys" (each setting has one owner: a key's models, residencies and policies are the gateway's);
secrets and certificates are files, never environment variables (repo-and-deployment §6); Decisions "No backward
compatibility before production" (variables and options can go in one step, with no fallbacks).

## 1. What a deploy asks for today

Measured on `main` at `81bed53c` (v0.5.1):

- **Portainer** (`deploy/portainer/compose.rag.template.yaml`): **48** `KALLA_*` variables, plus 5 shared with the
  gateway (`FORS_OIDC_AUTHORITY`, `FORS_ORGANISATION_NAME`, `FORS_OTEL_ENDPOINT`, `_TRACES_SAMPLER`, `_SAMPLER_ARG`),
  9 image pins and `COMPOSE_PROFILES`. `stack.env.example` marks **11 as CHANGE**. Then the deployer makes **two round
  trips**: they issue three gateway credential keys in the gateway's admin and paste them in (docs/portainer.md step
  4–5), and create Open WebUI's client in Källa's admin and paste its key and id (step 7). The feedback import takes a
  third, by hand with `curl`.
- **Compose** (`compose.prod.yaml`, `compose.ingestion.yaml`, `compose.openwebui.yaml`): 41 `KALLA_*` variables in
  `deploy/.env`, written by `init-deployment.sh` (723 lines, 30 options), plus host paths and firewall values that
  Portainer doesn't have.
- **Secret files** come from `kalla-bootstrap` (Portainer) or `init-deployment.sh` (Compose). Most are generated
  (certificates, database passwords, peppers, the blob key, the renderer path, Open WebUI's secrets). Five follow a
  variable: the three `Gateway__Key`, `Oidc__ClientSecret` and `OpenWebUi__AdminApiKey`.

**How the code reads them** (checked in `src/`; the PR lists the file:line evidence):

- Read **once at start** (options bound into DI, an HttpClient, the OIDC handler, Kestrel): `Gateway:BaseUrl`,
  `Knowledge:PublicBaseUrl`, every `Oidc:*`, `OpenWebUi:ClientId` and `JwtSecret` (the JWT check),
  `OpenWebUi:FeedbackImportInterval`, `ClamAv:*`, `Crawl:AllowedPrivateRanges`, `Admin:OrganisationName`,
  `Render:Endpoint`, `Parser:BaseUrl`, `OTEL_*`, `Rag:Roles`, `BlobStore:Root`, `DataProtection:CertificatePath`.
- Read **per use** (`IOptionsMonitor`): `Gateway:Key` (reloads since #659), `Gateway:IntegrationApiUrl`,
  `OpenWebUi:BaseUrl` and `AdminApiKey`. `Gateway:MaxResidency` is mixed: per request in the residency handler, once in
  `ChatService`, `GatewayEndpoints` and the embedding canary.
- Read **once by kalla-migrate**: `EmbeddingProfile:*`, inserted only when no profile of that name exists; an empty model
  seeds nothing. The profile is already editable in the admin UI (Platform → Gateway models), but there is **no create
  endpoint**.
- **No database setting exists** for any of them. The settings row (`config."Settings"`, `PUT /api/settings/{section}`,
  `pg_notify('rag_config')`, each service's `ConfigVersionMonitor`) has sections for network, retention, crawl,
  retrieval, chat, PII and worker only.

**What the gateway gives** (read at `origin/main` = v0.10.1, not the checked-out tree):

- `/v1/fors/context` has `gateway.admin_url` (from the gateway's `FORS_ADMIN_PUBLIC_URL`), the credential key's
  `allowed_residencies` and `require_max_residency`, departments with `member_groups`, teams and assigned keys. It has
  **no** Integration API URL, OIDC settings, role mapping or organisation name.
- `/v1/models` gives each model's `fors.kind` (`Embedding` among them), residency and `context_window`, but **no
  embedding dimensions**.
- The Integration API (`/integration/v1/usage`, `/budgets`, scope `usage.read`) is served **only by adminapi**, which
  isn't on the `llm` network. That is why the variable exists.
- Integrations and credential keys are created only through the admin API with an **OIDC cookie and antiforgery** (a
  gateway admin): there is **no machine-to-machine path**, no bootstrap token and no seeding. The gateway's
  `bootstrap.sh` makes certificates and passwords only.
- The gateway enforces a credential key's residencies server-side (the effective set is the assigned key's ∩ the
  credential key's; `x-fors-max-residency` only narrows).
- The gateway's own group→role mapping is environment-only, like Källa's. Its Keycloak realm import takes Källa's
  clients from `rag-additions.json` in its additions directory.

## 2. Where each variable goes

Five destinations: **deploy** (stays a deployment knob), **setting** (`Settings.<Section>.<Key>`), **auto** (set up
without the deployer), **gateway** (read from the gateway), **drop** (removed: derived or a fixed default). Names are
the Portainer ones; the Compose name is the same unless noted.

### 2.1 Stays deploy-time

What a process needs before it can reach the database or sign anyone in, or what is wiring and a security boundary.

| Variable(s) | Why it stays |
|---|---|
| `KALLA_API_HOST`, `KALLA_ADMIN_HOST` | Certificate SANs, written before anything starts |
| `KALLA_{API,ADMIN,OPENWEBUI}_{PORT,BIND}` (6) | Published ports |
| `COMPOSE_PROFILES` | Which containers run (and, after step C3, the ClamAV mode) |
| `KALLA_*_IMAGE`, `OPENWEBUI_IMAGE` (9) | Pinned by the stack file; nobody sets them |
| `KALLA_GATEWAY_URL` | Internal default `https://gateway:8443`; only for a gateway elsewhere (Compose) |
| `FORS_OIDC_AUTHORITY` / `KALLA_OIDC_AUTHORITY`, `KALLA_OIDC_CLIENT_ID`, `KALLA_OIDC_CLIENT_SECRET` (secret file) | Sign-in to the admin UI: a wrong value in the UI would lock every admin out, and the OIDC handler caches its options. The secret must equal the one in the Keycloak import |
| `KALLA_OIDC_{ROLE,DEPARTMENT,TEAM,GROUP}_CLAIM`, `_EXTRA_SCOPE`, `_TENANT_ID`, `_REFUSE_GUESTS` | Token shape and tenant checks; defaults fit Keycloak, set only for Entra ID. Security boundary |
| `KALLA_OIDC_GROUPS_PLATFORM_ADMIN` | Break-glass for the first platform admin when the token carries no `rag-platform-admin` role (D2) |
| `KALLA_CRAWL_ALLOW` | The crawler's SSRF boundary into private networks; `CrawlOptions` says "nothing else can lift it" on purpose, and in Compose it mirrors the host firewall. Proposed to stay (D4) |
| `KALLA_OPENWEBUI_HOST`, `_OIDC_CLIENT_ID`, `_OIDC_CLIENT_SECRET`, `_OIDC_PROVIDER_URL`, `_OIDC_PROVIDER_NAME`, `_PASSWORD_AUTH` | Open WebUI reads them from its environment at start (`ENABLE_PERSISTENT_CONFIG=false`) |
| `FORS_OTEL_ENDPOINT`, `_TRACES_SAMPLER`, `_SAMPLER_ARG` | The OpenTelemetry SDK reads them at start. Shared with the gateway, so they cost the deployer nothing extra |

### 2.2 Becomes a setting

| Variable | Setting | Read | Notes |
|---|---|---|---|
| `KALLA_OIDC_GROUPS_{DEPARTMENT_ADMIN,PIPELINE_EDITOR,REVIEWER,VIEWER}` (and `PLATFORM_ADMIN`'s further groups) | `Settings.Identity.RoleGroups` (role → groups) | At each sign-in (`ApplyClaimMapping`) through the settings snapshot, not `IOptions` | Only a platform admin edits it; audited; the validator's rule (`rag-` names or Entra GUIDs) stays. The variable for platform admin stays as break-glass and is merged in, never replaced (D2). With the shared Keycloak the roles arrive in the `roles` claim (Decisions "Shared sign-in with the gateway"), so most deployments leave all of them empty |
| `KALLA_OPENWEBUI_FEEDBACK_IMPORT_INTERVAL` | `Settings.Chat.FeedbackImportInterval` | Per tick (the importer re-reads it) | Low value; or drop the variable and keep the default (D6) |
| `FORS_ORGANISATION_NAME` (Källa's `Admin:OrganisationName`) | `Settings.Branding.OrganisationName`, shown before sign-in by `/bff/user` | Per request (cached with the settings snapshot) | Or from the gateway, which already stores an organisation name in its `Branding` table: G4 would put it in the context, and then this becomes **gateway** (D5) |

### 2.3 Set up automatically

| Variable(s) | How | Needs |
|---|---|---|
| `KALLA_{API,ADMIN,WORKER}_GATEWAY_KEY` | The gateway provisions the integration *Källa* and its three credential keys from a declared manifest, writes the secrets to a volume the stack shares (as `fors-certs` is shared today), and `kalla-bootstrap` copies each into `<service>/Gateway__Key`. Källa's side is built: the files reload | **Gateway change G1** (the user's repo). Compose with a gateway elsewhere keeps the manual path, made shorter by G2 |
| `KALLA_OPENWEBUI_API_KEY`, `KALLA_OPENWEBUI_CLIENT_ID` | With the `openwebui` profile on, `kalla-bootstrap` generates a `kalla-sk-` key into `openwebui/rag-api-key` (Open WebUI's wrapper exports it as `OPENAI_API_KEYS`, as it does the JWT secret). `kalla-migrate`, which already holds the pepper, upserts **the** Open WebUI client (kind `open-webui`, a fixed name) with that key's hash. kalla-api finds the client by kind instead of `OpenWebUi:ClientId` | Källa only. The client is marked *managed by the deployment*: the admin UI doesn't rotate its key; a rotation is a bootstrap re-run (D3) |
| `KALLA_OPENWEBUI_ADMIN_API_KEY`, `KALLA_OPENWEBUI_BASE_URL` | The base URL is `http://kalla-openwebui:8080` whenever the profile is on (bootstrap writes `OpenWebUi__BaseUrl`). The service account's key: a one-shot `kalla-openwebui-setup` job after Open WebUI is healthy, or kalla-api's importer signing in as the service account itself. **Which one works with password sign-in off is unverified at v0.12.5**: step C6 starts with a spike against the pinned image. **Outcome (C6, #689):** the one-shot job; the importer can't sign in, since `POST /api/v1/auths/signin` answers 403 first with password sign-in off | Källa only; the spike decides the route |
| `KALLA_CLAMAV_MODE` | Follows `COMPOSE_PROFILES`: `kalla-bootstrap` writes `ClamAv__Mode` (`on` with the `clamav` profile, else `off`) into the worker's secrets directory, as it writes `Render__Endpoint` for `render`. The mismatch the checklist warns about can no longer happen | Källa only. Compose: `init-deployment.sh --clamav` already sets both |
| `KALLA_EMBEDDING_MODEL`, `_DIMENSIONS`, `_REQUEST_DIMENSIONS`, `_MAX_INPUT_TOKENS`, `_MAX_RESIDENCY` | No seed. The admin UI's first-run checklist asks a platform admin to **create the embedding profile**: a picker of the synced models whose kind is `Embedding`, the ceiling from the model's residency, the input limit from its `context_window`, and the dimensions **measured** by the existing test call (the length of the vector it returns, or the requested size when the model takes `dimensions`). Then saved, as profiles are edited today | A create endpoint and the picker (stream b API, stream a UI). The gateway has no dimensions field; G3 would add one, but measuring is enough |

### 2.4 Comes from the gateway

| Variable | Source | Notes |
|---|---|---|
| `KALLA_GATEWAY_INTEGRATION_API_URL` | **G5:** adminapi's Integration API also served on the gateway host (`https://gateway:8443/integration/v1`), so kalla-admin uses `Gateway:BaseUrl` and the variable goes. **Without G5:** default to the context's `admin_url` (the Integration API is at its root), which needs the gateway's `FORS_ADMIN_PUBLIC_URL` and works when kalla-admin can reach the public admin address; an override stays as `Settings.Gateway.IntegrationApiUrl` | D7 picks the route |
| `KALLA_GATEWAY_MAX_RESIDENCY` | The highest of the credential key's `allowed_residencies` in the context (empty list = all = `External`). The gateway already enforces it; Källa's ceiling stops being a second copy that can disagree. Every caller reads it per use from the context cache; before the first context arrives the ceiling is `OnPrem` (fail closed) | Decisions "Residency per level" changes: "`Gateway:MaxResidency` only lowers it" becomes "the credential key's residencies only lower it" (D1) |

### 2.5 Dropped

| Variable | Why |
|---|---|
| `KALLA_API_PUBLIC_URL` | Required today; becomes optional with the default `https://${KALLA_API_HOST}` (Compose's nested default, as `KALLA_OIDC_AUTHORITY` already does). Set it only for a port or path |
| `KALLA_OPENWEBUI_PASSWORD_AUTH` | Only the manual feedback-key steps need it; it goes if C6's route works without it. **Gone in C6 (#689)** |

**Net effect on the Portainer stack** (Källa's part): the CHANGE lines go from 11 plus two round trips to **6**:
`KALLA_API_HOST`, `KALLA_ADMIN_HOST`, `KALLA_OIDC_CLIENT_SECRET`, `KALLA_OPENWEBUI_HOST`,
`KALLA_OPENWEBUI_OIDC_CLIENT_SECRET`, `COMPOSE_PROFILES` (the last three only with Open WebUI), and no round trip
once G1 exists. Without G1 one round trip stays (the keys). The rest of the work moves into the admin UI's first-run
checklist: create the embedding profile, and optionally map groups to roles.

## 3. Decisions for the user

**Decided** (user, 2026-11-13): every row as its option **A**; D9 nginx with the operator's certificate files, in Källa's
template for all five host names.

| # | Question | Options (recommended first) |
|---|---|---|
| D1 | The residency ceiling's owner | **A:** from the credential key's residencies in the context (one owner, the gateway; the variable goes). B: a setting `Settings.Gateway.MaxResidency`. C: keep the variable |
| D2 | Group → role mapping | **A:** a setting edited by platform admins, with `KALLA_OIDC_GROUPS_PLATFORM_ADMIN` kept as break-glass (merged, never replaced) and the checklist no longer demanding it when the token carries roles. B: keep all five variables. C: a setting with no break-glass (the first admin must come from the token's `roles`) |
| D3 | Open WebUI's client in Källa | **A:** created by kalla-migrate from a bootstrap-generated key; the client is deployment-managed and the UI won't rotate it. B: keep it manual |
| D4 | The crawler's private ranges | **A:** stay a deployment knob (security boundary, mirrors the firewall). B: a setting a platform admin can widen |
| D5 | The organisation's name | **A:** from the gateway's branding through the context (G4). B: a Källa setting. C: keep the variable |
| D6 | The feedback import's interval | **A:** drop the variable; keep the 15-minute default. B: a setting |
| D7 | The Integration API's address | **A:** the gateway serves it on the gateway host (G5); the variable goes. B: the context's `admin_url`, with a setting as override |
| D8 | Automatic gateway credential keys | **A:** ask for G1 (manifest provisioning in a shared stack) and G2 (a one-step "add Källa" in the gateway's admin for other deployments). B: G2 only. C: neither; keep the manual steps |
| D9 | The TLS proxy profile (C8) | **A:** nginx, matching the gateway's and Källa's `nginx.conf.example`, with the operator's certificate files in a volume. B: Caddy with automatic ACME (needs public reachability, which internal municipal hosts may not have). Either way: where the proxy lives when the gateway is in the same stack (Källa's template fronting the gateway's hosts too, or each repository its own) |
| D10 | Plan form | **A:** this proposal stays the reference; the agreed steps become queue items (stream a and b) with the plan linked. B: queue items only |

## 4. Gateway changes (the user's repository)

| # | Change | For |
|---|---|---|
| G1 (llmgateway#87) | **Integration manifest:** the gateway reads a declared integration (name, owner team code, credential keys with scopes, residencies, address list) at migration or start, creates it if absent (and the owner team if the manifest names one that doesn't exist), and writes each new key's secret to a file in a volume the stack shares; it never rewrites an existing key. A machine path that needs no admin session | C7, D8 |
| G2 (llmgateway#88) | A one-step "Add integration *Källa*" in the gateway's admin: the integration with its three credential keys, scopes and residencies preset, the three secrets shown once together | D8, Compose deployments |
| G3 (not asked) | Optional: an embedding model's dimensions in `/v1/models` (`kalla.dimensions`) | C5 (measuring is enough without it) |
| G4 (llmgateway#89) | The organisation name from the gateway's branding in `/v1/fors/context` (`gateway.organisation_name`) | D5 |
| G5 (llmgateway#90) | The Integration API also on the gateway host (`https://gateway:8443/integration/v1`), same auth and rate limit | D7 |

The contract is the gateway's `docs/openapi/integrations.json` and changes by addition only (Decisions "Gateway keys"),
so G3–G5 are additions; G1 and G2 are admin-side.

## 5. Steps

Small steps in order; each is one PR or less. Streams by `.github/stream-ownership.txt`: `deploy/**`, `docs/**`,
`.github/**` and `src/admin-ui/src/platform/**` are stream a; the rest of `src/**`, `labels.ts`/`messages.ts` and
`deploy/keycloak/**` are stream b. A step that crosses both goes on a `main-session/` branch or is split.

| # | What moves | Paths | Done when | Stream | Waits on |
|---|---|---|---|---|---|
| C0 | **Boot-test the Portainer stack in the release** (deploy-ease 5): `release.yml` builds the stack from the release's images, boots it with `docker compose` on the runner (profiles `clamav,openwebui`; the gateway from its pinned release), and checks every service healthy, `kalla-bootstrap`'s checklist, kalla-api's `/v1/models` through Open WebUI's key once C4 exists; a tag release attaches `compose.portainer.yaml` and opens the PR that pins `deploy/portainer/` to it | `.github/workflows/release.yml`, `deploy/tests/portainer-boot.sh`, `deploy/portainer/build-stack.sh` | A release dry run boots the stack green; a deliberately broken template fails it | a | – (first, so every later step is tested by it) |
| C1 | `KALLA_API_PUBLIC_URL` optional (default from the host); the checklist no longer demands `KALLA_OIDC_GROUPS_PLATFORM_ADMIN` (it says how to check the token's roles instead) | `deploy/portainer/`, `deploy/compose.prod.yaml`, `docs/portainer.md` | Stack boots with neither set; citation links are `https://<api host>/…` | a | – |
| C2 | Open WebUI's feedback base URL automatic with the profile; the interval variable dropped (D6) | `deploy/portainer/bootstrap.sh`, template, `compose.openwebui.yaml`, `init-deployment.sh` | Both variables gone from `stack.env.example`; the import runs with the profile on | a | D6 |
| C3 | ClamAV mode follows the profile (bootstrap writes `ClamAv__Mode`) | `deploy/portainer/` | `KALLA_CLAMAV_MODE` gone from the stack; with and without `clamav` the worker reports the right mode | a | – |
| C4 | Open WebUI's client automatic (D3): key generated, client upserted by kalla-migrate, kalla-api finds it by kind, the UI shows it as deployment-managed | `src/Kalla.MigrationService/`, `src/Kalla.Api/` (b); bootstrap, template, `init-deployment.sh` (a); the client page's text (b) | A fresh stack with `openwebui` answers a chat through Open WebUI with no client created by hand; `KALLA_OPENWEBUI_API_KEY`/`_CLIENT_ID` and `--openwebui-api-key-file`/`--openwebui-rag-client-id` gone | b, then a | D3 |
| C5 | The embedding profile created in the admin UI: create endpoint with measured dimensions, the picker, a first-run checklist item; the seed and its five variables removed | `src/Kalla.AdminApi/` (b), `src/admin-ui/src/platform/` (a), `src/Kalla.MigrationService/` (b), deploy files (a) | On a fresh stack a platform admin creates the profile from the gateway's embedding model and the canary passes; no `EMBEDDING` variable left | b, then a | – |
| C6 | The feedback service account's key automatic: spike at Open WebUI v0.12.5 first (which route works with password sign-in off), then the route it picks | spike in the scratchpad; then deploy files (a) or `src/Kalla.Api/` (b) | Feedback from Open WebUI reaches Källa on a fresh stack with nothing done by hand; `KALLA_OPENWEBUI_ADMIN_API_KEY` and `_PASSWORD_AUTH` gone | a or b | spike |
| C7 | Gateway credential keys automatic (G1). **As the gateway accepted it (#580, 2026-11-14):** the gateway's MigrationService provisions the integration from a manifest, create-only (an existing key is never changed or re-issued; differences are logged), and writes the keys to a dedicated volume, not `fors-certs`; kalla-bootstrap copies them from there. Rotation stays in the gateway's admin UI. The variables stay only for a gateway elsewhere. The one-step add (G2, llmgateway#88) is generic with a preset: Källa owns `deploy/integration-presets/kalla.json` in the gateway repo and sends changes there | `deploy/portainer/` | A fresh stack reaches `act-as-key` with no key pasted | a | G1 released and pinned |
| C8 | Optional TLS proxy profile (deploy-ease 4, D9): `proxy` in `COMPOSE_PROFILES` fronts the five host names with the operator's certificates | `deploy/portainer/`, `deploy/proxy/`, `docs/portainer.md` | C0's boot test also runs with `proxy` and reaches each host by name | a | D9 |
| C9 | Residency ceiling from the gateway (D1): every reader takes it per use from the context; the variable and `Gateway:MaxResidency` removed; Decisions and identity §4.7 updated by main first | `src/Kalla.Infrastructure/Gateway/`, readers in Api/AdminApi (b); deploy files (a) | With a credential key limited to `OnPrem`+`Eu` no call sends `External`; before the first context every call is `OnPrem` | b | D1 |
| C10 | Role groups as a setting (D2): `Settings.Identity.RoleGroups` with its page; sign-in reads it per sign-in; four variables removed | `src/Kalla.AdminApi/`, settings model (b); the settings page (a) | A platform admin maps a group and a new sign-in gets the role without a restart; the break-glass still works | b, then a | D2 |
| C11 | Integration API address from the gateway (D7). **G5 was declined (#580):** the API stays on the gateway's adminapi (its ADR 021); instead the context gets `gateway.integration_api_url`, so Källa reads the address from its context and `KALLA_GATEWAY_INTEGRATION_API_URL` goes; in a combined stack adminapi joins a network shared with Källa's services | `src/Kalla.AdminApi/IntegrationApiClient.cs` (b); deploy files (a) | The read-back works on a fresh stack without the variable | b | G5 or D7 B |
| C12 | Organisation name (D5). **G4 accepted (#580):** read `gateway.organisation_name` (null when empty) from the context, behind the feature `organisation-name` | `src/Kalla.AdminApi/` (b) | The sign-in page names the gateway's organisation; the variable gone | b | G4 or D5 B |
| C13 | `init-deployment.sh` loses the options whose values moved, and docs follow (`production-setup.md`, `portainer.md`, upgrade guide) | `deploy/init-deployment.sh`, `docs/` | `init-deployment.sh --help` lists only deploy-time options; `deploy-checks` green | a | the steps above |

C0–C3 need no decision and no gateway change; they can start once the plan is agreed. C4–C6 are Källa-only.
C7, C11 and C12 wait on the gateway. Main updates the designs (identity §4.7 and §5.2, repo-and-deployment, admin §2.3)
and Decisions before the step that changes a rule (C4, C9, C10).

## 6. What it makes simpler for the person deploying

- **One pass instead of three.** Today: fill 11 values, deploy, issue three keys in the gateway, paste them, deploy,
  create a client in Källa, paste its key and id, deploy, and for feedback mint a key with `curl`. After: fill up to
  6 values (host names and the two OIDC client secrets that the Keycloak import also holds), deploy, and finish in the
  admin UI's checklist.
- **No values that must agree by hand:** the ClamAV mode and profile, the residency ceiling and the gateway key's
  residencies, the Open WebUI key on both sides, the feedback base URL.
- **No gateway model id or embedding dimensions to look up:** the admin picks the model the gateway lists, and the test
  call measures the size.
- **A stack every release has booted** (C0), with the proxy as an option instead of a separate set-up (C8).
- **What stays:** host names, ports, the OIDC client secrets, the crawler's private ranges, profiles. Those are the
  things a process can't start without, or that a compromised admin account must not be able to change.
