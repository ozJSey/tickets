# BD-1 — bigdecimal-string 1.2.1: natural-scale operands, a separator standard, honest guards

**Owner, 2026-09-16**, deciding the audit's P0/P1 fixes one by one:
- Operands: *"we don't need to silently fail them, we can just call decimal significant digit instead"*
  — i.e. parse a raw operand at its **natural scale**, never coerce it to the receiver's.
- Separators: *"We should have a standart perhaps, default values to `.` as decimal separated and
  `,` as thousands but people should be able to modify."*
- `setScale(-1)` guard: proposed, not vetoed — in scope.

Context: the audit (2026-09-16, findings confirmed by planner probes against a fresh build AND the
published 1.2.0 tarball) — P0 operand coercion, P1 comma corruption, P1 stale CHANGELOG heading,
P2 negative-scale garbage, P2 dead `BigDecimalConfig` export, P2 test gaps.

## A. Operands parse at their natural scale — the P0, a one-line class of fix

Six call sites coerce with `new BigDecimal(other, this.scale)` (add:151, subtract:161, multiply:185,
divide:221, mod:259, compareTo:279 in `src/big-decimal.ts`). Drop the second argument: the operand
keeps its own scale and `alignScales`/the result-scale rules do their existing jobs. The wrapped
path is untouched; the string path becomes identical to it.

**Regression tests = the four confirmed probes**, pinned exactly:
- `bd("100.00").multiply("0.005")` → `"0.500"` (was `"1.00"`)
- `bd("1").divide("0.003")` → a real quotient (was: throws "Division by zero")
- `BigDecimal.sum("0.001","0.001","0.001")` → `"0.003"` (was `"0.00"`)
- `bd("0.10").add("0.005").add("0.005")` → `"0.110"` (was `"0.12"`)
Plus the gap class the auditor named: every arithmetic suite gains cases where the operand has
MORE decimals than the receiver — that shape being absent is exactly why 122 tests stayed green.

## B. Separator standard: `.` decimal + `,` grouping by default, configurable, round-trippable

Revive the dead `BigDecimalConfig` export (types.ts:11) as the real config:

```ts
interface BigDecimalConfig { decimal?: '.' | ','; group?: ',' | '.' | ' ' | '' }
```

- **Default:** `decimal: '.'`, `group: ','`.
- **Grouping is validated, not stripped.** `1,234.56` → 1234.56; `12,34.5` and `1,23` **throw**
  under the defaults — naive comma-stripping would read a European-habit `1,23` as `123`, which is
  the silent-corruption bug wearing a different coat. Accept only: no separators at all, or
  first group 1–3 digits with every subsequent group exactly 3.
- **Same symbol twice / decimal appearing before a group separator** → throw with a message that
  names both readings (`"1,23" is ambiguous: 1.23 (decimal comma) or malformed grouping…`).
- **Configurable at two levels:** per-call (`bd("1.234,56", { decimal: ',', group: '.' })` /
  constructor options arg) and app-wide default (`BigDecimal.setConfig({...})`), per-call winning.
  With flipped config, `1.234,56` parses as 1234.56.
- **`toFormat` reads the same config**, and the round-trip becomes a pinned test:
  `bd(x.toFormat(), cfg).equals(x)` for both dialects. Today the library cannot re-read its own
  formatted output; after this it can, by construction.

## C. Guards — small, loud, in scope

- `setScale` / `toFixed` / constructor precision: **non-negative integer or RangeError.**
  (`setScale(-1)` currently returns `"1.2"` from 123.45 — garbage, not tens-rounding. Real
  negative-scale semantics are out of scope; the guard closes the wrong-output hole now.)
- While in `parse`: `bd("1e")` currently returns `"0.10"` silently — reject trailing/dangling
  exponent input with the same clear error shape (this is finding #9's worst case; full input
  validation polish can wait, silent wrong numbers cannot).

## D. CHANGELOG truth

The published 1.2.0 tarball's own CHANGELOG says `[1.2.0] — UNRELEASED / Not on npm`; the registry
says 2026-09-14T10:01:33Z. Fix the heading, add `[1.2.1]` with A–C, and keep the house rule the
file itself states: nothing written down that the registry can't support.

## Version + sequencing

**1.2.1** — owner rule, 2026-09-16: *"We version too generously, stop it, we literally released
all these on Monday, we don't have many users."* A fixes provably wrong outputs (a patch by any
reading); B's config surface rides along rather than earning a minor. Publish via
`scripts/publish.mjs`; the owner runs `--publish`, never an agent.

## Acceptance — standing criteria (BOARD.md) apply

1. The four probe regressions above, red-before/green-after (negative control: revert the six
   call sites, watch exactly those go red).
2. Separator matrix: both dialects × {plain, grouped-valid, grouped-invalid, ambiguous, multi-sep}
   — invalid/ambiguous rows assert the THROW, not a value.
3. Round-trip property test over both dialects.
4. Negative HALF_UP/HALF_DOWN/HALF_EVEN ties pinned (probed correct today, unpinned — audit #6).
5. `readme-claims.spec.ts` stays the gate it is: README examples updated to show operand equality
   (`add("0.005")` === `add(bd("0.005"))`) and the separator standard, executed verbatim.
6. jsdom is sufficient here (pure BigInt) — no UNPROVEN rows expected.
