# Källa

RAG, crawling and chat platform for Umeå kommun: small .NET 10 services on top of Fors, the team's LLM gateway
([thekim1/fors](https://github.com/thekim1/fors)), a Python Docling parser and PostgreSQL with pgvector,
orchestrated with Aspire 13.6. The design lives beside the code: start at [`AGENTS.md`](AGENTS.md) and
[`HANDOVER.md`](HANDOVER.md).

**Status: P1 (pilot), built and released; the live block is ahead.** The last release is
[`v0.9.0`](https://github.com/thekim1/kalla/releases/tag/v0.9.0) (2026-11-21), on Fors `v0.12.1`;
[`CHANGELOG.md`](CHANGELOG.md) has every release. P1 was built on FakeGateway, fixtures and the dev Keycloak first; the
live parts (the real gateway, the shared Keycloak, the umea.se crawl, the gold set, the pilot member, the test host)
come as one block at the end. Built:
- ingestion end to end: URL lists, crawls (one frontier per source, robots, sitemaps, freshness and removal checks,
  JavaScript pages through the renderer) and uploads (ClamAV, fail closed, or off by the deployment's choice) through
  parse, clean, chunk, personal data, embed and the index writer;
- the parser for PDF, Office/ODF, images and scans (Docling, OCR), the Sitevision preset and the umea.se profile;
- the engine: runs and finalizer, scheduler, drift sweep, `reprocess`/`retry-failed` with an optional cache bypass,
  source settings-change effects, preview and preflight, publish with per-document swap and rollback within 14 days;
- evaluation: on-demand three-way runs, publish reports, nightly runs with the recall@10 alert, generated question sets;
- retrieval (hybrid Swedish keyword + vector search, optional rerank), the knowledge API and the chat API
  (OpenAI-compatible, for Open WebUI and other clients);
- the admin UI: knowledge bases, sources, assistants, apps and keys, costs, settings, and the setup wizard for sign-in;
- deployment: production Compose, one Portainer stack together with the gateway, Kubernetes manifests, backups,
  point-in-time restore and key escrow.

The plan ahead: [`plans/roadmap.md`](plans/roadmap.md). Overview for stakeholders (Swedish):
[`docs/phase1-guide.html`](docs/phase1-guide.html).

## Requirements

.NET SDK 10.0.1xx or later (CI uses 10.0.401), Docker 28+, Node 22 or later (admin UI; CI uses 24) and uv (parser).

## Build and test

```bash
dotnet build Kalla.slnx -c Release --nologo
dotnet test --solution Kalla.slnx -c Release --no-build --filter-not-trait "Category=Flaky"
```

`--filter-not-trait` skips tests marked `[Trait(TestCategory.Name, TestCategory.Flaky)]`, as CI does. The integration
tests start the pinned pgvector image with Testcontainers, so Docker must be running. `KALLA_TEST_POSTGRES_IMAGE`
runs them against another image (CI: the built `kalla/postgres`).

The container tests skip unless their image is named. Build the images locally (Testcontainers wants a dot in a local
registry name): the worker's tests use ClamAV and the renderer, the contract tests the parser.

```bash
docker build -t kalla.local/clamav:dev images/clamav
docker build -t kalla.local/renderer:dev images/renderer
KALLA_TEST_CLAMAV_IMAGE=kalla.local/clamav:dev KALLA_TEST_RENDERER_IMAGE=kalla.local/renderer:dev \
  dotnet test --project tests/Kalla.Worker.Tests -c Release --no-build
docker build -t kalla.local/parser:ci src/parser
KALLA_TEST_PARSER_IMAGE=kalla.local/parser:ci \
  dotnet test --project tests/Kalla.ContractTests -c Release --no-build
```

The pre-commit hook runs the admin UI lint and the parser's ruff check on staged changes. Enable it once per clone:

```bash
git config core.hooksPath .githooks
```

## Run locally

```bash
aspire run      # or: dotnet run --project src/Kalla.AppHost
```

The Aspire dashboard opens at `https://localhost:17470`. The AppHost starts postgres (stock pgvector image), keycloak,
the fake gateway, the parser, ClamAV and the renderer (built from their Dockerfiles), the migration service, api,
worker, adminapi, the admin UI (`https://localhost:5174`) and Open WebUI (`http://localhost:7410`). Its switches
(`Parser:Mode`, `ClamAv:Mode`, `Render:Mode`, `OpenWebUi:Enabled`, `FakeGateway:Enabled`, `Ollama:Enabled`,
`Redis:Enabled`, `Oidc:Authority`, `Setup:Try`) are in [`docs/local-development.md`](docs/local-development.md). Set
them as AppHost user secrets:

```bash
dotnet user-secrets --project src/Kalla.AppHost set Parser:Mode off
```

To run against the interim gateway instead of FakeGateway, see [`docs/p1a-local-run.md`](docs/p1a-local-run.md).
Addresses and keys go only in user secrets or secret files, never in the repository.

On Linux or WSL, `aspire start --apphost src/Kalla.AppHost/Kalla.AppHost.csproj` runs the same stack in the
background (`aspire describe`, `aspire logs <resource>`, `aspire stop`); `Seed__Enabled=false` in the environment starts
it without demo data.

Container health probe: every .NET service image runs its own probe with `--health-check` (exit 0 healthy, 1 not),
against `FORS_HEALTHCHECK_URL` (default `https://localhost:8443/health/ready`).

## Docs

- [`docs/introduction.md`](docs/introduction.md) and [`docs/how-it-works.md`](docs/how-it-works.md): what Källa is and how its parts work together.
- [`docs/local-development.md`](docs/local-development.md): the dev stack in detail and the start scripts.
- [`docs/p1a-local-run.md`](docs/p1a-local-run.md): the local P1a run, against FakeGateway now and the interim gateway later.
- [`docs/compose-quick-start.md`](docs/compose-quick-start.md): the gateway and Källa on one host with Docker Compose, the short path.
- [`docs/production-setup.md`](docs/production-setup.md): the Compose reference: host sizing, `init-deployment.sh`, sign-in, backups, Open WebUI.
- [`docs/portainer.md`](docs/portainer.md): the gateway and Källa as one Portainer stack, a first deploy and each upgrade.
- [`docs/kubernetes.md`](docs/kubernetes.md): the same services as Kustomize manifests.
- [`docs/runbook.md`](docs/runbook.md): backups, restore, key escrow, certificates, operations.
- [`docs/network-and-ports.md`](docs/network-and-ports.md): listeners, outbound destinations and firewall/proxy rules per setup.
- [`docs/test-host.md`](docs/test-host.md): the pilot test host, its caps, backups and the restore drill.
- [`docs/keycloak.md`](docs/keycloak.md): Keycloak as the sign-in provider.
- [`docs/security-events.md`](docs/security-events.md): the security event ids.
- [`docs/upgrade-guide.md`](docs/upgrade-guide.md) and [`docs/zero-downtime-updates.md`](docs/zero-downtime-updates.md): upgrading between releases, and which releases update without a stop.
- [`docs/ci-runner.md`](docs/ci-runner.md): the self-hosted runners and working with the workflows.
- [`docs/accessibility-statement.md`](docs/accessibility-statement.md): the admin UI's accessibility statement.
- [`docs/phase1-guide.html`](docs/phase1-guide.html) (Swedish) and [`docs/stakeholder-report.html`](docs/stakeholder-report.html): overviews for stakeholders.

## CI

`verify.yml` runs on every pull request that is not a draft (on open, each push, and when it leaves draft) and by
hand, not on a push to `main`. Its jobs are filtered by the paths a pull request changes:
- `ownership` (stream paths) and `doc-refs` (links and section references in the documents);
- `backend` (build and tests) and `evaluation-smoke`;
- `images`, `renderer` and `deploy-checks` (Compose files, `init-deployment.sh`, the host scripts, the release bundle);
- `admin-ui`.

`parser.yml` runs the `parser`, `parser-image` and `contract` jobs only when the parser or its contract changed.
`audit.yml` checks the dependencies of `main` for known vulnerabilities every night, the one workflow on a schedule.
`release.yml` builds, tests, pins and publishes a release on a `v*.*.*` tag (dispatch: a dry run that pushes nothing).
`e2e.yml` runs the browser tests, by hand or from `release.yml` when asked. `perf.yml` (the R1 release test) and
`evaluation.yml` (the evaluation on the test deployment) run by hand; their schedules are off.
`entra.yml` plans the Entra registrations. `pages.yml` publishes the website (`site/`, with `docs/` and the changelog)
on a push to `main` that changes them.

**CI runs on self-hosted runners** (Decisions "CI on the self-hosted runner";
[`docs/ci-runner.md`](docs/ci-runner.md)): GitHub-hosted minutes are used up, so nothing runs on GitHub's runners.
**`main` is protected:** a change goes in through a pull request with one approving review and green required checks
(`ownership`, `doc-refs`, `backend`, `admin-ui`, `deploy-checks`, `images`, `renderer`, `evaluation-smoke`; a job
that the path filter skips counts as passed).

Before a pull request, run the test projects the change touches (`AGENTS.md`, *Tests*) and the ownership check:

```bash
git diff --name-only origin/main...HEAD | python3 -I scripts/check-ownership.py <branch>
```

## Packages

Every package version is pinned exactly and images are pinned by digest (`AGENTS.md`). The 7-day rule (a version at
least 7 days old when pinned) is suspended during development and applies again before the first production
deployment, when every younger pin is reviewed (Decisions "7-day rule in development").

## Licence

Copyright © 2026 Kim Vallén and Nichlas Löfdahl. Licensed under the Apache License, Version 2.0 (Apache-2.0), see
[`LICENSE`](LICENSE) and [`NOTICE`](NOTICE). Källa is built by two private developers for Swedish municipalities; the
examples use Umeå kommun, where we work by day. Releases up to and including 0.8.0 (as Ume.Rag) were published under the
European Union Public Licence v. 1.2 (EUPL-1.2). Third-party components keep their own licences.
