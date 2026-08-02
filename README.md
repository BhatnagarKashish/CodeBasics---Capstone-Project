# AtliQ Sales Intelligence — prototype build

A working implementation of the core v1 loop from `AtliQ_Sales_Intelligence_PRD.pdf`
(unified data capture → AI intelligence layer → proactive notifications →
team visibility → empty-state handling), built against the three real
data sources provided: `crm_export copy.csv`, `emails copy/`, and
`meeting_notes copy/` — plus several rounds of UX iteration on top of that
core loop: filters (including a shared date range), pagination, an
actionable notification strip, confidence-scored AI insights, an inline
Quick Analytics view, and a few fixes that came out of reviewing the
output along the way.

## Run it

No dependencies beyond the Python 3 standard library.

```bash
python3 main.py
open output/dashboard.html   # or just double-click it
```

That's it — `output/dashboard.html` is a single self-contained file
(data + CSS + JS inlined). No server, no build step, no external
requests.

## What it does, mapped to the PRD

| PRD section | Where it lives |
|---|---|
| 8.1 Multi-channel ingestion | `atliq/ingest_crm.py`, `ingest_emails.py`, `ingest_meetings.py` |
| 8.1 Empty-state rule | `atliq/config.py` (`CHANNELS_WITHOUT_DATA`) + the channel strip in the dashboard header |
| 8.2 Unified lead dashboard | `atliq/dashboard_builder.py` → lead table, with search + stage/owner/channel/date-range filters and pagination (15 rows/page) |
| 8.3 AI intelligence layer (notifications) | `atliq/ai_notifications.py` |
| 8.3 AI intelligence layer (historical correlation) | `atliq/ai_correlation.py` — insight cards now carry a match-confidence % and, where relevant, a cross-sell probability % (see below) |
| 8.4 Notification centre | bell icon → drawer, grouped/coloured exactly per the PRD's table, plus a "Needs action today" hero strip on the main view (see below) |
| 8.5 Lead detail view | click any row → drawer: AI summary (now a full dated history, not just the latest touchpoint), insight card with confidence %, action items, timeline, team notes |
| 8.6 Empty states | exact PRD copy used verbatim throughout (`dashboard_builder.py::EMPTY_STATE_MESSAGES`, JS empty-state branches) |
| §9 "basic counts" pipeline analytics | Quick Analytics — an inline monthly-trend + channel-breakdown view (see below); stays inside the §12 out-of-scope line by sticking to descriptive counts, no forecasting |

## The interesting part: entity resolution

The PRD's entire value proposition is "stop making a person manually
figure out which conversation belongs to which lead." `atliq/resolve.py`
does that automatically, in priority order:

1. **Explicit lead ID** — meeting notes in this dataset sometimes cite
   `L-1003` directly; if a note mentions several lead IDs (e.g. the
   internal pipeline-review notes), it's attached to *all* of them.
2. **Exact contact-email match** — an email thread with a client's
   known address.
3. **Company alias match** — the CRM company's first significant word
   (`Meridian Healthcare` → `meridian`) matched against the interaction's
   **filename and title only**, deliberately not full body text — bodies
   often mention other companies for comparison ("heard about the
   Sunrise Foods portal from a distributor they share"), which would
   otherwise misattribute the whole note.
4. **Off-CRM gap detection** — if nothing matches but there's a real
   external counterpart, or the text explicitly says a company "is not
   in the CRM" (a convention the dataset uses itself), it's surfaced as
   a synthetic lead rather than silently dropped. The pipeline finds 5:
   **Lumino Fitness, Harbor & Finch, PixelWorks, Nimbus Lending,
   Ridgeline Foods** — none of which exist in the CRM export.

It also flags CRM messiness the PRD is explicitly designed to
compensate for: two pairs of likely-duplicate CRM rows (Meridian
Healthcare/Meridian Health LLC, and GlobalMart entered twice), and
per-lead CRM completeness against required fields. Both are now
surfaced in the UI, not just computed — see below.

## The AI layer is a deterministic rule engine, on purpose

`ai_notifications.py` and `ai_correlation.py` implement the PRD's
"AI-generated" notifications and historical-correlation insight cards
as explainable, swappable rule engines rather than calling an external
model — the PRD itself leaves the AI provider as an open question
(§13, Q1/Q3). Each is a single narrow entry point
(`generate_notifications()`, `build_insight()`) so a real LLM-backed
version can replace the internals later without touching ingestion,
resolution, or rendering. Concretely:

- **Overdue follow-up / Unanswered proposal** — true last-contact date
  computed across *all* channels (not just the CRM's often-stale
  `last_contact_date`), compared against the PRD's 7-day thresholds.
  Never fires for a **closed** (Won/Lost) deal — see below.
- **Revisit signal / Cross-sell signal** — regex-based phrase detection
  tuned to how this dataset actually writes these moments (e.g. "check
  back", "New opportunity, not logged anywhere").
- **Deadline approaching** — parses both structured `- [ ] ... — Owner —
  by 15 Jul` checkboxes in meeting notes and inline prose commitments
  ("security questionnaire due 24 Jul").
- **Historical correlation** — scores open leads against closed
  (Won/Lost) CRM rows on industry classification, service line, and
  text-token overlap. A match only counts once it clears
  `MATCH_THRESHOLD = 5` in `ai_correlation.py` — calibrated so that
  keyword overlap alone (capped at 4) can never trigger a match by
  itself; it needs industry+service agreement, or one of those plus
  real keyword overlap. (Earlier this threshold was `score > 0`, which
  matched 100% of leads — not a meaningful signal. It now matches 31 of
  45.) Leads that don't clear it get the PRD's exact fallback copy: "No
  similar past deals found — this may be a new opportunity type for
  AtliQ." The matched card also shows a **match confidence %**
  (`score / MAX_SCORE`, so a bare-threshold match reads ~50% and a
  perfect industry+service+keyword hit reads 100%).
- **Cross-sell probability** — when a cross-sell phrase is detected
  (`ai_notifications._CROSS_SELL_TRIGGER_RE`), the insight card also
  shows a probability % via `ai_correlation.estimate_cross_sell_confidence()`:
  a 45% base for the signal itself, +15% per **Won** deal AtliQ already
  has in that same industry (capped at +35%, so max 90% — deliberately
  never near-certain, since a text-pattern hit is a lead worth a human
  look, not a confirmed deal). This is the literal "same-domain" signal
  requested when the feature was added: it only rewards *proven*,
  successfully-closed experience in that industry, not just any past
  activity there.

## Dashboard UX

Beyond the core PRD loop, the dashboard got a pass aimed at "can someone
act on this in one view" rather than just "is the data all there":

- **Filters** — search, stage, owner, a date-range filter (All time /
  Last 30 / Last 90 / This quarter / custom, scoped to `created_date`),
  an "Off-CRM gaps only" / "Possible duplicates only" checkbox pair, and
  clickable channel chips that filter the table to leads touched by
  that channel.
- **Pagination** — 15 leads per page; any filter change resets to page 1.
- **Quick Analytics** — a header button toggles an inline section (same
  pattern as the hero strip below: lives in the page flow between the
  toolbar and the table, not a popup; open/closed state remembered per
  browser). Two charts, hand-rolled SVG with no charting library, built
  against the dataviz skill's rules: a **monthly lead-creation trend**
  (line, zero-filled across every month in range so gaps show as real
  gaps) and a **leads-by-channel** breakdown (horizontal bars — a
  lead's channel is whichever channel first captured it: the channel of
  its earliest resolved interaction, or "CRM only" if it has none).
  Both bars and line deliberately use **one hue**, not a rainbow — per
  the skill's own rule, coloring nominal categories by hue when there's
  only one measure just re-encodes what the bar length already shows.
  Both charts share the toolbar's **date range and owner filters**
  rather than getting their own separate copies, specifically so the
  table and the charts can never quietly disagree about what's in
  scope — verified end-to-end (filtering to a single owner shows the
  same lead count in the toolbar-driven table and in the channel
  chart's bar totals). Each chart has hover tooltips, a crosshair on
  the line chart, and a "Show as table" fallback.
- **"Needs action today" hero strip** — the notification centre's red
  (immediate) items, surfaced directly under the header instead of
  buried in a drawer, with a pulsing indicator on the bell and on each
  card. Every notification (hero, drawer, or a lead's own detail view)
  has real buttons: **View lead** and **Acknowledge**. Acknowledging
  persists to `localStorage` so it stays dismissed on reload, with a
  "show N acknowledged" undo link in the notification centre. The hero
  strip itself can be hidden/shown via a toggle (also remembered).
- **Clickable stat tiles** — "Off-CRM gaps found" jumps straight to the
  filtered table; "Possible duplicate CRM entries" opens a side-by-side
  comparison drawer (every field, from both records) with a recommended
  fix, since this prototype can flag duplicates but can't merge CRM rows
  for you.
- **Closed-deal handling** — a Won/Lost lead gets its own dark-green
  "closed" indicator instead of the red/amber/green staleness badge
  (which would misleadingly read as "needs attention"), and is fully
  excluded from every notification type, including the historical
  correlation card in the global notification feed (it's still shown in
  the lead's *own* detail view, for historical reference).
- **"Connect a channel" panels** — clicking an empty-data chip
  (WhatsApp/LinkedIn/Cold Calls) opens a static panel describing what
  that channel would capture and what a real integration would require
  (API/provider, credentials, consent) — see `CHANNEL_CONNECT_INFO` in
  `render_html.py`. This is intentionally not a working OAuth flow; it
  exists to show the roadmap without pretending the prototype does
  something it doesn't.

## Known limitations (prototype scope)

- **Quick Analytics only inherits date range and owner from the
  toolbar** — search, stage, and the gaps/duplicates checkboxes do not
  scope the charts. Deliberate (a channel-trend chart under "search:
  Acme" has no obvious meaning), but worth knowing if a chart's total
  doesn't match what's currently typed into the search box.
- **"Channel" in the leads-by-channel chart means data-capture
  channel** (Email vs. Meeting Notes vs. CRM-only), not the CRM's own
  `source` field (Referral/LinkedIn/Conference/Website/Cold Outreach) —
  i.e. which system first has evidence of the lead, not how the lead
  was acquired. Both are legitimate "channel" readings; this one was
  picked to match how the feature was originally described. The CRM
  `source` breakdown would be an easy second chart if lead-origin
  attribution turns out to be what's actually wanted.
- **Referral emails double-count.** When an existing contact
  introduces a *different* company (e.g. PayTrack's Shreya introducing
  Nimbus Lending), the email correctly creates a Nimbus Lending gap
  *and* stays in PayTrack's own timeline — so PayTrack's AI summary can
  occasionally surface an event that's actually about the referral,
  not PayTrack's own deal. Fixable with a proper subject classifier;
  not attempted here given the scope.
- **Team notes and acknowledged notifications are per-browser only**
  (`localStorage`), not shared — PRD Goal 5 (team continuity) needs a
  real backend; this is a UI placeholder for what that would feel like.
- **"Connect a channel" panels are informational only.** No live
  OAuth/webhook flow exists behind them — see Dashboard UX above.
- **WhatsApp, LinkedIn (as a message channel) and Cold calls** have no
  data in this dataset, so they're wired in as connected-but-empty
  sources to exercise the PRD's empty-state rule (8.1) rather than
  omitted outright.
- Industry classification and keyword lists in `ai_correlation.py` are
  heuristic and dataset-specific; extending the dataset (per the data
  dictionary's own suggestion) will need matching keyword additions.

## Project layout

```
atliq/
  config.py            paths, thresholds, reference "today" (2026-07-10)
  models.py             shared dataclasses
  ingest_crm.py          CRM CSV parser
  ingest_emails.py        email-thread markdown parser
  ingest_meetings.py       meeting-note markdown parser
  resolve.py              entity resolution / gap & duplicate detection
  ai_notifications.py      proactive notification rules (PRD 8.3/8.4)
  ai_correlation.py        historical correlation engine (PRD 8.3)
  ai_summary.py            per-lead dated history, stage, next step
  dashboard_builder.py      assembles it all + empty-state handling
  render_html.py           single-file HTML/CSS/JS renderer — dashboard UX, filters, and the hand-rolled SVG charts
main.py                  CLI entrypoint
output/dashboard.html    generated artifact (not committed logic, just output)
```
