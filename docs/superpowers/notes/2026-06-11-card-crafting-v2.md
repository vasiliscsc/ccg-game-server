# Card Crafting — v2 feature sketch + v1 compatibility audit

**Date recorded:** 2026-06-11 (session 9, from the deferral scan — crafting was in the locked brainstorming scope but had zero spec presence and no recorded disposition)
**Status:** **NOT a v1 feature.** Recorded so v1 machinery supports it later — or can with slight modification. **All design forks below stay deliberately undecided until a v2 spec.** Do not implement anything from this note.

---

## The vision (user's sketch, 2026-06-11)

- A third per-player pool alongside health and mana: **aether**. Gained by **killing neutral minions**, via **artifact conditions**, as **battlecries**, etc.
- A **crafting phase** (trigger mechanism undecided: every X turns / meeting a condition / cast from a spell) in which a player **spends aether to purchase components** of a card:
  1. First pick the card kind — **minion / spell / artifact / sigil**;
  2. Then subsequent choices add **triggers, stats, deathrattle/battlecry** (for minions), etc.;
  3. Each option has an **aether cost, scaling with quality**.
- At session end the player receives the crafted card **in hand, costing no mana**.
- Phase placement undecided: **parallel intermission between turns** (both players craft simultaneously) vs **part of a player's turn** (shared timer).

## v1 compatibility audit — what already supports this

| v2 need | v1 mechanism that covers it |
|---|---|
| Multi-step choice sequence | `PendingChoice.context` stores an effect continuation — a chain is a continuation issuing the next `StartChoiceAction`. Works today by construction. |
| Choice visibility | Directed `ChoicePromptEvent` (full options to the chooser) + public thin `ChoiceStartedEvent` (`optionCount`) — the opponent never sees the components you browse (session-8 split). |
| Parallel two-player phase | **Mulligan precedent**: `GamePhase.Mulligan` + `MulliganState {player1Completed, player2Completed}` + per-player directed events. An intermission crafting phase is the same shape. |
| Composed card behavior | Cards are **data-driven per instance**: `Card.definition: JsonElement` + `handlerKey?` + the `DefaultCardHandler` recursive JSON descent (borrow-list Item 2). A crafted card = an assembled definition JSON handled generically. |
| Zero mana cost | Mint with `baseManaCost = 0`. Nothing special. |
| Aether as a pool | Additive `PlayerState.aether: int` + a `ModifyAetherAction`/`AetherChangedEvent` pair — the artifact `charges` precedent (single mutation path for a generic accumulator). |
| Aether from artifact conditions / battlecries | Ordinary triggers/effects enqueueing the aether action — no new machinery. |

## The one mechanism to reserve (the "slight modification")

**A match-local definition-registry overlay.** The v1 backbone says `definitionKey` is the *sole* link into the card library — fabrication, graveyard snapshots, resurrection, handlers, traces all lean on it. Crafting must NOT break this, and doesn't have to: the crafting session **mints a fresh key** (e.g. `crafted:p1:3`) and **registers the composed definition into a per-match overlay** of the library. Everything downstream then works unchanged — graveyard recast, resurrection, bounce, `GiveCardAction`, trace records — because the key resolves. Replay stays deterministic: the overlay's contents derive from logged crafting choices (the input log), same as any other choice. `definitionKey` remains the sole link; the library just gains a match-scoped layer.

Consequence for v1: **keep the library access behind a seam that *could* take an overlay** (a lookup interface, not a static dictionary baked into handlers). This is the only thing v1 implementation should consciously do for crafting — and it is good hygiene anyway.

## Seams flagged for v2 (do not build now)

- **Kill attribution** ("gain aether when you kill a neutral minion"): `MinionDiedEvent` carries no killer. v2 either adds a `killingSourceId` or correlates with the preceding `DamageTakenEvent {sourceId}` (the session-7 fatigue idiom — identify by the preceding event). Open.
- **Priced options:** `ChoiceOption` gains an optional aether `cost` + spend-validation on `SubmitChoiceAction`. Additive.
- **Phase trigger:** every-X-turns = a Turn Lifecycle step; spell-cast = an ordinary action; intermission = the mulligan-shaped parallel phase. All three have homes; choosing is a v2 design call.
- **Trace:** `state` records elide definition bodies (`definitionKey` only) — crafted keys need the overlay dumped once per mint (one new trace record kind, or a field on the minting `action` record).
- **Product-model residual:** the overlay makes composed definitions *library-resident*, so the old "pre-registered (Kazakus) vs parameterized (Zombeast)" fork collapses to "what the crafting UI assembles" — a v2 content/design question, not an architecture one.

## Foreclosure check (v1 must not...)

Nothing in v1 currently forecloses the sketch. Watch-items: keep `GamePhase` extensible (it is — an enum); keep `PlayerState` additive (it is); keep library lookup behind the seam above (action item for Epic 01/02 implementation); do **not** let any handler cache definitions by key at match start in a way that ignores a later overlay registration.

## Plan impact

**None for v1 epics** beyond the library-lookup-seam hygiene note (fold into the Epic 01/02 reconciliation as a one-line implementation constraint). Crafting itself = a future v2 epic, written when the v2 spec exists.
