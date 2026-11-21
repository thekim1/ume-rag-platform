# Roadmap

Where Källa stands and what comes next: **the one place for the plan ahead.** The binding rules are in
[`decisions.md`](decisions.md) and the design documents ([`README.md`](README.md)); the work items are in
[`work/`](work/), and `HANDOVER.md` links here instead of restating it. The main session updates this page when a step
lands or an input arrives.

## Done

| Release | Result |
|---|---|
| `v0.1.0` (2026-10-31) | **P0 foundation:** repo, AppHost, schema, Postgres image with pgBackRest, Keycloak, FakeGateway, parser contract, CI; deploys from a digest-pinned bundle, restore drilled locally ([`archive/p0/`](archive/p0/README.md)) |
| `v0.2.0` (2026-11-08) | **P1a + P1b code:** crawl, upload, parse, embed, hybrid retrieval with citations, chat API behind Open WebUI, admin UI for the pilot journey, evaluation, publish and rollback, JavaScript rendering, optional rerank. Demo 2026-11-01 on a 200-page umea.se slice ([`archive/p1/`](archive/p1/README.md), [`archive/p1b/`](archive/p1b/README.md)) |
| `v0.3.0` (2026-11-10) | Deployment options matching the gateway's, upload knowledge bases, legacy Office formats, the #499–#513 review fixes |
| `v0.4.0` (2026-11-11) | Answer length from the model, editable embedding profiles, prompt texts as settings, keyword statistics in the worker, lower idle cost |
| `v0.5.0` (2026-11-13) | **Gateway integration phases 1–3** (#580): one credential key per service, a gateway key per assistant and knowledge base that every call runs as, organisation and models from the gateway's context (no Data API), **cost attribution** stages 1 and 2 (built; stage 2's live done-when waits on step 0 below) with the gateway's cost and budgets beside Källa's, refusals in chat and admin, content retention, malware scan optional |
| `v0.5.1` (2026-11-13) | Easier deploys with Portainer: the gateway and Källa as one stack with every image pinned, Open WebUI in it, a variables file, a checklist of what is still missing, gateway keys that apply without restarts |
| `v0.6.0` (2026-11-15) | A deploy that needs only what a process can't start without (M-17): the gateway creates Källa's credential keys, the rest comes from the gateway, the admin UI or the stack; apps and keys connected in one go (M-16); the admin UI review (M-19); the fixes of the 2026-11-14 review (#694) |
| `v0.7.0` (2026-11-16) | **The setup wizard** (M-20): sign-in set in the browser with a setup code and a test sign-in; a new Portainer stack needs one typed value, `FORS_DOMAIN`, and no secret, with the gateway's wizard first (gateway `v0.11.0`); retired deployment variables named (M-21); three hot paths of #764 |
| `v0.8.0` (2026-11-19) | **The deploy simplification** (M-22): one service file and one bootstrap for Compose, Portainer and Kubernetes, upgrades in one step from 0.5.x, one escrow set, Open WebUI off unless asked for; continue a crawl run and its live progress; the images' security updates and a daily image scan (A-27, #818) |
| `v0.9.0` (2026-11-21) | **Källa and Fors** (#899): the rename from Ume.Rag, licence Apache-2.0, on Fors `v0.12.1`, with an upgrade path for Compose, Portainer and Kubernetes; delete knowledge bases, sources and assistants, with what they held purged (#878); source dates, escalation and topics in the chat API (F4, F3, F8); the images' security updates from the first scan of 0.8.0 |

What is left before go-live is live work, and every step of it waits on an input from people. **P2 code is built
beside it since `v0.9.0`** (user, 2026-11-20; Decisions "P2 starts before go-live"): *P2 Broaden* below.

## Now: the test chatbot (before the umea.se pilot)

Decided by the user, 2026-11-13 (Decisions "Test chatbot before the umea.se pilot"): a test chatbot on **DoIT's support
documents**, uploaded, for a **small test group** in Open WebUI, on the user's own deployment of Källa and the
gateway (the VM, on `v0.5.0` today). It shows how well uploaded documents answer real questions before the umea.se pilot. Case plan:
[`cases/support-assistant/`](cases/support-assistant/README.md) (`07-rollout.md` §0b and §1).
**The user's team sets up and runs the test chatbot themselves** (user, 2026-11-14): the agents don't plan or prepare
T2–T6; they fix what the team reports and answer questions.

| # | Step | Input it waits on | Owner | Status |
|---|---|---|---|---|
| T0 | The gateway with Källa's integration and its three credential keys (`rag-api`; `rag-admin` with `organisation.read`, `usage.read`; `rag-worker` capped at `OnPrem`+`Eu`), the EU chat and embedding models, DoIT's department and team, and an assigned key for the assistant and the knowledge base (`docs/production-setup.md` *Gateway credential keys*) | the gateway deploy | user | open |
| T1 | Källa `v0.5.0` deployed as one Portainer stack with the gateway (`docs/portainer.md`; Keycloak realm, Open WebUI, ClamAV on), checked with its read-backs. It moves to `v0.9.0` on Fors `v0.12.1` in one update (`docs/portainer.md`, *Upgrade*) | T0 | user | **deployed** (user, 2026-11-13: "0.5.0 deployed fine", with the Portainer stack and the `openwebui` and `clamav` profiles); read-backs to confirm |
| T2 | The test group's accounts (Keycloak or the gateway's sign-in) and their Open WebUI access; DoIT's admin in the admin UI | T1 | user | open |
| T3 | DoIT's documents: the source inventory, then an upload knowledge base `support` (`internal`) with preview, first sync, run reports and the personal-data report (`02-knowledge-base.md` §5) | DoIT's files | user | open |
| T4 | Assistant `support`: the instructions of `04-assistant.md` §3, the knowledge base's key (V39), general knowledge `fallback`, a not-found text with the real escalation links; checked in the test panel with about twenty anonymised or invented tickets | T3 | user | open |
| T5 | The test group uses it in Open WebUI; feedback import on | T2, T4 | user | open |
| T6 | Findings back: answer quality per ticket, misses and their documents, upload gaps (U1–U7) seen on real files, the condense model, the live checks of the gateway integration (B-42–B-46, A-34, A-35) | T5 | user | open |

**Open question (user, with the data protection officer):** whether the test group's use needs the DPIA (G1), or part of
it, before T5: the documents are internal, Open WebUI keeps its own copy of each chat (W2), and real tickets wait for G1
in the case plan.

## Then: the umea.se pilot

**Go-live** (P1b done): the gate is met on hand-graded gold-set answers (correct ≥ 80 %, citation precision ≥ 90 %,
false not-found ≤ 10 %, off-topic answered "not found" ≥ 80 %), the DPIA (G1) is finished with W2 inside it, and one
pilot department uses its assistant in Open WebUI. The steps, in order; a step starts when its input exists.

| # | Step | Input it waits on | Owner | Status |
|---|---|---|---|---|
| 0 | The gateway and Källa as deployed for the test chatbot (T0, T1), or the test host's own | T0–T1 | user | after the test chatbot |
| 1 | Gateway and Keycloak for the pilot: pilot accounts, the key that allows the model `turkos` is published on | step 0 | user | open |
| 2 | U1 with the web editors, then the umea.se crawl in full and the `lastmod` sample's first pass (A-L1, A-L4) | U1 page URL, owner, contact | user + web editors | open |
| 3 | Gold set written with DoIT, entered, run three ways (`scripts/grading/`); not-found thresholds set from it | the gold set and its reviewers | user + DoIT | open |
| 4 | Test host: deploy from the release bundle, restore drill, R1 performance test, a week of nightly evaluation (A-L5–A-L7) | test host, backup target, key-escrow location, runner (R1) | user, IT security | open |
| 5 | Checks on the crawled collection: `lastmod` result, freshness and HNSW recall, personal-data dry run (A-L8, A-L9) | steps 2, 4 | stream a | open |
| 6 | Live feedback import, banner and public-records route | G1 | user + data protection officer | open |
| 7 | Before go-live: the 7-day package rule applies again; GitHub Actions pinned by SHA with automated bumps (A-20, A-27); `release.yml` and `entra.yml` on a runner of their own under a separate user (#704, Decisions "CI on the self-hosted runner"); LibreOffice placement with the parser sandbox question (Decisions "Legacy Office and RTF uploads") | – | streams, main | postponed on purpose (2026-11-08) |
| 8 | The gate graded by department reviewers; the pilot goes live (M-4) | steps 1–7 | user + main | open |

**Owners and dates** (umea.se pilot). The phase-start rule (`postponed-decisions.md`) asks for a named person and a date on every
input that blocks a phase. Still unnamed: A1, G1, U1, W2, R1 and the test host. Until the user names them, the
table says "user".

## Meanwhile: work that needs no input

P2 wave 1 (*P2 Broaden* below) is the code work. Beside it, what is open without an input:

- **Checks before go-live:** done 2026-11-16 (results in [`work/results-main.md`](work/results-main.md)) but for
  row 96, the NVDA pass, which needs a Windows screen reader. The question to KB (row 110) is the user's call. The
  live block's rows are listed in [`to-verify.md`](to-verify.md) under each step.
- **Step 3 starts on working tools:** the grading tools ran end to end on a FakeGateway stack (2026-11-15: `answer` →
  graded CSV → `totals`, which exits 1 on a failing gate and 2 on an inconsistent file). The pilot assistant needs a
  description, a contact and a gateway key of its department that reaches both the chat model and the collection's
  embedding model at the assistant's ceiling.
- **Upload gaps** ([`proposals/upload-quality-review.md`](proposals/upload-quality-review.md), U1–U7): measured on 14
  public IT guides (2026-11-15). U1 is the largest loss: 18 % of the text is inside screenshots (19–76 % in
  step-by-step guides), none of it kept. U5 and U7 cost little. The same script (`scripts/upload-gaps/measure.py`)
  runs on DoIT's files at T6.
- **Not tried anywhere yet:** Portainer's own *Update the stack* on an upgraded stack (the VM's move to `v0.9.0` is
  the first), and both setup wizards completed against a real provider in one stack. CI boots and upgrades the stack
  with Compose.
- **CI:** what the test collections of one `Kalla.AdminApi.Tests` process wait for (not CPU, thread pool, disk or
  GC; a profiler would say); the project runs as five processes meanwhile.

## Waiting on people

The inputs of the critical path are in its table. The other open calls:

| What | Who | Blocks |
|---|---|---|
| W1 relation to `chat.umea.se` | user, product owner | go-live |
| Upload gaps U1–U7: suggested picture descriptions (U1, needs a decision entry) and rerank on (U4) first | user | nothing in P1b |
| Ask-a-file workspaces (P2): owned by teams and individuals, against "Organisation" (Decisions "Ask-a-file workspaces") | user | P2 |
| Product calls: #333 finding 1 (`exactScanMaxChunks`, after the recall check on the test host); [`proposals/architecture-plans.md`](proposals/architecture-plans.md) items 1, 2, 4–7; the W3 grant step; a Git/Markdown connector | user | nothing in P1b |
| #511 splitting the worker into libraries | stream b | only if it keeps growing in P2 |
| Paths under one host name (`example.se/rag`), Decisions "Host names": not built | user | nothing; only if an organisation gets exactly one name |
| Fewer hand steps after an install: one wizard hand-off and a first assigned key from the manifest, proposed to the gateway (thekim1/fors#108, #109) | user (Fors) | nothing |

Every postponed decision with its stopgap and deadline: [`postponed-decisions.md`](postponed-decisions.md).

## P2 Broaden

Several departments, internal sources, tools, and the first multi-host deployment. **P2 code is built beside the
live block since `v0.9.0`** (Decisions "P2 starts before go-live"); go-live keeps its gate. P2 runs in waves, each a
set of queue items in `work/stream-a.md` and `work/stream-b.md`; the next wave is picked by the user when one is
nearly done.

**Wave 1: assistants and answer quality** (user, 2026-11-20). Designed; no input needed, built on FakeGateway:

| Item | What | Design |
|---|---|---|
| B-54 | Assistant templates: three seeded starting points, picked when an assistant is created | admin §4.5 (`/api/assistant-templates`), §5 step 9; data-model `AssistantTemplates`; architecture plan item 3 |
| B-55 | Copy an assistant | admin §4.5 (`POST /api/assistants/{id}/copy`) |
| B-56 | Version diff: **Compare** in Versions, the diff in `draft_base_changed` | admin §4.5 (`…/diff?against=`) |
| B-57 | Test questions per assistant: the set, the runner, the report, the run after publish | admin §7.4, V33; data-model `AssistantTestQuestions` |
| B-58 | The nightly answer run (`answers`): faithfulness and citation precision per assistant | processing §14.8; admin §7.2 |
| B-59 | Home's *Needs attention*: the attention items of every design that names one | admin §5.5.1 |

Each later item needs the earlier ones only where its row says so in the queue. The live check of wave 1 is the test
chatbot's assistant (T4) once it exists; until then FakeGateway and the demo seed.

**Later waves: themes that need no input** (the user picks the order):

- **Tools and MCP:** skills with know-how and tools, the in-process tool hub (`hosted` servers, read-only tools), the
  agent loop, `/mcp` with `search_knowledge`, the mock municipal server `ume-mcp-kommun` (tools design; the gateway
  passes function tools already). The SDK and validator re-checks
  of M1 and V1 come first (`postponed-decisions.md`).
- **Ingestion breadth:** e-mail (`.eml`/`.msg`) and archives (Z1), site-profile learning and re-learning,
  parse-profile proposals, a crawl frontier shared between sources, publish plans from effect tags, adaptive
  embedding concurrency; the renderer's replay check and path rules (after measurement) and, for external
  single-page apps, a per-page burst of the page's own subrequests and an asset cache kept across runs (A-13).
- **Assistants, the rest:** the drawn Map, unused-assistant attention items, the `TimeSeriesChart` trend.
- **Chat:** a public widget.
- **Operations:** flake-hunt, scale and fuzz workflows.

**Themes that wait on people** (each with its input):

| Theme | Waits on |
|---|---|
| A second department or a group audience: gateway release 2 (member groups from claims), cost attribution stage 2's done-when ([`platform/cost-attribution.md`](platform/cost-attribution.md) §6), owner-team edit scoping; one Open WebUI per department until W3 and E1 are solved | the gateway team; W3's grant step |
| Entra ID with Källa's own registrations, OBO and the E1 spike | a tenant and its admin |
| SharePoint | tenant access and an internal-content pilot (architecture plan item 9) |
| Stored chat content (history modes, content encryption, conversation review), chat attachments | A1, G1 |
| Ask-a-file workspaces | the user's calls on ownership (Decisions "Ask-a-file workspaces"); W3's grant step |
| Multi-host topology (leader leases, Redis, batched dequeue, a/b slots), Postgres standby (D1), object store (S2) | IT, a second host |
| `halfvec` embeddings (#502; to-verify #117 found no recall cost), an on-prem reranker behind the gateway's `/v1/rerank`, on-prem models | the full crawl's evaluation set; M2 hardware |

## Later: P3 Scale and refine

Driven by measurements: a Swedish retrieval benchmark (B1), OpenSearch if a trigger fires (O1), a separate
`kalla-tools` service, municipal and external tool hosting, per-skill MCP endpoints, Teams client, Entra OBO for
user-delegated tools, write tools with confirmation, a visual pipeline builder if its trigger fires (C2).

## Reference that still binds

- The capacity model, scaling triggers and scale tests of the
  [performance pass](archive/passes/performance-pass.md): §2 (capacity model), the sizing tables of §3.1–§3.3 and
  §3.9, §6 (scaling triggers) and §7 (load and scale tests), as its header lists.
- The ledgers of the archived passes say where each of their rules went; the rules themselves now live in the design
  documents.
