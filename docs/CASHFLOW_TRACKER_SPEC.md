# Acacia Cash-Flow Tracker — Data Model Spec (draft v1)

Purpose: turn the current "Cash Flow Dashboard" (an AR/AP aging + margin snapshot) into a
forward-looking cash-flow tracker that answers "will we have money in the bank in week N?"

Core principle:
**Projected cash = starting cash (QBO) + expected inflows − expected outflows, on a weekly
timeline.** The critical distinction is RETAINED vs PAID OUT — money Acacia keeps (office
fee, Acacia profit share) is NOT a cash outflow and must never be subtracted as one.

---

## 0. Build status (as of 2026-06-22)

**Phase 1 — SHIPPED & LIVE** (job-submittal `refreshCashFlowDashboard`, deploys @76 → @77 → @78):

| Deploy | What shipped |
|--------|--------------|
| @76 | Cash Flow Dashboard v2 — revenue on billable basis (Invoice when >0 else Contract); honest hard cost (Total Material Cost ×1.2 waste + 4-day PM synced to job-cost-calc); crew-vs-material split ($75/sq labor); per-job office fee / commission / Acacia-retained via tolerant Rep-Tier parse (T1 15% / T2 8% / T3 0%, blank→15% with ⚠️); PER-JOB PROFITABILITY table; $44K nut KPI; monthly "Est. Margin" → "Hard-Cost Margin" |
| @77 | Owner/house jobs (Rep Email in `OWNER_REPS` = joshua@/kelcie@/henry@) → office fee 0, commission 0, retained = full net; labeled "Owner" |
| @78 | Legacy/no-cost-basis rows (`hasCostBasis = TotalMaterialCost>0 \|\| TotalSquares>0`) excluded from profitability + monthly math; gray "N legacy jobs excluded" note |

Phase 1 verified live: nut KPI honest at ~+$12K surplus; owner jobs at 100%; 12 cost-less rows (11 legacy 2025 imports + Gang Liu's cleared row) correctly benched. PM model is the **4-day** version (1/2/3/4 days at 50/100/150 install-sq bands, $300/day, cap 4) — synced byte-for-byte between job-cost-calc and the dashboard.

**Not yet built:** inflow staging, company-fixed-cost *projection over time*, ancillary jobs, QBO actuals, the 13-week forward view (Phase D, §2/§3/§5). Phase 1 is a smarter snapshot; it is not yet the forward forecast.

---

## 1. Per-job waterfall (the engine)

Order of operations matters — office fee comes off the top, before the 50/50 split.

| Step | Line | Formula / source | Cash treatment |
|------|------|------------------|----------------|
| 1 | Collected revenue (true) | reconciled: invoice − WNP + supplements (NOT raw contract) | inflow |
| 2 | − Office fee | tier% × collected revenue (15% / 8% / 0% for Tier 1/2/3) | RETAINED by Acacia |
| 3 | − Direct job costs | material + crew/labor + lead cost + PA fee (insurance jobs) + permits/dumpster/chargebacks | PAID OUT |
| 4 | = Net job profit | step1 − step2 − step3 | — |
| 5 | Rep commission | 50% × net job profit | PAID OUT (at close) |
| 6 | Acacia profit | 50% × net job profit | RETAINED by Acacia |

Derived per job:
- **Cash out** = material + crew + leads + PA + rep commission
- **Acacia retained** = office fee + Acacia profit (step 2 + step 6)

---

## 1a. Job hard-cost engine (per roof job) — finalized

**The tracker should CONSUME job-cost-calculator's computed cost, not re-derive it.** The calc
already models all the line items below. The rates here are the authoritative source-of-truth
values pulled from `job-cost-calculator/index.html` (verified, not estimated). Lead cost, PA fee,
and permits/chargebacks are added on top to reach total direct job costs.

**Per-square shingle cost (blended material + labor — there is NO separate split in the code):**

| Shingle line | $/sq | | Shingle line | $/sq |
|---|--:|---|---|--:|
| IKO Cambridge (Lifetime) | 213 | | GAF Timberline HDZ | 263 |
| IKO Dynasty (Lifetime) | 223 | | Owens Corning Supreme | 263 |
| IKO Nordic (Class 4) | 255 | | Owens Corning Oakridge | 269 |
| Tamko Heritage | 224 | | GAF Natural Shadow | 273 |
| Atlas Pinnacle Pristine | 256 | | OC TruDefinition Duration | 284 |
| GAF Royal Sovereign | 258 | | GAF ArmorShield | 298 |
| | | | OC TruDefinition Duration Storm | 298 |

> The $138 material / $75 labor split is a Cambridge-only mental allocation (sums to $213); the
> code carries one blended number per line. Use blended for total cost.

**Cost stack (all from the calc):**
- `baseShingleCost = tearOffSquares × blended_rate`
- Pitch premiums: `8/12 ×$5 + 9/12 ×$10 + 10/12 ×$15 + 11/12 ×$20` per square
- Two-story: `+$10/sq` · Hip/ridge upgrade: `$2.51/LF` · Ridge-vent upgrade: `$6.50/LF`
- Extra shingle/felt layers: `tearOffSquares × rate` · Gutters: protect $200 / replace $1,500
- Satellite dish: trash $75 / reset $310 · Class 4 hip/ridge upcharge `$3.46/LF` (IKO Nordic only)
- **Waste = 20%**, applied to base shingle + pitch + upgrades + two-story + extra layers + (gutters if replace)
- **PM = 4-day model** (shipped + synced, see below)

> ✅ **PM model (RESOLVED — synced between job-cost-calc and the dashboard).** `installSquares =
> tearOffSquares × 1.2`; `$300/day`; days by band: `>150 → 4` (cap), `>100 → 3`, `>50 → 2`,
> `else → 1`. Both `job-cost-calculator/index.html` (`calculateCosts`) and
> `refreshCashFlowDashboard` use the identical chain — keep in sync if either changes. Example:
> a >100-install-sq roof = 3 days = $900 in both.

**Sanity backstop:** material ~28–31% of revenue, ~60% of hard cost. Outside that → bad inputs.

> ⚠️ **Margin correction:** the dashboard's ~70% "Est. Margin" subtracts material ONLY. With
> labor + PM in, true hard-cost margin is ~50% — and that's before office fee, commission, and
> the $44K nut. The tracker must net all cost lines or it repeats the overstatement.

Resolved this session: Nordic = $255, Dynasty = $223 (was TBD). Pitch/two-story ARE modeled as
separate add-ons (base labor flat; pitch drives the premium). Open: the PM per-day vs flat-$300
decision above.

---

## 2. Inflow timing — split each receivable by payment stage

A single "Balance Due" is useless for timing. Each open job's expected collections break into
dated buckets:

| Stage | Trigger / expected date | Notes |
|-------|------------------------|-------|
| ACV (first insurance check) | at/after approval | often the largest first slug |
| Deductible | at completion | homeowner pays directly |
| Recoverable depreciation (2nd check) | after completion + NOC/docs submitted | frequently weeks–months later; 20–40% of job |
| Financing (Hearth) | lump from lender | faster + lump-sum; different timing than insurance |
| Cash/retail jobs | per contract terms | no insurance staging |

Each receivable → one or more dated rows feeding the weekly inflow line.

---

## 3. Outflow timing

| Outflow | Timing source |
|---------|--------------|
| Material | per-job due date (currently ~60-day section) |
| Crew/labor | per-job due date (currently ~30-day section) |
| Rep commission | at job close — ⚠️ see risk flag below |
| PA fee | when PA invoices (confirm timing) |
| Lead costs | ongoing/monthly |
| Overhead | monthly (§4) |
| Owner salaries | monthly (§4) |

---

## 4. Company-level monthly fixed outflows (NEW — currently missing entirely)

| Item | Amount | Treatment |
|------|--------|-----------|
| Office overhead | ~$10,000 / mo | fixed monthly outflow |
| Owner salaries (incl. taxes) | $34,000 / mo | fixed monthly outflow |
| **Total monthly nut** | **~$44,000 / mo** | — |

**Break-even line:** Acacia-retained cash (office fees + Acacia profit share across all jobs in
the month) must exceed ~$44K before the company is cash-positive. Surface this as a headline
KPI: "Retained this month vs $44K nut."

ASSUMPTION (confirm): overhead + owner salary are company-level only and are NOT in the rep
commission base. Reps' 50% is on per-job net profit (revenue − office fee − direct job costs).

---

## 5. The net projection (13-week rolling — the actual cash-flow view)

Per week:

```
  Starting cash (from QBO actual bank balance)
+ Expected inflows      (dated receivables by stage, §2)
− Job outflows          (material + crew + commission + PA + leads, §1/§3)
− Company fixed         (overhead + salaries spread to the week, §4)
= Projected end-of-week cash
```

Flag any week where projected cash goes negative or below a minimum buffer.

---

## 6. Risk flags to surface on the dashboard

- **Commission-before-RD float:** rep commission is often paid at close, before the recoverable
  depreciation (2nd check) lands. Track "commission paid but RD not yet collected" exposure —
  this is the classic shortfall pattern in storm-restoration roofing.
- **Paid-in-Full with a balance:** status/dollars disagree (data bug — several exist today).
- **Payments > contract:** supplement raised the invoice but original contract still shown.
- **Collected < contract:** likely WNP (work not performed) billed below contract.

---

## 7. Source map

| Number | Source |
|--------|--------|
| Contract / invoice / status / dates | Master Job Tracker (contract col 15, invoice col 18) |
| Final invoice / WNP / supplements | Final Invoices sheet |
| Material, crew, lead cost, PA fee, commission | Rep Job Costs |
| Rep tier (→ office fee %) | Reps sheet (cumulative commissions: $50K→T2, $250K→T3) |
| Ancillary contracts/costs/commissions | Ancillary Jobs sheet (currently excluded — add) |
| Actual cash balance + actual payments received | QBO (via existing QBO app) |
| Overhead $10K/mo, owner salary $34K/mo | new input cells (single source of truth) |

---

## 8. Resolved + open items

**Resolved this session (2026-06-22):**
- Office fee base = billable revenue, off the top before job costs. ✓
- Tier % = 15/8/0 for T1/T2/T3 (matches acacia-portal `REP_TIERS` + CLAUDE.md). ✓
- Overhead + owner salary are company-level, NOT in rep commission base. ✓
- Owner/house jobs retain 100% (no fee, no commission). ✓
- PM = 4-day model, synced between job-cost-calc and dashboard. ✓
- Shingle costs are blended material+labor (single `data-cost`/`SHINGLE_PRICING`), authoritative. ✓

**Parked — needs work, not blocking (rough priority):**
1. **Recompute gap (Phase A — highest).** Nothing recomputes `Total Material Cost` when squares/
   shingle/rate are hand-edited (no onEdit trigger). Any correction silently leaves cost stale →
   corrupts dashboard. Fix: shared `computeMaterialCost(squares, shingleType, hipRidgeLF)` helper
   consumed by job-submittal + dashboard + Brain's `executeTrackerUpdate`; either onEdit recompute
   or derive material cost live in the dashboard (more robust). This is the keystone.
2. **Re-extract portal (Phase C).** Take a corrected adjuster PDF, re-run the *existing* extraction,
   patch the existing Job ID — no new email/calendar/Job ID. Diff-preview + selective write +
   recompute. Reuses `extractDocumentDataForForm` + the already-built-but-unwired
   `JobsService.updateJobFields` (acacia-portal, audited, schema-whitelisted, zero callers). Makes
   hand-jamming obsolete.
3. **Gang Liu (2026-0031).** Bad extraction; currently (correctly) excluded as no-cost-basis. It's a
   live 2026 job — hand-jam the full set together (Total Squares, Full Contract, Invoice, Shingle
   Type, Material Cost/Sq, AND recomputed Total Material Cost), then it rejoins the math.
4. **Legacy "Partial Payment" status bug.** The 11 imported 2025 rows are paid to $0 balance but
   stuck at "Partial Payment" — should be "Paid in Full." Status-hygiene cleanup.
5. **Contract/invoice discrepancy diagnostic.** Per job, compare contract (col 15) vs invoice
   (col 18) vs Final Invoices vs payments to classify each mismatch as supplement (over) vs WNP
   (under). Informs TODO #36. Standalone read-only pass, run anytime.
6. **supplement-agent defect.** Writes to a tab literally named `Jobs` (not Master Job Tracker), and
   its `SUPPLEMENT_STATUS = col 41` collides with the live schema's Claim Type = col 41. Latent.
7. **PA fee timing** — when does it actually leave the account? (needed for outflow timing, §3).
8. **Commission state** — generated-but-unpaid vs paid (TODO #22) so the projection knows which
   commissions are still future outflows (needed for Phase D).
9. `docs/about-page-content.md` still untracked in the repo — commit, gitignore, or delete.

**Phase D (the real forward forecast — own multi-session workstream):** inflow staging by insurance
stage (§2), company fixed costs spread over the timeline (§4), ancillary jobs, QBO actuals via the
QBO app (§5), the 13-week rolling projection, and the commission-before-RD float flag (§6).

