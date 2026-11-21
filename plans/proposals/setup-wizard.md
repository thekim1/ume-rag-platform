# Setup wizard: sign-in and the last deploy values set in the browser

**Status:** **agreed** (user, 2026-11-15: W1–W10 answered, the design agreed after an independent review, §10). The
binding rules are in Decisions "Setup wizard" and "Deploy configuration", identity §19, admin §5.11 and
repo-and-deployment §6.3 and §7.1. **Built and released in `v0.7.0`** (M-20, in `archive/work/to-v0.7.0.md`). The main session builds every step, stream b's
paths included (user, 2026-11-15). Written by the main session, session 40.

**Goal** (the user's idea, 2026-11-15): the platform is easy to set up. What a deployer still types into Portainer's
stack variables, `deploy/.env` or the Kubernetes values moves into a first-run wizard in the admin UI wherever that is
sound.

**Builds on** [`deploy-configuration.md`](deploy-configuration.md) (M-17), which took the Portainer stack from 48
variables and three paste round trips to one pass. This proposal is about the values its §2.1 kept.

## 1. What is left to type today

`deploy/portainer/stack.env.example` at v0.6.0 marks **9 values CHANGE**:

| Whose | Values |
|---|---|
| The gateway (4) | `FORS_OIDC_AUTHORITY` (shared with Källa and Open WebUI), `FORS_GATEWAY_HOST`, `FORS_GATEWAY_PUBLIC_URL`, `FORS_ADMIN_HOST` |
| Källa (3) | `KALLA_API_HOST`, `KALLA_ADMIN_HOST`, `KALLA_OIDC_CLIENT_SECRET` |
| Open WebUI (2) | `KALLA_OPENWEBUI_HOST`, `KALLA_OPENWEBUI_OIDC_CLIENT_SECRET` |

So: five host names, one URL that repeats a host name, one sign-in provider address and two client secrets (three
with a confidential `fors-admin`). The two secrets are typed into Portainer's variable list, where they stay readable.
Not marked, but edited by some: Open WebUI's client id, and the owner department's code inside the one-line JSON of
`FORS_INTEGRATION_MANIFEST`.

**Checked in the code and the stack** (2026-11-15, `main` at `de0c3a41`):

- Every service reads its secrets directory as key-per-file with reload on change
  (`InfrastructureExtensions.cs:43`). A settings change notifies kalla-api and the worker; **kalla-admin runs no
  configuration monitor** (`AdminApi/Program.cs:42`) and reads the database where it needs a current value.
- kalla-admin copies `Oidc:Authority`, `ClientId` and `ClientSecret` into the OpenID Connect handler's options once
  (`AdminAuthentication.cs:206`), and **refuses to start without an authority** (`[Required]` with `ValidateOnStart`,
  `AdminOptions.cs:83`, `Program.cs:55`).
- **Data Protection is registered only in kalla-admin** (`Program.cs:41`). Values are encrypted with keys of the key
  ring in `config."DataProtectionKeys"`; the certificate (`data-protection.pfx`) protects that ring at rest
  (`InfrastructureExtensions.cs:193-211`). kalla-api mounts the certificate and may read the table
  (`app-roles.sql:25`) but registers nothing. Encrypted values in the database have a precedent (revealable key
  secrets).
- The Portainer template and the Kubernetes base always set the client id and the token-shape keys with their
  defaults (`compose.rag.template.yaml:281-290`, `base/kustomization.yaml:33-39`), and kalla-admin runs as two replicas
  on Kubernetes (`base/admin.yaml:10`). `deploy/kubernetes` has no Open WebUI.
- Logs are exported over OTLP when a collector is set (`ServiceDefaults/Extensions.cs:76`).
- kalla-api trusts the user tokens of the deployment's own Open WebUI (`ChatCallers.cs:32`), so whoever sets Open
  WebUI's sign-in provider decides who its users are.
- The TLS proxy verifies its upstreams by their internal names (`proxy_ssl_name kalla-api`,
  `deploy/proxy/nginx.conf.example`), so the public host names in the internal certificates' SANs
  (`bootstrap.sh:51-52`) serve only a client that reaches a service past the proxy.
- Open WebUI gets `OAUTH_CLIENT_ID`, `OAUTH_CLIENT_SECRET`, `OPENID_PROVIDER_URL`, `WEBUI_URL` and
  `OPENID_REDIRECT_URI` from its environment, with `ENABLE_PERSISTENT_CONFIG=false`; it already starts through a wrapper
  that exports three secrets from files (template line 471).
- `kalla-bootstrap` runs with `network_mode: none`: it can't read the database, so it can't know whether setup is done.

## 2. How other products do it

A survey of 14 self-hosted products, from their source and official documentation (Grafana, GitLab, Nextcloud,
Portainer, Keycloak, Home Assistant, Open WebUI, Dify, Onyx, RAGFlow, Authentik, Gitea, Mattermost, Harbor). The
points that shaped the answers; three were re-checked by the main session in the fetched files (Portainer's setup
token, Keycloak's localhost rule, Onyx's last-provider guard):

- **First-run access comes in three forms.** An open race, where the first visitor wins (Nextcloud, Gitea, Home
  Assistant, Open WebUI, Onyx, Authentik, Mattermost). A deploy-time secret (Grafana's `admin`/`admin`, Harbor,
  Authentik's bootstrap variables). Proof of host access: GitLab's password file, RAGFlow's password file, Keycloak's
  localhost-only page, and Portainer, whose `develop` branch adds a 256-bit setup token printed in the start-up log
  (`X-Setup-Token`) on top of its 5-minute window.
- **The newest code tightens the open race** (Portainer, RAGFlow, Keycloak's temporary bootstrap admin).
- **Re-opening always needs host access**: a CLI in the container, a helper container, a file. Never the network.
- **Every product with sign-in settings in the UI keeps a way back in.** Mostly a local admin password. Onyx alone
  enforces it server-side: it refuses to remove the last working sign-in method.
- **A test before saving is rare**: Harbor's *Test OIDC Server*, Mattermost's *Test Live URL*, Onyx's discovery fetch
  on save. None requires a real sign-in round trip.
- **Variables against stored values, three camps:** the variable wins and the field is locked (Mattermost, Harbor,
  Gitea); the variable seeds the first start and is ignored afterwards (Open WebUI, whose documentation has a
  troubleshooting section for it); the stored value overrides and the variable is the fallback (Onyx, Grafana's SSO).
- **The public URL is deploy-time almost everywhere**; Mattermost is the exception.
- **Signing and encryption keys are never editable in a UI.**

## 3. Decided by the user (2026-11-15)

Each was put as four alternatives with a recommendation; the user took the recommendation in all but W2.

| # | Question | Answer |
|---|---|---|
| W1 | Which products | **The whole stack, Källa first:** one design for the gateway, Källa and Open WebUI; Källa's and Open WebUI's part is built first, the gateway's part is issues in its repository |
| W2 | Which deploy forms | **All three:** Portainer, Compose and Kubernetes (the recommendation was Portainer only) |
| W3 | Who may use the wizard before sign-in works | **A setup code from the log:** reading it proves host access; the wizard is open only while no working sign-in is stored and locks itself when done |
| W4 | Lock-out | **A test sign-in before saving, and the setup code re-opens:** no passwords; after setup the sign-in provider changes only through the wizard with a new code, never from an ordinary admin session |
| W5 | A value set both by the deployment and in the wizard | **The deployment wins and the field is locked**, shown as *set by the deployment* |
| W6 | Where wizard values live | **The database:** plain values as settings, secrets encrypted with Data Protection (the key ring the deployment's certificate protects, §4.5) |
| W7 | Host names | **One base domain variable** (`FORS_DOMAIN`); the five names default under it, each still overridable. They stay deploy-time |
| W8 | Open WebUI | **Its wrapper fetches its sign-in settings from kalla-api** at start and exits when they change |
| W9 | The crawler's private ranges | **Stay deploy-time** (Decisions "Deploy configuration", D4 unchanged); the wizard shows the value read-only |
| W10 | How a saved value takes effect | **Reload in place**; only Open WebUI restarts itself |

## 4. The design

### 4.1 What the wizard sets

| Value | Today | After |
|---|---|---|
| Sign-in provider address | `FORS_OIDC_AUTHORITY` / `KALLA_OIDC_AUTHORITY` | `Settings.Identity.Oidc.Authority`; in a shared stack the default comes from the gateway (§6, WG2) |
| kalla-admin's client id | `KALLA_OIDC_CLIENT_ID` (default `kalla-admin`) | `Settings.Identity.Oidc.ClientId`, same default |
| kalla-admin's client secret | `KALLA_OIDC_CLIENT_SECRET` → `admin/Oidc__ClientSecret` | an encrypted setting (§4.5) |
| Token shape: role, department, team and group claims, extra scope, tenant id, refuse guests | seven `KALLA_OIDC_*` variables, defaults fit Keycloak | `Settings.Identity.Oidc.*` under *Advanced*, with a preset per provider (Keycloak, Entra ID) |
| The break-glass platform-admin groups | `KALLA_OIDC_GROUPS_PLATFORM_ADMIN` | the same variable (it stays a deployment value, Decisions "Role groups as a setting"); the wizard shows it and the test sign-in says whether it applied |
| Open WebUI's provider address, client id, client secret, provider name | four `KALLA_OPENWEBUI_OIDC_*` variables | `Settings.OpenWebUi.Oidc.*`, the secret encrypted; the provider address defaults to kalla-admin's. Set in setup mode only, like kalla-admin's (§4.7) |

**Not in the wizard:** host names (W7), ports and binds, profiles, image pins, telemetry, the gateway's address, the
crawler's private ranges (W9), every generated secret and key. The wizard's last page lists them read-only with their
source, so the deployer sees the whole configuration in one place.

### 4.2 Setup mode and the setup code (W3)

- **Setup is open** in two cases only, both explicit: it has never been completed (`SetupCompletedAt` is null in the
  settings row), or a re-open was asked for (§4.4). Nothing else opens it. A missing settings row, a secret that can't
  be decrypted or a deployment value that disappears leaves setup **closed**; kalla-admin logs why, sign-in fails with a
  plain message, and the way forward is the re-open variable. (Deriving "open" from "no sign-in values" would open on
  such faults, and would never close for a public client, whose secret is empty by design.)
- **A deployment that declares its sign-in** (an authority and a client secret from variables or files; the client id
  has a default and can't tell) counts as completed at first start: setup never opens, as today. That is also what an upgrade of an existing deployment does.
- **The code** is derived, not stored: an HMAC of a **setup secret file** (generated by `kalla-bootstrap`,
  `init-deployment.sh` and `create-secrets.sh` like every other generated secret, never overwritten) over a nonce in
  the database that is made when setup opens. Every replica computes the same code, a recreated container can print it
  again, and a new opening has a new code. It is valid for one hour from the opening; after that a restart of
  kalla-admin makes a new nonce.
- **Where it is shown:** kalla-admin writes it straight to its standard output at start while setup is open, with the
  wizard's address. It never goes through the logger or a security event, so it isn't exported to a telemetry
  collector. The deployer reads it in Portainer's container log, `docker compose logs kalla-admin` or `kubectl logs`
  (any pod shows the same code). The code is worth platform admin while setup is open, since its holder chooses the
  provider; the docs say so.
- **The wizard** is the admin UI's `/setup` page, served before sign-in like the rest of the single-page app; its API
  is under `/bff/setup` and answers 404 while setup is closed. Entering the code gives a setup session (a cookie, 30
  minutes, the antiforgery token the BFF already gives anonymous callers).
- **Wrong codes:** a limit on attempts for the whole deployment, counted in the database (not per address: before
  the trusted proxies are set every caller has the proxy's address, and the limiter in memory is per replica). A
  128-bit code needs no more. Wrong codes never void the code: that would let anyone who reaches the page shut the
  deployer out.
- **Closing:** a saved, tested sign-in sets `SetupCompletedAt`, clears the nonce and ends the setup session.
- **Audit:** opening, wrong codes, each test and the save are security events with the address; the save's audit row
  names the person who passed the test sign-in.

### 4.3 The test sign-in (W4)

Saving the sign-in values requires a passed test, made with the values as typed (not yet saved):

1. kalla-admin fetches the provider's discovery document and says what is wrong in plain words (not reachable, not a
   discovery document, the certificate isn't trusted). The fetch is `https` only, follows no redirect and returns
   nothing of the response, so a code holder can't use it to read internal addresses (private addresses can't be
   refused: the provider is often internal).
2. **Test sign-in** opens the provider in a new window with the candidate values and the redirect URI kalla-admin
   already uses, so nothing extra is registered at the provider. The deployer signs in as themselves. The candidate
   values are kept encrypted in the database with the setup session, so the callback may land on another replica.
3. kalla-admin exchanges the code with the candidate client secret and runs the same checks as a real sign-in (tenant,
   guests, claims). The wizard shows the result: who signed in, the roles and groups the token carried, the departments,
   and **whether this person would be a platform admin**.
4. **Save** is offered only when the test passed and the tested person is a platform admin. That removes today's
   separate step "the first platform admin needs the role `rag-platform-admin`": the wizard says so before anything is
   locked, and names the claim it looked in.

The values are then in force (§4.6), and the wizard hands over to the normal sign-in and the *Get started* checklist
(the embedding profile is next).

### 4.4 Re-opening: a sign-in that broke, or a planned change (W4)

- After setup, no page and no API of an ordinary admin session changes the sign-in provider, the client or its secret.
  A stolen platform-admin session can't move sign-in to another provider.
- **To re-open**, the deployer sets `Setup:Reopen` (`KALLA_SETUP_REOPEN`) to any new value, for example today's
  date, and updates the stack. kalla-admin compares it with the last value it consumed (in the database). A new value is
  consumed **when setup opens**, not when it is saved: it opens setup once, for one hour, with a new code. An abandoned
  re-open therefore ends by itself, and a restart doesn't open it again. Changing a stack variable is the same proof of
  host access as reading the log, needs no shell in the container, and works on all three deploy forms.
- **After a database restore** the consumed value may be older than the variable's, which would open setup once more
  for an hour. The restore steps say to remove the variable first.
- While re-opened, the existing sign-in keeps working; the wizard shows the stored values (never a secret) and the
  same test applies before a save.
- **A client secret that expires** is the common case. The provider doesn't tell kalla-admin when (Entra ID would need a
  Graph permission), so the wizard takes an optional expiry date beside the secret, and Home reminds platform admins
  30 days before it, with these steps.
- **The other way out** is W5's: set the value in the deployment and restart. It always wins.

### 4.5 Storage (W6)

- **Plain values** are `Settings.Identity.Oidc.*` and `Settings.OpenWebUi.Oidc.*`, in the settings row with its
  version, `pg_notify('rag_config')` and audit, as Decisions "Settings rule" says.
- **Secrets** (the two client secrets) are stored encrypted with ASP.NET Data Protection under a purpose of their own.
  The keys are the key ring's (`config."DataProtectionKeys"`), which the certificate each deployment already has
  (`data-protection.pfx`) protects at rest. They are write-only in the API: a read answers "set" and when, never the
  value.
- **kalla-api gets Data Protection** for §4.7, read-only: the same application name and certificate, key generation
  off (its role may only read the key table). A purpose of its own keeps Open WebUI's secret apart from kalla-admin's.
- **Replacing the certificate** would make the stored secrets unreadable, as it does admin sessions and revealable
  keys today: the replacement steps keep the old certificate as an unprotect certificate, or the secrets are typed
  again after a re-open. Development has no certificate and stores the ring unprotected, as today.
- **Backup** needs nothing new: the database and `data-protection.pfx` are already on the list
  (`docs/portainer.md`, *First deploy* step 8).
- **This changes a rule:** repo-and-deployment §6.3's "secrets and certificates are files, never environment
  variables" becomes "files, or settings encrypted with Data Protection; never environment variables". Only values a person types in the wizard are stored this way; every generated secret stays a file.
- The same storage on Portainer, Compose and Kubernetes, so no service needs write access to a secrets volume or to
  the Kubernetes API.

### 4.6 Precedence and taking effect (W5, W10)

- **The deployment wins, per value.** A value the deployment sets is used, and the wizard shows the field locked with
  *set by the deployment*. "Set" is read from the raw configuration, where an empty value counts as unset; the bound
  options can't tell a default from a set value.
- **So the deploy files stop setting defaults** (step S6): the template's and the Kubernetes base's `Oidc__ClientId`
  and token-shape lines go, and the code's defaults apply. A deployer who had changed one of those variables is told in
  the upgrade guide to keep the line, or to enter the value in the wizard.
- **In stage 1 a shared stack still gets the provider address from `FORS_OIDC_AUTHORITY`** (the gateway needs it), so
  that field is locked and the wizard asks for the client secret and runs the test. That is the smallest wizard, and
  the right one for that stage.
- A deployment that sets authority and client id never opens setup (§4.2): Compose with `init-deployment.sh`'s
  options and Kubernetes with declared values keep working as today.
- Each field reports its source to the UI: `deployment`, `wizard` or `default`.
- **Reload in place:** kalla-admin's start no longer requires an authority. At each sign-in challenge and callback it
  compares the settings version it built its sign-in options from with the database's (it already reads the database
  at sign-in) and rebuilds the options when they differ. No listener, and both replicas follow within one sign-in.
  Sessions that exist stay valid until they end.
- The test sign-in (§4.3) runs on the same code path with candidate values, so "the test passed" and "sign-in works"
  can't differ.

### 4.7 Open WebUI (W8)

- kalla-api serves Open WebUI's sign-in settings on an internal route, to the deployment's own Open WebUI client only
  (`IsManagedByDeployment`, the key in `openwebui/rag-api-key`). The client secret leaves kalla-api only there, inside
  the stack's network.
- Open WebUI's start wrapper fetches them before `start.sh` (with `python3`, which the image has) and exports them
  with the three secrets it already exports. Its own URL and redirect URI come from the host name (W7).
- **Without values Open WebUI starts anyway**, with OIDC sign-in off, and its log says so: it is healthy, nobody can
  sign in. The release boot test and `up --wait` keep working unchanged.
- The wrapper asks again every minute and exits when the answer changed; `restart: unless-stopped` starts Open WebUI
  with the new values. No Docker socket. The boot test's "no restart while starting" check still holds, since nothing
  changes during a boot.
- **Open WebUI's provider is a trust boundary too:** kalla-api accepts the users Open WebUI signs in. Its provider
  address, client and secret are therefore set in setup mode only, as a step of the wizard with the same test
  (discovery, then a round trip with Open WebUI's client and redirect URI). The step may be skipped at first run;
  doing it later is a re-open. Only the provider's display name is an ordinary setting.
- **Kubernetes has no Open WebUI today**; when it gets one, it takes the same wrapper.

### 4.8 Host names (W7)

- **`FORS_DOMAIN`** (for example `example.se`) gives the defaults `rag.<domain>`, `rag-admin.<domain>` and
  `chat.<domain>`; the gateway's `llm.<domain>` and `llm-admin.<domain>` follow when its stack file takes the same
  variable (§6, WG1). Each `UME_*_HOST` still overrides its own name. Nested defaults
  (`${KALLA_API_HOST:-rag.${FORS_DOMAIN:?…}}`) resolve in Compose v5 (checked by the reviewer with `compose config`);
  Portainer's bundled Compose is checked by the boot-only run in S0.
- **Either the domain or the names:** a stack that sets `KALLA_API_HOST` and no domain keeps today's rule (the
  admin's name falls back to the API's); the default under the domain applies only when the domain is set, so no name
  ends as `rag-admin.`. The expression is written once per use in the template (seven places) and tested by S0.
- Compose: `init-deployment.sh --domain`. **Kubernetes keeps its host names in the Ingress** (Kustomize can't derive
  them from one value); `FORS_DOMAIN` is for Portainer and Compose.
- The names stay deploy-time: they must be in DNS, in the operator's certificate and in the provider's redirect URIs
  before a wizard can be reached, and the proxy routes by them.
- The internal certificates keep their SANs as they are (no change needed for this proposal).
- The wizard's first page shows the five names and tests that each answers through the proxy (*Test live URL*, as
  Mattermost), and shows the redirect URIs to register at the provider, ready to copy.

### 4.9 The three deploy forms (W2)

| | Portainer | Compose | Kubernetes |
|---|---|---|---|
| Setup secret file | `kalla-bootstrap` | `init-deployment.sh` | `create-secrets.sh` |
| Setup code | kalla-admin's log in Portainer | `docker compose logs kalla-admin`; `init-deployment.sh` prints the command | `kubectl logs deploy/kalla-admin` (either pod); `deploy.sh` prints the command |
| Declared instead | leave the variables set | `--oidc-*` options stay, now optional (no prompt); set, they lock the fields | values and `create-secrets.sh` stay; set, they lock the fields |
| Host names | `FORS_DOMAIN` | `--domain` | the Ingress, as today |
| Re-open | `KALLA_SETUP_REOPEN` in the stack | the same in `deploy/.env` | the same on the Deployment |
| Open WebUI | the wrapper in the template | the same wrapper in `compose.openwebui.yaml` (it replaces `secrets/openwebui.env`'s two OAuth lines) | none today |

The code is one path: setup mode, storage, precedence and reload are in kalla-admin and kalla-api, and no deploy form has
logic of its own beyond where a variable is written.

## 5. What it changes for the person deploying

**Today** (Portainer, v0.6.0): fill 9 values, two of them secrets, into the stack's variables; deploy; read
`kalla-bootstrap`'s checklist; make sure the first admin has the role; sign in; create the embedding profile.

**After stage 1** (Källa and Open WebUI): fill **5** values, none of them a secret of Källa or Open WebUI:
`FORS_OIDC_AUTHORITY`, `FORS_GATEWAY_HOST`, `FORS_GATEWAY_PUBLIC_URL`, `FORS_ADMIN_HOST` (the gateway's four) and
`FORS_DOMAIN`. Deploy, copy the setup code from kalla-admin's log, open `https://rag-admin.<domain>/setup`. The wizard
shows the redirect URIs, takes the two client secrets, tests each sign-in and tells the deployer whether they are a
platform admin. Then the checklist, as now.

**After stage 2** (the gateway's part, §6): fill **1** value, `FORS_DOMAIN`, and deploy. The gateway's wizard takes the
provider address and its client; Källa's wizard gets the address from the gateway and asks for its two client
secrets.

The count falls by four in stage 1, three of them from `FORS_DOMAIN`. The wizard's own gain is in what follows.

What gets better beyond the count:

- **No secret in Portainer's variable list**, where every stack editor can read it.
- **A wrong value is found before it locks anyone out.** Today a wrong authority or secret shows as a failed sign-in
  with the reason in a container log.
- **"Am I a platform admin?" is answered by the wizard**, not by a failed first sign-in.
- **The redirect URIs are shown ready to copy**, instead of built by hand from the docs.

What stays as it is: host names (as one value), ports, profiles, the crawler's ranges, and everything a declared
deployment sets on purpose.

## 6. Gateway changes (the user's repository), for stage 2

Proposed as issues in thekim1/fors once this proposal is agreed; none blocks stage 1.

| # | Change | For |
|---|---|---|
| WG1 | **`FORS_DOMAIN` in the gateway's stack file:** `FORS_GATEWAY_HOST` defaults to `llm.<domain>`, `FORS_ADMIN_HOST` to `llm-admin.<domain>`, `FORS_GATEWAY_PUBLIC_URL` to `https://<gateway host>` | 5 host values → 1 |
| WG2 | **The sign-in provider's address in `/v1/fors/context`** (`gateway.oidc_authority`, an addition as `organisation_name` was): Källa's wizard uses it as the default, so the address is typed once for the stack | the shared authority |
| WG3 | **The same setup mode in the gateway's admin:** a setup code in adminapi's log, a test sign-in before saving, the deployment's value wins, re-open by a variable. This proposal's §4.2–§4.6 is the contract; the gateway's role groups are its own to decide | `FORS_OIDC_AUTHORITY`, `FORS_OIDC_CLIENT_ID`, `FORS_OIDC_CLIENT_SECRET` |
| WG4 | Optional. **The manifest without an edit:** the owner department chosen in the gateway's admin after the first sign-in (the integration created without an owner, or in a default department), so `FORS_INTEGRATION_MANIFEST` can ship as a default in the stack file | the department code inside the JSON line |

With WG3 the gateway's wizard runs first (the gateway is the base product, Källa its add-on) and Källa's wizard
links to it when the context has no provider address yet.

## 7. Risks and how the design answers them

| Risk | Answer |
|---|---|
| A stranger reaches `/setup` first | They need the code from the container's output; attempts are limited for the whole deployment; the code lasts an hour; setup is open only before its first completion and after a re-open |
| A fault opens setup (an empty or restored database, an unreadable secret, a removed variable) | Open is an explicit state, never derived from missing values; a broken sign-in stays closed and needs the re-open variable. A restore with a newer re-open value is the one exception, and the restore steps cover it |
| Someone locks the deployer out by guessing | Wrong codes never void the code |
| A wrong value locks every admin out | A save needs a passed round trip by a platform admin; a value that breaks later is fixed by re-opening or by a deployment value |
| A compromised platform admin moves sign-in to their own provider | Sign-in settings change only in setup mode, which needs host access |
| A compromised admin reads a client secret | Secrets are write-only in the API and encrypted in the database |
| The database is restored without the certificate, or the certificate is replaced | Already the case for sessions and revealable keys; the wizard's secrets are typed again after a re-open, or the old certificate is kept for unprotecting |
| The setup code in a log collector | It is written to standard output only, never through the logger, so the OTLP export doesn't carry it; a collector that ships container output would. It lasts an hour and is void once setup closes; the docs say so |
| A compromised platform admin points Open WebUI at their own provider and mints users | Open WebUI's provider and client change only in setup mode |
| The test's discovery fetch is used to probe the network | `https` only, no redirects, no response content returned, and only a code holder can call it |
| Two sources for one value confuse | One rule (the deployment wins), shown per field |
| The test sign-in shares the real callback | Built first as a spike (step S2); if one callback can't serve both, the test gets a callback path of its own and the wizard shows two redirect URIs |

## 8. Steps

One PR per step or smaller; streams by `.github/stream-ownership.txt`: `deploy/**`, `docs/**` and
`src/admin-ui/src/platform/**` are stream a; the rest of `src/**` (the admin UI's router, `labels.ts` and
`messages.ts` among it), `tests/**`, `contracts/admin/**`, `docs/security-events.md` and both `app-roles.sql` are
stream b. The designs (identity §5.2 and §11, repo-and-deployment §6.3 and §7.1, admin §5.4) and Decisions are updated
by main before S1.

| # | What | Paths | Done when | Stream | Waits on |
|---|---|---|---|---|---|
| S0 | `FORS_DOMAIN`: the three names default under it in the template (seven places) and `kalla-bootstrap`, `init-deployment.sh --domain`; today's fallback kept when only the names are set | `deploy/**`, `docs/portainer.md` | The boot-only run passes with only `FORS_DOMAIN` for Källa's names, and with only the names; each `*_HOST` overrides | a | – |
| S1 | Encrypted settings: the store (Data Protection purpose, write-only API shape), `Settings.Identity.Oidc.*`, `SetupCompletedAt`, effective values with their source read from raw configuration (deployment, wizard, default) | `src/Kalla.Infrastructure/`, `src/Kalla.AdminApi/`, `contracts/admin/` | Tests: a deployment value wins; an empty one counts as unset; a secret never appears in a response or the audit row | b | the designs |
| S2 | Spike, then build: kalla-admin starts without an authority; its sign-in options come from the effective values and are rebuilt when the settings version differs at a challenge or callback; the test sign-in with candidate values kept in the database (one callback path for both, or a second one if the spike says so) | `src/Kalla.AdminApi/` | A changed authority applies to the next sign-in on both of two instances without a restart; a test with a wrong secret fails with the reason and saves nothing | b | S1 |
| S3 | Setup mode: the explicit open state, the derived code written to standard output, the attempt limit in the database, the setup session under `/bff/setup`, `Setup:Reopen` consumed at opening, the security events; the setup secret file in `kalla-bootstrap`, `init-deployment.sh`, `create-secrets.sh` and the AppHost | `src/Kalla.AdminApi/`, `src/Kalla.AppHost/`, `docs/security-events.md` (b); `deploy/**` (a) | `/bff/setup` answers 404 once completed; a deleted settings row or an unreadable secret doesn't open it; a re-open value opens it once for an hour; two instances print the same code; wrong codes never void it | b, then a | S2 |
| S4 | The wizard's pages: code, host names with *Test live URL* and the redirect URIs, sign-in with the test and its result, the read-only summary; the route and the texts | `src/admin-ui/` | A fresh FakeGateway stack with no `Oidc:*` set is taken from the printed code to a signed-in platform admin in the browser | b (a `main-session/` branch if stream a builds pages under `platform/`) | S3 |
| S5 | Open WebUI: Data Protection in kalla-api (read-only), the internal route for the managed client, the wrapper (fetch, start without sign-in when empty, exit on change) in the template and `compose.openwebui.yaml`, the wizard's step | `src/Kalla.Api/`, `deploy/postgres/app-roles.sql` if a grant is missing (b); `deploy/**` (a); `src/admin-ui/` (b) | With the values saved, a user signs in to Open WebUI with no `KALLA_OPENWEBUI_OIDC_*` set; a changed secret restarts Open WebUI by itself; the boot test passes unchanged without values | b, then a | S1, S4 |
| S6 | The deploy files lose what moved: the default `Oidc__*` lines in the template and the Kubernetes base, `stack.env.example`, `init-deployment.sh` (options optional, no prompt); `docs/portainer.md`, `production-setup.md`, the upgrade guide (an existing deployment keeps its variables and never sees the wizard; who changed a token-shape variable keeps that line), the restore steps; `kalla-bootstrap`'s checklist points at the wizard | `deploy/**`, `docs/**` | `stack.env.example` has no secret and 5 CHANGE values; the boot test passes with no `Oidc` value set and reads the setup code from kalla-admin's output | a | S4, S5 |
| S7 | The browser test of the wizard in `e2e.yml`, where a Keycloak runs (the release boot test has none): the code from the output, the test sign-in, the lock, a re-open | `tests/e2e/` | Green on a manual `e2e.yml` run | b | S6 |
| S8 | The gateway issues WG1–WG4, and Källa's side of each when released (the context's `oidc_authority` as the default; `FORS_DOMAIN` for all five names) | issues; `src/Kalla.Infrastructure/Gateway/`, `deploy/**` | `stack.env.example` has 1 CHANGE value | main, then b and a | the gateway |

**As built:** the test sign-in moved from S2 to S3 (it has no caller before the setup API exists); S2 keeps the spike
and reports its answer. S0 is #760, S1 #761.

S0 needs nothing and is useful by itself. S1–S3 are the security-relevant part and get the main session's review of
every line (handbook, *How we work*). Most of the code is in stream b's paths; **the main session builds all of it** on `main-session/` branches (user,
2026-11-15), announced to stream b on an issue so it stays clear of the same files.

## 9. What changes in the binding documents (on agreement)

- **Decisions "Deploy configuration":** "the OIDC authority, client ids and client secrets" leave the list of what a
  deployment must set. New text: they are set in the setup wizard (a setup code from kalla-admin's log, a test sign-in
  before saving, changed afterwards only in setup mode) unless the deployment sets them, which wins; host names
  default under `FORS_DOMAIN`; `Crawl:AllowedPrivateRanges` stays as it is.
- **A new entry "Setup wizard"** with W1–W10 in the present tense.
- **Decisions "Settings rule"** gains: a typed secret may be a setting, encrypted with Data Protection.
- **Decisions "Deploy configuration"** also gains: Open WebUI's sign-in provider is set in setup mode only.
- **repo-and-deployment §6.3** (the secrets rule, §4.5 above) and §7.1 (`--domain`, optional OIDC options);
  **identity** §5.2 and §11 (setup mode, the test sign-in, the security events); **admin** §5.4 (the wizard before
  step 1, the summary page).
- A dated line under *Later decisions* in `plans/archive/decisions-log.md`.

## 10. Review (2026-11-15)

An independent agent read this proposal against the code, the deploy files and the decisions, with W1–W10 as fixed.
The main session re-checked each finding below in the files before changing the text.

| Finding | Changed |
|---|---|
| Voiding the code after wrong guesses lets anyone lock the deployer out; a limit per address is useless before the trusted proxies are set | §4.2: one limit for the deployment, in the database; no voiding |
| The code was lost with the container's log and differed per replica (kalla-admin runs twice on Kubernetes) | §4.2: derived from a setup secret file and a nonce, the same on every replica |
| A logged code leaves the host through the OTLP log export | §4.2, §7: standard output only, one hour |
| "Open when no sign-in values exist" opens on faults and never closes for a public client | §4.2: an explicit `SetupCompletedAt`; faults stay closed |
| A re-open consumed only at save never ends | §4.4: consumed at opening, one hour; the restore case named |
| Open WebUI's provider is a trust boundary (kalla-api accepts its users) | §4.7: setup mode only (the first version let a platform admin change it) |
| kalla-api has no Data Protection; values are encrypted with the key ring, not the certificate | §1, §4.5, S5 |
| kalla-admin has no settings listener, and requires an authority at start | §1, §4.6, S2: a version check at sign-in; the requirement goes |
| "The deployment wins" would lock nearly every field, since the deploy files set the defaults | §4.6, S6: raw configuration, empty is unset, the default lines go |
| A waiting Open WebUI breaks the boot test and `up --wait`; the boot test has no identity provider; Kubernetes has no Open WebUI | §4.7, §4.9, S5–S7 |
| 9 CHANGE values, not 11; stage 1 reaches 5 | §1, §5, S6 |
| `FORS_DOMAIN`: a name ending in a dot when only the names are set; Kustomize can't derive names | §4.8, S0 |
| Path ownership of the wizard's route, texts and browser tests (stream b) | §8 |
| `/setup` can't answer 404 (the single-page app answers every path); the API belongs under `/bff`; Entra ID doesn't report a secret's expiry | §4.2, §4.4 |

**Found sound by the reviewer:** the page can be served before sign-in; antiforgery works for an anonymous caller;
encrypted values in the database have a precedent; no other service reads `Oidc:*`; the AppHost, tests and CI declare
their sign-in and are unaffected.

**Not reviewed:** the gateway's side (§6), the identity, admin and deployment design texts that §9 changes,
`init-deployment.sh`'s prompts, and whether Portainer's bundled Compose resolves nested defaults (S0 tests it).

**Second review (2026-11-15, of S3's code, by an independent agent; each finding checked by the main session):** a
code holder could have had the stored or the deployment's client secret sent to a provider of their choice (fixed: a
secret goes only to the provider it belongs to); ten wrong codes shut out the right one for the opening (fixed: a
per-minute throttle, nothing stored); a test could be finished in another person's browser (fixed: a cookie binds it);
a missing settings row opened setup (fixed: closed, as §4.2 says); the body of a provider's answer had no timeout;
abandoned sessions kept typed secrets. The rules are in identity §19.
