# Main session handbook

How the main session works, the local stack, and the lessons of earlier sessions. Stable: it changes when something is
learned, not every session (the session's state is in [`../../HANDOVER.md`](../../HANDOVER.md), the plan ahead in
[`../roadmap.md`](../roadmap.md)). Stream agents' gotchas are in
[`../../docs/local-development.md`](../../docs/local-development.md) (*Working notes*) and
[`../../docs/ci-runner.md`](../../docs/ci-runner.md) (*Working with the workflows*).

## How we work (user preferences, binding)

- **Good and easy to use:** look at pages in the browser, not only tests; fix friction or file it.
- **Demo first; admin UI focus.** No extra negative proofs or flake hunts unless a change needs one. No tests that
  depend on timing ("wait until X, fail after 60 s" is fine).
- **Every change is a pull request** (ruleset "Protect main", 2026-11-16): one approving review and the required
  checks. The main session owns `plans/**`, root `*.md` and `.github/stream-ownership.txt` and changes them on a
  `main-session/<slug>` branch; a design line that belongs to a code change goes into that pull request. Stream-a
  work is on `a/<id>-<slug>` branches; a pull request that touches another stream's paths uses `main-session/`.
- **Who approves and merges.** The session approves nothing on this machine: the permission check stops
  `gh pr review --approve` as `ume-nichlas` and as `Crotalus`, also with the user's go-ahead. Review as a comment,
  say that it is not the approval, and give the user the line to run
  (`gh auth switch -u Crotalus && gh pr review <n> --approve && gh auth switch -u ume-nichlas`). A pull request goes
  in on the user's approval, on `thekim1`'s approval, or by `thekim1`'s `--admin` on a green one (allowed by the user,
  2026-11-19). Never `gh pr merge --admin` as `ume-nichlas`. **Merge an approved, green pull request without
  asking** and tell the user afterwards. Design, scope and destructive actions go to the user.
- **A decision for the user** is asked as four alternatives with one recommendation (the question tool).
- **Handle GitHub on your own:** answer new or updated issues and pull requests; close verified-done issues with an
  evidence comment.
- **Two machines.** `thekim1`'s runs stream b, often stream a, and a coordinating session of its own; talk only
  through GitHub (labels `stream-b`, `stream-a`, `main`). It merges its own pull requests; review their
  security-relevant parts. Before taking stream b's paths, look for open `b/…` work and say so in an issue.
  **Look at the open pull requests before each step** (`gh pr list`), and say on the other machine's pull request
  what you are about to write. `gh` here acts as `ume-nichlas`: run `gh auth status` before every push or post.
- **Lean CI:** pull-request jobs are path-filtered; the full set runs before a release, plus a release dry run;
  `e2e` runs alone on the runners; agents run only the test projects they touch; no re-dispatching `verify` on
  `main`. **Fix a failing release check in isolation**, not with another 30-minute dry run (user, 2026-11-15).
- **Subagents:** own worktree (`../rag-<id>`), full brief, no touching the live stack unless the brief says so. Their
  report is a claim: read the diff, re-run the tests, check CI's test totals, check live, spot-check line numbers.
- **Shared files** (`Directory.Packages.props`, `README.md`, `Kalla.slnx`, `.gitignore`): small pull request,
  append-only for the streams.
- **Release notes are files, not edits** (user, 2026-11-17): a pull request adds `changes/<branch>.md` and, when an
  operator has something to do, `changes/<branch>.upgrade.md`; `scripts/unreleased.py` checks and folds them.
- **Secrets never go into the repo, a log or a chat.** Verify claims against real sources.
- **No backward compatibility before production**, but an upgrade path for deployments where one is feasible
  (`AGENTS.md`, *Conventions*).

## Local stack (this machine)

- **State (2026-11-21):** the dev AppHost is stopped and has not run since the rename. Its data is still under the
  old names: user secrets under the id `ume-rag-apphost-dev` (the AppHost now reads `kalla-apphost-dev`, which is
  empty), volumes `ume-rag-m1-postgres` (named by the old secret `DevelopmentVolumes:Postgres`),
  `ume-rag-openwebui-data` and `ume-rag-clamav-db` (the AppHost now mounts `kalla-openwebui-data` and
  `kalla-clamav-db`), helper scripts in `~/ume-rag-tools/`. Moving them over is the user's call. The old secrets
  also hold a `Gateway:Keys` section, `Parameters:gateway-key-{eu,onprem,external}` and `Parameters:data-api-key`,
  which the AppHost refuses or no longer reads (`docs/local-development.md` lists the current parameters).
- **What the development data holds** (department DoIT; in `ume-rag-m1-postgres`): the collections `turkos` (1,166
  documents, about 10,000 chunks, a 100-question evaluation set), `renderer-demo`, `a13-klimatkollen`, `a19-e-umea`
  and `Personalhandbok` (an upload source made as test data); the assistant `turkos`; the embedding profile
  `pilot-embedding`; crawl `hostOverrides` for `www.umea.se` only (2 req/s, 2 connections).
- **A stack on the FakeGateway needs no secrets and no dev data:** `export FakeGateway__Enabled=true
  Seed__Enabled=false Parser__Mode=off Render__Mode=off OpenWebUi__Enabled=false ClamAv__Mode=off
  DevelopmentVolumes__Postgres=<a throwaway volume name>`, then start; afterwards stop and `docker volume rm` it.
- **Start:** `aspire start --apphost src/Kalla.AppHost/Kalla.AppHost.csproj --non-interactive` (Docker up:
  `docker info`). **Stop:** `aspire stop --all --non-interactive`; it returns before the persistent containers have
  stopped, so wait a few seconds before `docker volume rm`. One service: `aspire resource worker restart --apphost …`.
  Logs and states: `aspire logs worker`, `aspire describe --apphost …`. The CLI is at
  `~/.nuget/packages/aspire.cli.linux-x64/13.6.1/tools/net10.0/linux-x64/aspire` (not on PATH; it starts the 13.6.5
  AppHost fine).
- **Browser tests locally without pwsh:** stop the stack; `docker volume rm kalla-e2e-postgres`; export
  `DevelopmentVolumes__Postgres=kalla-e2e-postgres Seed__Enabled=true FakeGateway__Enabled=true E2E__Enabled=true`;
  start; `aspire wait admin-ui --apphost …`; in `tests/e2e`: `npm ci`, `npx playwright install chromium`,
  `npx playwright test` (about 10 minutes).
- **Ports:** dashboard `https://localhost:17470`; admin UI `https://localhost:5174` (user `rag-platform-admin`); api
  7401, admin API 7402, worker 7403; Open WebUI `http://localhost:7410`, Keycloak 7411, Postgres 7412, parser 7413,
  ClamAV 7414, renderer 7417. Platform pages are under `/plattform/<resource>`.
- **The parser** goes cold after 15 idle minutes; `PARSER_IDLE_STOP_S=0` and `NER_IDLE_STOP_S=0` keep it warm.
- **Scripting the admin API:** `~/ume-rag-tools/` (outside the repo, no secrets in it): `admin.py` (BFF login from
  the AppHost user secret, XSRF), `pf.py`, `runwait.py`, `chat.py`; run them from that folder. Upload slots are
  numbered from 1 (`PUT /api/upload-batches/{id}/files/1`) and the commit takes the slot numbers; an empty commit
  answers `upload_batch_too_large`. A published assistant needs a description, a contact and a gateway key (V39)
  whose residencies reach the collection's embedding model too.
- **Postgres:** `docker exec -e PGPASSWORD="$(docker exec <postgres-…> printenv POSTGRES_PASSWORD)" <postgres-…> psql
  -h localhost -U postgres -d ragdb` (schemas `config`, `content`, `chat`).
- **UI checks on realistic data:** copy the dev volume to a throwaway one (`docker run --rm -v <dev>:/from:ro -v
  <copy>:/to alpine cp -a /from/. /to/`), start the FakeGateway stack on it, and remove it afterwards.
- **Fors:** `~/git/llmgateway` is a clone of `thekim1/fors` on a merged feature branch, fetched through `v0.12.1`:
  read a release with `git show v0.12.1:<path>`. Where its documents and its code differ, follow the code.

## Lessons

### Pull requests and GitHub

- **Stacked pull requests:** merging the base with `--delete-branch` *closes* the stacked one instead of retargeting
  it (#579, #732). Before a merge run `gh pr list --base <branch-being-merged>` and retarget each hit. Better: branch
  dependent work from the other branch but target `main`, and rebase after the first is squash-merged.
- **`gh pr merge --delete-branch` also removes the branch's local worktree.** Stop any stack running from it first,
  and merge without `--delete-branch` while an agent still reads it.
- **Squash-merged branches look unmerged to git:** check the pull request's state and `git rev-list --count
  main..<branch>` before `git branch -D`.
- **Read a watcher's result before merging:** `gh run watch … --exit-status; echo exit=$?` printed `exit=1` and the
  merge went ahead anyway (#582). In a polling loop test for an empty `conclusion`, not for `status != COMPLETED`.
- **The permission check** also stops `commit --amend` with a force push (push a follow-up commit) and a force push
  to another session's branch (push a plain merge commit, or `gh pr update-branch <n>`).
- **Switch the `gh` account in a command of its own:** chained with `&&` behind a command that failed it did not
  run, and what came after went out under the wrong account.
- **A branch that joins a rebuilt line to its old one loses every approval** (#906): the release preparation was
  rebuilt on the new `main` and merged with its first version, and GitHub dismissed two approvals within seconds
  with *The merge-base changed after approval*, with no push in between. The pull request's stored base
  (`gh api …/pulls/<n> --jq .base.sha`) was still the old `main`; that this is the cause is a reading, not checked.
  Rebuild such a branch as a new branch and a new pull request.
- **`gh pr edit` fails here** (GitHub's *Projects (classic)* error) and changes nothing: edit a title or body with
  `gh api -X PATCH repos/thekim1/kalla/pulls/<n> -f title=… -F body=@file`.
- **After a GitHub outage** runs can sit `queued` with jobs that have a start time but no runner: cancel and re-run
  them (the user allows cancelling stuck CI). A run that fails at an artifact upload with `Unexpected token '<'`:
  look at https://www.githubstatus.com first.

### CI and releases

- **A new GitHub advisory turns every backend job red** through `dotnet list package --vulnerable` (NU1903) with no
  code change: look at the audit step first when an unrelated pull request fails.
- **Run `e2e.yml` at least once between releases, and only with the runners idle.** Left alone for a week it drifted
  by seven UI changes and took nine rounds to clear; beside a pull request's `backend` job Chromium loses its page
  (`net::ERR_NETWORK_CHANGED`).
- **Six pull requests at once fill both runners for an hour:** open the small ones first, and dispatch a dry run only
  after the pull request's own checks are done. Under that load `backend` and vitest fail at random in untouched
  tests: run again alone before calling a failure real.
- **Test a stack-file or boot-test fix with the boot-only run** (`gh workflow run release.yml --ref <branch> -f
  portainer_boot=X.Y.Z`, about 2 minutes). It uses that version's images on ghcr.io, so a change that needs new
  images takes a dry run from the branch.
- **The pin pull request that `release.yml` opens starts no checks** (it is opened with the workflow's token): close
  and reopen it, wait for `deploy-checks`.
- **An upgrade's steps are not final until the upgrade test has run them** (0.9.0): in the joined Portainer stack
  `kalla-bootstrap` waits for the gateway's `migrations`, so whatever makes those fail keeps all of Källa from
  starting, `kalla-postgres` included. The upgrade as first written was a dead end that only the dry run showed.
- **A boot test's fixture is the earlier release's own example file with real-looking values:** 0.5.1's kalla-admin
  refuses a platform-admin group that doesn't start with `rag-` and restarts for ever.
- **Workflows:** a step's `if:` without a status function implies `success()`, so a step meant to run after a
  failure needs `!cancelled() &&` (#774). The runners have no `gh`: download a release asset with
  `scripts/ci/release-asset.sh`. Every workflow that pulls from ghcr.io sets its own `DOCKER_CONFIG`, because the
  runners share one user's `~/.docker/config.json` and a stale login there denies pulls (#572, #748).
- **Compose:** `up --wait` fails on a one-shot that exits 0 unless a service waits for it
  (`service_completed_successfully`); the boot test waits only on the long-running services (#751). An inline
  (`content:`) config can't go into a `read_only` service: copy the file into `kalla-config` from the bootstrap.
- **A stack-file refactor is shown by rendering, not by reading** (#800): render the committed and the regenerated
  `compose.portainer.yaml` with `docker compose config --format json` and compare; only the embedded bootstrap
  script may differ. `deploy/portainer/build-stack.sh` regenerates the stack; a change to
  `deploy/postgres/app-roles.sql` or the template leaves it stale until someone runs it.
- **Shell tests that pass here and fail on the runner:** `join` and `sort` follow the locale's collation (compare by
  exact name in `awk`, or set `LC_ALL=C`), and `producer | grep -m1` fails under `pipefail` when grep closes the
  pipe early (read the output into a variable first).
- **A slow test project: compare one process with two before adding collections** (#775): the collections of one
  `Kalla.AdminApi.Tests` process wait for one another, so CI runs it as five processes
  (`scripts/ci/backend-tests.sh`). Database classes go into one of `AdminDatabaseGroup1`..`4`, the shortest by
  measured time; the test executable's `-xml <file>` gives every test's time.

### Code and deployment

- **Design sections are never renumbered:** code and plans cite §-numbers (about 2,800 times in `src/` and
  `tests/`). A section whose content moves keeps its number and heading with a one-line pointer; a new section takes
  the next free number. Cite stable IDs (V39, a rule's name) where one exists.
- **`scripts/check-doc-refs.py`** resolves a bare `name.md §N` by file name: a new file with a common name captures
  old references, so give plan files distinctive names. A `.venv` left by `uv run` in `src/parser` breaks it.
- **A rename must not touch what is stored** (#902, thekim1/fors#116): Data Protection's application name and
  purposes, AES-GCM associated data, HMAC contexts, check-constraint values and applied migrations are format, not
  product text. Fresh-database tests cannot see the damage. Review a rename by grepping the old name in
  `CreateProtector`, `SetApplicationName`, `GetBytes(` and `Migrations/`.
- **SQL in `src/Kalla.Infrastructure/` uses positional parameters** (`$1`): the API's SQL contract prepares every
  string constant there and reads `@name` as a column (`SqlSchemaContractTests`; regenerate with
  `KALLA_UPDATE_CONTRACTS=1`).
- **New worker background services open connections on the worker role:** count them against data-model §1.1's
  budget; `DispatcherTests.Under_load_…_role_cap` catches an extra session.
- **`deploy/.env` holds every value in single quotes:** a script that reads a value with `sed` compares the quotes
  too (#905). Read a value the way `init-deployment.sh`'s `get` does.
- **What marks a copy as done is written last** (#905). Test a resumable step by making it fail part-way and running
  it again.
- **A pinned base image with no newer build:** pin the one fixed package in our own Dockerfile, with a comment that
  the line goes when the base has the fix. `docker buildx imagetools inspect` shows whether a tag was rebuilt.
- **`labels.ts` and `messages.ts` are stream b's paths:** a stream-a UI item that needs a text goes on a
  `main-session/` branch.
- **Gateway documents and code can differ** (403 documented where the code answers 401): check the behaviour Källa
  depends on in the code at the tag.

### Agents

- **Brief:** say which facts are assumptions (two wrong statements in a brief came back as built). For a review, give
  the code and the threat model, not your conclusions, and ask for producer and consumer read on both sides; check
  each finding in the files before acting on it. A second agent's reading finds what the first one built wrong.
- **Agent worktrees start from local `main`,** not from the branch that is checked out: merge the prerequisite first,
  or name the commit the agent needs.
- **Implementor agents often refuse to commit:** read the diff and commit yourself. To bring the work to a pull
  request: `git worktree add ../rag-<id> -b <branch> origin/main`, `git cherry-pick -n <agent branch>`, build, re-run
  the tests there, commit, push; then remove the agent's worktree and its `worktree-agent-…` branch.
- **Splitting a feature into an API agent and a UI agent** against a contract written first in the design worked
  well (#532): integrate on one branch and check both sides agree.

### Browser checks and this harness

- **Drive a new page in a real browser before the pull request:** a Node script under the scratchpad that requires
  `tests/e2e/node_modules/playwright` with `ignoreHTTPSErrors: true`, reads `Parameters:dev-user-password` from the
  AppHost user secrets itself (never on a command line) and writes screenshots. It found a usability gap that 400
  unit tests didn't. The Playwright MCP browser rejects the dev certificate.
- **Sign-in in a script:** click `locator('a,button').filter({ hasText: 'Sign in' }).last()` and wait for the
  Keycloak URL together with the click.
- **A `cd` into another worktree can move the session there for good:** use absolute paths and `git -C <path>`.
- **`rm` with a shell variable in the path is stopped by a safety check:** use literal paths, or `git rm`.
- **The Edit tool refuses a file outside the working directory that was not read first.** A commit made in the same
  step then goes out without the change: check `git show --stat`.
- `dotnet dev-certs https --export-path x.pem --format Pem --no-password` also writes `x.key`, the dev certificate's
  private key: delete it afterwards. Don't `pkill -f` a pattern that also matches your own shell command.

## Plan files (how they are kept)

- **The plan ahead is only in `plans/roadmap.md`.** HANDOVER links to it; when a step lands or an input arrives, update
  the roadmap's critical path first.
- **At each release** move the queues' ticked items to `plans/archive/work/to-v<version>.md` (unchanged, links fixed
  for the depth), fold the release notes (`python3 scripts/unreleased.py fold`), and move the CHANGELOG's and upgrade
  guide's *Unreleased* to the version.
- **Designs hold the specification.** History (answered questions, deferred items, milestones) goes to
  `plans/archive/design-history/<area>.md`; a moved section keeps its number and heading with a `Moved to [...]` line,
  which `scripts/check-doc-refs.py` follows. Non-binding papers go to `plans/proposals/` with a status line.
- **`to-verify.md` is sorted by when a check can run:** the live block under the roadmap step whose input it needs,
  before go-live (no input needed), and P2 and later. When a step's input arrives, its section is that step's
  checklist; a new row goes where it can run.
- **Left for when each area is next worked on:** cite stable IDs (V39, a rule's name, a heading anchor) in new
  citations rather than bare §-numbers; move the *Phases and planning* section of `decisions.md` into the roadmap,
  keeping only binding rules there.

## Environment notes

- `rag_key.txt` in the repo root holds the interim-gateway key (mode 600, excluded in `.git/info/exclude`). Never
  commit, print or move it.
- **Git over HTTPS:** `git -c credential.helper= -c 'credential.helper=!gh auth git-credential' fetch|pull|push …`,
  in Fors's clone too: a plain `git push` waits for the Windows credential manager and never returns.
- Ports 5000 and 5173 are held by other programs: Fors's IntegrationTests need `ASPNETCORE_URLS=http://127.0.0.1:0`.
- `172.30.0.0/16` is taken on this machine; test networks use 10.21x.0.0/24.
- **`actionlint`** is installed (`/snap/bin/actionlint`): run it on every workflow change, with `-shellcheck=
  -pyflakes=` (it hung once without). `shellcheck` and `pwsh` are not installed.
- **`grep` here is ugrep**, which reads `$` mid-pattern as an anchor (`grep '${A:-'` doesn't match `${A:-x}`). Use
  `grep -F` for literal `${...}`; the runner has GNU grep.
