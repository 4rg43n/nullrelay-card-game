# AGENTS.md — NullRelay Card Game

## Purpose

This repository implements the original **NullRelay** tactical board/card game. Read **[GAME_DESIGN.md](GAME_DESIGN.md)** *before* planning or changing gameplay. This file tells future human/AI contributors how to preserve the design; GAME_DESIGN.md is the canonical game specification.

This project is **in the design / pre-implementation phase**. The initial implementation is intended to be playable with **P1 human, P2–P4 AI**, plus a hostile station AI faction; multiplayer can follow. The user owns game-design decisions.

## Non-negotiable collaboration rules

1. **Do not invent or silently 'fix' gameplay rules.** Preserve everything marked **LOCKED** in GAME_DESIGN.md. Treat **PROVISIONAL** values as defaults suitable for balancing, not immutable constants. Treat **OPEN** items as questions, not permission to decide.
2. **Ask before changing mechanics.** If implementation cannot proceed without an open decision, make the smallest possible isolated assumption, label it clearly **temporary / not approved**, and expose it via configuration or TODO; prefer asking the owner first.
3. **Preserve phase ordering:** Start one Process → Update one Process → Mod one location → Activate one Process. Each is optional, at most once, and in this order. Commands and eligible Operator abilities never reorder the slots.
4. **Favor a pure deterministic game-rules core** independent of rendering, network, persistence or AI. UI should issue legal action intents and display the authoritative resulting state; it must not calculate ownership, damage, resources, or rule legality by itself.
5. **Do not hardcode 'current' balance defaults** throughout the engine. Use board/config rules for grid dimensions, starting resources and income cap; card data for printed stats and costs.
6. **Keep precise zone semantics:** Discard and Graveyard are distinct. Process discarded from hand is not the same as Process destroyed on board. Attached Updates must be cleaned up.
7. **Do not overwrite work belonging to another contributor.** Inspect the branch/diff, use focused commits, and explain breaking changes. No implicit force pushes or deletion of unrelated files.
8. **Keep documentation synchronized.** When the user approves a new rule, edit GAME_DESIGN.md in the same change as any related code/tests. Record unresolved questions rather than concealing them.

## Canonical baseline in brief

- Standard map: **9×9**, orthogonal connection graph. Player Injection Points at the four edge midpoints; hostile station Injection Point at center. Only Processes occupy cells; maximum one Process per cell and one Module per location.
- **End-of-owner-turn** occupation captures territory; ownership persists after departure. Capture a player's Injection Point to eliminate them; the hostile center cannot be captured normally.
- **Station phase (LOCKED v3.0):** after every surviving player has taken a turn, the hostile station faction acts in its own once-per-round phase, then the round ends. All hostile deployed forces belong to the station; the station does not occupy a position in randomized player order. **Every deployed hostile Process receives one activation per Station phase (LOCKED v3.1)**. The station escalates irreversibly through provisional force-size tiers **3 → 5 → 6+** when players advance toward its center. The exact trigger metric is **OPEN**: distance-to-center 6 is crossed before distance-to-center 3, so do not silently implement thresholds as radial distances.
- 30 cards/deck; max 3 copies of a card; 1 Operator outside deck. Start 4 cards; at each round start optionally discard any number and draw **up to 4**, no hand limit. Draw pile reshuffles from Discard when exhausted; Graveyard stays separate.
- Round-start income **1 per owned cell, max 5 *income per round***, stored resources uncapped. Standard starting bank 2; initial owned Injection Point yields 1 on round 1.
- Each player activates **one** Process per turn. Slow/Normal/Fast all Move at most one edge; Slow cannot Move on deployment round or after moving previous round, Normal cannot Move on deployment round, Fast can.
- Process Attack is a **single stat**; damage persists. Entering an enemy-occupied connected neighboring cell initiates simultaneous melee. Shooting allows simultaneous retaliation only if defender can reach shooter.
- Operator has exactly one passive and one activated ability; activated ability at most once/round and does not consume a fixed action. **Activated abilities are restricted to their Operator owner's turn (LOCKED v2.9); reactive off-turn play belongs to Commands.**
- Commands can be used during anyone's turn, as responses, globally by default; paid on play, LIFO response resolution, canceled Commands go to Discard. Some detailed priority rules remain open.
- Provisional scoring: +1/owned cell at end of player's turn; +1/per boss damage; +50 to surviving player with largest total contribution when boss is destroyed.

Refer to GAME_DESIGN.md for important exceptions and definitions. This summary is intentionally insufficient as an implementation spec.

## Suggested architecture (guidance, not a mandated framework)

Define concepts explicitly:
- **GameState:** players, turn order, round, current player/phase, ruleset, board graph, pending Command stack, central enemy, card zones, score, event history.
- **Board/Location:** stable ID, row/column, graph edges, type, owner, occupying Process ID, optional Module ID.
- **PlayerState:** Operator, banked resources, hand/deck/Discard/Graveyard, eliminated flag, per-match boss contribution, action-slot completion and per-round ability usage.
- **Card definitions vs instances:** immutable printed definitions and stable instance IDs for damage, attachments, temporary status and provenance. Card effects should be testable without UI.
- **Action intents:** explicit typed inputs to an authoritative validator/resolver (start, update, mod, move/melee, shoot, ability, pass, Command, Command response, pass priority, end step).
- **Events:** deterministic outputs usable by UI, logging, AI planning, replay and tests (damage dealt, card moved, capture, elimination, income, score, stack push/resolve, etc.).
- **RNG:** seedable and isolated for deck shuffles and initially randomized turn order. Keep the randomized player order fixed after setup.
- **AI:** consumes the same state/legality/action API as the human client; never bypasses costs, range, step sequence or information boundaries.

Do not implement a speculative networking layer or large content system before the core rules are verified.

## Rules-engine invariants and test priorities

Write tests at the rule layer. Cover at minimum:

1. **Board setup:** 9×9, four edge Injection Points and one center enemy objective; links orthogonal by default; occupancy max one.
2. **Turn phases:** round-start discard/draw/income order; player turns in fixed randomized order; exactly four optional action slots in fixed sequence; at most one player Process activates; **one station phase after all player turns** with one activation for **each deployed hostile Process** (not an extra ordinary player turn). Escalation tiers only rise; store the highest attained tier.
3. **Economy:** income cap applies to *income*, not storage. Resources bank across turns; 2 initial + 1 first-round income. No income from locations captured later that round.
4. **Spawn/ownership:** deployment only at empty owned spawns, end-turn capture, location ownership persists without occupant, Modules destroyed on capture, player elimination effects.
5. **Movement:** each type's deployment-round eligibility, Slow previous-round cooldown including blocked melee moves, exactly one graph edge of movement.
6. **Combat:** simultaneous attacks; ranged retaliation only in range; persistent damage; no defense stat; both-dead melee stays empty and location ownership unchanged.
7. **Zones:** hand-discarded Process → Discard; destroyed board Process → Graveyard; Update → Discard; replacement cleanup; reshuffle excludes Graveyard.
8. **Commands:** any-turn play, LIFO nested responses, paid-before-resolution, canceled Commands to Discard, global default targeting with card exceptions.
9. **Operator abilities:** passive effects disabled upon elimination, once-per-round activated usage, not consuming fixed actions; **owner-turn-only is confirmed; reject off-turn Operator activations**.
10. **Station escalation:** each deployed hostile Process activates once during the Station phase; escalating tier never decreases after Process retreat/death. Test thresholds once their distance/progression definition is approved.
11. **Scoring:** boss damage accumulated per player throughout match; only surviving top contributor receives provisional +50; territory points at end of player's turn.

Add regression tests before fixing a bug. For every new mechanic, test both an ordinary case and at least one counterexample.

## Documentation and design-change process

For a new decision:
1. Identify the relevant **OPEN** item or affected **LOCKED** rule.
2. Have the owner explicitly agree; if it changes a locked rule, call out the change.
3. Update GAME_DESIGN.md and this file if an implementation invariant changes.
4. Add or update tests before merging behavior changes.
5. Summarize the gameplay impact in the commit or PR.

Use concise, legible Markdown and specific examples. Prefer stable terminology (**Operator, Process, Command, Update, Module, Injection Point, Integrity, Attack, Range, Slow/Normal/Fast, Discard, Graveyard**).

## Scope / initial development priorities

1. Data model and seedable deterministic simulation, including board graph and card-zone management.
2. Setup, turn/round transitions, legality checks, resource bank and territory control.
3. Process deployment, activation, movement, melee, shooting and destruction.
4. Fixed action slots, Updates and Modules, then Commands and stack/priority.
5. Operator passive/activated effects with owner-turn-only activated timing.
6. Station-faction action policy, escalation triggers and force-size/spawning rules, central objective and scoring once unresolved detailed rules are agreed.
7. Playable single-player UI and AI opponents using the same rules interface; future multiplayer.

These are suggested implementation milestones, **not permission to decide open gameplay rules**.

## Known unresolved design decisions

See **GAME_DESIGN.md §11**, especially central enemy behavior/stats, contribution ties, Command priority details, range/line-of-sight and the initial card catalog. An agent should surface these prominently when they block implementation.

## Repository housekeeping

- Respect the repository's default branch and any instructions that the owner adds later.
- Use feature branches/PRs for substantial work when practical; keep commits reviewable.
- Do not add copyrighted card art, names or other third-party assets without permission.
- Avoid exposing secrets, personal identifiers or credentials in code, sample data, logs or docs.
