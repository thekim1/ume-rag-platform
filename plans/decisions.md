# Decisions

The decisions in force for Källa: one entry per topic, in the present tense. Every entry binds and wins over the
design documents; where a design says otherwise, the design is a bug to report. Cite an entry by its name
(`Decisions "Cost attribution"`); an older name is on the entry's *Also* line. *From* names the rows of the dated
history, [`archive/decisions-log.md`](archive/decisions-log.md), that the entry was written from (on 2026-11-08, from
the table that stood in `01-general-idea.md` §10). Paths are relative to `plans/`, except root files (`AGENTS.md`,
`NOTICE`, the root `README.md`) and `docs/`, `deploy/` and `.github/`.

**Recording a decision:** change the entry in place, or add one under its area, with a Detail link and who decided it
when; then add a line under *Later decisions* at the end of the log (date, who, entry, what changed). Decisions put
off on purpose are in [`postponed-decisions.md`](postponed-decisions.md).

## Phases and planning

### Product names

The RAG platform is **Källa** (written **Kalla** where only ASCII fits) and the LLM gateway is **Fors**; the working
names Ume.Rag and Ume.LlmGateway are gone. Code, images, documents and deployment names follow, with an upgrade path for
existing deployments ("One deployment bootstrap"): `Kalla.*` projects, `KALLA_*` variables (the gateway's `FORS_*`),
`kalla-*` services, volumes and networks (copied once from the `rag-*` volumes), `kalla_*` database roles,
`ghcr.io/thekim1/kalla/*` images and the repository thekim1/kalla. Generic `rag` names that describe a function stay:
`rag.*` metrics, the `rag-*` roles and role groups, and the credential keys `rag-api`, `rag-admin` and `rag-worker`.
The names on the wire change too, in both products in one step (no backward compatibility before production): Källa's
own Chat API members `ume.*` became `kalla.*` (`kalla.topics`, `kalla.escalation`, `kalla.follow_ups`, …), and the
gateway's `x-ume-*` headers, `/v1/ume/context` and the `ume.*` model facts became `x-fors-*`, `/v1/fors/context` and
`fors.*`; Källa needs Fors 0.12 or later. Keys issued before the rename (`ume-rag-sk-`, `ume-rag-pk-`) stay valid; new
ones are `kalla-sk-` and `kalla-pk-`. The names that are part of stored ciphertext keep the old name, as format
constants nobody sees (`StoredFormatNames`: the Data Protection application name `Ume.Rag`, its purposes `Ume.Rag.*`
and the blob key wrap's `ume-rag-wrap:`), so what an installation has encrypted stays readable (thekim1/fors#116).
The same holds for a stored value under a constraint of an applied migration, and for a name that data is bound to and
that cannot be renamed: the setup test's target `rag-admin` (`CK_SetupTests_Target`) and the Kubernetes blob claim
`rag-blobs`. An applied migration is never edited for a name.
The public host-name defaults (`rag.`, `rag-admin.`, `chat.<domain>`) and the host units (`rag-*.service`,
`rag-ingestion.slice`) stay too. The rename comes before `v0.9.0`. This answers N1.

*From:* decided by user, 2026-11-20 (#894, #899: deployment names, the key prefixes and how far `kalla-*` goes).

### First version

The pilot covers URL lists, a website crawl and uploads, the Chat and Knowledge APIs, and Open WebUI for one pilot
department. Detail: `roadmap.md` "Done", "Now".

*From:* log row 5; decided by user, 2026-10-29.

### Test chatbot before the umea.se pilot

Before the umea.se pilot, a **test chatbot** runs on the latest release and the gateway: DoIT's support documents
uploaded into one knowledge base, one assistant, used by a small test group in Open WebUI. The user deploys the gateway
and Källa by hand. The umea.se pilot (full crawl, gold set, go-live gate) follows it. The case plan is
`cases/support-assistant/` (non-binding); tickets in it are anonymised or invented until G1 allows real ones. Detail:
`roadmap.md` "Now".

*From:* decided by user, 2026-11-13.

### Code first, live testing last

P1 code is built against FakeGateway, fixtures and the dev Keycloak first; the live block (real gateway and EU routes,
umea.se crawl, gold set, shared Keycloak, first pilot answer, test-host deploy and restore drill) runs at the end in
one go. Detail: `roadmap.md` "Now", track 2.

*From:* log row 85; decided by user, 2026-10-31.

### P2 starts before go-live

P2 code starts once `v0.9.0` is released, without waiting for P1b's go-live. The live block and go-live (the
roadmap's critical path) go on in parallel as their inputs arrive, and their live checks may be postponed. The
phase-start rule binds go-live, not the start of P2 code. P2 runs in waves: wave 1 is **assistants and answer
quality** (assistant templates, copy, version diff, per-assistant test questions, the nightly answer run, Home's
attention items); the themes that wait on people (gateway release 2, Entra ID, SharePoint access, stored chat content
under A1, ask-a-file workspaces, multi-host) are listed with their inputs. Detail: `roadmap.md` "P2 Broaden".

*From:* decided by user, 2026-11-20 (the first wave picked from four themes).

### Two streams

The code lives in this repository. Stream A owns the platform and worker side, stream B the serving side and all admin
UI and admin API work; they meet on written contracts in `contracts/`. Items that need other people are the user's.
Detail: `AGENTS.md` "Working in a stream"; `.github/stream-ownership.txt`.

*From:* log rows 61, 76; decided by user, 2026-10-30.

### Phase-start rule

Every item that blocks the next phase has a named person and a date before that phase starts; for P2 it binds
go-live, not the start of P2 code ("P2 starts before go-live"). Detail: `postponed-decisions.md` (header).

*From:* log row 66; decided by user, 2026-10-30.

### M2 and test hosting

Until on-prem model hardware exists (M2), every model call is EU-hosted through the gateway; moving to on-prem BGE-M3
re-embeds every chunk, and the on-prem capacity run waits for the hardware. Tests use the cheapest host; the test-host
deploy, restore drill, R1 and the cgroup v2 check (to-verify #90) run before go-live. Detail: `platform/repo-and-deployment.md` §11 (#14, #16, #22);
`postponed-decisions.md` M2.

*From:* log rows 66, 77; decided by user, 2026-10-30.

### P1a plan answers

Accepted risk: with no on-prem hardware, a DPIA refusal of EU processing leaves the pilot waiting for hardware or
another provider. The embedding ceiling is `EmbeddingProfiles.MaxResidency`. Detail: `identity/design.md` §4.1;
`platform/data-model.md` §2.1.

*From:* log row 78; decided by user, 2026-10-30.

### Pilot set-up inputs, P1a item 20

The pilot collection is `Turkos` (`public`), owned by DoIT, with `piiMode: report`. Scope patterns match the URL
before the site profile's identity rule. Detail: `ingestion-worker/design-connectors.md` §5.2.

*From:* log row 93; decided by user, 2026-11-01.

### P1b briefs

E1 moves to P2 with Entra and the W3 spike runs in P1b (O3); `evaluation.yml` uses the deployment's own profile (O4);
the `lastmod` sample starts once U1 is done (O5); H8 is no P1b gate (O6); the preview kind is `stage-preview` (O9);
the go-live host (O7) is open. Until W3 and E1
are solved, each department gets an Open WebUI of its own. Detail: `identity/design.md` §16; `roadmap.md` "Waiting on people".

*From:* log rows 66, 86; decided by main session, 2026-10-31.

### M-3 settlements

The test panel is as built; its extras (collection subset, condense detail, token budget, filtered texts, `ttftMs`,
`costUsd`), the Overview attention items and editors' site-profile read access are P2. The interaction-audit
benchmark is dropped. Retrieval latency over budget at 100,000 chunks in one scope is accepted for the pilot. Detail:
`admin/design.md` §4.9; `chat-api/design.md` §16.2.

*From:* log row 119; decided by user, 2026-11-07.

### P0 as built

P0's deviations are written into the designs as built: blob keys use only `a–z0–9-_.`, and blob tombstones are
database rows. Open for P2 (H7): `Group.Create` or `Group.ReadWrite.All` for the Entra deployer. Detail:
`platform/data-model.md` §9; `identity/design.md` §12.4.

*From:* log rows 79, 80; decided by main session, 2026-10-31.
*Also:* "P0 integration, stream A", "P0 integration, stream B"

## Development rules

### Design documents: one layer

These decisions and the area's design document bind; archived passes do not, except the performance pass's
capacity model, its §3 tables, scaling triggers and scale tests. Design text keeps no PR history; section numbers and linked headings never change, and a
file moves only by a scripted rewrite of its links, checked by `scripts/check-doc-refs.py`. Open checks:
`to-verify.md`. Detail: `AGENTS.md`; `README.md`; `archive/doc-restructure.md`.

*From:* log rows 52, 97; decided by user, 2026-11-01; file moves, 2026-11-08.

### Settings rule

Deployment knobs are `Section:Key`; run-time values are `Settings.<Section>.<Key>` (database, admin UI). A secret a
person types in the setup wizard may be a setting, encrypted with Data Protection (Decisions "Setup wizard"). Detail:
`AGENTS.md` "Conventions".

*From:* log row 52; decided by user, 2026-10-30.

### No backward compatibility before production

Until the user declares production, schemas, contracts, payloads, APIs and config keys change in one step: no
expand/contract, fallbacks, tolerant readers or shims; development data is rebuilt. The gateway's real API still binds,
and each PR lists the changes and hand-offs. Detail: `platform/data-model.md` §10; `AGENTS.md`.

*From:* log row 96; decided by user, 2026-11-01.

### 7-day rule in development

Until the first production deployment newer releases may be pinned, still as exact versions and digests; before
go-live every pin younger than 7 days is reviewed. Detail: `platform/repo-and-deployment.md` §2.3.

*From:* log rows 70, 81; decided by user, 2026-10-31.

### Admin UI in English

Admin UI text is English (`lang="en"`), every string in `utils/labels.ts` and `utils/messages.ts`, so Swedish is a
later translation of those files; no i18n library. Swedish names are marked `lang="sv"`. Dates and numbers follow the
gateway's `format.ts`; `x-labels` show their `en` text. Open WebUI is unaffected.
Detail: `admin/design.md` §1 (row 3).

*From:* log row 99; decided by user, 2026-11-02.

### Licence

Källa is Apache-2.0, copyright Kim Vallén and Nichlas Löfdahl, as Fors is; releases up to and including 0.8.0 (as
Ume.Rag) were published under EUPL-1.2. The third-party notices are in `NOTICE`. Detail: the root `README.md` "Licence";
`NOTICE`.

*From:* log row 82; decided by user, 2026-10-31; changed to Apache-2.0 by user, 2026-11-20 (#899).

### Low CPU during development

Base Compose files cap every container at 2 CPUs or less; `compose.host-24g.yaml` carries the pilot caps, which R1
re-sets before go-live. Detail: `platform/repo-and-deployment.md` §6.2.

*From:* log row 84; decided by user, 2026-10-31.

## Database and hosting

### Postgres hosting

Källa has its own Postgres, never the gateway's: the digest-pinned stock pgvector image plus pgBackRest (WAL archiving,
point-in-time restore), or managed Postgres with pgvector ≥ 0.8, which is not planned for now. CI's `backend` job builds the image and runs
integration tests on it. Blobs are write-once and long-lived keys escrowed. A standby (D1) is required before
production beyond the pilot. Detail: `platform/repo-and-deployment.md` §5.2, §7.4, §8.2.

*From:* log rows 7, 13, 55, 58, 68; decided by user, 2026-10-30; managed Postgres not planned, 2026-11-08.

### Data model

One `ragdb` with schemas `config`/`content`/`chat`/`audit` and one role per service. Swedish compounds are split in
.NET with the `sv_SE` word list. Config changes go through a versioned outbox, serialised, failing closed after a 90 s
grace window. The HNSW rebuild runs on `ConnectionStrings:ragdb-maintenance`. No data tiers: `sensitive` isolation would be a separate deployment. Embeddings are stored as `vector` with a `halfvec` HNSW index; `halfvec` storage follows in P2 (after to-verify #117). Search is Postgres only;
OpenSearch only when a measured trigger (O1) fires. Detail: `platform/data-model.md` §1, §3.1, §5, §6;
`knowledge-api/design.md` §13.

*From:* log rows 6, 24, 30, 55, 73; decided by user, 2026-10-30; `halfvec` storage in P2, 2026-11-08.

### Scale-out

Target ≈ 10× the pilot on 3–5 hosts, beyond by measured triggers: Compose per host with roles, one shared Postgres,
Redis and S3-compatible object store, the queue behind `IJobQueue`, no PgBouncer up to five hosts. The queue stays in Postgres (a broker only
on a trigger), blobs are files on a single host and an S3-compatible store beyond it (the municipality's, or SeaweedFS;
never MinIO), Kubernetes is optional later, and the BGE-M3 query lane comes on a trigger. The worker's gateway key
carries a `MaxConcurrentRequests` cap. In P2 Open WebUI
gets its own VM. Detail: `platform/repo-and-deployment.md` §6.4; performance pass §2, §6, §7.

*From:* log rows 52, 53, 54; decided by user, 2026-10-30.

## Containers and images

### Pilot containers

Seven long-running containers (`kalla-api`, `kalla-admin`, `kalla-worker`, `kalla-parser`, `kalla-clamav`, `kalla-renderer`,
`kalla-postgres`) and the one-shots `kalla-bootstrap` and `kalla-migrate`; no Redis until the first multi-host deployment, and then
Källa's own, never the gateway's. Admin sessions are in Postgres; rate limits and the query-embedding cache stay in
process behind `IRateLimitStore`/`ISharedCache`; pgBackRest runs inside the Postgres image, not as a sidecar. Merging
containers (api with admin, ClamAV with the parser, the worker with the api) was considered and rejected. `kalla-clamav`
runs only where the malware scan is on (Decisions "Malware scan optional"). Detail:
`docs/production-setup.md`; `platform/repo-and-deployment.md` §6.2.

*From:* log rows 13, 58, 115; decided by user, 2026-11-07.

### Repo and deployment

`Kalla.Api` is one binary with roles; tests use `Kalla.FakeGateway`. Production Compose has resource zones and
fixed networks with firewall egress rules; releases ship a digest-pinned bundle. Detail: `platform/repo-and-deployment.md` §3,
§4.3, §6, §7.

*From:* log rows 17, 31; decided by user, 2026-10-30.

### Deploy configuration

A deployment sets only what a process can't start without: host names (on Portainer and Compose they default under
one `FORS_DOMAIN`), ports and binds, the crawler's private ranges (`Crawl:AllowedPrivateRanges` stays a deployment knob:
a security boundary), profiles, image pins and telemetry. The sign-in provider's address, the client ids and the
client secrets, kalla-admin's and Open WebUI's, are set in the setup wizard unless the deployment sets them (Decisions
"Setup wizard"). The rest is set up automatically or comes from the gateway: the gateway
credential keys from a gateway integration manifest in a shared stack (gateway G1, llmgateway#87; elsewhere a one-step
integration in the gateway's admin, G2, #88); Open WebUI's client created by the deployment (key generated by
`kalla-bootstrap`, client registered by `kalla-migrate`, found by its kind; shown as managed by the deployment); the admin test panel's `admin-console` client seeded by `kalla-migrate` the same way (#712); the
feedback import's address with the `openwebui` profile and a fixed 15-minute interval; the ClamAV mode from the
`clamav` profile; the embedding profile created in the admin UI from the gateway's embedding models, its dimensions
measured; the organisation's name and the Integration API from the gateway (G4 #89, G5 #90). The Portainer stack has
an optional `proxy` profile: nginx with the operator's certificates, in Källa's template, for all five host names.
Detail and order: `proposals/deploy-configuration.md`; built as M-17 (`archive/work/to-v0.9.0.md`).

*From:* decided by user, 2026-11-13 (session 34, D1–D10); sign-in values to the setup wizard and `FORS_DOMAIN`, decided
by user, 2026-11-15.

### Setup wizard

Sign-in is set up in the browser. While setup is open, kalla-admin prints a one-time setup code on its standard output
(derived from a generated setup secret file and a nonce in the database, the same on every replica, valid one hour,
never sent through the logger); the admin UI's `/setup` asks for it. Setup is open only before its first completion
and after a re-open; a fault never opens it, and a deployment that declares an authority and a client secret counts as
completed. Saving sign-in values needs a passed test sign-in with those values by a person who comes out as a platform
admin. Afterwards kalla-admin's and Open WebUI's sign-in provider, client and secret change only in setup mode, never
from an admin session; the deployer re-opens it by giving `Setup:Reopen` a new value, which is consumed when setup
opens. A value the deployment sets wins and is shown locked (an empty value is unset). Wizard values are settings; the
typed client secrets are stored encrypted with Data Protection and are write-only. kalla-admin rebuilds its sign-in
options when the settings version differs at a sign-in; Open WebUI's start wrapper fetches its sign-in settings from
kalla-api and exits when they change. It covers Portainer, Compose and Kubernetes; the gateway's part follows in its own
repository. The main session builds it, stream b's paths included (user, 2026-11-15). Detail:
`identity/design.md` §19; `admin/design.md` §5.11; `proposals/setup-wizard.md`; work: `work/main.md` M-20.

In the Portainer stack (stage 2, one typed value, `FORS_DOMAIN`):

- **The integration manifest is the stack file's default.** `deploy/portainer/build-stack.sh` rewrites one line of the
  gateway's part, its bootstrap's `FORS_INTEGRATION_MANIFEST`, to carry `deploy/portainer/integration-manifest.json` (no
  owner) as the default for an unset variable; a set value wins and an empty one provisions nothing. The rest of the
  gateway's part is copied unchanged, and the build stops when that line is not found.
- **kalla-admin's address of the gateway's admin** defaults to `https://<the gateway admin's host name>` (port 443, as
  behind the TLS proxy) and is empty when the admin has no host name of its own; `FORS_ADMIN_PUBLIC_URL` overrides it.
- **The release's boot test sets no sign-in value,** so Open WebUI starts without sign-in there. Open WebUI with OIDC
  sign-in is shown by the browser tests (`e2e.yml`) before a release.

*From:* decided by user, 2026-11-15 (session 40, W1–W10; agreed after an independent review); the stage 2 points
2026-11-16 (session 44).

### Upgrades in one step

An existing deployment moves to a new release in one step, from any release still supported, and stops nothing by
hand on one slot.

- **The migration step stops what it must itself.** A migration that the previous version of a service can't run
  beside declares the service roles it stops; `kalla-migrate` holds those roles out of the schemas until a version that
  contains the migration starts (data-model §10, *Service fences*). The upgrade guide still marks the release
  *Rolling update: no*: it means a short outage of those services, not a step for the operator.
- **Two slots and Kubernetes** stop or scale down the old instances of such a release first (the grant is per role, so
  an old instance beside an admitted new one would get access again).
- **A release may be skipped.** The oldest release an upgrade is tested from is `UPGRADE_FROM` in `release.yml`
  (0.5.1 today); the release boots that release's Portainer stack and applies the new file over it. The guides have
  one upgrade section, not one per release; what a deployment from an older release must do by hand is a numbered
  step there, and steps for releases before `UPGRADE_FROM` are dropped.
- **The joined Portainer stack sets what it knows:** the Integration API's address is the stack's own `adminapi` by
  default (`build-stack.sh` rewrites the gateway's line as it does the manifest's), so `FORS_INTEGRATION_API_URL` is
  typed by nobody.

*From:* main session, 2026-11-17 (session 47; the user asked for M-22's remaining points with the recommended choices),
`proposals/deploy-simplification.md` S4b and S4c.

### One deployment bootstrap

Every deployment form gets its certificates, secrets and derived files from one script, `deploy/portainer/bootstrap.sh`
(the `kalla-bootstrap` service), and its services from one definition.

- **Files live in Docker volumes in Compose too:** `kalla-certs`, `kalla-secrets` and `kalla-config`, as in the Portainer
  stack. Host scripts (key escrow, backups, certificate checks) read them from the volumes; what an operator supplies
  as a file (the gateway's CA, PKI certificates, the TLS proxy's certificate) is a host path mounted read-only.
- **`deploy/init-deployment.sh` writes only `deploy/.env`:** the domain, the gateway's address and where its keys are,
  the backup target and the optional parts. It generates no certificate or secret, and its `--help` is the option
  list.
- **One Källa service file with the `rag-` names.** Compose runs it directly; `build-stack.sh` merges it with the
  gateway's file for Portainer. What only a Compose host has (pgBackRest, the fixed subnets of the host firewall, the
  ingestion slice, rolling slots, the 24 GB sizes) is in overlay files. The Portainer stack's service, volume and
  variable names stay.
- **Kubernetes:** `create-secrets.sh` runs the same `bootstrap.sh` into a local directory and uploads the result.
- **The setup wizard is the same in every form** (Decisions "Setup wizard"): the bootstrap generates the setup
  secret, ends its log with the wizard's address, and takes the declared sign-in values from the same variables;
  `init-deployment.sh` asks for no sign-in value, except the provider's address where the host firewall overlay is on
  (kalla-admin reaches only the provider named there).
- **Published ports bind `0.0.0.0` in every form** (kalla-api, kalla-admin and `kalla-proxy`; Open WebUI `127.0.0.1`, for a
  proxy on the host); the `*_BIND` variables change one.
- **Host hardening is an optional step on a Compose host** (`compose.hardened.yaml` with `deploy/host/install.sh`: the
  egress firewall, the ingestion slice, the renderer's seccomp profile). A first install runs without it, with the
  Portainer stack's posture; a production Compose host adds it once the installation works, and the guides say so.
- **Gone without a replacement:** forcing a certificate re-issue before its last 30 days, Open WebUI on a VM of its
  own, and a stack without the ingestion services.
- **Upgrades:** a deploy change may break an existing deployment (Decisions "No backward compatibility before
  production"), but it comes with an upgrade path wherever one is feasible: a step in the upgrade guide, or a
  migration in the bootstrap. *Install again* is the answer only when no reasonable path exists, and the release says
  so.

Detail: `proposals/deploy-simplification.md` (S1); work: `work/main.md` M-22. Detail of the files:
`platform/repo-and-deployment.md` §6 and §7.1.

*From:* decided by user, 2026-11-16 (session 46, four points with four alternatives each; the wizard and the upgrade
path the same day; the bind default and what is gone confirmed, and hardening as an optional step chosen by the main
session on the user's "do what you think is best", 2026-11-16).

### Host names

Each service keeps a host name of its own: the gateway API, the gateway's admin, kalla-api, Källa's admin and Open
WebUI. They are not served under paths of one shared name. Separate names keep the two admin consoles and the APIs
apart as browser origins (their session cookies and scripts do not reach each other), and need no change in either
product.

The guides ask for as little DNS as that allows: every name is one label under `FORS_DOMAIN`, so one wildcard record
and one wildcard certificate for a sub-domain of the installation's own cover them all. A test host needs only its
own name: without a proxy the services share it on their ports (the Compose quick start).

Paths under one name are not built. If an organisation can get exactly one name, the APIs go first (prefixes the
proxy strips, since both serve `/v1`), Open WebUI keeps a name of its own, and the admin consoles need a base path in
each UI and admin API, in both products, with the shared origin decided explicitly.

*From:* decided by user, 2026-11-20 (session 49, after the quick start was run end to end).

### Deployment values across releases

A release may add, move or retire deployment variables without stopping a deployment that upgrades from the guides.

- **A new deployment variable** has a default in the deploy files or is generated by the bootstrap
  (`kalla-bootstrap`, in every deployment form). A value the platform can start without is a setting instead (Decisions
  "Settings rule"): with a default, or, when a person must choose, the feature that needs it stays off and Home tells
  platform admins until it is set (as the embedding profile). A variable that is required and has no default needs the
  user's approval; the bootstrap then stops with one line that names it, and the upgrade guide has a numbered step.
- **A variable that becomes a setting:** from production on, the variable stays readable as the deployment's value,
  which wins and is shown locked (as Decisions "Setup wizard"). Before production it may go in one step (Decisions "No
  backward compatibility before production").
- **A retired variable** (no longer read) goes on `kalla-bootstrap`'s retired list in the change that stops reading it,
  with a few words on what replaced it. While a stack still sets one, `kalla-bootstrap` names it on its checklist and
  Home shows platform admins a deployment notice (`GET /api/overview`, `deploymentNotices`, code `retired_variables`);
  nothing stops. The upgrade guide lists it. Compose runs the Portainer stack's service file, so it has the same
  list and notice, and `init-deployment.sh` refuses to write a retired variable into `deploy/.env`; Kubernetes'
  values are explicit, so it has no list.

Home's deployment notices are for platform admins and are not the attention items of admin design §5.5.1, which stay
P2. Detail: `admin/design.md` §5.5.1; `deploy/portainer/bootstrap.sh`.

*From:* decided by user, 2026-11-16 (session 43).

### .NET runtime settings

Every .NET container sets `DOTNET_GCHeapHardLimitPercent=0x4B` (hexadecimal). `AuditLog.Service` is the OTel service
name; the query-embedding step is `x-fors-step: query-embed`. Detail: `platform/repo-and-deployment.md` §6.2;
`platform/data-model.md` §2.4.

*From:* log row 70; decided by main session, 2026-10-30.

### ClamAV updates

freshclam updates from `database.clamav.net`; no internal mirror. Detail: `platform/repo-and-deployment.md` §5.5.

*From:* log row 11; decided by user, 2026-10-29.

## CI/CD, releases and tests

### CI/CD

GitHub Actions run build, test and release. RAG's own Entra registrations are Bicep with the Graph extension, applied
by `entra.yml`. Pinning actions by commit SHA with automated pin bumps (A-20, A-27) is postponed until before go-live. Detail: `platform/repo-and-deployment.md` §8.1.

*From:* log rows 8, 10, 31; decided by user, 2026-10-29; SHA pins postponed, 2026-11-08.

### CI on the self-hosted runner

Every workflow runs on the self-hosted runners; `verify.yml` and `parser.yml` run on every non-draft PR, each job only
when its paths change, and the full set before a release. Agents run only the test projects
they touch. The main session
merges on local evidence, then checks `main` after a batch; a red `main` is fixed first. `perf.yml` and
`evaluation.yml` schedules stay off until the test host; its runner awaits IT security. Pull-request jobs share the
runners' OS user, so a tag release builds without the build cache and with its own empty NuGet and npm caches (dry runs
keep the cache); before go-live, `release.yml` and `entra.yml` move to a runner of their own under a separate user,
labelled `release`, which runs no pull-request code (#704, with #506 and #507). Detail:
`platform/repo-and-deployment.md` §8, §12 (#4).

*From:* log rows 12, 88, 90, 91; decided by user, 2026-10-31; path-filtered jobs, 2026-11-05; release isolation,
2026-11-15 (#704).
*Also:* "CI on the self-hosted runner for every pull request", "Merge on local evidence, then check main".

### E2E optional on release

`e2e.yml` never runs on PRs, pushes or a schedule; it runs by hand or from `release.yml` when asked (`RELEASE_E2E`, the
dry run's `e2e` input). A skipped run lets the release go on; a failed one stops it. A done-when that needs the browser tests is
ticked on a green run. Detail: `platform/repo-and-deployment.md`
§8.2.

*From:* log rows 104, 105; decided by user, 2026-11-06.
*Also:* "E2E only on release or by hand".

## Cost attribution

### Cost attribution

Every assistant and every knowledge base runs on an **assigned gateway key** of its own department, chosen by id from
the department's assigned keys; Källa stores the key's id, never a secret. Every call of a chat turn (condense, query
embedding, rerank, answer, follow-ups) and the admin's test panel run as the assistant's key; indexing, re-embedding and
evaluation of a knowledge base run as the knowledge base's key. The gateway charges, limits and budgets each call as
that key, its team and department, and is the record of cost: Källa takes the cost from `x-fors-cost-usd`, from the
stream comment it asks for with `x-fors-stream-cost: 1`, and from the Integration API's `usage.read`, and keeps its own
finer split per step, joined on `x-fors-run-id` and `x-fors-step`. A per-assistant budget is the key's budget in the
gateway, shown read-only in Källa; `OwnerTeamId` keeps one meaning, the edit scope. An object without a key runs as
the service's credential key and is charged to the integration's owner team; once the gateway's `features` contain
`act-as-key`, publishing an assistant needs a key. Källa's own record (stage 1) is one row per day, object, step
(and client, for an assistant) in `chat."AssistantDailyUsage"` and `content."CollectionDailyUsage"`; a call without a
gateway cost is priced from the model mirror and kept apart as an estimate, never mixed into the gateway's figure; a
row belongs to the object's department at the time of the call. Calls made for no assistant or knowledge base
(Knowledge API callers without an assistant, the admin's checks and canary) are counted only in the gateway. The pilot
needs no hard budget (Q3). Detail: `platform/cost-attribution.md` §4–§6; `gateway-changes/design.md` §5.

*From:* log row 120; decided by user, 2026-11-11 (#580); stage 1's record and no pilot budget, 2026-11-12 (M-9).

## Sign-in and roles

### Identity model

Källa copies the gateway's identity model (OIDC code + PKCE BFF, claims, `KALLA_OIDC_*` keys, key and Data API patterns, security events) with its own clients and secrets (`kalla-admin`). Member groups are group claim values; if they cannot be read, scope narrows to claims only, never fails open. Policy denials above the ceiling or outside the audience answer 404. The audit actor is `oid` on Entra, else `sub`. Detail: `identity/design.md` §1.1, §1.3–§1.6, §2.6, §4.3.

*From:* log rows 29, 32, 55; decided by user, 2026-10-30.

### Role groups as a setting

Which IdP groups grant which Källa role is `Settings.Identity.RoleGroups`, edited by platform admins and read at
each sign-in. `Oidc:RoleGroups:rag-platform-admin` (`KALLA_OIDC_GROUPS_PLATFORM_ADMIN`) stays as a break-glass for
the first platform admin and is merged with the setting, never replaced; the other four variables go. Roles in the
token's role claim still count. Built in M-17 C10 (#693).

*From:* decided by user, 2026-11-13 (deploy-configuration D2).

### Identity providers: AD and Entra ID through Keycloak, Källa standalone later

In production AD and Entra ID must both work, brokered through Keycloak (Entra ID as an OIDC provider, AD by LDAP federation or AD FS) and directly through `Oidc:*`; the brokered route is tested with a stand-in upstream provider. Entra ID comes in P2 as configuration, not code, with the Graph group lookup; On-Behalf-Of is designed now and
built later. Källa's realm part needs only the `departmentCodes` and `teamCodes` claims from the base realm, so Källa can run standalone. Detail: `identity/design.md` §11.2, §11.3.

*From:* log rows 21, 55, 71, 72; decided by user, 2026-11-05.

### P1 pilot sign-in

The pilot uses the gateway's Keycloak (its own Compose project on the gateway's production host, defined in the gateway repo): one realm `fors` with both systems' clients and roles, `departmentCodes` shared, *full scope allowed* off on every client. Källa delivers `rag-additions.json`, applied by keycloak-config-cli after the gateway's base file (`no-delete`), which holds password policy, brute-force protection and the user profile. The gateway operator administers the realm. Accounts are local (LDAP to AD later), without OTP; MFA returns with brokered Entra ID or AD.
Keycloak's database is `keycloak` in the gateway's Postgres, under its backups; each repo keeps its own dev Keycloak in
its AppHost. `kalla-api` is an audience only. Detail: `identity/design.md` §11.2.

*From:* log rows 71, 73, 74, 83; decided by user, 2026-11-06.

### Shared sign-in with the gateway

One group membership gives roles in both products: operator-made Keycloak groups map realm roles of both systems (e.g. `ume-admins` → `gateway-admin` + `rag-platform-admin`), never in `rag-additions.json`. Each system's client token carries only its own roles. On the direct Entra route each product has its own app registration with app roles, and one security group is assigned in both. Neither product's code names the other's roles; Källa's role groups keep the `rag-` names. The two
products are separate Compose projects with the same layout, and Källa's ports stay clear of the gateway's. Detail: `identity/design.md` §11.3.

*From:* log rows 74, 118; decided by user, 2026-11-07.

## Keys and the gateway

### Key service

Master keys are protected files (file key provider) while no `sensitive` data exists; Azure Key Vault is the target, adopted later with a re-wrap. Long-lived keys are escrowed offline. `public` blobs are not encrypted. Client keys are `kalla-sk-…`
(secret) and `kalla-pk-…` (public). Detail: `postponed-decisions.md` K1.

*From:* log rows 9, 30, 55; decided by user, 2026-10-30.

### Gateway keys

Each Källa service (api, admin, worker) holds **one credential key** of the gateway integration *Källa*, as a secret
file, with `AllowedIpRanges` and `RequireMaxResidency`. A call for an assistant or a knowledge base sends
`x-fors-on-behalf-of: <assigned key id>` and runs as that key (its models, residencies, PII policy, limits, budgets and
usage record); the answer's echo of the header is checked, and a missing or different echo fails closed
(`Gateway:RequireDelegationEcho`). Every call still sends `x-fors-max-residency` from the data's level, and the worker
never sends `External`. The gateway bounds an acted request by the credential key too: `RequireMaxResidency` applies
when either key requires it, and the effective residencies are the assigned key's ∩ the credential key's (decided by
user, 2026-11-12, #580; built in thekim1/fors#84, feature `credential-residencies`). So each service's
credential key is capped at the highest level it serves (the worker's at `Eu`). An assigned key cannot
authenticate by itself, so nothing per assistant is copied, stored or rotated in Källa. Each setting has one owner:
departments, teams, models with their limits and prices, and a key's models, residencies, PII policy, content guard,
rate limits, output cap and budgets are the gateway's and shown read-only; which key an object runs on, prompts,
retrieval and answer settings, who may call an assistant and the content's level are Källa's. Configuration comes
from the gateway's data plane with the credential key, not the Data API: the gateway's features, the assigned keys and
the models with their limits and prices from `GET /v1/fors/context` and `GET /v1/models`, polled with `If-None-Match`;
departments and teams from the same context, where `kalla-admin`'s credential key (and only it) holds the scope
`organisation.read` (every department and team, inactive ones included; feature `organisation`), so the Data API key
goes. Department member groups come from the same context (`member_groups` per department, feature `member-groups`,
gateway v0.10.0) and are never edited in Källa (the user chose to wait for the gateway rather than build another
source, 2026-11-12, #580). The contract is the gateway's OpenAPI document `docs/openapi/integrations.json` (pinned at a
release tag), which changes by addition only. Each service reads the gateway's `features` from its own
`/v1/fors/context`, so it acts as a key only once its own gateway instance says `act-as-key`. From then on a published assistant always has a
key of its own department: publishing or restoring one without a key is refused, and so is a save that clears the key of a
published assistant (V39, `gateway_key_required`); to drop its key, an admin unpublishes it or picks another key, so it never
runs on kalla-api's credential key outside its department's budget (confirmed by user, 2026-11-13). Detail:
`gateway-changes/design.md` §5, §13.4; `identity/design.md` §4.7; `admin/design.md` §4.10.

*From:* log rows 33, 55, 66, 120; decided by user, 2026-11-11 (#580); the organisation from the context and member
groups by hand until the gateway has them, decided by user, 2026-11-12 (#580); V39's required key on a published assistant confirmed by user, 2026-11-13.

### Gateway releases

The gateway never depends on Källa; any key may send a residency ceiling. Release 1 (live since gateway v0.9.1): `x-fors-max-residency` (also on the admission path), FakeLlm `dimensions`, embeddings checked per input. Release 2 (P2): member groups from claims, tenant and guest refusal. The integration features follow the gateway's own phases: phase 1 credential and assigned keys, `GET /v1/fors/context` and the model facts on `GET /v1/models`; phase 2 acting as an assigned key (`x-fors-on-behalf-of` and its echo), run metadata and the stream cost; phase 3 the Integration API (`usage.read`). Källa tests a capability by the context's `features`, never by version. Rerank is a separate optional endpoint. Källa retries `502`/`504` with the same ceiling, never a wider one; `403 max_residency_not_met` is final; `503` + `Retry-After` means in-ceiling targets are drained; a bad or (with `RequireMaxResidency`) missing header is `400`. Detail: `gateway-changes/design.md` §1, §3.3, §5, §13.

*From:* log rows 33, 34, 55, 66, 74; decided by user, 2026-11-06; the integration phases decided by user, 2026-11-11
(#580).
*Also:* "Gateway release 1 and shared Keycloak as built"

### Interim gateway for P1a

A test/dev gateway on a fixed HTTPS host with EU embedding and chat models is the interim gateway (J10); its Keycloak holds the shared realm `fors` (J11). FakeGateway serves CI and tests. Its address and keys are only in user secrets, `deploy/.env` and secret files, never in the repository. Detail: `gateway-changes/design.md` §13.4.

*From:* log rows 78, 83; decided by user, 2026-10-31.

## Classification and residency

### Classification

Levels are `public`, `internal` and `sensitive`; `sensitive`, the municipality's highest category, uses on-prem models only. Rows carry a plain `Classification`; there are no data tiers, and `sensitive` data would get a separate deployment (T1). The pilot holds no `sensitive` data. Detail: `platform/data-model.md` §6.

*From:* log rows 9, 14, 22, 55; decided by user, 2026-10-30.

### Residency per level

Every model call sends an explicit ceiling from its data's level: `public` `External`, `internal` `Eu`, `sensitive`
`OnPrem`. The worker never sends `External`, so its calls run `public` at `Eu`; embedding calls use the profile's
`MaxResidency`. Each client in the registry has its own classification ceiling. The ceiling of each service is its gateway credential key's `allowed_residencies` from `/v1/fors/context` (empty: all), so the gateway owns it and `Gateway:MaxResidency` goes; before the first context every call is `OnPrem` (user 2026-11-13, deploy-configuration D1; built in M-17 C9, #691). Under register M2 the keys allow `Eu` at most, so everything is EU-hosted. Chat runs at the assistant's level; with a client that keeps history every turn runs at max(assistant level, client ceiling). The answer's `x-fors-residency` is checked before use. Detail: `identity/design.md` §4.7.

*From:* log rows 14, 17, 25, 55, 77, 100; decided by user, 2026-10-30.

## Chat history and data protection

### Chat history

History mode, retention, encryption and review rules are set per level within platform-admin limits; retention defaults to 90 days per assistant. Until A1 (archive law) is answered the pilot runs history mode `none` (metadata and ratings only): no bucket keys, content encryption, crypto-shredding or conversation review are built. Stored content will use one data key per (level, expiry month), with key destruction off until A1 allows it. Detail: `chat-api/design.md` §11.2, §12.3, §13.

*From:* log rows 4, 23, 55, 65; decided by user, 2026-10-30.
*Also:* "Weakness review 4"

### DPIA and the pilot client

The DPIA (G1), owned by the user with the data protection officer, is finished before go-live. It covers Open WebUI's store as the primary copy of pilot chats and decides W2; public-records requests are answered from that copy; a fixed Open WebUI banner warns against confidential input. The pilot client keeps `internal` only for EU providers under a processor agreement listed in G1, otherwise on-prem models only; with no on-prem hardware the pilot then waits. Detail: `chat-api/design.md` §13.3; `identity/design.md` §4.1.

*From:* log rows 65, 77, 78; decided by user, 2026-10-30.

## Model routes

### Pilot embedding model, U-b

The pilot profile is `text-embedding-3-large` from an `Eu` account at 1024 dimensions, `MaxInputTokens` 8191, with `EmbeddingProfiles.MaxResidency` as its ceiling. Every call for the profile (embed, canaries, query embedding, evaluation) sends `dimensions`; a BGE-M3 profile never does. Moving to on-prem BGE-M3 re-embeds every chunk. Detail: `ingestion-worker/design-processing.md` §11; `platform/data-model.md` §2.1.

*From:* log rows 77, 78, 89; decided by user, 2026-10-31.

### Embedding canary on demand

The platform makes no periodic model calls: the embedding canary runs only when a platform admin asks (`POST /api/gateway/canary-check`). The result is stored on the profile; cosine below 0.999 refuses the profile in the worker until a later check passes. The canary is recorded once per profile after the model sync; check after every gateway model change. Detail: `ingestion-worker/design-processing.md` §11.

*From:* log row 102; decided by user, 2026-11-03.

### Editable embedding profiles before use

A platform admin edits an embedding profile in the admin UI (`GatewayModel`, `Dimensions`, `RequestDimensions`,
`MaxInputTokens`, `MaxResidency`) while no chunk set uses it; once one does, only `MaxResidency` stays editable, within
the platform ceiling, and a model or dimension change is the P3 profile migration. Every save first runs one canary
embedding with the new values and is refused if it fails, then stores that canary. **Test** runs the same call without
saving and names the outcome in plain words (the gateway's refusal code, the residency, or the dimension count
returned). A platform admin creates the profile in the admin UI; nothing seeds it (Decisions "Deploy configuration"). Detail: `admin/design.md` §4.5;
`platform/data-model.md` §2.1; `ingestion-worker/design-processing.md` §11.

*From:* decided by user, 2026-11-11 (#547).

## Pipelines and the engine

### Pipelines

Ingestion runs one fixed linear pipeline per source type: template + parameters, no graph or user-added stages. The query side is a fixed linear step sequence per assistant, set in a form with a test panel. The platform team owns templates; department admins pick one and set parameters; `rag-pipeline-editor` edits their parameters. A visual builder is P3 behind trigger C2 (two departments, or a recurring platform-team request, need a flow parameters cannot express). Detail: `ingestion-worker/design-engine.md` §3, §13.

*From:* log rows 1, 2, 55, 57; decided by user, 2026-10-30.

### Ingestion engine

A document belongs to one source; reuse across sources is by content-addressed caches only. Work is reconciliation (`EnsureCurrent` + nightly drift sweep) with an execution-id fence, at-least-once with effectively-once effects, per-department fairness, short chunk-write transactions and counters derived from run items. Cron with per-source jitter; defaults: URL lists nightly, crawls weekly at night. Detail: `ingestion-worker/design-engine.md` §2.3, §7, §9, §10.

*From:* log rows 37, 38, 55; decided by user, 2026-10-30.

### Preview, publish and rollback

Preview (`stage-preview`) runs the engine isolated: no index or cache writes. A publish re-processes every live document of the affected sources (unchanged stages come from cache), then shows an advisory evaluation report; the owner keeps it or rolls back in one click. Rollback is never gated; superseded sets are kept 14 days. Plans from effect tags are P2. Detail: `ingestion-worker/design-engine.md` §11, §12.

*From:* log rows 38, 55, 62, 86; decided by main session, 2026-10-31.

## Site profiles and learning

### Site profiles

A versioned profile per (level, host) sets HTML extraction, identity rule, excluded types (images and video by default) and `lastmod` reliability, set only after a 500-URL sample passes. HTML goes by the profile, PDFs by `/v1/inspect`. In P1 profiles are manual (umea.se: the hand-checked Sitevision preset), edited and rolled back by platform admins. Nothing is learned and applied automatically: site-profile learning and parse-strategy proposals are P2, as is editors' read access. Detail: `ingestion-worker/design-connectors.md` §5.6; `ingestion-worker/design-processing.md` §3.5.

*From:* log rows 55, 56, 62, 64, 119; decided by user, 2026-11-07.
*Also:* "Learning"

## Crawler and connectors

### Crawler

Own crawler, not Firecrawl or Crawlee; a hosted service only as an `IRenderer` backend for anti-bot sites. User agent `UmeaKommunBot/1.0 (+about page)`, robots.txt honoured (both changeable per host). `noindex`/`nofollow` honoured on external hosts, ignored on municipal ones. Defaults 10,000 pages, depth 10; off-site links only when an `include` names the host, as leaves. P1 has one frontier per source; the shared frontier is P2. Detail: `ingestion-worker/design-connectors.md` §5.1, §5.2, §6.0.

*From:* log rows 39, 40, 41, 62, 116, 117; decided by user, 2026-11-07.
*Also:* "Crawler (step 6b questions)", "Crawler build vs buy"

### Configurable politeness per host

Politeness rules (rate, connections, `Crawl-delay`, latency throttle, night window, back-off, breaker, robots.txt, User-Agent, renders per host) are defaults. A platform admin may change them per host in `Settings.Crawl.HostOverrides` (presets, audited) within `Crawl:MaxHostRate` 50 req/s and `Crawl:MaxHostConnections` 16; robots.txt may be ignored only with a reason (its groups are still
matched against `UmeaKommunBot`); the User-Agent is free. 429/503 are honoured up to a configurable cap. Detail: `ingestion-worker/design-connectors.md` §6.6.

*From:* log rows 53, 116; decided by user, 2026-11-07.

### Document identity

A URL's identity is its final URL after redirects, mapped by the profile's identity rule only when verified; `rel=canonical` folds an alias only on equal content hash. The worker's `UrlCanonicalizer` normalises; the admin refuses exact duplicates only. Scope patterns match before the identity rule. An upload is (source, slot) with the content hash as version; uploads need no topics. Detail: `ingestion-worker/design-connectors.md` §3, §4, §9.

*From:* log rows 41, 55, 56, 73, 93; decided by user, 2026-11-05.

### Freshness and deletions

Scheduled crawls are full traversals with conditional GETs. A traversal is complete at ≥ 99 % of seeds resolved; failed seeds are never deletion candidates in that run, and deletions pass a guarded reconcile phase. URLs leaving the sitemap are re-requested, `Gone` on 404/410. Retrieval has per-collection age decay (on for PDFs in `website`) and `demote` patterns. Detail: `ingestion-worker/design-connectors.md` §5.4, §13; `knowledge-api/design.md` §6.5.

*From:* log rows 41, 56, 64; decided by user, 2026-10-30.

### Egress guard and ClamAV

Fetches go through a DNS-pinned egress guard, with the host firewall and a startup canary behind it. ClamAV scans every upload and every fetched file other than HTML, text and CSV, fail-closed, unless the deployment turns the scan off (Decisions "Malware scan optional"); the UI shows malware metadata only. SharePoint ACLs are not mirrored (P2). Detail: `ingestion-worker/design-connectors.md` §7, §11.

*From:* log row 41; decided by main session, 2026-10-29.

### Malware scan optional

The malware scan is a deployment's choice: `ClamAv:Mode` is `on` (default) or `off`, a deployment knob like
`Parser:Mode`. With `on` the scan is as before, fail-closed, and outside Development a worker without `ClamAv:Host`
refuses to start. With `off` no `kalla-clamav` container runs, and the worker stores files without a scan and says so:
a warning and security event 5106 `rag.malware.scan_off` at every start; the health check stays Healthy with the text
"malware scan off"; an upload slot is `unscanned`, never `clean`, and has no `ScannedAt`; the run's report carries
`malwareScan: "off"`; and the admin UI shows *Not scanned for malware* on those files and runs. Files stored while the
scan was off are not scanned when it is turned on; an automatic re-scan may come later. Detail:
`ingestion-worker/design-connectors.md` §9.1, §11; `platform/repo-and-deployment.md` §5.5, §7.1; `admin/design.md`
§5.5.7.

*From:* decided by user, 2026-11-09 (#535); the `unscanned` state, the health text and no re-scan by user, 2026-11-12.

### JavaScript rendering

A `browser` source (`fetchMode`, optional `readySelector`) is rendered by `kalla-renderer`, a browser-only container with no route out, driven by the worker, which fetches every browser request through its guard. Fresh context per page; images, fonts, media, trackers blocked; a page is ready when no routed request has been open for `Render:QuietPeriod`, and with a `readySelector` once the
selector holds text and then that quiet rule holds; up to 4 renders per worker (`Render:MaxConcurrent`) and per host
its `renderPerHost` (default 1) up to its `maxConnections`, a slot taken only when free. Scripts and stylesheets are cached in the worker per run. There is no fallback to the raw shell: a failed render
keeps the last good content and links; `browser` pages render every traversal. The preflight compares raw and
rendered text as a hint only. The renderer is no go-live gate for the server-rendered umea.se pilot. Detail: `ingestion-worker/design-connectors.md` §8.

*From:* log rows 41, 55, 115; decided by user, 2026-11-07; render cap and ready rule by user, 2026-11-08.

### Crawlee review: ideas adopted

Adopted: meta refresh as a redirect hop; `redirect.cookie_required`; sniffing a missing or `octet-stream` type; per-host soft-404 fingerprint; challenge pages transient, never evaded; sitemap retries, text sitemaps, `/sitemap_index.xml`; in the renderer bounded operations, navigations, popups and downloads as hops or links, per-page cookies, `scrollToLoad`. Shadow-DOM flattening and same-host iframe inlining
come later, when a source needs them: its rendered text falls well short of what a browser shows because of web
components or same-host iframes (none of the measured sites does). Detection by counted verdicts comes later. Not
adopted: its URL normalisation, retries without backoff, fail-open robots, in-memory per-host delay, sessions,
fingerprints, proxies and clicking elements. Detail: `ingestion-worker/design-connectors.md` §3.2, §5.3, §8.3.

*From:* log row 117; decided by main session, 2026-11-07; shadow DOM and iframes later, user, 2026-11-08.

## Parser and processing

### Parser

P1 formats: PDF, OOXML, ODF, JPG/PNG/TIFF, text, Markdown, CSV, HTML; uploaded legacy Office (`.doc`, `.xls`, `.ppt`) and RTF are converted to OOXML by LibreOffice in the parser image (headless, forced import filter, a profile per job, process-group kill at the timeout); the same types when crawled, and e-mail (`.eml`, `.msg`), P2. **Before go-live:** LibreOffice in the parser or a separate converter container, decided together with the parser's sandbox; archives and macro files refused. The worker parses only HTML, text, Markdown, CSV; every PDF goes to the Docling parser (pypdfium2 fast path). Pictures give captions and OCR; Tesseract `swe`, explicit. The parser is its own supervisor with pre-warmed
processes, and per-page parse rules act on `/v1/inspect` signals. The worker stores `result.json` and the
`DoclingDocument`. NER model in the image; port 8000, TLS in production; a permissively licensed JSON Schema validator, no `JsonSchema.Net` binaries; logs only in P1. Detail: `parser/design.md` §3, §7.6, §12, §13.2.

*From:* log rows 35, 36, 55, 67, 69, 114; decided by user, 2026-11-07; legacy Office and RTF uploads by user, 2026-11-09 (#526, #527).
*Also:* "Parser telemetry is logs only in P1"

### Chunking

In P1, flat heading-split chunks, by default 400 tokens including the breadcrumb › heading prefix and ~15 %
whole-sentence overlap (template parameters `chunkTokens`, `overlapPercent`), tables as row groups, BGE-M3 tokenizer in .NET; the Knowledge API expands sections. A noise-only change re-keys the live set. Detail: `ingestion-worker/design-processing.md` §6, §12.

*From:* log rows 37, 43, 55; decided by user, 2026-10-30.

### Personal data

Published municipal content is public information. .NET patterns run on every chunk set; KBLab BERT NER (`/v1/ner`) only when names can change the flag, inside the job. Counts, fired types and flag live on the set, never values. Source `piiMode`: `hold` (default) holds a flagged set in a `public` collection; `report` lets it go live, raising 5103 for the editors' report, except personnummer and samordningsnummer. The umea.se source uses `report`. `Settings.Engine.PiiOwnDomains` is set to `umea.se`, `vakin.se`,
`umeaenergi.se`, `visitumea.se` in the admin, subdomains included (the seed has `umea.se` only). Detail: `ingestion-worker/design-processing.md` §9.

*From:* log rows 37, 42, 43, 55, 73, 92, 93, 94; decided by user, 2026-11-05.
*Also:* "Personal data on the municipality's own published site", "Municipal companies' e-mail domains, `PiiOwnDomains`"

## Retrieval

### Knowledge API (step 7)

Retrieval is hybrid in the shared `Kalla.Retrieval` engine, used unchanged by the knowledge API, the evaluator and
the test panel: keyword and vector candidates, RRF in .NET, then per-collection age decay (off by default) and source
`demote` patterns, at most 3 chunks per document by default (1–10). Top-k is 10 (max 50), the query is used as given, results are not
cached, query text is never stored, and a failed embedding degrades to keyword-only. Detail: `knowledge-api/design.md`
§1, §6.

*From:* log rows 44, 45, 64; decided by user, 2026-10-30.

### Search cost at scale

A scope of up to 10,000 chunks, counted after topic filters, is searched exactly, a larger one through HNSW, whose
recall the worker samples per enumerable scope and restores by rebuild on `ConnectionStrings:ragdb-maintenance`, never through a
pooler. The keyword search drops lexemes found in more
than 30 % of the scope's chunks; the worker keeps the per-collection statistics for it (`KeywordStatistics`), so
kalla-api runs no corpus-wide `ts_stat`, while the cheap scope-size count stays in kalla-api. Latency over budget at
100,000 chunks in one scope is accepted for the pilot and re-measured in R1. Detail: `knowledge-api/design.md` §6.2, §6.4.

*From:* log rows 73, 87, 119; decided by user, 2026-11-07; statistics in the worker: main session at the user's go-ahead, 2026-11-11 (M-10).
*Also:* "Keyword stop list before the pilot crawl", "M-2 open design calls"

### Rerank pulled forward

Rerank is optional everywhere and no go-live gate: with no rerank model configured there is no call and no degraded
status. An assistant turns it on (`chat` or `gateway`) when the gold set shows a gain over hybrid; the reranker scores
the top 20 fused candidates at the turn's ceiling within 3 s (`ChatModelReranker`) or 1.5 s (`GatewayReranker`), and
an error keeps the fused order. `ChatModelReranker` uses a small chat model chosen in the platform settings,
without `temperature`; `GatewayReranker` calls the gateway's optional, Cohere-compatible `POST /v1/rerank` (Berget AI
`bge-reranker-v2-m3` at `Eu` until on-prem), which answers 404 or 400 when no rerank model is configured. Detail: `knowledge-api/design.md` §6.7.

*From:* log rows 74, 75, 110; decided by user, 2026-11-06.
*Also:* "Gateway rerank time limit 1.5 s"

## Chat

### Chat API (step 8)

The client's messages are authoritative for the prompt (last 10, at most 4,000 tokens; no SQL on the hot path).
Residency is checked before the first byte, and a history-keeping client runs every turn at max(assistant level,
client ceiling). Only cited sources are shown, renumbered by first citation, linked to the public URL or a 7-day signed
download grant. An output filter removes images and de-links hosts outside the allow-list; clients other than Open WebUI get signed
conversation ids. Grant links work on the staff network only, and access is re-checked at every download. Open
WebUI's own task generation is off; attachments are refused in P1. Detail: `chat-api/design.md`
§1, §8, §9.

*From:* log rows 44, 46, 47, 55; decided by user, 2026-10-30.

### Condense on the EU route

Condense rewrites a follow-up that is not self-contained into a search query, for retrieval only, at `Eu` until
on-prem models exist and never above the turn's ceiling: `Settings.Chat.CondenseModel` is set to `gpt-6-luna` with
`reasoning_effort` `none` (`Settings.Chat.CondenseReasoningEffort`, sent only when set), no `temperature`, `max_tokens`
96, within `Chat:CondenseTimeout` 1,200 ms; on a timeout the raw question is searched. A question over 2,000 characters
(up to 16,000) is condensed on every turn within 3 s, else cut to 2,000 characters; the answer model always gets the
full question. A failed condense never yields the not-found text. Detail: `chat-api/design.md` §5, §5.1.

*From:* log rows 46, 77, 98, 108; decided by user, 2026-11-06.
*Also:* "Long questions condensed into the search query"

### Not-found gate

Coverage is `low` only when vector and keyword side are both weak (the vector side alone until the keyword floor is
calibrated), with both thresholds set from answerable and unanswerable gold-set questions. Then the assistant gives its
fixed text with contact links and no model call, unless its general-knowledge level says otherwise. Detail:
`knowledge-api/design.md` §7; `chat-api/design.md` §6.2, §10.

*From:* log rows 46, 55, 63; decided by user, 2026-10-30.

### General knowledge per assistant

`answer.generalKnowledge` is `off` (default), `supplement`, `fallback` or `open`, a versioned draft field set by anyone with `edit` on the
assistant, each with a locked frame; no level
allows invented municipal facts or cites general knowledge. With `supplement` the answer says which parts do not rest on the sources, and coverage `low` still gives
the not-found text; with `fallback` and `open`, coverage `low` gets an answer without sources behind a fixed platform
caveat, stored as `answered-general` and counted apart from not-found. "Search unavailable" is the fixed text at every
level. On a `public` assistant a level other than `off` is a warning and needs
explicit confirmation on publish, restore and a level change to `public` (V36). A client other than Open WebUI may
ask for a level per chat request (`kalla.general_knowledge`); the turn runs at the stricter of the request's level and
the assistant's, so a request only tightens and V36 keeps holding (#726). The admin's test panel takes the same
optional level. Detail: `chat-api/design.md` §3.8, §7.3, §10.4; `admin/design.md` §4.6, §4.9.

*From:* log rows 103, 111; decided by user, 2026-11-07; the per-request level by user, 2026-11-15 (#726).
*Also:* "General knowledge on a public assistant"

### Related documents and the `kalla` chat extension

A client other than Open WebUI may add `kalla` to a chat request to get the turn's `outcome` and `degraded`, plus on
request `related_documents` (1–10, from the same retrieval, empty on coverage `low`) and `follow_ups` (1–5, one
condense-model call within 2 s, never stored, lines with markers, links or personal data dropped), and
`general_knowledge`, a stricter general-knowledge level for the turn (Decisions "General knowledge per assistant"),
and `escalation` (Decisions "Escalation hand-off"). Unknown members are refused. Each related document carries its
dates (Decisions "Source dates on citations"). Detail: `chat-api/design.md` §3.8, §9.7, §9.8.

*From:* log rows 106, 107; decided by user, 2026-11-06.
*Also:* "Follow-up suggestions"

### Escalation hand-off

A client other than Open WebUI may ask for `kalla.escalation`. When the turn ends `not-found`, `answered-general` or
`unavailable`, the answer then carries the escalation route: the reason, the assistant's contact links (its not-found
links, else the platform's), and a short hand-off summary of the user's problem for a second-line ticket. The summary
comes from one condense-model call at the condense ceiling, as the step `escalation` on the assistant's gateway key, within
`Chat:EscalationTimeout` (2 s; 0 turns the summary off, and the route still comes). Links are removed and personal data
masked. It is never stored or logged, and `null` when skipped or failed. An `answered` turn gets `escalation: null`.
Its instructions are an editable prompt text, with locked data, personal-data and no-links lines. Writing back to a
ticket system stays later. Detail: `chat-api/design.md` §3.8, §9.9.

*From:* support chatbot proposal F3; picked by user, 2026-11-19 (#857).

### Source dates on citations

Every citation and related document carries two dates. `modified` is when the source says it last changed: the chunk's
date, else the document's, else for an upload the upload time. A crawled page that states no date has none; its fetch
time no longer stands in. `checked` is when a crawl last confirmed a crawled page: fetched, `304`, or an unchanged
sitemap `lastmod`. An upload has no `checked`. The citation label shows both, in the turn's language
("2026-03-02, kontrollerad 2026-11-18"). There is no staleness threshold or warning. Detail:
`knowledge-api/design.md` §10.1, §10.4; `chat-api/design.md` §9.3, §9.7.

*From:* support chatbot proposal F4; picked by user, 2026-11-19 (#857).

### Topics per request

A client other than Open WebUI may send `kalla.topics` with a chat request: the turn then searches only chunks that
carry one of those topics, inside what the assistant's links allow. A link with topics keeps the ones also asked
for, and is left out when none is; a link without topics is searched for the asked topics. A request narrows and
never widens. A topic that matches no link is not an error: the answer's `kalla.topics` lists the ones that were
searched, and a turn with nothing left to search ends `not-found`. Detail: `chat-api/design.md` §3.8;
`knowledge-api/design.md` §4.2.

*From:* support chatbot proposal F8; picked by user, 2026-11-20.

### Chat rate limit per key

A key that is itself the chat subject is limited by `Keys.RequestsPerMinute`, else by the per-user value; Open WebUI
users always get the per-user value. Knowledge routes do not read it in P1. Detail: `chat-api/design.md` §4.1 step 4.

*From:* log row 109; decided by user, 2026-11-06.

### Feedback and conversation review

Ratings come through `POST /v1/feedback` and an Open WebUI import. Comments are read only by `rag-reviewer` and
`rag-department-admin` after an audited reason, whole conversations only by `rag-reviewer` (`sensitive`: a second
approver); both exist only once chat content is stored. Detail: `chat-api/design.md` §14; `admin/design.md` §8.

*From:* log rows 46, 50, 65; decided by user, 2026-10-30.

### Chat prompt texts configurable

The platform's own prompt texts are platform settings (`Settings.Chat.Prompts`, Settings page, platform admins), each
optional with today's text as its built-in default. Editable: the frame's identity line (`{assistant}`), the sources
label, the rule lines of each `generalKnowledge` level with and without sources, the conversational and degraded-search
lines, both condense prompts, the follow-up and escalation prompts' instructions, and the degraded and general-knowledge caveats (sv,
en). Locked in code, because the marker parser, the output filter and the injection defences rely on them: the
citation line, the no-invented-numbers/no-links line, the `<sources>`-is-data line and the block's data note, the
language line, the follow-up prompt's data, personal-data and no-citations lines, and the escalation prompt's data,
personal-data and no-links lines. Texts are platform-wide and apply at once to every assistant; an assistant's own
`SystemPrompt` still follows the frame. Detail: `chat-api/design.md` §5, §5.1, §7.2–§7.3, §9.8, §9.9, §10.2, §10.4; `admin/design.md` V38.

*From:* decided by user, 2026-11-11 (#548).

### Answer length from the model

`QuerySteps.answer.maxOutputTokens` and `answer.contextTokens` are optional overrides; absent (the default, *Use the
model's limit*), they take the model's `MaxOutputTokens` and `ContextWindow` from the `GatewayModels` mirror. The
request's `max_tokens` is the override, else the model's limit less the estimated prompt within the context window,
else left out when the limit is unknown (the prompt budget then keeps its 1,024-token answer reserve). There is no fixed
4,096 ceiling. An optional platform-wide ceiling, `Settings.Chat.MaxAnswerTokens` (unset by default), guards against
long, costly answers on models with large output limits: when set, it caps every answer's `max_tokens`, override
included, and `max_tokens` is then always sent, also when the model's limit is unknown. A platform admin enters both
limits on the mirror row until the gateway sends them (`fors.context_window`, `fors.max_output_tokens` on
`GET /v1/models`); then the sync fills them and the manual fields go. The assistant's gateway key's output cap
(`max_output_tokens` in `/v1/fors/context`), when set, caps every answer too. The test panel shows the `max_tokens`
sent and whether the answer stopped at the limit. Detail: `chat-api/design.md` §6.3,
§8.1; `admin/design.md` §4.9, §4.10, V17, §5.5.4; `platform/data-model.md` `GatewayModels`, `AssistantVersions`;
`gateway-changes/design.md` §7.3.

*From:* decided by user, 2026-11-11 (#548); the platform ceiling decided by user, 2026-11-11, from the main session's
review of #548; the limits from the gateway and the key's output cap decided by user, 2026-11-11 (#580).

## Evaluation

### Gold set drafted by the main session, reviewed by DoIT

The pilot gold set leads every report: about 120–150 answerable, 30–50 unanswerable and at least 10 temporal
questions, drafted by the main session in residents' wording and stored as `human` questions once DoIT approves them.
Go-live and the pilot's end need hand-graded answers: correct ≥ 80 %, citation precision ≥ 90 %, false not-found
≤ 10 %, off-topic not-found ≥ 80 %. Hybrid and the decompounder stay only if they beat vector-only. The full Swedish benchmark stays postponed. Detail:
`ingestion-worker/design-processing.md` §13.4, §14.

*From:* log rows 15, 63, 95; decided by user, 2026-11-01.
*Also:* "Weakness review 2"

### Evaluation models at the collection's ceiling

Retrieval evaluation is advisory and never blocks: no gates, no `building` state, no floor. Questions point at
documents; generated sets (up to 100 judge-filtered questions) are made and judged at the collection's ceiling
(`sensitive` `OnPrem`, `internal` and `public` `Eu`, as the worker never sends `External`; lowered to the
worker's credential-key ceiling, "Residency per level") by any gateway chat model listed there (`Settings.EvaluationModel`, `EvaluationJudgeModel`),
without `temperature`, JSON output, a fenced reply accepted. recall@10 and MRR run on the full collection after each
publish and nightly; question embeddings come from the cache. Feedback enters only as editor-rewritten questions without personal data; answer judging is P2.
Detail: `ingestion-worker/design-processing.md` §13, §14.

*From:* log rows 27, 42, 43, 55, 62, 100; decided by user, 2026-11-02.

## Admin and assistants

### Admin API and UI

The admin copies the gateway's BFF and CSP; a write is one transaction with `xmin` concurrency and cross-rules in
`ConfigRules`. UI text is English, all in `labels.ts` and `messages.ts`, so Swedish can be added later; WCAG 2.2 AA, with one NVDA
pass of the pilot journey before the pilot. Pilot department admins set up their own collections and assistants,
with the platform team pairing.
P1 has pages for the department journey only, platform objects use generic forms and Scalar, except clients and keys, which have a page of their own (user, 2026-11-13; `admin/design.md` §5.9). An app is connected in one go (its assistants, its first key and how to connect, §5.9.1), every platform page is in the side menu and Settings is a page of sections (§5.10; user, 2026-11-13, M-16). The test panel runs
chat's code on the draft, unpublished collections included; its extra detail and the Overview attention items are P2.
From the admin UI review (user, 2026-11-15): Home shows sync health and a **Live** card per published assistant beside the
setup checklists, which fold away once published (§5.5.1); the knowledge base and assistant lists show last fetch,
quality, knowledge bases, model and use, and every list row is a real link (§5.5.2, §5.5.4); the audit log shows
names linked to their objects, events in plain words and pickers for its filters (§9.1); the assistant page keeps
its draft form, with wording that says what saving does (§5.5.4).
Detail: `admin/design.md` §1, §4.9, §5, §9.1.

*From:* log rows 50, 51, 55, 99, 119; decided by user, 2026-11-07; clients and keys page, user 2026-11-13; connect flow, menu and Settings page, user 2026-11-13; admin UI review UI1 B, UI2 B, UI3 A, UI4 B, user 2026-11-15 (`proposals/admin-ui-review.md`).

### Assistants

An assistant's knowledge is only its own collection links, never skills. There is one assistant type; it runs the agent
loop (P2) when a linked skill has tools. Edits go to a draft; publish, history and restore are in P1, with a
description and contact per assistant; diff, test questions, templates, copy and owner-team scoping are P2. Detail:
`admin/design.md` §1, §5.5.4.

*From:* log rows 60, 62; decided by user, 2026-10-30.
*Also:* "Weakness review 1"

### Organisation

Departments own all content; the platform team owns none. A knowledge base, with a contact person, is shared with its
own department (default), selected departments or the whole municipality, used read-only through the linking department's assistants,
and that department sees only its name, owner, level, contact and topics; "whole municipality" means every active
department in the gateway's catalogue (municipal companies only when it lists them); narrowing is refused while another department's published assistant links
it. Topics are path-rule tags, optional on links and uploads. Teams are hidden in create forms: the owner team is the creator's first team in the department,
changeable under ⋯. The Map is a table in P1b. Detail: `admin/design.md`
§3.2, §5.5.6.

*From:* log rows 62, 73, 86, 101; decided by user, 2026-11-05.
*Also:* "M-2 open design calls", "Organisation: department-owned, municipality-shared knowledge bases with topics"

### Moving to another department

A platform admin can change the owner department of a knowledge base or an assistant, to correct one created under the
wrong department; department admins cannot. The move keeps every link that works today working: it shares the
knowledge base with each active department whose assistants would otherwise lose it (the old department when a
knowledge base moves; the new one, for each linked knowledge base it cannot reach, when an assistant moves), and those
shares stay until someone removes them under Sharing (V35 still guards that). A link that doesn't work today stays
broken, and client keys of other departments scoped to the object keep their scope; the admin sees both before
confirming. The moved object takes an owner team of the new department: the one the admin
picks, which is required when the department has several active teams, else its only team (none when it has no team).
It takes a gateway key of the new department, or none (V39 unchanged: a published assistant needs a key once the gateway
acts as keys). A knowledge base whose sources use
a pipeline of the old department is not moved until they use a platform pipeline or one of the new department. History
stays where it was: cost and usage rows keep the department of their calls, and audit entries their department; the
move itself is audited under both. The admin sees what the move changes and confirms it. Detail: `admin/design.md`
§3.2, §4.5.

*From:* decided by user, 2026-11-17 (#811): platform admins only, shares keep the links; the owner-team rule from
the main session's review on #811.

### Deleting knowledge bases, sources and assistants

A knowledge base, a source and an assistant can each be deleted in the admin UI, and a deleted one's name is free at
once: a slug is unique among the objects that are not deleted.

- **An assistant** is deleted directly (`Manage`), from any state, after a confirmation that lists the apps it is
  offered to and its recent use and asks for its name. It is a soft delete: no client lists or reaches it from that
  moment, as with archive, and it does not come back. Its conversations and feedback stay until their own retention
  removes them, and its statistics for their own time; where the assistant's row is gone they read *Deleted
  assistant*. The row, its versions and audiences are purged 30 days later, once no conversation of it remains.
- **A knowledge base** is deleted when no assistant that is active or archived links it (published version or draft;
  a deleted assistant does not block) and it has no source that is not deleted. **A source** is deleted at any time.
  Either way the documents leave the index at once and are purged by the rules for removed documents (14 days) and
  their files by the blob purge delay.
- **Nothing is restored** in the UI. A delete is in the audit log with the object's last values.

Archive stays, for an assistant that may come back.

*From:* #878; decided by user, 2026-11-20 (six questions with four alternatives each).
*Also:* replaces "a slug is never reused" (the unique index over deleted rows too).

### Level change in the edit drawer

A level is changed in the item's edit drawer under `edit`; the API's checks run as before and their refusals show
after Save, naming the affected objects. Detail: `admin/design.md` §5.5.2, §5.5.4.

*From:* log row 112; decided by user, 2026-11-07.

### Model pickers show what the key may use

Model pickers mark each mirror model available or not from the gateway's `/v1/models`, asked with the service's
credential key at the relevant ceiling and, for an assistant or knowledge base with a gateway key, as that key
(`x-fors-on-behalf-of`, once the gateway's `features` contain `act-as-key`), cached 5 minutes per ceiling and key. A model
the key may not use is greyed out and cannot be chosen; if the gateway cannot be asked, all stay choosable with a hint.
V8 (publish), V32 (save) and V39 (the object's key) stay authoritative. Detail: `admin/design.md` §4.5, §4.6, §4.10.

*From:* log rows 51, 113; decided by user, 2026-11-07; the object's key decided by user, 2026-11-11 (#580).

### Upload knowledge bases

A knowledge base of uploaded files is a first-class case in the admin UI: one step creates the knowledge base with its
upload source, and its **Files** tab has the drop zone, live progress, search, state filter, sort, folder grouping,
*Retry failed* and file details. Folder paths are kept: `X-File-Name` carries the relative path, which becomes the
display name, so path rules can set topics per folder. Detail: `admin/design.md` §4.5, §5.5.7.

*From:* decided by user, 2026-11-09.

## Skills, tools and MCP (P2)

### Skills

Skills come in P2; P1 assistants are instructions plus collections. A skill is know-how and tools in the Agent Skills
format (no scripts), with one served version for every linking assistant, an advisory gate, sharing with named
departments and a department grant per tool link; the platform team owns platform skills. Detail: `tools/design.md`
§1, §3.

*From:* log rows 18, 28, 48, 49, 55; decided by user, 2026-10-30.

### Tools and MCP

Tools are read-only first; write tools later always need the user's confirmation. Policy is deny by default, a run's
level is fixed at its start, and a composition that would combine private data, third-party text and an outbound
channel is refused. Tool definitions are pinned with drift checks: a pre-registered change is pinned, any other needs
a two-person re-approval. Agent limits per user message are 8 calls, 60 s, $0.05 (caps 20, 120 s, $0.25). Tool calls
raise security events 5012–5015. The platform serves `/mcp`, calls MCP tools through an in-process hub, and hosts .NET municipal servers (hosted only, the first `ume-mcp-kommun`); an MCP
client's ceiling is `public`, raisable to `internal`, never `sensitive`. Detail: `tools/design.md` §1, §5–§8.

*From:* log rows 19, 20, 26, 48, 49, 55; decided by user, 2026-10-30.

### Ask-a-file workspaces

In P2, knowledge bases from uploaded files, owned by teams and individuals, are a light kind of collection on the
whole pipeline, managed in a self-service area open to all staff (`Kind = workspace`, not Open WebUI's built-in Knowledge). Each
workspace appears to its members as a model in Open WebUI, with whole-document tasks from the start: per-document
summaries made on-prem at indexing, and full-text mode when a workspace fits the context. Detail:
`archive/area-plans/workspaces.md` (plan only).

*From:* log row 59; decided by user, 2026-10-30.

## Not carried over

Rows of the log with nothing left that binds:

- row 3 "Learning": replaced by the simplification pass; what holds is in "Site profiles".
- row 16 "Plans": the first plan layout; replaced by "Design documents: one layer".
