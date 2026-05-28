# Contract Builder — Specification

**Locked:** 2026-05-28 (initial draft, Phase 1A scope only)
**Project:** contract-builder (standalone Apps Script web app, sibling to ancillary-submittal and job-submittal)
**Replaces:** Roofr (~$400/mo, ~$4,800/yr savings)

## 1. Purpose

Generate branded contract PDFs for remote customer signing. The rep meets a new customer, types fields into the web app, picks a template, and the app produces a filled Google Doc that the rep reviews and sends via Google's native eSignature feature. Customer signs remotely; the sealed PDF + audit trail land in a per-rep Drive folder.

In-person contracts continue to use paper. This tool is for the remote-signing case only.

## 2. Architecture

- **Project type:** Standalone Apps Script project deployed as web app
- **Owner:** henry@acaciaroofs.com
- **Topology:** `executeAs: USER_DEPLOYING`, `access: DOMAIN`
- **Auth:** Rep authenticates via @acaciaroofs.com login (`Session.getActiveUser().getEmail()`)
- **Configuration:** Script Properties (no hardcoded CONFIG object — matches ancillary-submittal pattern)

**Required Script Properties:**

- `CONTRACT_ROOT_FOLDER_ID` — Drive folder ID for "Acacia Contracts" root
- `MASTER_TRACKER_SHEET_ID` — `1r3vrE6uGXNlEforLs3znzo1BpsJbeK89hZ9HH9EA0Cs`
- `TEMPLATE_CONTINGENCY_DOC_ID` — Google Doc ID for the contingency template (Phase 1A)
- `TEMPLATE_CASH_PROPOSAL_DOC_ID` — Google Doc ID for the cash proposal template (Phase 1B, optional in Phase 1A)
- `OFFICE_EMAIL` — office@acaciaroofs.com
- `IS_TEST_MODE` — true / false

**Coupling points:**

- Master Tracker spreadsheet (new "Contracts" tab created on first run)
- Drive: contract root folder + per-rep subfolders (auto-created on first use per rep)
- Google Doc templates (one per contract type, stored in a templates subfolder)
- No coupling to job-submittal, ancillary-submittal, or final-invoice-calculator

**OAuth scopes:** spreadsheets, drive, docs, scriptapp, userinfo.email

## 3. User flow (Phase 1A: contingency only)

1. Landing → rep clicks "New contract"
2. First-time onboarding (per rep, one-time): "What's your full name as you want it to appear on signed contracts?" → stored as Script Property `REP_FULLNAME_<email>`. Subsequent visits skip this step.
3. Contract type picker → currently shows only "Contingency Agreement" (Phase 1A). Phase 1B adds "Cash Proposal" as a sibling option. Trade-picker pattern from ancillary-submittal.
4. Contingency form → fills in customer fields, scope blanks, scope checkboxes. Submit.
5. Generate → app copies template Doc, replaces placeholders, saves to per-rep folder, logs row on Master Tracker Contracts tab.
6. Review screen → shows: "Contract generated. Click below to review and send for customer signature." with a link to open the filled Doc in a new tab.
7. Rep manually: opens Doc → reviews → Tools → eSignature → enters customer email → sends. (Google handles the signing flow.)
8. End: rep can generate another contract or close the tab.

Phase 1A leaves the signed-PDF detection + Master Tracker status update to manual (rep changes Status column to "Signed" when they receive Google's confirmation). Phase 3 automates.

## 4. Contingency contract — fields

Mirrors Doc 1 (Contingency_Contract_blank_copy.pdf) provided 2026-05-28.

### 4.1 Customer fields (universal, required)

- Customer name
- Address
- City
- State (dropdown: TX default, full US state list)
- Zip code
- Email (required — used for Google eSignature send)
- Phone
- Claim number
- Policy number
- Insurance company (free text — full carrier name)

### 4.2 Date

Auto-filled to today's date. Not editable. Format: Month DD, YYYY (matches Doc 1's DATE___________ line).

### 4.3 Scope fill-in-blanks

Dropdowns where the option set is known; free-text otherwise.

- **Felt paper type** (dropdown):
  - Synthetic
  - 15 lb
  - 30 lb
- **Decking allowance** (free-text, numeric — number of sheets)
- **Valley material** (dropdown):
  - Ice and Water Shield
  - Closed-cut shingle valleys
  - Open metal valleys
- **Shingle warranty years** (dropdown):
  - Lifetime
  - 50
  - 30
  - 25
- **Shingle color/manufacturer** (free-text — rep types the combined spec, e.g. "Weatherwood — GAF Timberline HDZ")
- **Ventilation** (dropdown):
  - Ridge vent
  - Static box vents
  - Power vents
  - Turbine vents
  - Same as current

### 4.4 Scope checkboxes

All checked by default. Rep unchecks any that don't apply to the job. Unchecked items render as omitted from the final document (not greyed-out — line is removed entirely; no visual hint that anything was removed).

- Tear off ALL existing shingle layers & felt layers
- Lay down new [felt paper type] felt paper
- Decking allowance [N] sheets
- Install [valley material] in the valleys
- Install starter strip around perimeter of roof
- Lay down new shingles [warranty years] year warranty
- Shingle color/manufacturer [spec]
- Install Duraflow pipe flashings around plumbing pipes
- Install [ventilation type] for ventilation
- Reflash chimney and skylights as needed
- Paint all metals and flashing to match
- Install drip edge 2" to match shingle color
- Remove all job related debris & nails
- 5 year labor warranty

Lines containing fill-in-blanks (felt paper, decking, valley, shingles warranty, color/mfr, ventilation) only render if the relevant blank has been filled. Empty fill = line auto-omitted.

### 4.5 Special instructions

Free-text. Optional. Renders in the "Special Instructions:" box on the contract.

### 4.6 Auto-stamped signature (Contractor block)

- Signature line: rep's stored full name (`REP_FULLNAME_<email>`) rendered in Mr Dafoe cursive font, size 18pt, embedded as inline image in the Contractor signature block
- Date line: today's date (auto-filled, matches the document Date field)
- Rep takes no action per contract; the stamp is automatic

### 4.7 Customer signature (Insured block)

- Left blank in the generated Doc
- Google eSignature template has the signature field pre-positioned over the Insured block
- Rep enters customer email → Google sends signing link → customer signs remotely
- Sealed PDF returns to Drive: `Signature requests/` (Google's default) and is filed by the rep into the per-rep folder (Phase 3 automates this)

## 5. Master Tracker integration

### 5.1 New tab: "Contracts"

Created on first contract submission via `bootstrapContractsTab()` (one-time, runnable from editor).

### 5.2 Columns (10)

| # | Column | Notes |
|---|--------|-------|
| 1 | Timestamp | Auto-filled |
| 2 | Rep Email | From `Session.getActiveUser().getEmail()` |
| 3 | Rep Name | From `REP_FULLNAME_<email>` Script Property |
| 4 | Customer Name | |
| 5 | Customer Address | |
| 6 | Customer Email | |
| 7 | Customer Phone | |
| 8 | Contract Type | Enum: Contingency / Cash Proposal (Phase 1B) |
| 9 | Drive Link | URL to the generated Google Doc |
| 10 | Status | Enum: Generated → Sent for Signature → Signed. Phase 1A all manual updates; Phase 3 automates. |

Per spec convention: schema may deviate from this draft at implementation; deviations documented in README.

## 6. Drive structure

```
Acacia Contracts/                           ← Root (CONTRACT_ROOT_FOLDER_ID)
├── _Templates/                              ← Hidden from reps; admin-managed
│   ├── Contingency Contract Template       ← Google Doc, eSig fields pre-placed
│   └── Cash Proposal Template               ← Phase 1B
├── Sara Smith/                              ← Auto-created on rep's first contract
│   ├── 2026-05-28 - John Doe - Contingency
│   ├── 2026-05-30 - Jane Roe - Contingency
│   └── ...
├── Henry Alford/
│   └── ...
└── ...
```

Per-rep folder naming: `<Rep Full Name>` (from `REP_FULLNAME_<email>`).
Per-contract Doc naming: `<yyyy-MM-dd> - <Customer Name> - <Contract Type>`.

**Permission model (Phase 1A):** All folders owned by deployer (henry@). Reps access contracts via the web app (which enforces auth) or via the Drive Link in the Master Tracker row. Folder-level Drive permissions per rep deferred to Phase 3 (acceptable tradeoff: a rep with full Drive access could browse other reps' folders, but app-auth gating prevents UI access).

## 7. Template engine

### 7.1 Placeholder syntax

Templates use `{{placeholder_name}}` tokens. Generation logic:

1. `DriveApp.getFileById(TEMPLATE_CONTINGENCY_DOC_ID).makeCopy(<name>, <repFolder>)`
2. Open the copy via `DocumentApp.openById()`
3. For each `{{token}}`, replace with the form value via `body.replaceText('\\{\\{token\\}\\}', value)`
4. For conditional scope lines: replace `{{line_xyz}}` with the rendered line OR empty string (line omitted)
5. For the Mr Dafoe signature: generate an inline image via the font-rendering helper (see §7.3), insert at the `{{rep_signature}}` token
6. Save & close

### 7.2 Placeholder inventory (Contingency)

| Placeholder | Source |
|-------------|--------|
| `{{date}}` | Auto-filled today |
| `{{customer_name}}` | Form |
| `{{customer_address}}` | Form |
| `{{customer_city}}` | Form |
| `{{customer_state_zip}}` | Form (concatenated State, Zip) |
| `{{customer_email}}` | Form |
| `{{customer_phone}}` | Form |
| `{{claim_number}}` | Form |
| `{{policy_number}}` | Form |
| `{{insurance_company}}` | Form |
| `{{felt_paper_type}}` | Form |
| `{{decking_allowance}}` | Form |
| `{{valley_material}}` | Form |
| `{{shingle_warranty_years}}` | Form |
| `{{shingle_color_mfr}}` | Form |
| `{{ventilation}}` | Form |
| `{{special_instructions}}` | Form (blank if empty) |
| `{{scope_lines}}` | Generated from checkbox state (multi-line replacement; uses `body.appendListItem` or similar) |
| `{{rep_signature}}` | Mr Dafoe-rendered image of rep's full name |
| `{{rep_signature_date}}` | Auto-filled today |
| `{{contractor_block}}` | Static — "Acacia Roofing" — could also be a placeholder if we want flexibility |

### 7.3 Rep signature rendering

The Mr Dafoe font signature is rendered server-side and embedded as an image. Implementation approach:

**Option A — HtmlService canvas rendering:** Render the rep's name in a tiny HTML page via `HtmlService.createHtmlOutput()`, capture as image using a hidden helper page. Risk: Apps Script doesn't have native HTML-to-image. Would need an external rendering service.

**Option B — Pre-rendered signature image per rep:** First time a rep onboards, generate their Mr Dafoe signature as a PNG (via an external rendering call or manual one-time process), store in a hidden Drive location. App pulls that PNG and inserts at `{{rep_signature}}`. Tradeoff: Per-rep one-time setup, but no runtime image generation needed.

**Option C — Insert as styled text in the Doc:** Replace `{{rep_signature}}` with the rep's name as Doc-level styled text using Mr Dafoe font directly in Google Docs (Mr Dafoe is available in Google's font picker). Tradeoff: Simplest. The "signature" is text, not an image, but visually identical when the Doc/PDF renders.

**Recommended:** Option C for Phase 1A simplicity. Apps Script can set font family + size on a text range via `DocumentApp.Body.editAsText().setFontFamily('Mr Dafoe').setFontSize(18)`. Verify Mr Dafoe is available in the Docs font picker (it should be — it's a Google Font). If not, fall back to Option B in Phase 1B.

**Open question for implementation:** Verify Mr Dafoe is accessible via DocumentApp programmatic font-setting before locking Option C.

## 8. Phase 1A — definition of done

- [ ] contract-builder Apps Script project scaffolded with Code.js, Constants.js, Form.html, appsscript.json, README.md
- [ ] First-time rep onboarding captures full name
- [ ] Contingency contract template Google Doc created with all `{{placeholders}}` and Google eSignature field placed over Insured signature block
- [ ] Web form renders: contract-type picker (only Contingency live), customer fields, scope blanks, scope checkboxes, special instructions
- [ ] Form validation: required fields enforced (customer name/address/email)
- [ ] On submit: app copies template, replaces placeholders, applies Mr Dafoe to rep signature text, saves to per-rep folder, writes Master Tracker Contracts row
- [ ] Review screen shows Drive link to filled Doc
- [ ] Smoke test: generate one contingency contract end-to-end with henry@acaciaroofs.com as the rep, send for eSignature to a test customer email, customer signs, signed PDF returned
- [ ] CLAUDE.md inventory entry
- [ ] Spec doc updated to reflect any implementation deviations

## 9. Out of Phase 1A scope (deferred)

- **Phase 1B:** Cash Proposal template (multi-page packet with Customer Guidelines, Warranty, COI appendices, payment-flow narrative without specific dollar amounts on breakdown lines per Option C decision)
- **Phase 2:** Other Roofr templates (you mentioned 4-5 total; identify and migrate the remaining 2-3)
- **Phase 3:** Automated signed-PDF detection + Master Tracker Status auto-update; Drive folder-level permissions per rep
- **Phase 4 (deferred indefinitely):** Job Cost Calculator integration to auto-fill contract total
- **Phase 5 (deferred indefinitely):** Pre-fill contract from existing Master Tracker job (currently always fresh-data per Hank 2026-05-28)

## 10. Lock decisions (Phase 1A)

1. Data is always fresh-entry. No Master Tracker pre-fill in Phase 1A or 1B. (Confirmed 2026-05-28.)
2. Pricing is a single number from the rep. No line items, no math, no calculation engine. Cash proposal payment-breakdown communicates the flow of money, not specific dollar amounts.
3. Customer signature via Google eSignature, remote only. In-person contracts continue to use paper. No native canvas signature pad.
4. Rep signature is auto-stamped via Mr Dafoe cursive font. No per-contract rep action. Set company-wide.
5. Per-rep Drive folders. Folder-level access control deferred to Phase 3.
6. Master Tracker Contracts tab created new by this project.
7. Phase 1A = Contingency only. Phase 1B = adds Cash Proposal. Same pattern as ancillary-submittal (gutters first).

## 11. Risks and open implementation questions

- **Mr Dafoe font availability via DocumentApp** — must verify before committing to Option C. If unavailable, fall back to pre-rendered PNG per rep.
- **Google eSignature field positioning on a copied template** — when `DriveApp.makeCopy()` duplicates the template, do the eSignature fields copy with it, or do they need to be re-placed each time? Test in Phase 1A scaffolding. If they don't copy, the workaround is to copy the template + tell the rep "now place the signature field on the Insured line" as part of step 7. Friction, but workable.
- **First-time rep onboarding UX** — if the rep tries to generate a contract without setting their full name, app forces the onboarding screen first. Confirm flow.
- **Cash Proposal "money flow" narrative wording** — Phase 1B will need final copy for the payment-breakdown section that conveys "supplements are owed to Acacia" without specific dollar amounts. Defer to Phase 1B kickoff.
