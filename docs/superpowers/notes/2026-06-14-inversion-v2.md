# Inversion — deferred to V2 (parked 2026-06-14)

**Decision:** Inversion (the Attack↔Health swap mechanic, formerly "the signature custom mechanic") is **parked as a v2 feature**. v1 builds **nothing** for it. This note is the v2 design seed — it carries every decision and open question reached before parking, so v2 resumes from here, not from scratch. (Precedent: `notes/2026-06-11-card-crafting-v2.md`.)

**Why parked:** Walking the inversion stat-math against pre-existing enchantments (the "1/6 buffed to 4/6, damaged to 4/2, then inverted" case) exposed that inversion is a deep, ramifying mechanic — enchantment-flip rules, keyword behaviour under inversion, the damage-memento typing (#10c), and card-definition `normal`/`inverted` sectioning all hang off it. It is the single most complex item in the design and touches the most surfaces, while being a *novel* mechanic with no Hearthstone reference to lean on. Pulling it to v2 lets v1 ship a coherent HS-faithful core; inversion returns as a designed-in-full expansion.

**What parking dissolved (v1 wins):**
- **#10c is moot.** The `StatModifier { sourceId: "inversion" }` sentinel existed *only* for the inversion damage-memento. No inversion → no memento → no sentinel-vs-`kind`-field question. The fix-pass batch-1 blocker is gone.
- The **enchantment-flip-on-invert** open question (below) is no longer a v1 blocker.

**What parking must NOT unwind:** the **cascade-settle death cadence** (pending-death / mortally-wounded lingering, §4) was originally clinched by *two* references — R1 Intervention × lethal **and** R2 Inversion × lethal. R2 is gone, but **R1 (heal a mortally-wounded minion back above 0 in the dying window) still fully justifies the cadence.** The death-wave design is unchanged. The spec's dying-window save examples were reworded from "invert to save" to "heal to save."

---

## v1 footprint removed (what v2 must re-add)

Spec (`2026-05-26-game-mechanics.md`) — all removed 2026-06-14:
- §1 `MinionOnBoard.isInverted`; `Card.isInverted`; the `definition` **`normal`/`inverted` two-section split** (v1 definitions are now **flat** — re-adding the `inverted` section in v2 is purely additive, no migration); `GraveyardMinion`/`GraveyardSpell` `isInverted`.
- §2A `InvertTargetAction`, `UnInvertTargetAction` rows; the inversion clause in `DestroyMinionAction`.
- §2B `MinionInvertedEvent`, `CardInvertedEvent`.
- §3 `ITrigger` catalog: the **On Invert** trigger type.
- Identity/replay: `isInverted` retain-table row + "copied through unchanged" replay line.

Plan: Epic 14 marked ⛔ deferred (file retained as the v2 seed); references scrubbed from Epic 03, Epic 16, README.

---

## Carried-forward design (decided before parking — reuse in v2)

### A. Stat-math — option 3 (PINNED 2026-06-11, full record)

`currentHealth` and `attack` **trade places, on aura-exclusive values; auras re-applied after** (the existing per-action ⑥ recalc rebuilds bonuses on the new orientation, so the handler operates on aura-stripped values).

`InvertTargetAction` ④ handler, in order:
1. **Strip auras:** `A = attack − auraAttackBonus`, `H = currentHealth − auraHealthBonus`, damage `D = (auraless maxHealth) − H`. The handler never touches `aura*` fields.
2. **Orientation flip:** base pair **swaps systemically** (`baseAttack ↔ baseHealth` — the inverted definition section defines *effects/triggers*, never its own stat line) **and every enchantment's deltas swap** (`attackDelta ↔ healthDelta`). Forced by self-consistency: makes the new auraless `maxHealth` equal exactly `A`.
3. **Trade:** `currentHealth ← A` (lands at **full flipped health**); the attack side absorbs the wound via a system `StatModifier { attackDelta: −min(D, auraless maxHealth), sourceId: "inversion" }` — damage persists as **reduced attack**, not as damage; attack stays derived, never stored negative.
4. **Emit** `MinionInvertedEvent`; card-handler swaps to the inverted section's triggers (On Invert fires); ⑥ re-applies auras on the new orientation (aura grants don't flip — a +1/+1 aura re-applies as +1/+1).

**Consequences (accepted):** re-inverting an unhealed minion re-trades the penalty back as a health delta (the wound returns — an R2-saved minion re-doomed if re-inverted before healing); **Silence forgives the traded damage** (clears the memento like any enchantment). **R2 × lethal:** a mortally-wounded minion (`H ≤ 0`) inverted in the dying window returns at full flipped health, 0 attack, survives ⑦ iff `A > 0`. Worked examples: 4/5 @ 7 dmg → 4/4 full, attack 0, lives; 4/5 @ 2 dmg → 4/4 full, attack 3. **Destroy-marked minions:** invert executes the trade but the mark survives (dies at settle). **Card-in-hand inversion** = the flip minus the trade (no `currentHealth`).

The damage memento is the home of the **#10c** question — re-evaluate sentinel (`sourceId: "inversion"`) vs a `StatModifier.kind: CardEffect|SystemMemento` field when v2 lands. The re-invert handler must distinguish the memento (reconvert to damage, preserve maxHealth) from a normal enchantment (flip its deltas), so the branch is load-bearing — mild argument for the `kind` field in v2.

### B. OPEN QUESTION — do enchantments flip on invert?

Surfaced 2026-06-14, **not resolved** (parking pre-empted it). The pinned stat-math (A) says enchantment deltas flip (`attackDelta ↔ healthDelta`) — so a +3-**attack** spell becomes +3-**health** after invert. Worked example: a 1/6 base, +3-attack enchanted (4/6), damaged 4 (4/2), inverted → **2/4** (base 6/1; flipped enchantment {atk 0, hp +3}; inversion memento {atk −4}).

The alternative reading: a spell that reads "+3 Attack" should *stay* +3 attack through inversion (re-applied like an aura). **Which reading matches design intent is the first v2 question to settle** — it determines whether enchantments are orientation-bound (flip) or label-bound (persist). The flip reading is what option-3's self-consistency math assumes; the persist reading would need different arithmetic.

### C. Keyword behaviour under inversion (explored 2026-06-14; design direction, not pinned)

The intent: inversion should flip a minion's keywords from the **defensive register into the offensive one**, mirroring the stat swap (attack↔health). Pairing keywords on the **offense↔defense axis** (not the same-attribute axis) yields four **native** pairs (both keywords already in v1):

| Defensive | ⟷ | Offensive | Shared mechanism, transposed |
|---|---|---|---|
| **Divine Shield** | ⟷ | **Poisonous** | one instance of damage is absolute: negate the first hit taken ⟷ kill with the first hit landed |
| **Enrage** | ⟷ | **Lifesteal** | damage converts into the other stat: taken→+attack ⟷ dealt→+health (this pair *is* the attack↔health swap as a keyword) |
| **Reborn** | ⟷ | **Windfury** | "you get a second one": a second life ⟷ a second attack |
| **Taunt** | ⟷ | **Charge** | dictate engagement: the fight must come to me, here ⟷ I bring the fight, now (Rush = minor-key Charge) |

Two keywords have **no native mirror** (would need invented anti-keywords if inversion is a *total* keyword-flip): **Stealth** ⟷ *Infiltrator* (can't be targeted ⟷ can't be blocked / ignores Taunt); **Spell Damage +X** ⟷ *Spell Mending +X* (damage output ⟷ heal output).

**Open v2 design choices on keyword-flip:**
- Whether inversion flips keywords at all, or only stats (smaller mechanic; an inverted minion keeps e.g. its Charge despite flipping identity).
- If it flips: the mapping must be an **involution** (invert twice = identity) — Tier-1 four qualify; **Charge↔Taunt is lossy** because Rush has no defensive counterpart (fold Rush into Charge, or invent "minions-only Taunt").
- Whether to design the invented anti-keywords (`Infiltrator`, `Spell Mending`) so the flip is *total*, or leave unpaired keywords untouched (asymmetric).
- The strongest argument *for* keyword-flip: **Divine Shield→Poisonous** and **Enrage→Lifesteal** aren't bolted-on rules — they're the same attack↔health transposition the stats already perform, re-expressed as keywords, so they'd feel systemically consistent.
