# tax-optimizer (Claude Agent Skill)

Cut taxes across accounts and years: broad tax optimization (asset location, tax-loss harvesting, charitable), multi-year ISO/conversion timing (AMT/NIIT/IRMAA-aware), Roth conversion ladders, mega-backdoor Roth space, advanced surtaxes (NIIT/AMT/state), tax-gain harvesting (how much long-term gain you can realize at the 0% capital-gains rate before NIIT/IRMAA bites), and retirement relocation / state-tax arbitrage (compare the lifetime after-tax cost of retiring in state A vs B). Thin orchestration over the planfi MCP.

> **Retirement relocation:** *"I'm retiring in California but thinking about Texas — $80k of IRA
> withdrawals, $40k Social Security, a $600k house, ~$60k spend. Worth the move?"* →
> `analyze_relocation` returns the annual + lifetime after-tax difference, a one-time state-estate-tax
> delta, and a `move` / `stay` / `marginal` call with the dominant driver named. (Fictional figures.)
> Pair with the **retirement-income** skill for the decumulation side once the state is chosen.

It's a **thin orchestration layer** over the public **planfi MCP** (`https://ai.planfi.app/mcp`,
public, no auth) — all the math and financial logic live server-side. The skill itself bundles no
engine; it just gathers inputs and calls the tools.

### MCP bootstrap (one time)

If the planfi tools aren't connected yet, run:

```
claude mcp add --transport http planfi https://ai.planfi.app/mcp
```

On **claude.ai**: Settings → Connectors → add a custom connector pointing at
`https://ai.planfi.app/mcp` (no auth). The skill also reminds you to do this if the tools are
missing when you invoke it.

## Install

### Quickest — skills.sh CLI (recommended)

```
npx skills add holdequity/planfi-tax-optimizer
```

### Claude Code — copy the folder (zero prerequisites)

User-level (available in every project):

```
cp -r skills/tax-optimizer ~/.claude/skills/
```

Or project-level (only this repo/project):

```
mkdir -p .claude/skills && cp -r skills/tax-optimizer .claude/skills/
```

Restart Claude Code (or start a new session). The skill auto-loads by its description.

### Claude Code — as a plugin (shareable one-liner)

```
/plugin marketplace add holdequity/planfi-tax-optimizer
/plugin install tax-optimizer@planfi
```

### claude.ai — upload as a skill

1. Zip the folder: `cd skills && zip -r tax-optimizer.zip tax-optimizer`
2. In claude.ai go to Settings → Capabilities → Skills and upload the zip.

## Notes & honest caveats

- All decimals are fractions (24% → `0.24`); all figures are today's (real, inflation-adjusted)
  dollars; tax brackets/limits are approximate ~2026 values.
- Every tool returns a structured `assumed_defaults[]` array (`{ field, assumed_value, note }`); the
  skill reads each entry back so you can correct any silent default. Each tool also returns a
  `share_url` when passed a `plan_id` that resolves a saved household.
- Not financial advice. Planning estimates only.

See `SKILL.md` for the full instructions, exact tool params, and output format.
Source + issues: <https://github.com/holdequity/planfi-tax-optimizer>.

## License

MIT — see [LICENSE](./LICENSE).
