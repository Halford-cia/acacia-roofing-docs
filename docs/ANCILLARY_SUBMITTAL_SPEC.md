# Ancillary Submittal — Spec (v1)

**Status:** Locked v1 — ready for CC scaffold
**Owner:** Hank Alford
**Related TODO:** #15 (Repair / non-roof job submittal flow)
**Source-of-truth doc location after commit:** `docs/ANCILLARY_SUBMITTAL_SPEC.md`
**Locked:** 2026-05-26

---

## 1. Purpose & scope

A sibling intake to `job-submittal` for non-roofing work: gutters, fencing, siding, painting, drywall. Reps use it to (a) request a bid from the office or (b) schedule work whose price is already known (covered by the adjuster's report, or self-priced by an experienced rep). Schedules land on the same Acacia crew calendar as roof installs.

**v1 trades:** gutters, fencing, siding, painting, drywall, + "Other" free-text.

**Out of v1 scope:** automated material ordering, per-trade calendar separation, financial linkage back to parent roof job's Contract Amount, chained questionnaires (e.g. drywall → painting).

---

## 2. Architecture

**Project:** new standalone Apps Script project, repo path `ancillary-submittal/`. Sibling to `job-submittal`. Does not share code with the roof intake form; couples only via spreadsheet IDs in Script Properties.

**Why separate project (not a toggle on job-submittal):**
1. Matches the existing modular pattern across all 10+ projects.
2. `job-submittal/Code.js` is already ~4,900 lines and at the edge of a single CC context window.
3. Different deploy cadence — roof intake is mature and fragile (PWI, ACV, NRD history); ancillary is greenfield and will iterate fast in early weeks.

**Deployment topology:** `USER_DEPLOYING` + `DOMAIN` (matches `bad-lead-app` and `job-submittal`); rep auth via `@acaciaroofs.com` domain check. _**Phase 1 spec correction (2026-05-27):** locked v1 spec said USER_ACCESSING; this was a misremembering of job-submittal's actual topology. Implementation runs as deployer (henry@) so reps can write to admin-owned sheets without direct access._

**Configuration:** Script Properties (intentional deviation from `bad-lead-app` / `job-submittal` hardcoded `CONFIG` pattern). Required keys: `MASTER_TRACKER_SHEET_ID`, `CREW_CALENDAR_ID`, `ANCILLARY_PHOTOS_FOLDER_ID`, `OFFICE_EMAIL`, `IS_TEST_MODE`. See `ancillary-submittal/Constants.js` header.

**Coupling points:**
- Reads Master Job Tracker (parent-job linkage dropdown) — same spreadsheet ID as job-submittal.
- Writes new "Ancillary Jobs" tab on Master Job Tracker.
- Writes to Acacia crew calendar (`c_a1f375a2cfce8a53ed9e73befd49b3ccb03e9e9cc896dabf47fb8527d6b3c36b@group.calendar.google.com`).
- Photo uploads land in dedicated Drive folder (`1UMS0Tjl90ES8jls9lcYAXwmRyoM5Rx7u`); one subfolder per submission, named `[Trade] - [Address] - [yyyyMMdd-HHmmss]`.
- Does NOT write Rep Job Costs — see §8.

---

## 3. User flows

### 3.1 Path A — Adjuster's report covers the work

Most common case. Side work is itemized on the same adjuster's report as the roof job.

1. Rep selects **Trade** (gutters, fencing, siding, etc.)
2. Rep selects **Submission Type: Ready to Schedule**
3. Rep links to **parent roof job** from a dropdown (Master Tracker rows from the last ~90 days, filterable). Customer name/address/phone/email auto-populate (copy-at-submit, no live linkage).
4. Rep fills the **trade questionnaire** (§4).
5. Rep uploads photos.
6. Rep picks an **Install Date** (≥10 days out, same lead-time rule as roof).
7. **Pricing source = "Adjuster Report"** — no contract amount entered by rep; office reconciles from parent adjuster report on their end (per §11 lock-decision 4).
8. Submit → row written to Ancillary Jobs tab, calendar event created, rep notification email. (No Rep Job Cost row — see §8.)

### 3.2 Path B — Office bid needed

Rep doesn't know what the work costs; office needs to price it.

1. Rep selects Trade.
2. Rep selects **Submission Type: Need Bid**.
3. Rep optionally links a parent job (for contact info convenience) OR enters customer info standalone.
4. Rep fills trade questionnaire.
5. Rep uploads photos.
6. Submit → row written with Status = `Bid Requested`. **No calendar event yet.** Notification email to office (Kelcie + Hank).
7. Office reviews row in Ancillary Jobs tab, types **Bid Amount** in col 15 + sets Status to `Bid Sent`.
8. **Status-edit trigger** auto-fires rep notification email with the bid amount.
9. Rep gets bid via email AND can ask Brain ("what's the gutter bid for Smith St?") — Brain reads the Ancillary Jobs tab.
10. When rep is ready to schedule, they return to the web app, click **"Schedule existing bid,"** select the row, pick Install Date → calendar event created, Status → `Scheduled`.

### 3.3 Path C — Rep self-prices

Experienced rep knows pricing and is ready to schedule immediately.

1. Same as Path A, except rep manually enters **Contract Amount** instead of inheriting from adjuster's report.
2. **Pricing source = "Rep Self-Priced"**.
3. All other steps identical to Path A.

---

## 4. Trade questionnaires

All trades share these **universal fields**:
- Scope summary (free-text, required)
- Photos (multi-upload, ≥1 required for Ready to Schedule; ≥3 recommended for Need Bid)
- Crew notes (free-text, optional)
- Preferred schedule window (Ready to Schedule only)

### 4.1 Gutters

_**Phase 2 refinement (2026-05-27):** dropped downspout size (derivable from gutter size: 5"→2x3, 6"→3x4) and fascia repair (never handled by gutter crew). Field list below reflects implementation._

- **Gutter linear feet** (numeric)
- **Downspout linear feet** (numeric)
- (Form computes Total LF = gutter LF + downspout LF for display + crew notes)
- Downspout count
- Material: aluminum / copper / steel / vinyl
- Size: 5" / 6" / 7"
- Color: white / brown / bronze / black / gray / custom (free-text)
- Gutter guards? Y/N + type (if Y)
- Tear-off existing? Y/N

### 4.2 Fencing

_**Phase 2 spec correction (2026-05-27):** v1 spec had a 4-option work_type select with a "Pressure wash + restain only" shortcut path. Phase 2 implementation refines this to a 9-option multi-select work scope with 5 conditional field groups, drops the wood-only "Finish" select, drops the per-gate walk/drive distinction (collapsed to a single free-text "Gates" field), and drops the auto-set "Tear-out existing" coupling (work scope multi-select covers it). Table below reflects the implemented questionnaire._

_**Phase 2 refinement (2026-05-27):** post type field dropped — fence posts are always concrete-set in practice._

- **Work scope** (multi-select checkbox — at least one required):
  - Replace pickets only
  - Replace posts only
  - Replace pickets + posts
  - Full tear-out + new install
  - Repair only
  - Pressure wash only
  - Stain / restain
  - Paint
  - Other (free text — detail required if checked)
- **Install field group** (shown if any of: pickets-only / posts-only / pickets-and-posts / full tear-out):
  - Linear feet (numeric)
  - Height: 4' / 6' / 8'
  - Material: cedar / pine / treated pine / vinyl / chain-link / wrought iron / composite
  - Style: privacy / picket / split-rail / shadowbox / board-on-board
  - Gates: count + sizes (free text — e.g. "2 gates, one 4-ft walk-through, one 10-ft drive-through")
  - HOA approval status: Approved / Pending / Not Required / Unknown
- **Condition group** (shown if stain or pressure-wash-only):
  - Current stain color (free text)
  - Condition notes (free text)
- **Stain brand / color preference** (shown if stain selected)
- **Paint brand / color preference** (shown if paint selected)
- **Repair description** (shown if Repair only selected — required when it is the sole selected scope)

### 4.3 Siding

_**Phase 2 spec correction (2026-05-27):** stories enumerated as 1 / 2 / 3+ (locked spec said "number of stories" without enumeration). Otherwise matches the locked spec._

- Square footage (numeric)
- Number of stories: 1 / 2 / 3+
- Material: vinyl / Hardie / wood / metal / engineered wood
- Profile: lap / shake / board-and-batten / panel
- Color (free-text)
- Trim work needed? Y/N (Y → trim description)
- Soffit / fascia included? Y/N
- Window / door wrap? Y/N
- Tear-off existing? Y/N (Y → pre-1978 lead paint flag required: Yes / No / Unknown)

### 4.4 Painting

_**Phase 2 spec correction (2026-05-27):** scope rendered as radio (single-select Interior/Exterior/Both) instead of free-text. Otherwise matches the locked spec._

- Scope: Interior / Exterior / Both (radio)
- Square footage (numeric)
- Surfaces (multi-select checkbox): walls / trim / doors / ceilings / exterior body / shutters
- Paint brand preference: Sherwin Williams / Behr / Customer choice / No preference
- Color count: Single / Multi-room
- Color codes (free-text, optional — "if known")
- Number of coats: 1 / 2 / 3
- Prep work (multi-select checkbox): pressure wash / scrape / patch / prime
- Trim color same as body? Y/N (N → trim color free-text)

### 4.5 Drywall

_**Phase 2 spec correction (2026-05-27):** "Other" cause requires explicit cause notes. Paint-match notes hint added but field stays optional. Otherwise matches the locked spec._

_**Phase 2 refinement (2026-05-27):** dropped "Type of work" and "Cause" fields — cause is implied by photos (hole vs water damage), and type-of-work isn't operationally useful at submit time. Removed the conditional cause-notes field. Drywall now has no required tradeFields validation._

- Square footage affected (numeric)
- Number of rooms affected (numeric)
- Texture match: Knockdown / Orange peel / Smooth / Popcorn / None
- Paint to match? Y/N (Y → paint match notes, optional; drywall crew handles paint themselves)
- Insulation work needed? Y/N

### 4.6 Other

_**Phase 2 spec correction (2026-05-27):** trade name is required (used as the calendar event title label). Photos requirement uses the same universal rules as other trades (≥1 for Ready to Schedule, ≥3 for Need Bid)._

- Trade name (free text, required) — used as the calendar title label (e.g. "Window Replacement @ 123 Smith St")
- Rough materials / measurements / specs (free text, optional)
- Scope summary (universal field above — main scope capture for Other)
- Photos (universal rules)

**Known gap (TODO #40):** when trade = `other`, the calendar event title uses the rep-typed trade name (e.g. `Window Replacement @ ...`) which does not match any of `ANCILLARY_BLOCKING_KEYWORDS` (`GUTTERS, FENCE, SIDING, PAINTING, DRYWALL`). Subsequent ancillary submissions won't see the "other" event as a date blocker. Dispatch can insert a manual `UNAVAILABLE` event if there's a real conflict. Low-volume gap; revisit when "Other" trade frequency justifies a stable prefix (e.g. `[ANC]`).

---

## 5. Data model — "Ancillary Jobs" tab on Master Tracker

_**Phase 1 spec correction (2026-05-27):** schema reduced from 23 → 21 columns. Added "Rep Email" at col 3 (stable key per CLAUDE.md gotchas around Rep Name format/casing inconsistency). Dropped "Rep Share" and "Acacia Share" — ancillary jobs are standalone commission projects (see §8). Dropped "Trade-Specific Crew Title" — derived from Trade + Customer Address at calendar-write time, not stored. Table below reflects the implemented schema._

| Col | Field | Type | Notes |
|---|---|---|---|
| 1 | Submitted At | datetime | Auto |
| 2 | Rep Name | string | From auth |
| 3 | Rep Email | string | From auth — stable key; reads/writes key off this, not Rep Name |
| 4 | Trade | enum | gutters/fencing/siding/painting/drywall/other |
| 5 | Submission Type | enum | Need Bid / Ready to Schedule / Schedule Existing Bid |
| 6 | Status | enum | Bid Requested / Bid Sent / Ready / Scheduled / Complete / Cancelled |
| 7 | Linked Parent Job | string | Master Tracker row ref ("Row 234") or "Standalone" |
| 8 | Customer Name | string | |
| 9 | Customer Address | string | |
| 10 | Customer Phone | string | |
| 11 | Customer Email | string | |
| 12 | Scope Summary | string | Free-text from rep |
| 13 | Trade-Specific Fields | JSON | All questionnaire answers as JSON blob |
| 14 | Photo Folder URL | string | Drive folder link |
| 15 | Source of Pricing | enum | Adjuster Report / Office Bid / Rep Self-Priced |
| 16 | Bid Amount | currency | Office types this in Path B |
| 17 | Bid Sent Date | date | Auto-stamped when Status flips to Bid Sent |
| 18 | Contract Amount | currency | What customer agreed to — what Commission Builder will compute against |
| 19 | Install Date | date | |
| 20 | Calendar Event ID | string | For later edit/cancel |
| 21 | Crew Notes | string | |

**Trade-specific fields stored as JSON, not 40+ sparse columns.** Adding/removing fields per trade is a code change, not a schema migration. Brain parses JSON for queries. The "Scope Summary" column gives office a human-readable glance without opening the JSON.

**Calendar event title is derived data**, not stored on the row. Computed at calendar-write time from Trade + Customer Address as e.g. `Gutters @ 123 Smith St` (or `[TEST] Gutters @ ...` when `IS_TEST_MODE=true`). If a downstream consumer needs the title from a row, recompute via the `crewTitleFor_` helper in `Code.js`.

---

## 6. Status state machine

```
[New Bid Request] → Bid Requested
                       ↓ (office enters Bid Amount)
                    Bid Sent
                       ↓ (rep returns + schedules)
                    Scheduled

[Ready to Schedule submit] → Scheduled

                    Scheduled
                       ↓ (job done, manual flip)
                    Complete

Any state → Cancelled (manual flip)
```

`Ready` state shown in schema is reserved for future use (e.g. office-approved bid awaiting rep action). Not used in v1.

---

## 7. Calendar integration

**v1 rule: everything blocks everything.** Roof installs and ancillary jobs share the Acacia crew calendar with no labor-pool distinction. One job per day until volume requires the split. (Per-trade calendar separation is logged as a future TODO — see §11.)

**Event format** (parallel to existing roof convention `Install @ [address]`):
- `Gutters @ [address]`
- `Fence @ [address]`
- `Siding @ [address]`
- `Painting @ [address]`
- `Drywall @ [address]`
- `[Other-trade-name] @ [address]`

**Event description** includes: rep name, trade, scope summary, key questionnaire fields (linear feet / sqft / material — whatever's most relevant per trade), customer contact, crew notes.

**Conflict handling:** rep's Install Date picker shows existing calendar bookings; submit fails if selected date is taken (matches roof intake behavior).

---

## 8. Commission

_**Phase 1 spec correction (2026-05-27):** v1 spec called for a Rep Job Costs row write at submit. This was wrong — ancillary jobs are **standalone commission projects**, not rep-job-cost incidents (the Rep Job Costs sheet records per-incident costs that get deducted from a rep's commission inside a parent project; ancillary jobs ARE the project). Commission generation moves downstream to Commission Builder (acacia-portal), which will consume the Ancillary Jobs tab's Contract Amount column directly._

**Phase 1 implementation:** ancillary commissions are computed **downstream via QBO data only** — no automated writes from this project to Rep Job Costs or any commission spreadsheet. Cols 19/20 (Rep Share, Acacia Share) removed from the schema as a result.

**Future TODO:** extend Commission Builder (acacia-portal) to read the Ancillary Jobs tab so ancillary commissions get computed alongside roof jobs at commission-generation time. See `docs/TODO.md`.

**Tier 1/2/3 fee structure** still applies (downstream, not here):
- Tier 1: 15% office fee on contract amount
- Tier 2 ($50K cumulative): 8% office fee
- Tier 3 ($250K cumulative): 0% office fee

**`onAncillaryStatusEdit` responsibility #3** (Contract Amount populated on a Path A row) was originally specced to write a Rep Job Costs row. Phase 1 wrap-up: this branch is now a logged no-op — the slot is retained so future Commission Builder integration has a clean place to drop in.

---

## 9. Brain integration

**New accessor:** `getAncillaryJobsByAddress(address)` and `getAncillaryJobsByRep(repEmail)` (parallel to existing `getRepJobCosts`).

**System prompt extension:** Brain's prompt updated so it knows the new tab exists and what's queryable ("ancillary work like gutters, fencing, siding, painting, drywall — query by customer address or rep").

**Sample queries Brain should handle after this lands:**
- "What's the gutter bid for 305 Arabian Colt Rd?"
- "Show me all my ancillary jobs awaiting bid."
- "What ancillary work is scheduled this week?"

---

## 10. Rep "My Ancillary Jobs" view

Lightweight selection-list UI in the same web app, accessible via top-nav toggle. Pattern mirrored from `bad-lead-app` rep selection screen.

**Columns shown:**
- Trade
- Customer address
- Status
- Bid Amount (if Bid Sent) or Contract Amount (if Scheduled/Complete)
- Install Date (if scheduled)

**Filters:** All / Outstanding (Bid Requested + Bid Sent) / Scheduled / Complete (last 30 days).

Brain remains the primary query interface; the list view is for at-a-glance browsing.

---

## 11. Locked decisions (resolved during 2026-05-26 spec session)

1. **Photo storage:** new "Acacia Ancillary Photos" Drive folder, with one subfolder per job. Subfolder naming convention finalized during scaffold (proposed: `[Trade] - [Address] - [SubmittedAt]`). Subfolder URL written to col 13 Photo Folder URL.
2. **Bid notification routing (Path B):** `office@acaciaroofs.com` only. Not Kelcie individually, not Hank individually, not Josh.
3. **"Schedule Existing Bid" entry point:** dropdown picker showing rep's outstanding `Bid Sent` rows. No bid-reference-number entry.
4. **Path A pricing inheritance:** Contract Amount left blank at submit; office reconciles from parent adjuster report on their end. Auto-extraction from parent deferred (see §12).
5. **Status edit trigger location:** `onAncillaryStatusEdit` lives in the new `ancillary-submittal` project. Ancillary Status enum is independent of any roof Master Tracker status field — they don't share a state machine.
6. **Phase 1 smoke-test outcome (2026-05-27):** all three paths (A/B/C) plus Schedule Existing Bid verified end-to-end on TEST deployment @4 after the date-serialization fix landed. Bug surfaced (raw `Date` objects in `outstandingBids[i].submittedAt` broke `google.script.run` IFRAME serialization once any row reached `Bid Sent` status); fix stringifies via `Utilities.formatDate` in `getOutstandingBidsForRep_`. All four prereqs (Script Properties, bootstrap, trigger install, smoke test) re-verified clean before Phase 1 wrap-up patches.

---

## 12. Known future TODOs (for `docs/TODO.md` after spec approval)

- **Per-trade calendar separation** (v2): once subcontractor labor pools become a scheduling concern, split Acacia crew calendar from per-trade calendars or add a labor-pool tag on events.
- **Automated material ordering for ancillary** (v2): mirror roof's IKO/Tamko/etc. supplier routing for gutters, siding, etc. — defer until volume justifies.
- **Live parent-job linkage** (v3): edits to parent customer info propagate to ancillary rows. Deferred indefinitely; copy-at-submit is sufficient.
- **Chained questionnaires** (v3): drywall→painting cascading if a separate painting crew is brought in. Currently moot because drywall crew handles paint-to-match.
- **Path A extraction-from-parent**: pull ancillary line items from parent adjuster's report's extracted data automatically. Deep rabbit hole; deferred.

---

## 13. Deploy plan (post-scaffold)

1. CC scaffolds `ancillary-submittal/` project structure.
2. Build & deploy in test deployment (separate deploymentId).
3. Manual end-to-end test by Hank: one bid-request submission, one ready-to-schedule submission, one schedule-existing-bid flow.
4. Kelcie heads-up email — explain the new tab on Master Tracker and the bid-entry workflow.
5. Add new project to `CLAUDE.md` Project Inventory with deploymentId.
6. Rep training: short Loom video or written walkthrough covering all three paths.
7. Promote to production deployment.
8. Monitor first week of real usage; iterate on questionnaire fields based on rep + office feedback.

---

## 14. Definition of done (v1)

- [x] All three paths (A/B/C) function end-to-end without manual intervention. _(Phase 1 — gutters only; @5 TEST deployment.)_
- [x] Calendar events created and visible to dispatch.
- [ ] ~~Rep Job Costs commission entries match roof intake math.~~ _Superseded — see §8 spec correction; ancillary commissions move to Commission Builder downstream._
- [ ] Brain answers ancillary queries. _Phase 4._
- [ ] "My Ancillary Jobs" view loads for any rep on any device. _Phase 2/3._
- [ ] Kelcie has used the bid-entry flow on 1+ real bid without confusion. _Pending production deploy._
- [x] Spec doc updated to reflect any v1 deviations from this draft. _(This update, 2026-05-27.)_

**Phase 1 scope:** gutters only. Phase 2 adds fencing, siding, painting, drywall, other. Phase 4 adds Brain. "My Ancillary Jobs" view + production cutover pending future phases.
