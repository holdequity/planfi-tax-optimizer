---
name: tax-optimizer
version: 1.7.0
description: Cut taxes across accounts and years by orchestrating the public planfi MCP. Use whenever someone wants to lower their tax bill, time ISO exercises or Roth conversions, build a Roth conversion ladder, find mega-backdoor / after-tax 401(k) space, check NIIT / AMT / state surtax exposure, optimize charitable giving — bunch donations into a donor-advised fund (DAF) to clear the standard deduction, take a QCD at 70½+ that lowers AGI/IRMAA/Social-Security taxation, or give long-term appreciated stock/RSUs in-kind to avoid capital gains while still deducting FMV — realize long-term gains at the 0% capital-gains rate in a low-income year (tax-gain harvesting) — how much gain can I harvest at 0% before NIIT/IRMAA? — harvest lot-level unrealized losses to offset realized gains and up to $3,000 of ordinary income, flag wash sales, suggest replacement securities (tax-loss harvesting) — weigh retirement relocation / state-tax arbitrage — "should I retire in / move to a lower-tax state? compare the lifetime after-tax outcome of state A vs B" — check whether you qualify for the federal Saver's Credit (up to 50% of retirement contributions for lower/moderate income) — or size a 72(t)/SEPP substantially-equal-payment stream for penalty-free access to retirement money before 59½ — e.g. "how do I cut my taxes with $900k in a 401k?", "convert my IRA to Roth between 60 and 70 filling the 12% bracket — how much each year?", "how much after-tax 401(k) space do I have?", "what's the full NIIT/AMT bite on an ISO exercise?", "I'm retiring in CA but thinking about TX — how much do I keep over my lifetime?".
---

# Tax Optimizer

A thin orchestration layer over the **planfi MCP** (https://ai.planfi.app/mcp, public, no auth).
All tax math, brackets, and limits live server-side. This skill only gathers inputs and calls the
tools — it does **not** compute anything locally and bakes in no defaults of its own. Read-only.
Every tool returns a structured **`assumed_defaults[]`** array (`{ field, assumed_value, note }`)
listing each input it had to assume — always read these back to the user.

**Related skills:** for high-W2 execs weighing an employer Nonqualified Deferred Comp (NQDC / 409A)
election — defer-now-vs-take-now, lump-vs-installment distribution, and smoothing the distribution
years' brackets / IRMAA — see **deferred-comp** (`analyze_deferred_comp`).

## Step 0 — Make sure the planfi tools are connected

This skill uses these tools (may be namespaced, e.g. `mcp__planfi__analyze_tax_optimization`):
`analyze_tax_optimization`, `optimize_multi_year_tax`, `analyze_roth_conversion`,
`analyze_mega_backdoor_roth`, `analyze_advanced_taxes`, `analyze_gain_harvesting`,
`analyze_tax_loss_harvesting`, `analyze_charitable_giving`, `analyze_relocation`,
`analyze_savers_credit`, `analyze_72t_sepp`, `analyze_hsa_retirement`, plus optional `generate_financial_plan`
(for `plan_id` chaining + a `share_url`). Use whichever name your environment exposes (bare or
`mcp__planfi__`-prefixed); below they are written bare.

If they're NOT available, tell the user to connect the MCP, then continue:

```
claude mcp add --transport http planfi https://ai.planfi.app/mcp
```

(On claude.ai: add a custom connector pointing at https://ai.planfi.app/mcp — no auth.)

## Step 1 — (Optional) build a plan first to chain context + get a share link

> **Feed it into the forecast (not just plan_id chaining):** `generate_financial_plan` now accepts `gain_harvesting` directly as a plan input, so it flows into net worth, FIRE %, and Monte-Carlo backtesting — the 0%-LTCG harvest is applied as a one-time tax effect. Use the standalone analyze tool below for a focused what-if; pass `gain_harvesting` into the plan to see its effect on the whole household forecast.

If the user has (or wants) a full household model, call **`generate_financial_plan`** once and
**capture the returned `plan_id`**. The plan-aware tools in this skill accept `{ plan_id }` (plus inline overrides),
so they can resolve balances, income, age, and filing status from the saved plan instead of you
re-sending every figure. Every tax tool also returns a **`share_url`** (planfi.app) **when you pass
a `plan_id` that resolves a saved household** — without a `plan_id` there is no plan to share, so no
`share_url` is emitted. `generate_financial_plan` is the way to mint a `plan_id` (and its own
`share_url`) when the session has no model yet.

This step is optional: every tax tool runs cold from raw inputs too. Prefer the plan path when the
session already has a model or the user wants a sharable artifact.

> **Engine facts to bake in:** all decimals are **fractions** (24% → `0.24`); all dollars are
> **today's (real) dollars**; brackets/limits are approximate **~2026** values. Override `tax_year`
> on any tool if the user needs a different year.

## Step 2 — Route by intent

Pick the tool that matches what the user is asking. Pass `{ plan_id }` when you have one; otherwise
pass the raw fields below. **Every field is optional with a sensible server default** unless marked
REQUIRED — so the tools run even from sparse input (every assumption is reported back in the
structured `assumed_defaults[]` array, see Step 3).

### "Lower my taxes broadly" → `analyze_tax_optimization`
Asset location + tax-loss harvesting + charitable bunching/QCD, with quantified annual $ savings.
Useful fields: `taxable_balance`, `tax_deferred_balance`, `roth_balance`, `ordinary_tax_rate`,
`capital_gains_tax_rate`, `age`; toggles `enable_asset_location` (default on), `enable_tlh`,
`enable_charitable`. For TLH add `realized_capital_gains`, `harvestable_losses`,
`loss_carryforward`. For charitable add `annual_charitable_giving`, `bunch_years`, `qcd_amount`.

```
analyze_tax_optimization({
  taxable_balance: 400000, tax_deferred_balance: 900000,
  ordinary_tax_rate: 0.24, age: 52,
  enable_tlh: true, realized_capital_gains: 30000, harvestable_losses: 12000
})
```

### "Time my ISO exercise + Roth conversions over several years" → `optimize_multi_year_tax`
AMT-crossover / NIIT-threshold / IRMAA-tier aware year-by-year plan.
REQUIRED: `baseline_ordinary_income`. Optional: `iso_shares_to_exercise` (total bargain element to
exercise-and-hold), `total_roth_to_convert`, `horizon_years`, `target_magi_ceiling`,
`filing_status`.

### "Roth conversion ladder in my gap years" → `analyze_roth_conversion`
Fills conversions to the top of a target bracket and models lifetime RMD tax avoided.
REQUIRED: `traditional_balance`, `current_age`. Optional: `conversion_start_age`,
`conversion_end_age`, `target_bracket_rate`, `birth_year` (sets RMD start age 73 vs 75),
`filing_status`, `other_taxable_income`, `state_flat_rate`, `life_expectancy`. ISO/NIIT layering via
`enable_amt` + `iso_bargain_element`, `enable_niit` + `net_investment_income`. Conversions raise
MAGI — flag the ACA-subsidy interaction and point pre-65 retirees to the retirement-income skill's
`analyze_healthcare_bridge` (this is your own suggestion, not a server `next_actions` edge — those
chain to `analyze_withdrawal_strategy`, `analyze_gain_harvesting`, and `analyze_relocation`).

```
analyze_roth_conversion({
  traditional_balance: 800000, current_age: 60,
  conversion_start_age: 60, conversion_end_age: 70,
  target_bracket_rate: 0.12, filing_status: "married_joint", birth_year: 1966
})
```

### "After-tax 401(k) / backdoor Roth space" → `analyze_mega_backdoor_roth`
415(c) total-addition limit, remaining after-tax space, and Roth-IRA phaseout / pro-rata detection.
REQUIRED: `age`, `annual_salary`, `employee_401k_contribution`. Optional: `employer_match` or
`employer_match_formula`, `magi`, `filing_status`, `existing_pretax_ira_balance` (triggers pro-rata
warning), `existing_nondeductible_ira_basis`.

### "Full surtax bite — NIIT / AMT / state" → `analyze_advanced_taxes`
The three surtax layers on top of ordinary tax.
REQUIRED: `ordinary_taxable_income`. Optional: `net_investment_income`, `magi` (NIIT threshold),
`iso_bargain_element` (AMT), `state_flat_rate`, `filing_status`. Good as a sanity check after a Roth
conversion or ISO exercise.

### "How much gain can I realize at 0%?" → `analyze_gain_harvesting`
Tax-**gain** harvesting (the mirror of tax-loss harvesting): how much long-term capital gain you can
realize at the **0%** LTCG rate (or up to the **15%** band) in a low-income year before the next
bracket, **NIIT**, or an **IRMAA** cliff bites. Returns the **harvestable amount at 0%**, the
**per-tranche tax cost** (0% / 15% / 20% bands), the **basis step-up benefit** of resetting basis at
0%, and the **binding cliff** to avoid (LTCG breakpoint / NIIT / IRMAA).
Useful fields: `unrealized_ltcg_gain` (or `position_value` + `cost_basis` to derive the embedded
gain), `ordinary_taxable_income`, `existing_realized_ltcg` (gains already booked this year that eat
0%-bracket room), `filing_status`, `magi` (drives NIIT + IRMAA), `age` (IRMAA only applies at the
≥63 look-back age), `future_ltcg_rate` (rate the step-up avoids), and `target_max_ltcg_rate`
(`0` to stay in the 0% band, `0.15` to also fill the 15% band). Complements the **TLH** path in
`analyze_tax_optimization` — losses offset gains, gain-harvesting books gains for free.

```
analyze_gain_harvesting({
  unrealized_ltcg_gain: 80000, ordinary_taxable_income: 50000,
  filing_status: "married_joint", magi: 115000, age: 55, target_max_ltcg_rate: 0
})
```

### "Should I bunch my donations? / front-load a donor-advised fund / give appreciated stock instead of cash / use a QCD from my IRA to cover my RMD / how do I give to charity tax-efficiently / clear the standard deduction by bunching / donate my RSUs / avoid capital gains by donating / lower my AGI with a QCD?" → `analyze_charitable_giving`
**Always CALL `analyze_charitable_giving` for these — do not answer from general knowledge or quote the standard-deduction / 60%-AGI / QCD-cap rules of thumb from memory. When the user gives the numbers, run it and lead with its real output (recommended strategy, estimated tax savings, bunching schedule, AGI-limit headroom).** This is the first-class charitable analyzer for high earners (32–37% brackets) with charitable intent and/or concentrated appreciated positions. It compares four levers and recommends the highest-savings one:
- **BUNCHING / DAF** — concentrate several years of giving into one year (or front-load a donor-advised-fund lump sum) so the bunch-year itemized deductions clear the 2026 standard deduction; itemize in the bunch year, take the standard deduction in off-years. The `bunching` block returns `bunchedDonation`, `deductionGain`, the per-year `schedule[]` (`{ year, donation, deductionTaken, itemizes }`), and `taxSavings` (zero benefit when per-year itemized already beats the standard deduction — the tool tells you).
- **APPRECIATED SECURITIES vs cash** — donating long-term appreciated stock at fair-market value **avoids the capital-gains tax AND NIIT** on the embedded gain *while still taking the full FMV deduction*. The `appreciated` block returns `unrealizedGain`, `capGainsAvoided`, `niitAvoided`, `fmvDeduction`, and `taxSavingsVsCash` (the in-kind edge over selling-then-donating, where the FMV deduction is identical).
- **QCD (age 70½+)** — a Qualified Charitable Distribution satisfies the RMD while **excluding** the amount from AGI (capped at the 2026 QCD cap). The `qcd` block returns `eligible`, `rmd`, `qcdApplied`, `cap`, and `taxSavings`.
- **AGI deduction limits** — the `agi_limits` block returns `cashLimit` (60% AGI), `appreciatedLimit` (30% AGI), `cashHeadroom`, `appreciatedHeadroom`, and any 5-year `carryforward` of excess.

Top-level returns: `recommended_strategy`, `estimated_tax_savings`, the four blocks above, and an `assumptions[]` list. Useful fields (every one optional, plan-resolved when omitted): `age`, `filing_status`, `adjusted_gross_income` (alias `agi`) — drives the 60%/30% AGI limits + NIIT, `ordinary_taxable_income` (the bracket base + QCD marginal-rate lookup — never a flat assumed rate), `ira_balance` (alias `rmd_amount`), `birth_year`, `annual_donation` (alias `cash_donation`), `years_to_bunch`, `other_itemized_deductions`, `appreciated_securities_value`, `appreciated_cost_basis`, `daf_contribution`, `qcd_amount`, `state_code`, `tax_year`, `plan_id`.

```
analyze_charitable_giving({
  filing_status: "married_joint", age: 72,
  ordinary_taxable_income: 380000, adjusted_gross_income: 400000,
  annual_donation: 15000, other_itemized_deductions: 8000, years_to_bunch: 3,
  appreciated_securities_value: 50000, appreciated_cost_basis: 10000,
  ira_balance: 500000, qcd_amount: 20000
})
// → recommended_strategy + estimated_tax_savings, with bunching{bunchedDonation,deductionGain,taxSavings,schedule[]},
//   appreciated{unrealizedGain,capGainsAvoided,niitAvoided,fmvDeduction,taxSavingsVsCash},
//   qcd{eligible,rmd,qcdApplied,cap,taxSavings}, agi_limits{cashLimit,appreciatedLimit,cashHeadroom,appreciatedHeadroom,carryforward}
```

**Chain note:** complements `analyze_gain_harvesting` (donate the most-appreciated lots instead of harvesting/selling them — the appreciated-securities overlap), `analyze_rmd` (a QCD satisfies the RMD it sizes — the QCD overlap), and `analyze_irmaa` (the QCD's AGI exclusion is what relieves an IRMAA tier).

### "Harvest my losses / offset my gains / what can I write off / will this trigger a wash sale / what do I buy instead / sell my losers for the tax break?" → `analyze_tax_loss_harvesting`
**Always CALL `analyze_tax_loss_harvesting` for these — do not answer from general knowledge or quote
the $3,000 / 30-day rules of thumb from memory.** When the user gives lots and a gain budget, run it
and lead with its real output (harvestable amount, tax saved, wash-sale flags, replacement
suggestions). Tax-**loss** harvesting is the mirror of gain-harvesting: it identifies lot-level
unrealized losses, nets them against realized ST/LT gains (IRC §1211/§1212 order), applies up to
**$3,000** against ordinary income, carries the rest forward, flags **wash-sale** windows (30-day,
cross-account incl. spouse/IRA, IRC §1091), and suggests correlated-but-not-substantially-identical
replacement securities. Returns `harvestable_loss`, `disallowed_loss`, `wash_sale_lots[]`,
`net_short_term` / `net_long_term`, `ordinary_offset`, `tax_benefit`, `niit_savings`,
`loss_carryforward_to_next_year`, `replacement_suggestions[]`, and a `netting_steps[]` audit trail.
Useful fields: `lots[]` (each `{ costBasis, marketValue, term: 'short'|'long', symbol?, account?,
recentPurchaseDates?[] }`), `realizedShortTermGain`, `realizedLongTermGain`,
`shortTermLossCarryforward`, `longTermLossCarryforward`, `ordinaryTaxableIncome`, `filingStatus`,
`magi` (NIIT), `harvestDate`, `age`. Federal-only in v1 (most states disallow/cap the $3k
net-loss-against-ordinary deduction). The replacement map never names the same CUSIP/index — the
substantially-identical judgment is the user's.

```
analyze_tax_loss_harvesting({
  lots: [
    { symbol: "VTI", costBasis: 60000, marketValue: 48000, term: "long" },
    { symbol: "VXUS", costBasis: 30000, marketValue: 22000, term: "short",
      recentPurchaseDates: ["2026-12-01"] }
  ],
  realizedLongTermGain: 40000, ordinaryTaxableIncome: 120000, filingStatus: "married_joint"
})
```

### "Should I retire in / move to a lower-tax state?" → `analyze_relocation`
Lifetime after-tax comparison of state A vs B: state income tax, capital-gains, retirement-income &
Social-Security taxation, property tax, state estate tax, plus a cost-of-living delta. Returns the
annual + lifetime difference, a one-time estate-tax delta, and a `move` / `stay` / `marginal`
recommendation with the dominant driver named.
REQUIRED: `from_state`, `to_state` (two-letter codes). Optional: `annual_retirement_income`,
`social_security_income`, `annual_capital_gains`, `annual_spend` (at COL index 100),
`real_estate_value`, `filing_status`, `current_age`, `life_expectancy` (sets the horizon),
`liquid_assets`, `mortgage_principal`, `estimated_growth_rate`, `tax_year`, and `plan_id` /
`overrides` (plan resolution). State income tax comes from the server's shared engine
(`state-tax.ts`): progressive bracket tables for all 50 states + DC, with first-class single and
married-filing-jointly (MFJ) brackets, so `filing_status` branches every state (no-income-tax states
report $0). There is **no** `state_flat_rate` field on this tool — brackets are server-side, not
user-supplied. Federal income & estate tax are state-invariant and excluded from the delta; figures
are real-dollar, undiscounted.

> **Shared bracket engine — `n/a (shared engine: state-tax.ts)`.** The progressive 50-state + DC
> single/MFJ bracket tables (with per-state surtaxes) live in one shared engine module
> (`state-tax.ts`) reused by every tool here and by the **`relocation-planner`** skill — there is no
> CA/NY/MA-only special-casing and no per-tool flat-rate fallback. For a relocation-only question
> (no broader tax work), the **`relocation-planner`** skill wraps the same `analyze_relocation`
> tool as a focused entry point.

Like every tool in this skill, `analyze_relocation` emits a structured `assumed_defaults[]`
array (every state-profile fallback it applied — no-SS-tax, $0 retirement-income exclusion, default
property rate, the 85% Social-Security convention) and a **`share_url`** when you pass `{ plan_id }`.
Read back the `assumed_defaults[]` and offer the link. Because this is a near-retiree decision, pair
it with the **`retirement-income`** skill (`analyze_withdrawal_strategy`, `optimize_social_security`,
`analyze_estate_exposure`) for the full decumulation picture once the state is chosen.

### "Do I qualify for the Saver's Credit?" → `analyze_savers_credit`
The federal Saver's Credit (Retirement Savings Contributions Credit, IRC §25B): a **non-refundable**
credit of **50% / 20% / 10%** of up to **$2,000 per person** of IRA + elective-deferral contributions,
phased out by AGI and filing status. The server owns every AGI breakpoint and tier — do **not** hardcode
thresholds here.
Useful fields: `agi`, `filing_status`, `retirement_contributions` (IRA + 401(k)/403(b) elective deferrals),
`age`, `is_student`, `is_dependent` (the three eligibility gates — under-18, full-time student, or claimed
as a dependent all disqualify). Returns the credit rate/band, eligible contributions counted, the gross and
allowed (liability-capped) credit, and whether it was capped by your tax liability.

```
analyze_savers_credit({
  agi: 34000, filing_status: "single",
  retirement_contributions: 2000, age: 27
})
// → 50% band → ~$1,000 credit on $2,000 of Roth IRA contributions (non-refundable, capped to tax owed)
```

### "Access retirement money penalty-free before 59½" → `analyze_72t_sepp`
72(t) **substantially-equal-periodic-payments (SEPP)**: how much you can pull from an IRA/401(k) each year
**penalty-free before 59½** by committing to a fixed-formula stream for the longer of **5 years or until age
59½**. The server owns the divisor table and the max(5%, 120% mid-term AFR) interest-rate cap — no thresholds
live here.
Useful fields: `account_balance`, `current_age`, `method` (`amortization` / `rmd` / `annuitization`),
`interest_rate`. Returns the annual distribution for the chosen method, all three methods side-by-side, the
commitment-window years, whether the rate is within the §72(t) max, and the retroactive-10%-penalty warning if
the SEPP is modified early.

```
analyze_72t_sepp({
  account_balance: 1000000, current_age: 52,
  method: "amortization", interest_rate: 0.05
})
// → fixed annual SEPP withdrawal, locked in for the longer of 5 yrs or age 59½
```

### "Should I max my HSA and invest it? / Is the HSA a good retirement account? / triple-tax-advantaged / save medical receipts and reimburse later / receipt shoebox / deferred reimbursement / how much can I contribute to my HSA in 2026 / family vs self HSA limit / age-55 HSA catch-up / can I keep contributing past 65 / HSA after Medicare / use HSA for non-medical after 65" → `analyze_hsa_retirement`
**Always CALL `analyze_hsa_retirement` for these — do not answer from general knowledge or quote the contribution-limit / triple-tax / age-65 rules of thumb from memory. When the user gives the numbers, run it and lead with its real output (recommended contribution + invest decision, projected tax-free medical reserve, receipt-banking advantage in $, age-65 withdrawal recommendation, lifetime tax saved vs spending annually).**

The HSA-as-retirement optimizer treats the HSA as the only **triple-tax-advantaged** investable retirement vehicle and models four levers deterministically (compounding + bracket math, no Monte Carlo):
- **MAX-FUND + INVEST vs spend** — 2026 family/individual contribution limits + the age-55 **$1,000 catch-up** (server-sourced from `tax-limits`, never hardcoded), deposit-then-grow compounding to retirement.
- **RECEIPT-SHOEBOX / DEFERRED REIMBURSEMENT** — pay current qualified medical out-of-pocket, bank the unreimbursed receipts, reimburse tax-free decades later after the balance compounds; quantifies the invest-and-defer advantage in dollars (`receipt_banking_advantage_dollars`) and the breakeven horizon.
- **6-MONTHS-BEFORE-MEDICARE contribution stop** and its interaction with working past 65 under an employer HDHP (`medicare_contribution_stop_age`).
- **AGE-65 PIVOT** — after 65 non-medical withdrawals are penalty-free but taxable (traditional-IRA-equivalent) while qualified-medical withdrawals stay tax-free (`age_65_withdrawal_recommendation`).

Useful fields (every one optional, plan-resolved when omitted): `current_age`, `retirement_age`, `coverage_type` (`individual`/`family`), `current_hsa_balance`, `annual_contribution`, `invest_balance` (boolean), `expected_real_return`, `annual_qualified_medical_oop` (the receipt-banking candidate, paid from cash), `marginal_tax_rate` (else derived from income), `filing_status`, `working_past_65`, `medicare_enrollment_age`, `tax_year`, `plan_id`, `overrides`. Returns `recommended_annual_contribution`, `invest_recommendation`, `projected_tax_free_medical_reserve_at_retirement`, `receipt_banking_advantage_dollars`, `age_65_withdrawal_recommendation`, `lifetime_tax_saved_vs_spending_annually`, `contribution_limit_2026`, `catch_up_eligible`, `medicare_contribution_stop_age`, and `breakeven_year`.

```
analyze_hsa_retirement({
  current_age: 35, retirement_age: 65, coverage_type: "family",
  invest_balance: true, expected_real_return: 0.07,
  annual_qualified_medical_oop: 2000, filing_status: "married_joint",
  marginal_tax_rate: 0.24
})
// → recommended max contribution + "invest", projected tax-free medical reserve at 65,
//   receipt-banking advantage in $, age-65 withdrawal recommendation, lifetime tax saved vs spending annually
```

> **Pairs with `retirement-income` and `financial-forecast`:** a 72(t) is an early-retirement decumulation
> bridge — pair it with the **`retirement-income`** skill's `analyze_withdrawal_strategy` /
> `analyze_healthcare_bridge` for the pre-Medicare income+coverage picture, and use the
> **`financial-forecast`** skill to see the SEPP floor inside a full household projection. The Saver's Credit
> matters most for early-career accumulators — fold it into a forecast via `financial-forecast` to see it land
> on the federal tax line.

## Step 3 — Surface results honestly

For whichever tool you called:
- **Lead with the headline dollar figure** — annual tax savings, per-year conversion amounts +
  lifetime RMD tax avoided, AMT/NIIT crossover, remaining after-tax space, total surtax bite.
- **Read back the assumptions verbatim.** Every tax tool returns a structured **`assumed_defaults[]`**
  array — each entry is `{ field, assumed_value, note }` for an input it had to assume (e.g. ordinary
  rate 0.24, cap-gains 0.15, bond allocation 0.2, standard deduction $29,200; Roth target bracket
  0.12, RMD age 73, life expectancy 92). Read each one back so the user can correct any silent
  default. (`disclosures.key_assumptions` is separate static explanatory prose — not the assumption
  list; the machine-readable record lives in `assumed_defaults[]`.)
- Honor `disclosures.not_advice` (a **boolean** flag, not a message) — present results as planning
  estimates, not tax advice.
- **Follow `next_actions[]`** — each is `{ tool, why, prefilled_args }` (carrying `{ plan_id }`
  when available). Use these server-suggested chains rather than guessing the next call.
- **For a share link:** every tax tool returns a `share_url` when called with a `{ plan_id }` that
  resolves a saved household; without a `plan_id` no link is emitted, so run `generate_financial_plan`
  (Step 1) first to mint a `plan_id` and surface its `share_url`.

## Recommended call sequence (typical session)

1. (optional) `generate_financial_plan` → capture `plan_id` (+ `share_url`).
2. Route by intent → one of the tools above (with `{ plan_id }` or raw fields).
3. Read back the headline + the structured `assumed_defaults[]`.
4. Follow `next_actions[]` (for these tools the edges chain into `analyze_advanced_taxes`,
   `analyze_gain_harvesting`, `analyze_withdrawal_strategy`, `analyze_estate_exposure`,
   `analyze_relocation`, `analyze_hsa_retirement`, or `analyze_self_employed_retirement`).

## Fictional examples

**1.** *"I'm 52, $900k in a 401k and $400k taxable, 24% bracket — how do I cut my taxes?"*
→ `analyze_tax_optimization({ tax_deferred_balance: 900000, taxable_balance: 400000,
ordinary_tax_rate: 0.24, age: 52 })`. Lead with the annual asset-location tax-drag savings; offer to
turn on `enable_tlh` / `enable_charitable` if they have realized gains or giving intent. Read back
the `assumed_defaults[]` (cap-gains rate, allocation, standard deduction).

**2.** *"I want to convert my traditional IRA to Roth between 60 and 70, MFJ, filling the 12% bracket
— how much each year?"* → `analyze_roth_conversion({ traditional_balance: <ask>, current_age: 60,
conversion_start_age: 60, conversion_end_age: 70, target_bracket_rate: 0.12, filing_status:
"married_joint" })`. Lead with per-year conversion + lifetime RMD tax avoided; flag the MAGI/ACA
interaction and suggest `analyze_healthcare_bridge` if pre-65.

**3.** *"We're retiring in California but thinking about Texas — $80k of IRA withdrawals, $40k Social
Security, a $600k house, ~$60k spend. Worth the move?"* → `analyze_relocation({ from_state: "CA",
to_state: "TX", annual_retirement_income: 80000, social_security_income: 40000, real_estate_value:
600000, annual_spend: 60000, filing_status: "married_joint" })`. Lead with the total lifetime
advantage and the `move`/`stay`/`marginal` call; name the dominant driver (often state income tax or
COL). Read back the assumptions (esp. the 85% Social-Security convention and any no-table state).

**4.** *"We retired early — basically $0 ordinary income this year, MFJ, and we're sitting on $300k of
unrealized long-term gains in a brokerage account. How much can we sell at 0% tax?"* →
`analyze_gain_harvesting({ unrealized_ltcg_gain: 300000, ordinary_taxable_income: 0,
filing_status: "married_joint", target_max_ltcg_rate: 0 })`. Lead with the harvestable-at-0% figure
(room up to the 0%/15% LTCG breakpoint), the $0 tax cost of that tranche, and the basis step-up
benefit of resetting cost basis at no cost. Name the **binding cliff** — here the **15% LTCG
breakpoint** (and, if `magi`/`age` push them near it, the NIIT or IRMAA threshold). Note this is the
mirror of tax-loss harvesting and pairs with a Roth conversion (both spend the same 0%-bracket room,
so sequence them).

**5.** *"We're 72, MFJ, ~$380k taxable / $400k AGI, give ~$15k/yr to charity, hold $50k of stock (basis $10k), and have $500k in a traditional IRA. Should I bunch into a DAF, give the appreciated shares, and do a QCD from our IRA?"* →
`analyze_charitable_giving({ filing_status: "married_joint", age: 72, ordinary_taxable_income: 380000, adjusted_gross_income: 400000, annual_donation: 15000, other_itemized_deductions: 8000, years_to_bunch: 3, appreciated_securities_value: 50000, appreciated_cost_basis: 10000, ira_balance: 500000, qcd_amount: 20000 })`. Lead with `recommended_strategy` and `estimated_tax_savings`; break out the levers — `bunching.deductionGain` × marginal rate (with the per-year `bunching.schedule[]`), the in-kind `appreciated.capGainsAvoided` + `appreciated.niitAvoided` (the gain-avoidance edge over selling-then-donating, where `appreciated.fmvDeduction` is identical), the `qcd.qcdApplied` AGI exclusion, and the `agi_limits` headroom (`cashHeadroom` / `appreciatedHeadroom` / any `carryforward`). Read back the `assumptions[]` (30%/60%-AGI ceilings, 5-yr carryforward, QCD cap and age 70½). Pairs with `analyze_gain_harvesting` (donate the most-appreciated lots), `analyze_rmd` (the QCD satisfies the RMD), and `analyze_irmaa`.

*(All examples use fictional figures — never reuse a real user's numbers in documentation.)*

## Notes

- All decimals are fractions; all dollars are today's (real) dollars; brackets/limits are ~2026
  (override `tax_year` as needed).
- Pass `{ plan_id }` to reuse a saved household model; any field you also pass is a shallow override.
- Every tax tool surfaces its assumptions as a structured **`assumed_defaults[]`** array
  (`{ field, assumed_value, note }`) — read each entry back. `disclosures.key_assumptions` is
  separate static prose and `disclosures.not_advice` is a boolean. Each tool also returns a
  `share_url` when passed a `plan_id` that resolves a household; with no `plan_id`, run
  `generate_financial_plan` for a sharable link.
- **Tax-loss harvesting** (`analyze_tax_loss_harvesting`) **is the mirror of tax-gain harvesting**
  (`analyze_gain_harvesting`): TLH nets lot-level losses against realized gains, deducts up to $3,000
  against ordinary income, flags wash sales, and suggests replacements; gain-harvesting books gains at
  the 0% rate. Call `analyze_tax_loss_harvesting` whenever the user has losing lots and a gain budget.
- **Tax-gain harvesting** (`analyze_gain_harvesting`) **complements tax-loss harvesting**
  (`analyze_tax_loss_harvesting`): losses offset realized gains, while gain-harvesting books
  long-term gains at the 0% rate and steps up basis for free. It also **pairs with
  `analyze_roth_conversion`** — both consume the same 0%-bracket / low-income headroom, so a household
  with limited room must choose how to spend it (sequence the two rather than double-count the space).
- **Charitable giving** (`analyze_charitable_giving`) is the dedicated optimizer for donors and
  holders of concentrated appreciated positions. Its three levers cross-link the rest of the suite:
  **in-kind appreciated-stock** donation is the better move when `analyze_gain_harvesting` shows a
  large embedded gain (donate the lot instead of selling it — you avoid the capital-gains tax AND
  NIIT *and* still deduct FMV); a **QCD** at 70½+ satisfies the RMD that `analyze_rmd` sizes while
  excluding it from AGI; and that AGI exclusion is exactly what relieves an IRMAA tier in
  `analyze_irmaa`. Call `analyze_charitable_giving` whenever the user mentions giving, a DAF, a QCD,
  or donating appreciated stock/RSUs — don't quote the 60%-of-AGI / QCD-age / standard-deduction
  rules of thumb from memory.
- Near-retiree weighing a move? Pair this with the **`retirement-income`** skill for the
  decumulation side (withdrawal order, Social Security claiming age, estate-tax exposure).
- Self-employed / S-corp owner? The **`self-employed-planner`** skill sizes Solo 401(k) / SEP / SIMPLE
  room, the §199A QBI deduction, and the S-corp reasonable-salary tradeoff.
- Not financial or tax advice. Planning estimates only.
