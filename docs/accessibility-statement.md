# Accessibility statement: draft for the pilot

Källa, admin UI, assessed automatically on 2026-11-05. This is a **draft for
the pilot**, not a published or approved municipal statement (tillgänglighetsredogörelse). The responsible unit, contact
route, publication date and reporting routine are filled in before publication, and the published statement is in
Swedish (the repository's documents are in English).

## Target and verified scope

The design target is WCAG 2.2 AA (including the 24 px target size of criterion 2.5.8), which contains the legal baseline
of DOS-lagen (2018:1937) and EN 301 549 (WCAG 2.1 AA). AAA rules are not checked. We **do not claim WCAG certification
or full conformance**: automated checks cannot decide every criterion.

Verified automatically (Playwright against a running stack, `tests/e2e`, admin design §10.1, §11.4):

- **Pilot journey** (admin design §5.4 steps 1–13): a department administrator creates a knowledge base, adds a website,
  checks and previews it, fetches it, adds a test question, runs the evaluation, publishes the knowledge base, creates,
  tests and publishes an assistant and makes it available in Open WebUI. Run twice: with the mouse, and with the
  **keyboard only** (Tab, Enter, Space and the arrow keys; tabs by the arrow keys as the ARIA tabs pattern says).
- **axe** rules for WCAG 2.2 A/AA on every routed page (list pages, platform objects, one knowledge base, source, run and
  assistant page, the no-access and not-found pages) in the three themes light, dark and Lumen, and on the create and
  add-source drawers.
- **Target size** (2.5.8) on the same pages.
- **Reflow** at 320 px and at 200 % text size on every page.
- **Keyboard and focus**: one main landmark and the main menu, focus to the page heading after navigation, the theme
  radio group by the arrow keys, the skip link on small screens.
- **Forced colours and reduced motion** keep a visible focus indicator.
- **No Content-Security-Policy violations** during the pilot journey and the page scans.

Language: English, with Swedish names marked `lang="sv"`. **No screen reader (NVDA, VoiceOver) has been run against the
admin UI yet**; nothing in this document should be read as saying such a check was made. The NVDA pass of the pilot
journey follows soon after the pilot starts: the statement was approved for the pilot on 2026-11-06 with this check
still open (`plans/to-verify.md` #96). The sign-in provider (Keycloak, later Entra ID) needs its own review.

## Remaining manual checklist

| Check | Status |
|---|---|
| NVDA with Firefox or Chrome | Pilot journey: headings, landmarks, tables, live regions (job progress, "the answer is ready"), dialogs and focus return: **not done yet**; approved for the pilot without it (2026-11-06), to be run soon |
| Keyboard only | The pilot journey is automated; the other role journeys (pipeline editor, platform administrator) remain a manual check |
| Focus appearance, focus not obscured | All focus targets in drawers, dialogs and scrolled states (2.4.11, 2.4.13): manual check remains |
| Zoom and text spacing | 200 % text and 320 px are automated; real browser zoom to 400 % and 1.4.12 spacing remain |
| Authentication | MFA without cognitive tests, the identity provider's help and errors: reviewed with the identity provider |
| Session time | Warning and explicit extension, also with assistive technology and unsaved forms: manual check remains |
| Language and understanding | Plain language, instructions, errors, consequence texts before destructive actions: user test needed |
| Colour and contrast | Real states, hover, disabled, Windows high contrast: manual check remains |
| Pointer and touch | 24 x 24 px checked automatically; nothing needs drag (rows move with buttons, files by a file input) |

Criteria about pre-recorded or live audio and video do not apply: the admin UI has no such content. Record a changed
applicability if media is added.

## Reporting barriers and supervision

Before the pilot: the responsible unit, an accessible e-mail, phone or form, the response routine, alternative formats
and a link to DIGG's reporting route are filled in. **No invented contact details are given in this draft.** During the
pilot test, use the municipality's existing helpdesk.

See [WCAG 2.2](https://www.w3.org/TR/WCAG22/) and [DIGG](https://www.digg.se/digital-tillganglighet).
