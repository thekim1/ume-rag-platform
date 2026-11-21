# Admin UI review: Home, lists, audit log and the assistant page

**Status:** **agreed** (user, 2026-11-15: UI1 B, UI2 B, UI3 A, UI4 B, each as recommended). The binding rules are in
Decisions "Admin API and UI" and admin design §5.3, §5.5.1, §5.5.2, §5.5.4 and §9.1. **Built and released in
`v0.6.0`** (M-19, in `archive/work/to-v0.6.0.md`). Written by the main session, 2026-11-15, after a review of the admin UI in a
browser.

## How the review was done

25 screens at 1440 px, plus phone width (390 px) and the dark theme. The stack was the FakeGateway stack on a copy of
the development data: Turkos with 1,169 documents, its runs, eight evaluation runs and a published assistant. The parser,
the renderer and Open WebUI were off, so upload, preview, create flows and the test panel's real answers were not tried.

**What works well:** every page loads in about 1.5 s with no console errors. The phone layout works (no sideways
scrolling, a bottom menu, table rows become cards), and the dark theme is consistent. Wording is plain and consistent
with §5.7.

**Already fixed** (#723): the Documents tab opened by link said "No documents yet"; archived assistants showed as
"Draft only"; the assistant page lost its tab on reload (both detail pages now keep it in `?tab=`).

## What the review found

1. **Home doesn't show what is running.** It is a stack of *Get started* checklists, one per unfinished knowledge base
   (four near-identical cards in the dev data, all test knowledge bases). The one published knowledge base and assistant
   (Turkos) don't appear on Home at all. This also falls short of the design: §5.3 gives P1's Home "setup checklist,
   sync health per department, queue age, org/model sync", and only the checklists are built. The attention items of
   §5.5.1 are P2 by the user's choice (2026-11-07, M-3), so they are not missing.
2. **The lists say too little to act on.** Knowledge bases show name, department, class, sharing, documents and state:
   no last fetch, run result or quality. Assistants show name, class and state: no knowledge bases, model, published
   version or use. Rows open on click, but they are buttons, not links, so a row can't be opened in a new tab or
   copied as a link.
3. **The audit log is hard to read.** The object column shows raw ids (`01a1b71e-fa04-…`), not names, with no link;
   events are code names (`source.sync`, `collection.preflight`); the filters for event, object type and object id are
   free text, so you must know the codes and ids; runs of identical events (six previews in a row) fill the page.
4. **The assistant page is ambiguous.** It opens on an editable form for a published assistant with no draft, next to
   an **Edit** button in the header (which edits name, description, contact and level). It isn't clear which starts a
   draft. The four *General knowledge* options come in two pairs with word-for-word identical descriptions (Off =
   Supplement, Fallback = Open), so the descriptions don't tell them apart. "All topics / Topics of Turkos" above an
   empty box isn't self-explanatory.

Smaller points that need no decision are listed at the end.

## Decisions for the user

### UI1: Home

| Option | Gain | Cost |
|---|---|---|
| **A.** Build §5.3's P1 Home as designed: checklists plus sync health per department (fetches in the last 24 h: done, with remarks, failed), queue age, organisation and model sync | Closes the gap against the design; no design change | Home still shows nothing about the assistants in use |
| **B.** A, plus a **Live** card per published assistant (7 days: questions, not found, thumbs up and down, cost; from `AssistantDailyTotals`, `AssistantDailyUsage` and the feedback import), and checklists folded once a knowledge base is published, with a *Hide* per checklist | Home answers "is it working?" for the test chatbot's team; data already exists | One `GET /api/overview` extension and a card; about a day |
| **C.** B, plus the cheap attention items brought forward from P2: runs with remarks or failed in the last 24 h, sources with ≥ 3 failed runs, personal data in `public` knowledge bases, a gateway key that is unusable or out of budget | Problems in one place instead of on each page | Reverses part of the 2026-11-07 deferral; `/api/overview` gets the attention list (§5.5.1); 2–3 days |
| **D.** Leave Home as it is; each signal stays on its own page | Nothing to build | The test team must visit each page to find a problem |

**Recommendation: B now**, and C's items after the test chatbot's first findings (T6) show which problems actually
occur.

### UI2: Lists

| Option | Gain | Cost |
|---|---|---|
| **A.** More columns. Knowledge bases: last fetch (time and result, linked to the run), quality (latest recall@10). Assistants: knowledge bases, model, published version and date, questions in 7 days | Each list answers "which one needs me?" | List endpoints return a few more fields |
| **B.** A, plus **rows as links**: the name cell is a real link (`DataTable` change, so every list gets it), opening in a new tab and copying a link work | Normal browser behaviour on every list | Small: one shared component |
| **C.** B, plus sorting and a remembered filter on each list | Helps once lists grow | Lists hold 3–20 rows in the pilot; little gain yet |
| **D.** Leave the lists | Nothing to build | Detail pages stay the only place to see state |

**Recommendation: B.**

### UI3: Audit log

| Option | Gain | Cost |
|---|---|---|
| **A.** Readable entries: the API returns the object's name (the snapshot's `Name`, else the current one) and the page links it to the object; events in plain words (a label per §9.2 action: "Fetch started", "Published"); event and object type as pickers; the object found by name | The log can be read and filtered without knowing codes or ids | API: name in the list DTO; UI: labels for about 80 actions in `labels.ts` |
| **B.** A, plus consecutive identical events by the same person on the same object shown as one row ("Preview ×6") | Less noise | Grouping across keyset pages is fiddly; the full list must stay one click away |
| **C.** Only the plain-word events and pickers (no names or links) | Cheapest | The object column stays unreadable |
| **D.** Leave it until P2's field-level diff viewer (§5.3) | Nothing now | Department admins get an audit log they can't use in the pilot |

**Recommendation: A.**

### UI4: Assistant page

| Option | Gain | Cost |
|---|---|---|
| **A.** A read view first: the Draft tab shows the published settings read-only with **Start a draft**; the form opens only for a draft. The header's button becomes **Edit details** | No doubt about what is live and what is a draft | Changes §5.5.4's wireframe (the draft form is always open there); the most work of the four |
| **B.** Keep the form as designed, and remove the doubt in words: the header's button becomes **Edit details**, the form's lead says "Saving starts a draft from version n; the published version stays live until you publish", one description per general-knowledge mode that says what differs (when general knowledge is used, and what happens when nothing is found), and the topic field labelled "Limit to topics (empty: all topics)" | Fixes the confusion within the current design | Wording only (`labels.ts`) and one label in §5.5.4 |
| **C.** A separate Settings tab (read-only) and a Draft tab only while a draft exists | Clear split | A tab that comes and goes; more navigation |
| **D.** Leave it | Nothing to build | The ambiguity stays |

**Recommendation: B.**

## No decision needed (done with the work above)

- **Settings:** an empty field shows the value in force as its placeholder ("20, the deployment's default"); short
  fields for small numbers.
- **Feedback and Costs:** one filter layout (labels above, even widths). "Answers 0 — found nothing" reads as "0
  answers, 0 not found".
- **Models:** a heading on the model table ("The AI gateway's models"); "Set here" as a link to the row's edit, or plain
  "Manual"; which assistants use each model.
- **Organisation:** "0 own · 0 selected · 0 whole organisation" spelled out, or a tooltip.
- **Knowledge base overview:** "No contact yet" with **Add contact**; "Used by" names linked to the assistants.
- **Source page:** the personal-data section as a table (flagged first, linked to the document), folded by default; the
  run list's "4 failed" linked to the run.

## Where it lands

- Design: admin §5.3 (Home row), §5.5.1 (the Live card; attention items only if UI1 C), §5.5.2 and §5.5.4 (list
  columns, labels), §9.1 (audit names and labels).
- Code: `src/admin-ui/src/**` and the list and overview endpoints in `src/Kalla.AdminApi/**`, both stream b's paths.
  Stream b is on review #694, so main can build it on a `main-session/` branch after announcing it on #667, as with
  #723, or queue it for stream b.
- Order: UI4 B and the small points (wording, a day), UI2 B, UI1 B, UI3 A.
