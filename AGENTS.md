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
- **End-of-owner-turn** occupation captures territory; ownership persists after departure. Capture a player's Injection Point to eliminate them; the hostile center cannot be captured normally. **Open locations economically belong to the station, not to a fifth player.** At the **end of the Station Phase**, each location occupied by a station Process becomes **Open**, removing any player's control and discarding a replaced/removed Module. Eliminated players' territory becomes Open.
- **Station phase (LOCKED v3.0):** after every surviving player has taken a turn, the hostile station faction acts in its own once-per-round phase, then the round ends. All hostile deployed forces belong to the station; the station does not occupy a position in randomized player order. **Every deployed hostile Process may activate once per Station phase (reaffirmed and LOCKED v3.5)**. The station plays its **own deck and resources** using the same card rules as players; escalation irreversibly increases its **draw-to hand-size target** from **4 → 5 → 6**, with **7** planned for an as-yet-undefined additional trigger. It does **not** automatically deploy a force of 3/5/6+ Processes on escalation. **Inward progress from starting edges** is LOCKED, and **dotted escalation-zone boundaries must be shown on the board**. **LOCKED (v3.3):** for the standard 9×9 board, Tier 1 begins in the inclusive central **5×5 square** (zero-based rows and columns **2…6**); Tier 2 begins in the central **3×3 square** (rows and columns **3…5**). **Any player's Process reaching a zone triggers it**; escalation never recedes. The two nested square perimeters are drawn as **dotted lines**, exactly as in the approved board diagram. A boundary cell counts as inside. For other boards, configure explicit zone geometry; do not use remaining distance-to-center or the superseded 3/6 step suggestions.
- 30 cards/deck; max 3 copies of a card; 1 Operator outside deck. Player starting hand 4; at each round start optionally discard any number and draw **up to 4**, no hand limit. The station instead draws up to its escalating **4/5/6** target (eventual 7 trigger OPEN). Draw pile reshuffles from Discard when exhausted; Graveyard stays separate.
- **Player** round-start income **1 per owned cell, max 5 *income per round***, stored resources uncapped. Standard player starting bank 2; initial owned Injection Point yields 1 on round 1. **Station** income depends on Open cells, with a **LOCKED minimum 2/round** and **PROVISIONAL** formula `max(2, min(5, ceil(openCells / 15)))`; don't mistake the provisional formula/cap for approved final balance. Accumulated station resources are not capped.
- Each player activates **one** Process per turn. Slow/Normal/Fast all Move at most one edge; Slow cannot Move on deployment round or after moving previous round, Normal cannot Move on deployment round, Fast can.
- Process Attack is a **single stat**; damage persists. Entering an enemy-occupied connected neighboring cell initiates simultaneous melee. Shooting allows simultaneous retaliation only if defender can reach shooter.
- Operator has exactly one passive and one activated ability; activated ability at most once/round and does not consume a fixed action. **Activated abilities are restricted to their Operator owner's turn (LOCKED v2.9); reactive off-turn play belongs to Commands.**
- Commands can be used during anyone's turn, as responses, globally by default; paid on play, LIFO response resolution, canceled Commands go to Discard. Some detailed priority rules remain open.
- Provisional scoring: +1/owned cell at end of player's turn; +1/per boss damage; +50 to surviving player with largest total contribution when boss is destroyed.

Refer to GAME_DESIGN.md for important exceptions and definitions. This summary is intentionally insufficient as an implementation spec.

## Suggested architecture (guidance, not a mandated framework)

Define concepts explicitly:
- **GameState:** players, turn order, round, current player/phase, ruleset, board graph, pending Command stack, station faction with its own card zones/resources and persistent escalation tier, score, event history.
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
2. **Turn phases:** round-start discard/draw/income order; player turns in fixed randomized order; exactly four optional action slots in fixed sequence; at most one player Process activates; **one station phase after all player turns**. **Station exception LOCKED v3.5:** each deployed hostile Process may activate once per Station phase, unlike the one-Process-per-turn player limit. Apply ordinary Process action legality and movement cooldowns to each hostile Process. Escalation tiers only rise; store the highest attained tier.
3. **Economy:** player income cap applies to *income*, not storage. Resources bank across turns; 2 initial + 1 first-round income for the standard player. Station income has floor 2 per round, provisional scaling with Open cell count and no storage cap. Captures/reclamation do not retroactively modify this round's income.
4. **Spawn/ownership:** deployment only at empty appropriate spawns, end-turn player capture, end-of-Station-Phase reclamation to Open at station-occupied cells, persistent ownership elsewhere, Modules destroyed on control change, elimination makes territory Open. Ensure the station can deploy to appropriate station spawn locations without treating all Open cells as spawn locations.
5. **Movement:** each type's deployment-round eligibility, Slow previous-round cooldown including blocked melee moves, exactly one graph edge of movement.
6. **Combat:** simultaneous attacks; ranged retaliation only in range; persistent damage; no defense stat; both-dead melee stays empty and location ownership unchanged.
7. **Zones:** hand-discarded Process → Discard; destroyed board Process → Graveyard; Update → Discard; replacement cleanup; reshuffle excludes Graveyard.
8. **Commands:** any-turn play, LIFO nested responses, paid-before-resolution, canceled Commands to Discard, global default targeting with card exceptions.
9. **Operator abilities:** passive effects disabled upon elimination, once-per-round activated usage, not consuming fixed actions; **owner-turn-only is confirmed; reject off-turn Operator activations**.
10. **Station activation exception:** each deployed hostile Process gets up to one legal activation in the Station phase; never cap total station activations at one, and never grant an individual hostile Process multiple activations by default. Test a station with 0, 1, and multiple deployed Processes.
11. **Station escalation:** station has its own deck/resources and regular draw-to target **4 initially, 5 at Tier 1, 6 at Tier 2**; a future 7-card tier lacks a trigger. Escalation does **not** force-spawn Processes or immediately draw cards. Standard board: entering both row and column ranges **2…6** reaches Tier 1; **3…5** reaches Tier 2. UI displays **dotted square zone boundaries**, separately from graph links. Test inclusive boundaries, direct Tier 2 entry, retreat/death, monotonicity, draw-to without hard hand cap, and no spawning on escalation.
12. **Station reclamation/economy:** Open is a station economic state; any surviving station Process occupying a player-controlled location when the Station Phase ends returns it to Open and destroys its Module. Test empty locations remaining controlled, destroyed station Processes not reclaiming, late-phase reclaim income applying next round, minimum 2 income, and provisional formula boundaries.
13. **Scoring:** boss damage accumulated per player throughout match; only surviving top contributor receives provisional +50; territory points at end of player's turn.

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
6. Station-faction card-driven action policy, detailed ordering of hostile activations, third escalation threshold, central objective and scoring once unresolved detailed rules are agreed.
7. Playable single-player UI and AI opponents using the same rules interface; future multiplayer.

These are suggested implementation milestones, **not permission to decide open gameplay rules**.

## Known unresolved design decisions

See **GAME_DESIGN.md §11**, especially central enemy behavior/stats, contribution ties, Command priority details, range/line-of-sight and the initial card catalog. An agent should surface these prominently when they block implementation.

## Repository housekeeping

- Respect the repository's default branch and any instructions that the owner adds later.
- Use feature branches/PRs for substantial work when practical; keep commits reviewable.
- Do not add copyrighted card art, names or other third-party assets without permission.
- Avoid exposing secrets, personal identifiers or credentials in code, sample data, logs or docs.
