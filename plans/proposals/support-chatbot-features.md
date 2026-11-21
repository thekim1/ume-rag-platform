# Support chatbot features: ten candidates beyond Eneo

**Status:** **proposal**, for the user to choose from (decided 2026-11-15 to write it up; nothing is built until a
feature is picked). **Picked and built** (released in `v0.9.0`): **F3** and **F4** (user, 2026-11-19, #857; Decisions
"Escalation hand-off" and "Source dates on citations", chat §9.9 and knowledge §10.1) and **F8** (user, 2026-11-20;
Decisions "Topics per request"). The others stay proposals. Each picked feature then gets a chat or admin design section and a Decisions line from the main
session before it is queued (`cases/support-assistant/06-gaps.md` preamble). Written by the main session, 2026-11-15,
from the comparison in [`../../platforms/eneo-vs-ume-rag.md`](../../platforms/eneo-vs-ume-rag.md) Part 2, which holds
the Eneo evidence (the greps) for each.

The case is the Support Assistant: an IT support chatbot for municipal staff (`cases/support-assistant/`). None of
these is built or queued, and none duplicates a planned item: per-assistant test-question sets, the nightly answer
judge, unused-assistant attention items, the public widget, `/mcp`, picture descriptions (U1, waits on DoIT's files at
T6) and the Git connector (SA-10) are already planned and left out. Sizes as in `06-gaps.md`: **S** is days, **M** a
week or two, **L** several weeks.

## At a glance

| # | Feature | Size | Builds on |
|---|---|---|---|
| F1 | Exact identifier search for error codes and product names | M | retrieval, RRF fusion |
| F2 | Ticket triage as structured output (`kalla.triage`) | S–M | the `kalla` extension, the condense model |
| F3 | Escalation hand-off (`kalla.escalation`) | S | `kalla.outcome`, the assistant's contact links |
| F4 | Freshness on every source (updated and checked dates) | S | crawl freshness, citations |
| F5 | Operational notices (`kalla.notices`) | M | a short-interval crawl of the IT status page, retrieval |
| F6 | Downvotes traced to the cited documents | M | feedback import, citations |
| F7 | Content-gap report from outcomes | M | `Messages.Outcome`, daily totals |
| F8 | Narrow by topic per request (`kalla.topics`) | S | topics in retrieval |
| F9 | Owner digest for knowledge at risk | S–M | run reports, deletion proposals, document states |
| F10 | Cost per outcome | S | `AssistantDailyUsage` per step, outcomes |

**Suggested order** (main session): the four S items first, because each reuses a built piece and the support app can
use them at once: **F3, F4, F8, F10**. Then **F7** and **F6** (the quality loop that tells the team which documents to
fix), then **F2** and **F1** once DoIT's real tickets show how often codes and triage matter (F1 needs a measurement
first, below), and **F5** and **F9** last.

## The features

### F1. Exact identifier search for error codes and product names (M)

- **User value:** pasted tickets carry `0x80070005`, `AADSTS50076`, `KB5034441`, printer models and app versions. The
  document that names that exact code is the best answer, and the Swedish stemmer and embeddings may both blur it.
- **Fit:** in `Kalla.Retrieval`, a third candidate list next to keyword and vector. Identifier-shaped tokens
  (regex: hex codes, letter+digit runs, versions) are extracted from the query and matched on a `simple`-config or
  trigram index over chunk text. The worker adds the index. The list joins `RrfFusion` with its own weight, and
  `CoverageEvaluator` may count an exact hit as coverage. Measure first: how the `swedish` config tokenises such codes
  is unverified.
- **Eneo:** has no keyword search in the answer path (`keyword_search` has no callers; grep `tsvector\|ts_rank\|pg_trgm`
  finds only audit indexes).

### F2. Ticket triage as structured output: `kalla.triage` (S–M)

- **User value:** with the answer, the app gets `{category, system, urgency_hint, missing: ["device", "error text"]}`.
  The frontend pre-fills the ticket fields and asks for the missing facts.
- **Fit:** in `Kalla.Answering`, one call on the condense model, run like `FollowUps.cs` (its own budget, second
  `kalla` chunk, `[]` on failure, never stored). It uses the same personal-data and no-link filters, with a category list
  set per assistant version. It runs on the assistant's gateway key, so its cost is attributed per step.
- **Eneo:** has no structured answer metadata in the conversation API (grep `triage`: only a help-assistant template;
  `response_format\|json_schema` in `B/conversations`, `B/assistants`: only OpenAPI deprecation flags).

### F3. Escalation hand-off: `kalla.escalation` (S)

- **User value:** when the outcome is `not-found` or `answered-general`, the app gets the escalation route (queue
  name, link from the assistant's not-found contacts) and a short hand-off summary to paste into the second-line
  ticket. Nobody retypes the ticket.
- **Fit:** reuses `NotFound` contact links (chat §10.1), `kalla.outcome` and the condense model for the summary. The
  output filter's `allowedLinkHosts` keeps the links honest. Ticket-system write-back stays "later" (`07-rollout.md`).
- **Eneo:** lacks it (grep `ticket\|escalat\|handoff\|servicedesk\|topdesk\|jira\|servicenow`: hits only in module
  login and template text, `B/modules/module.py:37`).

### F4. Freshness on every source (S)

- **User value:** each citation and related document shows "updated 2026-03-02, checked yesterday". A stale guide is
  visible before staff follow it.
- **Fit:** the worker already keeps the visible date (`HtmlExtractor` date facts), `Last-Modified`, the content-hash
  version and the last successful fetch. Add `updated_at` and `checked_at` to citation and `related_documents` entries
  (chat §3.8). Retrieval's age decay is already per collection.
- **Eneo:** lacks it (no dates in `frontend/apps/web-next/src/features/chat/activity-sources.tsx`, `answer-links.tsx`;
  grep `updated\|modified\|ago` there).

### F5. Operational notices: `kalla.notices` (M)

- **User value:** during an outage ("Teams down since 08:10"), every matching question first gets the notice and the
  status link, before stale self-help steps.
- **Fit:** a `notice` knowledge base kept fresh by a short-interval crawl of the IT status page (the URL-list source
  with a 5-minute schedule) or by admin-entered items with an expiry. A matching notice (topics or keyword hit) is
  returned in `kalla.notices` and placed in the prompt as data. Reuses the source types, the scheduler, topics and the
  output filter.
- **Eneo:** lacks it (grep `incident\|outage\|announcement\|banner\|maintenance\|motd`: only a connection banner and
  a crawl notice in `frontend/apps/web-next/src/features/knowledge/notices.tsx`).

### F6. Downvotes traced to documents (M)

- **User value:** the knowledge-base editor sees "this guide was cited in 7 thumbs-down answers this month" and fixes
  the guide, not the prompt.
- **Fit:** `chat."Feedback"` (API and Open WebUI import) already references the message. `MessageSource` already holds
  the cited document ids, even in history mode `none` (`ChatEntities.cs:155-199`). Add a per-document aggregate and an
  attention item on the knowledge-base page. It works without stored content.
- **Eneo:** partial. It counts feedback per assistant (`B/analysis/analysis_router.py:355`) and lists questions with
  their feedback, but has no per-source aggregation (grep `feedback` in `B/analysis` with `blob\|source\|reference`:
  nothing).

### F7. Content-gap report from outcomes (M)

- **User value:** "18 % of questions this week ended `not-found`, mostly near the topic *VPN*". DoIT knows which
  guide to write next.
- **Fit:** outcomes are stored per message (`MessageOutcomes`, `ChatEntities.cs:27-38`). Add the nearest-hit topic and
  collection for low-coverage turns. Query text stays unstored, so the report is counts per topic and day. Example
  questions only come later through the A1 path, or as editor-rewritten questions (Decisions "Evaluation models …").
  It feeds the gold set.
- **Eneo:** has no outcome classification (grep `answer_outcome\|unanswered\|no_answer`: prompt text only) and no gap
  report (grep `content.?gap\|knowledge.?gap`). Insights Q&A over stored conversations (`analysis_router.py:667`) is a
  manual, partial substitute.

### F8. Narrow by topic per request: `kalla.topics` (S)

- **User value:** the support app has a "Printers / Teams / Accounts" selector, and the search stays inside that topic.
  This raises precision on short tickets.
- **Fit:** topics are already GIN-indexed tags in the retrieval SQL (`SearchScope.cs:26-30`). A request may only
  narrow the assistant's linked topics, never widen them, the same rule as `kalla.general_knowledge`.
- **Eneo:** has no metadata or tag filter at search time (`B/info_blobs/source_metadata.py:1-7`). The TOOL mode's
  `within=` narrows to one source chosen by the model, not by the client (`B/internal_mcp/knowledge.py:706-709`).

### F9. Owner digest for knowledge at risk (S–M)

- **User value:** the knowledge base's contact person gets a weekly mail listing three things: heavily cited documents
  that went `Gone`, failed to parse, or haven't changed in 12 months; deletions the guard is holding; and the F6
  downvote counts.
- **Fit:** run reports, `DeletionProposals`, document states and the contact per knowledge base already exist
  (Decisions "Organisation", connectors §13). A worker sweep and a mail or webhook sender (new) are needed. The Overview
  attention items in admin design are a P2 theme. This feature is the outbound digest, not the page.
- **Eneo:** lacks it (grep `stale`: only failed SharePoint subscriptions, `sharepoint_subscription_worker.py:6`; no
  owner notification).

### F10. Cost per outcome (S)

- **User value:** "the support assistant costs $0.004 per answered ticket and $0.001 per not-found". This gives a
  basis for chargeback and for choosing between models on the gold set.
- **Fit:** `AssistantDailyUsage` already splits cost per step from the gateway's `x-fors-cost-usd`, and outcomes are
  stored per message. Join them on the run (`x-fors-run-id`) into a per-outcome column on the Costs page.
- **Eneo:** has no per-space or per-assistant cost (`B/token_usage` routers: tenant and user only). Cost is estimated
  in the browser (`usage-page.tsx:183`).
