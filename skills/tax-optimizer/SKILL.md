---
name: tax-optimizer
version: 1.5.0
description: Cut taxes across accounts and years by orchestrating the public planfi MCP. Use whenever someone wants to lower their tax bill, time ISO exercises or Roth conversions, build a Roth conversion ladder, find mega-backdoor / after-tax 401(k) space, check NIIT / AMT / state surtax exposure, realize long-term gains at the 0% capital-gains rate in a low-income year (tax-gain harvesting) — how much gain can I harvest at 0% before NIIT/IRMAA? — harvest lot-level unrealized losses to offset realized gains and up to $3,000 of ordinary income, flag wash sales, suggest replacement securities (tax-loss harvesting) — weigh retirement relocation / state-tax arbitrage — "should I retire in / move to a lower-tax state? compare the lifetime after-tax outcome of state A vs B" — check whether you qualify for the federal Saver's Credit (up to 50% of retirement contributions for lower/moderate income) — or size a 72(t)/SEPP substantially-equal-payment stream for penalty-free access to retirement money before 59½ — e.g. "how do I cut my taxes with $900k in a 401k?", "convert my IRA to Roth between 60 and 70 filling the 12% bracket — how much each year?", "how much after-tax 401(k) space do I have?", "what's the full NIIT/AMT bite on an ISO exercise?", "I'm retiring in CA but thinking about TX — how much do I keep over my lifetime?".
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
`analyze_tax_loss_harvesting`, `analyze_relocation`,
`analyze_savers_credit`, `analyze_72t_sepp`, plus optional `generate_financial_plan`
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
   `analyze_relocation`, or `analyze_self_employed_retirement`).

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
- Near-retiree weighing a move? Pair this with the **`retirement-income`** skill for the
  decumulation side (withdrawal order, Social Security claiming age, estate-tax exposure).
- Self-employed / S-corp owner? The **`self-employed-planner`** skill sizes Solo 401(k) / SEP / SIMPLE
  room, the §199A QBI deduction, and the S-corp reasonable-salary tradeoff.
- Not financial or tax advice. Planning estimates only.
