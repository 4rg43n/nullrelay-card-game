# NullRelay — Game Design

> **Design baseline:** v3.18 · 2026-11-02  
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
- **Ownership:** **Open** (the station's economic territory) or controlled by one of up to four players. Open is not a fifth player ownership color.
- **Occupant:** zero or **one** Process (of any faction).
- **Installed Module:** zero or one.

An Injection Point is initially a faction's spawn location. Players can deploy at any **empty owned spawn** location; a captured spawn must be vacated before deployment. Ordinary locations do not automatically serve as deployment sites. **Only Processes occupy the grid.**

**Station spawning — LOCKED (v3.9):** The station's **central Injection Point (4,4)** is its **initial spawn location** on the standard 9×9 board. During its once-per-phase Start action, it may deploy **one** paid Process from hand to that location if empty, or to another **empty spawn location available to the station**. Other explicitly marked spawn locations may become available through station reclamation or card/board effects. **Open does not mean spawn:** ordinary Open corridors are station economic territory but are never automatically legal deployment sites. Open spawn locations are available to the station; player-controlled spawns are not, unless a card explicitly overrides this rule. No free Process appears simply because the station escalates.

**Reclaimed spawn locations — LOCKED (v3.10):** When a station Process occupies a player-controlled **spawn** location at the end of the Station Phase, station reclamation changes its control to **Open** without changing its location kind. That spawn becomes eligible for future station Starts **automatically**, provided it is **empty at deployment time**. If the reclaiming station Process still occupies the location, it cannot spawn another Process there until that Process leaves or is removed. The station may use the reclaimed spawn on a later Station Phase's Start action, not retroactively in the Start slot already completed for the phase in which reclamation occurred. This does not turn normal Open locations into spawn locations.

### Capturing and eliminating players — LOCKED

At the **end of the active player's turn**, a location occupied by one of that player's Processes becomes owned by that player. Ownership remains after the Process leaves until another player captures it or the owner is eliminated. A Process being destroyed does **not** by itself change the location to Open.

Capturing another player's Injection Point eliminates that player. Their Operator is turned face down; they cease taking turns, earning resources or scoring, and cannot receive the boss bonus. All their owned territory becomes Open (and therefore contributes to station economy from the next income calculation). All their deployed Processes and attached Updates and their installed Modules leave play; destroyed/on-board Processes go to the **Graveyard**, while Updates and Modules go to **Discard**. A captured player Injection Point becomes a regular spawn.

The central hostile Injection Point **cannot be captured by ordinary location control**. The game ends only when the central hostile enemy is destroyed. The design must separately represent enemy health/state and its central objective; do not treat it as an ordinary capturable player Injection Point.

## 3. Victory and scoring

**LOCKED:** Destroying the hostile central enemy ends the game. Eliminated players cannot win; surviving players are ranked by accumulated score. The large boss-kill contribution bonus goes to the **surviving player who inflicted the most cumulative damage to the central enemy during the match**, not necessarily the player delivering the final blow.

**PROVISIONAL score values:**
- +1 point per owned location at the end of the player's turn.
- +1 point per point of damage inflicted on the central enemy.
- +50 points to the eligible surviving player with the highest lifetime cumulative boss damage.

**OPEN:** tie-break for boss contribution, ranking ties, enemy stats/actions, overkill/mitigation accounting for contribution, and exact victory timing relative to other pending effects.

## 4. Resources — LOCKED player economy; station economy v3.6

- At the **start of each round**, every surviving player gains **1 resource per location owned**, capped at **5 resources of income per round** (configurable).
- Income is evaluated **once** at round start. A location captured during the round does not produce income until the following round. A subsequent loss does not retroactively subtract income.
- **Unspent resources carry over indefinitely; there is no storage cap.** The value 5 limits *income per round*, not holdings or card costs.
- Starting bank is set by board. Standard default **2**, so with a starting Injection Point providing 1 income, a standard player has **3 resources available during round 1**.
- All non-Operator card types have a resource cost. Operator cards themselves cost 0; activated abilities may have resource costs.

### Station economy and reclamation — LOCKED mechanic, PROVISIONAL formula (v3.6)

- **Every Open location supports the station economically**, without being represented as an ordinary player-owned color or scoring as player territory. The station uses its own resource bank and pays for cards in the usual way.
- **Station reclamation:** at the **end of each Station Phase**, each location occupied by a surviving station Process changes to **Open** if previously player-controlled. Remove any installed Module on a location whose control changes, sending it to its owner's Discard. This is analogous to end-turn player captures but makes a location Open rather than player-owned. Empty player-controlled locations are not automatically reclaimed.
- **LOCKED (v3.7): Station starts with 0 banked resources at setup**, then collects its normal station income during the **first round's income step** before its first Station Phase. Do not treat 0 starting resources as 0 income, or give it a separate opening bonus.
- **Station income has a guaranteed minimum of 2 resources per round**, preventing complete economic starvation even if players control almost the whole map. Income is computed at the normal round-start income step from the number of Open locations; the exact scaling is for balancing.
- **PROVISIONAL starting formula:** station income = **max(2, min(5, ceil(Open location count / 15)))**. This has a floor of 2, a cap of 5 income per round, and no cap on the station's accumulated bank. It is an initial balancing suggestion, **not a locked formula**. Under the provisional formula, player territory gains reduce station income, while eliminating a player or station reclamation may increase future station income.
- The station's own Injection Point counts as Open for economic purposes unless a future specific board rule says otherwise; player-held Injection Points count for their players. This avoids silently treating the center as a separate fifth ownership type.

**OPEN:** whether the provisional station-income formula and its income cap should become permanent, and precise timing of its round-start discard/draw/income relative to players. The two-resource minimum is locked; exact gains beyond that are not.

**Design guidance, not a legality restriction:** Process/Update/Module cost 1–2 = basic, 3–4 = moderate, 5+ = powerful. Command costs need not conform to these tiers.

## 5. Card types — LOCKED

| Type | Function | Placement and limits |
| --- | --- | --- |
| **Operator** | Faction/character identity and abilities | Separate from 30-card deck; exactly one per player; no grid occupancy |
| **Process** | Board unit; can move, shoot, melee and use abilities | Only board occupant; one Process per location; one attached Update max |
| **Command** | Instant, reactive effect | Can be played on any player's turn; does not consume a fixed turn action |
| **Update** | Modify a friendly Process | Only during owner's turn, using the single Update action; one per Process, replacement discards old Update |
| **Module** | Modify an owned location | Only during owner's turn, using the single Mod action; one per location, replacement discards old Module |

**Station Updates, Modules and Commands — LOCKED (v3.12):** The station uses **exactly the same rules for Updates, Modules and Commands as player factions**. It may attach one Update to one of its own Processes using its one paid Update slot per Station Phase, subject to the same replacement, attachment and card-text rules. It may install one Module using its one paid Mod slot; Open locations count as the station's controlled locations. Its Commands are playable at the same times as player Commands—including during player turns, between actions, in response to attacks and other Commands—and follow identical costs, legal targets, global range defaults, LIFO stack/priority rules and Discard handling. There are **no station-specific bonuses or additional plays** for any of these three card types. The station's special activation rule does not grant additional Update or Mod slots.

**Station Modules — LOCKED (v3.11):** The station treats **Open** locations as its controlled locations for Module placement. It may install a Module at **any Open location** during its one Mod action each Station Phase, without needing a station Process to occupy the space. It cannot install on player-controlled locations unless explicit card text overrides the rule. When a player captures an Open location, the station's installed Module goes to its Discard, exactly as an opponent's Module would on any control change.

An existing Module is destroyed/discarded on a change of location ownership. Modules persist when Processes leave. Process destruction sends its attached Update to Discard. See section 10 for destinations.

### Operator abilities — LOCKED

Each Operator has exactly **one passive** and **one activated** printed ability. The passive is active while that Operator remains in play. The activated ability can be used **at most once per round**, can have a resource cost, and **does not consume** any of the four fixed turn actions. An eliminated Operator has neither ability active.

**LOCKED (v2.9):** An Operator's activated ability can only be used **during that Operator owner's turn**, at any legal point without consuming a fixed turn action, and at most once per round. It cannot be activated reactively on an opponent's turn. **Commands** are the instants used to intervene outside one's turn or in response to other effects. Operator passive and triggered effects still function whenever their conditions occur, unless their card text restricts them.

### Station Operator subtype — LOCKED (v3.13)

The station has **one Operator of its own**, separate from its deck, like each player. **Station Operator** is a distinct **Operator card subtype / eligibility keyword** reserved for the station; player-controlled factions (human or AI) use regular Player Operators and cannot select Station Operators. Design Station Operator abilities **specifically around station mechanics**—such as escalation, Open locations, reclamation, the station economy or Station Phase—rather than assuming a player Operator's abilities are interchangeable.

A Station Operator **remains an Operator** and by default has exactly **one passive and one activated ability**, the same zero card cost and once-per-round activation limit. Its activated ability may be used only during the **Station Phase** (the station's own turn) and does not consume Start, Update, Mod or Process-activation slots. Passives follow the normal passive rules. Being a Station Operator does **not** itself grant additional actions or new timing permissions; any exceptional abilities must be explicitly printed and agreed.

**First official Station Operator — LOCKED (v3.15): `REMOVE/REPLACE`.** This is the approved baseline Station Operator, unavailable to player factions.

- **Passive — REPLACE:** Whenever the station **reclaims a player-controlled location containing a player Module** at the end of the Station Phase, remove that Module to its owner's Discard as normal. The station **may** then install a Module from its own hand on that now-Open location, **paying the Module's normal resource cost**. The Module must be legal for that location. This is a **card-effect exception** to the normal one-Mod action limit: a triggered REPLACE installation **does not spend or require the Station Phase's Mod action**, and can occur even when that action was already used. **The passive triggers immediately when its condition is met** during the end-of-Station-Phase reclamation step, and resolves before that phase finishes. Each qualifying reclaimed location is evaluated separately, with a cost paid for each Module actually installed. It is optional and never creates a free card. **Card text overrides the default action timing:** the station may use its regular Mod action **and** install a Module through REPLACE in the same Station Phase (or more than one triggered replacement if multiple qualifying locations are reclaimed). These triggered installations do not grant extra ordinary Mod actions or allow arbitrary Module placement outside their triggering locations.
- **Activated — REMOVE:** **Once per round during the Station Phase**, choose an **enemy Module installed on a location adjacent to at least one deployed station Process** and pay **resources equal to that Module card's printed cost** to destroy it, moving it to its owner's Discard. The variable cost is intentional: removing a cost-2 Module costs 2; removing a cost-5 Module costs 5. The station must have enough resources, and this Operator ability does **not** consume Start, Update, Mod, or Process activation slots. Enemy Modules include player-installed Modules, not the station's own Modules. **Adjacent** uses the board's normal connected orthogonal neighbors (not diagonals); the station Process must be present when REMOVE is activated. REMOVE cannot target Modules anywhere else on the board.

**LOCKED (v3.16):** REMOVE targets only a Module in a location **adjacent to at least one deployed station Process**. A Module on a station Process's own location is not adjacent. Targeting requires a connected orthogonal edge; diagonals and remote Modules are not eligible. This spatial restriction overrides any assumption that Operator abilities have the global range used by Commands.

**Narrative design direction:** Keep **NOXENGINE** and **VOIDSYNC** enigmatic and morally ambiguous; do not make either a standard Station Operator or canonically establish either as the villain. Prefer terminal/process-style names that imply mechanical identity. `KILL/ERASE`, `SCAN/TRACE` and `LOCK/DENY` are still **candidates**, not approved cards or final names.

**Implementation note:** represent the subtype/eligibility as card data (e.g. `operatorKind: "station"` versus `"player"`), not a mandatory separate software class. Share common Operator state, validation and ability handling; station-only effect implementations may be distinct. **OPEN:** whether future scenario variants should have additional subtype-specific constraints.

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

**After all surviving player turns**, the station acts **once per round**, in a dedicated **Station phase**. The station is the central enemy faction itself; every hostile Process or other force it deploys belongs to that single station adversary. The station is **not** inserted into the randomized player turn order. Once its phase completes, the round ends and the next round begins.

**LOCKED (v3.8) — station action limits and order:** The station follows the same ordered action slots as a player, with only the final activation slot expanded:

1. **Start one Process** — optional, at most one legal, paid deployment at an empty eligible station spawn.
2. **Update one Process** — optional, at most one legal, paid Update to its Process.
3. **Mod one location** — optional, at most one legal, paid Module on **any Open location** (LOCKED v3.11), whether or not a station Process occupies it. As for players, this uses the single Mod slot for the phase, costs resources, replaces an existing Module if present, and must obey card-specific restrictions. The location must be Open at the time the Module is installed.
4. **Activate deployed Processes** — each deployed hostile Process may take **one legal activation**, rather than selecting a single Process as a player does. No Process activates more than once in this phase by default.

Actions 1–3 are limited to once per Station Phase, not once *per hostile Process*. Their ordering is fixed, and the station cannot skip forward then return to an earlier slot. Commands remain instant/reactive under normal Command rules, not additional Start/Update/Mod actions. Exact ordering among multiple hostile Process activations remains open.

### Station escalation — LOCKED zones and expanding station hand (v3.4)

**LOCKED (v3.4):** The station is a **card-playing adversary**, using its **own deck, hand, resource bank, Processes, Commands, Updates, Modules, Discard and Graveyard** under the same underlying game rules as the player factions, except for explicitly agreed station-specific mechanics. Its deployed force emerges from legal card plays, rather than automatic escalation spawning.

**LOCKED (v3.5) — station activation exception:** **Every deployed hostile Process may activate once during each Station phase.** This is an explicit exception to the player rule limiting a player to one activated Process per turn. Each hostile Process still performs only **one legal activation action** (Move, Shoot, ability or Pass), subject to its movement restrictions, costs and card text. The station uses the same card legality, costs and board rules as players otherwise. A station Process not deployed on the board cannot activate. The exact order in which hostile Processes activate remains an AI/priority implementation detail to define.

**LOCKED (v3.4):** Escalation increases the station's **draw-to hand-size target**, not its automatically deployed Process count. The station begins at **4**, increases to **5** at the first board threshold, **6** at the second, and has an intended later target of **7** at a **not-yet-defined third escalation trigger**. With the currently approved two dotted zones, the standard board supports 4 → 5 → 6; **do not invent a third dotted boundary** for 7.

At its regular round-start draw step, the station draws **up to its current target**; existing cards remain in hand, and hand size has no hard maximum. An escalation increases future draw capacity, **not an immediate draw or an automatic Process deployment**. The former suggested fixed hostile force-size tiers **3 → 5 → 6+** are **superseded as station spawning/force requirements**; actual deployment depends on legal cards, resources, locations and station decisions.

**Escalation is irreversible.** Once a tier is triggered, the station never decreases its escalation level because players retreat, change position or are eliminated. Store the highest escalation tier reached in game state; do not derive the current tier solely from current Process positions.

**LOCKED (v3.2) — inward progression, visually marked:** Escalation triggers as player-controlled Processes advance **inward from the player starting edges toward the station core**, not by measuring a remaining-radius value outward from the center. Show **dotted boundary lines on the board** separating escalation zones so players can anticipate when their advance will increase the station's alert tier. Once a zone triggers, its escalation remains active even if the triggering Process retreats or dies. Dotted zone markers are board UI guidance; do not confuse their appearance with traversable board-graph connections.

**LOCKED (v3.3) — precise zones on the standard 9×9 board:** Use zero-based coordinates **(row, column) = 0…8**, center **(4,4)**. The approved diagram defines two nested square boundaries, **not circular radii or graph-distance rings**:

- **Tier 0, initial:** outer region, station draw-to hand-size target **4**.
- **Tier 1:** when **any player-controlled Process reaches** a cell with **row 2…6 and column 2…6** (inclusive), i.e. the central **5×5 square**, escalation rises to Tier 1, station hand-size target **5**. Draw a dotted perimeter around this 5×5 zone.
- **Tier 2:** when any player-controlled Process reaches a cell with **row 3…5 and column 3…5** (inclusive), i.e. the central **3×3 square**, escalation rises to Tier 2, station hand-size target **6**. Draw a second, distinct dotted perimeter around this 3×3 zone.

Thus the boundaries are **2** and **3** squares inward from the closest outer board edge, respectively. The approved 5×5 / 3×3 diagram is authoritative for zone geometry; **a boundary cell counts as inside**. Escalation is global and monotonic: entering the inner zone implies both thresholds have been attained; retreat, death or changes in ownership never lower an attained tier. For nonstandard boards, use explicitly configured equivalent escalation regions rather than hardcoding 9×9 coordinates.

**OPEN:** location/timing of a third escalation to hand-size **7**; station round-start discard/draw/income timing relative to the player round-start procedure; detailed card-driven deployment and AI behavior; when in the movement/Command resolution sequence to evaluate zone entry; and the detailed sequencing of the individual hostile Process activations.

**Important:** 'play Updates/Modules during your own turn' does **not** imply unlimited plays or free timing in violation of the ordered actions. Their owner-turn permission and their once-per-turn fixed action slots both apply. Commands can be played between those steps and as responses without altering the fixed order.

**OPEN:** detailed timing of phase boundaries, priority passing, ordering of simultaneous or chained triggered effects, start/end hooks and what happens if a player is eliminated mid-round.

## 8. Process activation, movement and combat — LOCKED

Only **one Process per player per round** is activated via that player's Activate action, regardless of board army size. **Exception:** during the Station phase, **each deployed hostile Process may activate once** (v3.5). The same Process may activate in consecutive rounds; there is no requirement to rotate units. Passive and triggered abilities do not require activation unless a card states otherwise.

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

## 9. Triggered abilities and Commands

### Triggered ability timing — LOCKED (v3.18)

**Triggered abilities from any card type (Operators, Processes, Updates, Modules, and Commands with triggered text) fire and resolve immediately when their stated condition occurs**, unless that card explicitly specifies a different timing. Triggered abilities are not ordinary activated actions and do not consume Start, Update, Mod or Process-activation slots merely because their effect resembles one of those actions. **Explicit card text overrides the default action restrictions and timing** only to the extent stated: for example, REPLACE may install a paid Module when reclamation triggers it even after the station's normal Mod action has already been used. It does not grant an additional unrestricted Mod action. Apply any costs, eligibility requirements and limits in the triggered effect's text.

**OPEN:** exact ordering if multiple triggers occur from the same event, how simultaneous triggers interact with pending Commands and response windows, and resolution of chained triggers. The default of immediate resolution does not pre-decide these conflict-resolution details.

### Commands: timing, stack and targeting — LOCKED

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
1. Station behavior: health, Process AI, card/resource-driven deployment, third hand-size-7 escalation trigger, exact round-start timing, order and sequencing of station Process activations, escalation trigger timing, attack logic, damage effects and precise destruction sequence; final income scaling beyond its locked 2-resource floor.
2. Boss contribution and scoring tie-breaks; interaction with eliminated contributors and overkill.
3. Exact multiplayer priority/Command stack response windows and trigger ordering.
4. Line of sight and range on boards with removed/added connections.
5. Additional spawn locations, and any board obstacles beyond the five initial Injection Points.
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
