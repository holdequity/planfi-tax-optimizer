---
name: tax-optimizer
version: 1.1.0
description: Cut taxes across accounts and years by orchestrating the public planfi MCP. Use whenever someone wants to lower their tax bill, time ISO exercises or Roth conversions, build a Roth conversion ladder, find mega-backdoor / after-tax 401(k) space, check NIIT / AMT / state surtax exposure, or weigh retirement relocation / state-tax arbitrage — "should I retire in / move to a lower-tax state? compare the lifetime after-tax outcome of state A vs B" — e.g. "how do I cut my taxes with $900k in a 401k?", "convert my IRA to Roth between 60 and 70 filling the 12% bracket — how much each year?", "how much after-tax 401(k) space do I have?", "what's the full NIIT/AMT bite on an ISO exercise?", "I'm retiring in CA but thinking about TX — how much do I keep over my lifetime?".
---

# Tax Optimizer

A thin orchestration layer over the **planfi MCP** (https://ai.planfi.app/mcp, public, no auth).
All tax math, brackets, and limits live server-side. This skill only gathers inputs and calls the
tools — it does **not** compute anything locally and bakes in no defaults of its own. Read-only.

## Step 0 — Make sure the planfi tools are connected

This skill uses these tools (may be namespaced, e.g. `mcp__planfi__analyze_tax_optimization`):
`analyze_tax_optimization`, `optimize_multi_year_tax`, `analyze_roth_conversion`,
`analyze_mega_backdoor_roth`, `analyze_advanced_taxes`, `analyze_relocation`, plus optional `generate_financial_plan`
(for `plan_id` chaining + a `share_url`). Use whichever name your environment exposes (bare or
`mcp__planfi__`-prefixed); below they are written bare.

If they're NOT available, tell the user to connect the MCP, then continue:

```
claude mcp add --transport http planfi https://ai.planfi.app/mcp
```

(On claude.ai: add a custom connector pointing at https://ai.planfi.app/mcp — no auth.)

## Step 1 — (Optional) build a plan first to chain context + get a share link

If the user has (or wants) a full household model, call **`generate_financial_plan`** once and
**capture the returned `plan_id`**. All six tools in this skill accept `{ plan_id }` (plus inline overrides),
so they can resolve balances, income, age, and filing status from the saved plan instead of you
re-sending every figure. `generate_financial_plan` also returns a **`share_url`** (planfi.app) —
the tax tools themselves do **not** emit a share link, so this is the way to give the user one.

This step is optional: every tax tool runs cold from raw inputs too. Prefer the plan path when the
session already has a model or the user wants a sharable artifact.

> **Engine facts to bake in:** all decimals are **fractions** (24% → `0.24`); all dollars are
> **today's (real) dollars**; brackets/limits are approximate **~2026** values. Override `tax_year`
> on any tool if the user needs a different year.

## Step 2 — Route by intent

Pick the tool that matches what the user is asking. Pass `{ plan_id }` when you have one; otherwise
pass the raw fields below. **Every field is optional with a sensible server default** unless marked
REQUIRED — so the tools run even from sparse input (the assumptions are reported back as prose, see
Step 3).

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
MAGI — flag the ACA-subsidy interaction and suggest chaining to `analyze_healthcare_bridge`
(retirement-income skill) for pre-65 retirees.

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

### "Should I retire in / move to a lower-tax state?" → `analyze_relocation`
Lifetime after-tax comparison of state A vs B: state income tax, capital-gains, retirement-income &
Social-Security taxation, property tax, state estate tax, plus a cost-of-living delta. Returns the
annual + lifetime difference, a one-time estate-tax delta, and a `move` / `stay` / `marginal`
recommendation with the dominant driver named.
REQUIRED: `from_state`, `to_state` (two-letter codes). Optional: `annual_retirement_income`,
`social_security_income`, `annual_capital_gains`, `annual_spend` (at COL index 100),
`real_estate_value`, `filing_status`, `life_expectancy` (sets the horizon), `tax_year`, plus
`overrides` for flat-rate states without a bracket table (CA/NY/MA have tables; PA/CO/etc. need a
flat rate, else their income tax is reported as $0 with an assumption note). Federal income & estate
tax are state-invariant and excluded from the delta; figures are real-dollar, undiscounted.

Unlike the other five tax tools, `analyze_relocation` **does** emit a structured `assumed_defaults[]`
array (every state-profile fallback it applied — no-SS-tax, $0 retirement-income exclusion, default
property rate, the 85% Social-Security convention) and a **`share_url`** when you pass `{ plan_id }`.
Read back the `assumed_defaults[]` and offer the link. Because this is a near-retiree decision, pair
it with the **`retirement-income`** skill (`analyze_withdrawal_strategy`, `optimize_social_security`,
`analyze_estate_exposure`) for the full decumulation picture once the state is chosen.

## Step 3 — Surface results honestly

For whichever tool you called:
- **Lead with the headline dollar figure** — annual tax savings, per-year conversion amounts +
  lifetime RMD tax avoided, AMT/NIIT crossover, remaining after-tax space, total surtax bite.
- **Read back the assumptions verbatim.** The five tax-optimization tools (`analyze_tax_optimization`,
  `optimize_multi_year_tax`, `analyze_roth_conversion`, `analyze_mega_backdoor_roth`,
  `analyze_advanced_taxes`) do **not** emit a structured `assumed_defaults[]` array — they apply
  silent Zod defaults (e.g. ordinary rate 0.24, cap-gains 0.15, bond allocation 0.2, standard
  deduction $29,200; Roth target bracket 0.12, RMD age 73, life expectancy 92) and expose what they
  assumed only as **prose in `disclosures.key_assumptions`**. `analyze_relocation` is the exception:
  it returns a structured **`assumed_defaults[]`** (read each one back). Either way, surface the
  assumptions so the user can correct any silent default.
- Honor `disclosures.not_advice` — present as planning estimates, not tax advice.
- **Follow `next_actions[]`** — each is `{ tool, why, prefilled_args }` (carrying `{ plan_id }`
  when available). Use these server-suggested chains rather than guessing the next call.
- **For a share link:** the five tax-optimization tools don't return one. `analyze_relocation` does
  return a `share_url` when called with `{ plan_id }`. Otherwise, run `generate_financial_plan`
  (Step 1) and surface its `share_url`.

## Recommended call sequence (typical session)

1. (optional) `generate_financial_plan` → capture `plan_id` (+ `share_url`).
2. Route by intent → one of the six tools (with `{ plan_id }` or raw fields).
3. Read back the headline + `disclosures.key_assumptions`.
4. Follow `next_actions[]` (often chains into another tax tool, the retirement-income
   `analyze_healthcare_bridge`, or `generate_financial_plan` for a share link).

## Fictional examples

**1.** *"I'm 52, $900k in a 401k and $400k taxable, 24% bracket — how do I cut my taxes?"*
→ `analyze_tax_optimization({ tax_deferred_balance: 900000, taxable_balance: 400000,
ordinary_tax_rate: 0.24, age: 52 })`. Lead with the annual asset-location tax-drag savings; offer to
turn on `enable_tlh` / `enable_charitable` if they have realized gains or giving intent. Read back
key_assumptions (cap-gains rate, allocation, standard deduction).

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

*(All examples use fictional figures — never reuse a real user's numbers in documentation.)*

## Notes

- All decimals are fractions; all dollars are today's (real) dollars; brackets/limits are ~2026
  (override `tax_year` as needed).
- Pass `{ plan_id }` to reuse a saved household model; any field you also pass is a shallow override.
- The five tax-optimization tools surface assumptions as **prose in `disclosures.key_assumptions`**,
  not a structured `assumed_defaults[]`, and they do **not** return a `share_url` — chain
  `generate_financial_plan` for a sharable link. (Server follow-up tracked in `SKILL_AUTHORING.md`.)
  `analyze_relocation` is the exception: it emits a structured `assumed_defaults[]` and a `share_url`
  (with `plan_id`).
- Near-retiree weighing a move? Pair this with the **`retirement-income`** skill for the
  decumulation side (withdrawal order, Social Security claiming age, estate-tax exposure).
- Self-employed / S-corp owner? The **`self-employed-planner`** skill sizes Solo 401(k) / SEP / SIMPLE
  room, the §199A QBI deduction, and the S-corp reasonable-salary tradeoff.
- Not financial or tax advice. Planning estimates only.
