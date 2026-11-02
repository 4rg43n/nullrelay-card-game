# NullRelay — Game Design

> **Design baseline:** v3.0 · 2026-11-02  
> **Status:** Pre-implementation design. Sections marked **LOCKED** reflect decisions made in the design conversation. **PROVISIONAL** values may be balanced. **OPEN** questions must not be silently decided by an implementation agent.

## 1. Concept and design goals

NullRelay is an original competitive tactical board/card game in the NullRelay science-fiction setting. Operators infiltrate and compete for territory on a space-station floor, deploy software-like **Processes** into physical locations, and use **Updates**, **Modules**, and reactive **Commands** to change the battle. The hostile **station itself** is the shared enemy; any hostile forces it deploys act as its extensions, not as independent enemy factions.

- Play on a spatial graph instead of an abstract row of creatures.
- Make the **Operator** materially define a deck's strategy.
- Keep cards readable and physical-tabletop-friendly: one primary Attack value, bounded attachments, simple movement and persistent damage.
- Preserve tactical interaction: fixed action sequence, contested space, responsive Commands and cumulative boss damage.
- Support up to four player factions; the initial solo prototype uses P1 human and P2–P4 AI, plus the central hostile AI. Multiplayer is a future goal.

No implementation language, UI framework, complete card catalog, lore text, or engine architecture has been approved yet.

## 2. Board, players and locations — LOCKED

**Standard board: 9×9** square grid. The four player **Injection Points** are placed at the midpoint of the north, east, south and west edges. The hostile enemy Injection Point is in the exact center. Other boards may vary dimensions, connections and starting resources.

The board is a **graph of locations**. Initially, only orthogonally adjacent north/east/south/west locations are linked; future boards/cards can create or break connections. Never assume that visually adjacent cells are necessarily connected.

Every location tracks:
- **Kind:** normal, spawn, or Injection Point.
- **Ownership:** neutral or one of up to four players.
- **Occupant:** zero or **one** Process (of any faction).
- **Installed Module:** zero or one.

An Injection Point is initially a faction's spawn location. Players can deploy at any **empty owned spawn** location; a captured spawn must be vacated before deployment. Ordinary locations do not automatically serve as deployment sites. **Only Processes occupy the grid.**

### Capturing and eliminating players — LOCKED

At the **end of the active player's turn**, a location occupied by one of that player's Processes becomes owned by that player. Ownership remains after the Process leaves until another player captures it or the owner is eliminated. A Process being destroyed does **not** by itself neutralize its location.

Capturing another player's Injection Point eliminates that player. Their Operator is turned face down; they cease taking turns, earning resources or scoring, and cannot receive the boss bonus. All their owned territory becomes neutral. All their deployed Processes and attached Updates and their installed Modules leave play; destroyed/on-board Processes go to the **Graveyard**, while Updates and Modules go to **Discard**. A captured player Injection Point becomes a regular spawn.

The central hostile Injection Point **cannot be captured by ordinary location control**. The game ends only when the central hostile enemy is destroyed. The design must separately represent enemy health/state and its central objective; do not treat it as an ordinary capturable player Injection Point.

## 3. Victory and scoring

**LOCKED:** Destroying the hostile central enemy ends the game. Eliminated players cannot win; surviving players are ranked by accumulated score. The large boss-kill contribution bonus goes to the **surviving player who inflicted the most cumulative damage to the central enemy during the match**, not necessarily the player delivering the final blow.

**PROVISIONAL score values:**
- +1 point per owned location at the end of the player's turn.
- +1 point per point of damage inflicted on the central enemy.
- +50 points to the eligible surviving player with the highest lifetime cumulative boss damage.

**OPEN:** tie-break for boss contribution, ranking ties, enemy stats/actions, overkill/mitigation accounting for contribution, and exact victory timing relative to other pending effects.

## 4. Resources — LOCKED

- At the **start of each round**, every surviving player gains **1 resource per location owned**, capped at **5 resources of income per round** (configurable).
- Income is evaluated **once** at round start. A location captured during the round does not produce income until the following round. A subsequent loss does not retroactively subtract income.
- **Unspent resources carry over indefinitely; there is no storage cap.** The value 5 limits *income per round*, not holdings or card costs.
- Starting bank is set by board. Standard default **2**, so with a starting Injection Point providing 1 income, a standard player has **3 resources available during round 1**.
- All non-Operator card types have a resource cost. Operator cards themselves cost 0; activated abilities may have resource costs.

**Design guidance, not a legality restriction:** Process/Update/Module cost 1–2 = basic, 3–4 = moderate, 5+ = powerful. Command costs need not conform to these tiers.

## 5. Card types — LOCKED

| Type | Function | Placement and limits |
| --- | --- | --- |
| **Operator** | Faction/character identity and abilities | Separate from 30-card deck; exactly one per player; no grid occupancy |
| **Process** | Board unit; can move, shoot, melee and use abilities | Only board occupant; one Process per location; one attached Update max |
| **Command** | Instant, reactive effect | Can be played on any player's turn; does not consume a fixed turn action |
| **Update** | Modify a friendly Process | Only during owner's turn, using the single Update action; one per Process, replacement discards old Update |
| **Module** | Modify an owned location | Only during owner's turn, using the single Mod action; one per location, replacement discards old Module |

An existing Module is destroyed/discarded on a change of location ownership. Modules persist when Processes leave. Process destruction sends its attached Update to Discard. See section 10 for destinations.

### Operator abilities — LOCKED

Each Operator has exactly **one passive** and **one activated** printed ability. The passive is active while that Operator remains in play. The activated ability can be used **at most once per round**, can have a resource cost, and **does not consume** any of the four fixed turn actions. An eliminated Operator has neither ability active.

**LOCKED (v2.9):** An Operator's activated ability can only be used **during that Operator owner's turn**, at any legal point without consuming a fixed turn action, and at most once per round. It cannot be activated reactively on an opponent's turn. **Commands** are the instants used to intervene outside one's turn or in response to other effects. Operator passive and triggered effects still function whenever their conditions occur, unless their card text restricts them.

### Process cards — LOCKED

Printed stats: **Cost, Integrity (health), Attack (one damage number), Range, Movement type** (Slow/Normal/Fast). There is **no default Defense stat**. One printed ability per Process (passive, triggered or activated); its one Update may add a second ability, for **two max**.

Damage is **persistent between rounds**, represented by counters against Integrity. Repair/heal cannot exceed printed maximum Integrity unless explicit card text says otherwise.

## 6. Deck, hand and card zones — LOCKED

- Deck: **30 cards**, maximum **3 copies of the same card**. One Operator is outside the deck.
- Starting hand: **4**.
- At the start of each round, each surviving player may **discard any number** of cards in hand, **including Processes**, then draws **up to** 4. A hand already at or above 4 draws nothing; **there is no maximum hand size**.
- When the draw deck empties, shuffle **Discard** to make a new draw deck. The **Graveyard** does not automatically recycle.
- **Discard:** voluntarily discarded hand cards, used Commands, removed/replaced Updates and Modules, and other discarded non-Process attachments.
- **Graveyard:** Processes destroyed while deployed on the board. A Process discarded from hand goes to **Discard**, not Graveyard.
- Future cards may explicitly interact with the Graveyard.

**OPEN:** edge case of needing to draw when both deck and Discard are empty (presumably draw as many as possible), hand availability/mulligan rules, and any deckbuilding restrictions beyond 30/three copies.

## 7. Round and turn sequence — LOCKED

### Setup
1. Each player selects one Operator, initially face down.
2. Randomize turn order **once** at game start; **keep this order throughout the entire game**.
3. Reveal Operators; draw 4 per player; set board-specific starting resources.
4. Begin round 1.

### Start of every round
For each surviving player in turn order:
1. Optionally discard any number of cards in hand to Discard.
2. Draw until hand contains 4 (if already ≥4, draw none).
3. Gain income from owned locations, capped at 5 for the round.

After the round-start procedure, players act in the fixed order.

### Player's turn — fixed action sequence

Each of these actions is **optional**, and each may happen **at most once**, in exactly this order:

1. **Start one Process:** pay its cost and place it at any empty, owned spawn.
2. **Update one Process:** pay and attach one Update to a friendly Process; an existing Update is discarded. Newly deployed Processes are eligible.
3. **Mod one location:** pay and attach one Module to an owned location; replacing its previous Module discards it.
4. **Activate one Process:** select one friendly Process, choosing **Move, Shoot, activated Ability, or Pass**. Other Processes remain idle.

After the actions: capture the locations occupied by the active player's Processes, award end-turn territory points (provisional value), then pass the turn.

### Station phase — LOCKED (v3.0)

**After all surviving player turns**, the station acts **once per round**, in a dedicated **Station phase**. The station is the central enemy faction itself; every hostile Process or other force it deploys belongs to that single station adversary. The station is **not** inserted into the randomized player turn order and does **not** take an ordinary player turn by default. Once its phase completes, the round ends and the next round begins.

**OPEN:** the station's exact number of actions/activations, decision policy, deployed forces, spawning, combat rules and effects. 'Acts once per round' means one **Station phase**, not an agreed limit of one individual enemy unit activation.

**Important:** 'play Updates/Modules during your own turn' does **not** imply unlimited plays or free timing in violation of the ordered actions. Their owner-turn permission and their once-per-turn fixed action slots both apply. Commands can be played between those steps and as responses without altering the fixed order.

**OPEN:** detailed timing of phase boundaries, priority passing, triggered effects, start/end hooks and what happens if a player is eliminated mid-round.

## 8. Process activation, movement and combat — LOCKED

Only **one Process per player per round** is activated via that player's Activate action, regardless of board army size. The same Process may activate in consecutive rounds; there is no requirement to rotate units. Passive and triggered abilities do not require activation unless a card states otherwise.

### Movement types

Each Move action travels at most **one connected orthogonally adjacent location**; Fast does *not* grant extra squares.

| Type | Deployment round | Later rounds |
| --- | --- | --- |
| **Slow** | Cannot Move | May Move only if it **did not Move in the preceding round** |
| **Normal** | Cannot Move | May Move once if selected to activate |
| **Fast** | May Move | May Move once if selected to activate |

All three may Shoot or use their activated ability **in their deployment round**, if chosen to activate. Slow tracks its movement cooldown per Process. Attempting to enter an enemy-occupied cell counts as **moving** for Slow cooldown even if the Process fails to enter.

### Melee

Trying to Move into a connected adjacent **enemy-occupied** cell triggers melee. The attacker and defender deal their printed Attack damage **simultaneously**, without requiring an additional defender activation. If defender is destroyed and attacker survives, attacker enters the cell. Otherwise attacker stays; if both die, the destination becomes empty and its existing ownership is unchanged. Friendly-occupied cells cannot be entered.

### Shooting

Shooting is the Process's one activation action and does not move it. Attack equals the printed Attack stat. A target shoots back **simultaneously** only when the initiating shooter is within the target's own Range; otherwise there is **no retaliation**. Range 0 denotes melee only; Range 1 permits shooting adjacent; Range N up to N connections. Use graph connectivity for range in the baseline; line-of-sight/walls and targeting across altered graphs need further definition.

**OPEN:** target selection constraints, obstruction/line-of-sight, whether base melee retaliation is impacted by other card effects, simultaneous death timing and general damage prevention rules.

## 9. Commands: timing, stack and targeting — LOCKED

- Commands may be played **before, between or after** the four fixed actions, during an opponent's turn, or **in response** to attacks, actions and other Commands, **before damage resolves**.
- Playing a Command does **not** consume the Activate action (or any of the other fixed actions); number played is limited by hand, cost and card-specific restrictions, not a fixed per-turn cap.
- Pay its resource cost when played. Upon resolution, put it in Discard; **canceled Commands are also discarded and their cost is not refunded**.
- Commands have **global board range by default**: any eligible target anywhere on the board regardless of distance or normal graph connections, unless card text imposes restrictions. All normal printed targeting restrictions still apply.
- Players may respond to a pending Command with another Command. Responses form a **last-in, first-out (LIFO)** stack: the most recently played Command resolves first after players pass. A response can cancel/interfere with an earlier pending Command. Example: damage Command → repair response → counter-repair response; counter-repair resolves first.
- Earlier 'active player resolves first when multiple Commands are played simultaneously' is a **priority-to-declare** rule, not a reversal of stack resolution. The active player has the first opportunity to act in simultaneous declaration scenarios; once the stack is built it resolves LIFO.

**OPEN:** complete priority order for non-active players, passing rules and reopened priority after individual resolutions; whether costs/targets are validated again on resolution; timing with simultaneous triggers. Avoid inventing a Magic-style comprehensive stack specification without approval.

## 10. Removal and cleanup — LOCKED

- Destroyed on-board Process → owner's **Graveyard**.
- Its attached Update → owner's **Discard**.
- Resolved or canceled Command → owner's **Discard**.
- Voluntarily discarded card (any type, including Process) → **Discard**.
- Replaced Update or Module → **Discard**.
- Module on a location that changes ownership → **Discard**.
- A player eliminated by Injection Point capture removes all their controlled cards from the board: their Processes to Graveyard, Updates/Modules to Discard; controlled territory neutralized; Operator turned face down.
- Empty locations retain ownership unless explicitly changed.

## 11. Design decisions still open

Do **not** convert these to rules without explicit design agreement:
1. Station behavior: health, number/types of actions during its once-per-round Station phase, spawning/deployed forces, attack logic, damage effects and precise destruction sequence.
2. Boss contribution and scoring tie-breaks; interaction with eliminated contributors and overkill.
3. Exact multiplayer priority/Command stack response windows and trigger ordering.
4. Line of sight and range on boards with removed/added connections.
5. Initial spawn positioning and any board obstacles beyond the five Injection Points.
6. Full Operator/Process/Command/Update/Module card catalog, costs, icons and keyword glossary.
7. Detailed rules for optional activation, card-specific exceptions, duration of effects and damage/heal timing.
8. AI strategy, difficulty, multiplayer networking and UI/UX.

## 12. Guiding examples / tests

- A newly deployed **Fast** Process can be chosen in the fourth action and Move one step; a newly deployed **Normal** or **Slow** Process cannot Move, but either can Shoot/use an activated ability if legal.
- A player holding 9 resources gains only up to 5 more at the start of a round; resources are **not capped at 5**.
- A Process moves onto neutral space but is destroyed before end-turn capture: the location **does not change owner**.
- A Slow Process attacks via attempted entry, fails to displace defender, and remains; its attempted Move still triggers next-round movement cooldown.
- A Process discarded from hand enters Discard, while a deployed Process destroyed in combat enters Graveyard.
- A new Process may be Updated in the second fixed action of the same turn and Activated in the fourth.
- A response Command can stop a pending Command; global targeting is valid unless printed text restricts it.
- If two Processes kill one another on melee entry, the destination remains empty and its prior ownership remains.

---

**Change policy:** When an agreed rule changes, update this document, note the change in the commit/PR, update tests, and do not silently rewrite other LOCKED rules. This document describes the *current* approved design, not a license to fill gaps by assumption.
