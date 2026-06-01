# Future Modifier System — design note (not yet specced)

**Date raised:** 2026-05-31 (during borrow-list Item 4 discussion)
**Status:** DEFERRED — design seam reserved, machinery intentionally not built. Do **not** implement until a card actually needs a player-scoped modifier.

---

## Why this note exists

Item 4 (Spell Damage +X) and **mana-cost reduction** turn out to be the *same shape*. The minion-bound and snapshot parts of spell damage were resolved under Item 4 and are local to spell resolution. The part that is **shared** with cost reduction — and therefore should be designed **once, for both** — was carved out to here so we don't bake a spell-power-specific version of it into spell damage now and refactor later.

## The shared shape

> A **scoped, filtered, lifecycle-bound modifier to a named quantity, aggregated at a read point.**

| Axis | Spell Damage | Cost Reduction |
|---|---|---|
| Quantity | damage of a spell | cost of a card |
| Source: minion-bound | "Spell Power +1" minion / aura | "while this minion's alive, spells cost 1 less" |
| Source: player persistent (this turn) | "all spells +1 this turn" | "all spells cost 1 less this turn" |
| Source: player consumable (next N) | "next spell +3" | "next card costs 1 less" / "next beast is free" |
| Filter | spells (maybe "Fireballs") | all cards / spells / a tribe / a specific card |
| Combine | add / double / set | add / set-to-0 ("free") |
| Consumed on | casting an applicable spell | playing an applicable card |

It's broader than these two — bonus draw, heal amplification, "your next minion has Rush" share the same skeleton.

## What is genuinely shared (the reusable substrate — build once)

- **Lifecycle / scope:** `Persistent(sourceId)` | `EndOfTurn` | `Uses(n)` (decrement-and-drop on each applicable use). End-of-turn clearing; consume-on-applicable-use.
- **Filter / predicate:** "which cards/spells does this apply to." **Reuse Item 2's `ITriggerCondition` vocabulary** (`CardTypeIs`, `MinionTypeIs`, `CostAtLeast`, …) — do **not** invent a third predicate language (Item 12 selectors are the other one; keep the family small).
- **Ordered combination:** add, then multiply, then set (non-commutative; needs an explicit order, not a flat sum).

Sketch (illustrative, not locked):
```
ActiveModifier {
  target: ModifierTarget          // CardCost | SpellDamage | (future)
  amount: int                     // or a dynamic provider
  combine: Add | Multiply | Set
  filter: <Item-2 condition tree>
  scope:  Persistent(sourceId) | EndOfTurn | Uses(n)
}
```

## What must stay per-quantity (do NOT unify the binding)

This is why it's a *bounded* abstraction, not a god-engine:

1. **Read cadence / binding differs.** Cost is a **standing value shown in hand**, recomputed on every board/hand change (aura cadence), and it **drives legality** (the action validator and Item 5 `GetLegalActions` read it). Spell damage is an **instantaneous snapshot at cast**, never displayed, never affects legality. Same aggregation, different storage + read site.
2. **Cost is already half-modeled.** Spec has `Card.modifiers: StatModifier[]` (`costDelta`) + `effectiveCost = max(0, baseManaCost + Σ…)`. A modifier system must *reconcile* on-card modifiers with player-scoped ones, not start clean.
3. **Auras don't reach hand cards today.** `IAura` emits `AuraEffect(TargetMinionId, …)` — board minions only. Both "spell power aura" and "spells cost 1 less while alive" need effects that reach *cards in hand / the player*. This is a **shared new capability** and the strongest argument for doing the two together.
4. **Floors:** both clamp at 0 ("free", non-negative spell damage).

## Recommended depth (when we build it)

Extract the **substrate** (lifecycle + filter + ordered combine), keep the **binding** per-quantity (cost rides the existing `StatModifier`/`effectiveCost` rail extended for player-scoped sources, recomputed; spell damage is the snapshot aggregate). **Defer the fully-general `ModifierTarget` enum** until a *third* quantity appears (rule of three; abstracting two un-built features is exactly when abstractions come out wrong).

## Trigger to revisit

The **first card** that needs a player-scoped / consumable / turn-scoped modifier — spell-damage *or* cost. At that point design the substrate once, informed by both, and wire whichever quantity the card needs. Until then: nothing to build.

## Connections

- **Item 2** (`ITriggerCondition`) — shared filter/predicate vocabulary. Reuse, don't fork.
- **Item 4** (Spell Damage +X) — resolved the minion-bound + snapshot parts; sent the player-scoped part here.
- **Item 5** (`GetLegalActions`) / action validator — consumers of the *cost* binding (legality reads `effectiveCost`).
- **Item 9** (source attribution) — who gets credited for a modifier, for display/tooltips.
- **Item 12** (targeting selectors) — the other predicate registry; keep the family unified.
