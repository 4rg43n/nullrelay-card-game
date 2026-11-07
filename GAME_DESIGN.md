# NullRelay — Game Design

> **Design baseline:** v3.33 · 2026-11-05  
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

- **Immediately before each player's turn**, calculate income as **1 resource per currently owned location**, capped at **5 income per preparation** (configurable). Use the board at that exact moment, after all preceding factions' actions have resolved.
- Income is evaluated **once per faction's own preparation**, not simultaneously at round start. Control gained or lost before that faction's preparation affects its income immediately. After income is set for that turn, later captures/losses do not retroactively alter its current allocation.
- **No ordinary resource banking across turns (LOCKED v3.33):** at the faction's next preparation, discard/expire any remaining ordinary resources, then replace its available resource pool with the newly computed income (plus any explicitly applicable first-turn setup allocation or card effects). There is no carryover or hoarding into another turn. Unspent resources **remain spendable on reactive Commands and triggered/optional effects during opponents' turns** until replaced at the faction's next preparation, so off-turn interaction remains possible. A card/Module may explicitly define special resource storage, distinct from this ordinary pool. Expiry/replacement is a resource reset, not a spending requirement.
- **First-turn setup allocation:** the standard player starts with **2 setup resources** in addition to the **1 income** from their initial Injection Point, for **3 resources on their first turn**. The setup allocation is one-time only; any unused portion is not carried into the next preparation. Thereafter the pool is replaced by that turn's income, subject to explicit card effects.
- All non-Operator card types have a resource cost. Operator cards themselves cost 0; activated abilities may have resource costs.

### Station economy and reclamation — LOCKED mechanic, PROVISIONAL formula (v3.6)

- **Every Open location supports the station economically**, without being represented as an ordinary player-owned color or scoring as player territory. The station uses its own resource bank and pays for cards in the usual way.
- **Station reclamation:** at the **end of each Station Phase**, each location occupied by a surviving station Process changes to **Open** if previously player-controlled. Remove any installed Module on a location whose control changes, sending it to its owner's Discard. This is analogous to end-turn player captures but makes a location Open rather than player-owned. Empty player-controlled locations are not automatically reclaimed.
- **LOCKED (v3.7, timing updated v3.33):** Station begins with **0 setup resources**, then receives normal station income **immediately before its first Station Phase**, without a bonus grant. Like player resources, its unspent ordinary pool expires when its next Station Phase preparation replaces it.
- **Station income has a guaranteed minimum of 2 resources per Station Phase preparation**, preventing complete economic starvation. Compute it **immediately before the Station Phase**, after all player turns and their end-turn captures; the Open-location count therefore reflects this round's changes. The formula is locked in SE01 (§14).
- **LOCKED (SE01):** Station income = **max(2, min(5, ceil(Open location count / 15)))**, based on current Open cells immediately before the Station Phase. This has a floor of 2 and a cap of 5 **per preparation**, with no ordinary carryover to the next station turn. Player captures before the station acts can lower its resources that same round. Station reclamation after preparation affects a later preparation.
- The station's own Injection Point counts as Open for economic purposes unless a future specific board rule says otherwise; player-held Injection Points count for their players. This avoids silently treating the center as a separate fifth ownership type.

**LOCKED (v3.33):** Each player and station performs **their own** pre-turn/phase preparation immediately before acting: optional hand discard, draw to applicable hand-size target, then reset ordinary resources and allocate income based on the **current** board. The station prepares after all player turns. No earlier universal round-start hand/income preparation remains. Ordinary resource storage between preparations is forbidden; a specifically printed storage Module/effect may create an exception.

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

**LOCKED (v3.24) — same-name non-stacking:** Effects from multiple **Updates** or **Modules** with the **same card name** do **not stack on the same affected target**. Duplicate copies cannot multiply a numeric bonus, penalty, counter or repeated continuous benefit on one target merely by existing simultaneously. This is distinct from the existing attachment limits (one Update per Process; one Module per location), which remain unchanged. The 3-copies-per-deck limit remains unchanged: same-named cards may still be included and played legally at different locations/Processes. **LOCKED (v3.25) — overlapping same-named abilities:** If multiple applicable sources grant an ability with the **same name** to one target, apply that named ability's **continuous benefit only once**, not once per source. Example: two `DEFENSE GRID` Modules each provide +1 Integrity to a Process within range 2; if both cover it, the Process receives **+1 total, not +2**. If one source stops applying, the other continues supplying the single benefit while eligible. No copy gains exclusive ownership of the effect; eligibility is recomputed from all active sources. This clarification applies to overlapping same-named ability effects, not only copies of the same physical card. Do **not** invent a global one-copy-per-board restriction.

**LOCKED (v3.26) — same-named triggered abilities:** When multiple eligible sources of a **same-named triggered ability** affect the **same target** in response to the **same event**, that named ability **triggers and resolves only once for that affected target**, rather than once per source. Example: if two `COUNTERMEASURE` Modules cover a Process and both say 'Whenever a friendly Process takes damage, deal 1 damage to the attacker', one damage event to that Process produces **one COUNTERMEASURE retaliation for 1 damage**, not two. The same named ability can still trigger separately for distinct targets or genuinely separate events, subject to the existing once-per-original-event safeguard. Multiple cards may still remain installed; they simply do not multiply their shared ability's effect on one target. This rule applies to same-named abilities, even on differently named source cards.

**LOCKED (v3.27) — strongest value for same-named abilities:** When multiple applicable sources give the same target the **same named ability at different numeric strengths**, use **only the highest applicable value**, rather than adding values. This applies to both continuous benefits and triggered effects. Example: `COUNTERMEASURE 1` and `COUNTERMEASURE 2` covering a single Process yield **one COUNTERMEASURE trigger dealing 2 damage**, not 3, for one qualifying damage event. If the stronger source stops applying, recompute using the strongest remaining eligible source. A numeric rank following an ability's name describes its strength, not a distinct ability name for stacking purposes.

**LOCKED (v3.29) — arithmetic modifiers and stat floor:** When several legally applicable differently named effects modify a numeric stat, add their signed bonuses and penalties to the printed/base value; e.g., printed Attack 3 + `POWER BOOST` 2 − `WEAKEN` 1 = **effective Attack 4**. Unless explicit card text permits an exception, the final effective value of a stat **cannot fall below 0**. Continue respecting same-named non-stacking and highest-strength rules before summing distinct effects. This does not by itself define maximum stat ceilings or treatment of current damage when maximum Integrity changes.

**LOCKED (v3.28) — different names stack:** **Differently named abilities apply cumulatively by default**, even if they affect the same stat or target. For example, `ARMOR 2` and `FORTIFY 1` both increase the same Process's stat if their printed effects say so, for a total benefit of +3. Explicit card text can prohibit or modify such stacking; obey that text. Resolve distinct triggered abilities according to the existing timing and trigger-order rules. The same-name highest-value/non-stacking restrictions remain unchanged.

**OPEN:** same-named abilities with genuinely different nonnumeric wording or incomparable effects; the highest-value rule does not decide which different effect text wins.

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

Damage is **persistent between rounds**, represented by counters against Integrity. Repair/heal removes accumulated damage (minimum 0) and cannot raise remaining Integrity beyond effective maximum Integrity. Maximum-Integrity changes follow S04 in §13.

## 6. Deck, hand and card zones — LOCKED

- Deck: **30 cards**, maximum **3 copies of the same card**. One Operator is outside the deck.
- Starting hand: **4**.
- **Immediately before their own turn**, each surviving player may **discard any number** of cards in hand, **including Processes**, then draws **up to** 4. A hand already at or above 4 draws nothing; **there is no maximum hand size**. The station follows this procedure immediately before its Station Phase with its current escalated target.
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

### Preparation immediately before each faction acts — LOCKED v3.33
In fixed initiative order, each surviving player performs this preparation **just before their own turn**, not at a shared round-start step. The station performs the same preparation **just before its Station Phase**:
1. Optionally discard any number of hand cards to Discard.
2. Draw until the hand reaches its current draw-to target (players: 4; station: escalated 4/5/6/7). If already at or above target, draw none.
3. Expire any unspent ordinary resources from the previous turn/phase, then allocate fresh income using **the current board state** (players: owned cells capped at 5; station: locked Open-cell formula). On a player's first turn add their one-time starting allocation of 2; the station has no starting allocation.

The faction then executes its normal ordered actions. Remaining resources may still pay reactive Commands until its **next** preparation, when they expire. Captures and reclamation before a faction prepares can change that faction's income in the same round.

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

Actions 1–3 are limited to once per Station Phase, not once *per hostile Process*. Their ordering is fixed, and the station cannot skip forward then return to an earlier slot. Commands remain instant/reactive under normal Command rules, not additional Start/Update/Mod actions. **SP04/§14:** station AI chooses the activation order, reevaluating after each activation. Newly Started station Processes are eligible to activate under standard Slow/Normal/Fast deployment-round rules (SP03).

### Station escalation — LOCKED zones and expanding station hand (v3.4)

**LOCKED (v3.4):** The station is a **card-playing adversary**, using its **own deck, hand, resource bank, Processes, Commands, Updates, Modules, Discard and Graveyard** under the same underlying game rules as the player factions, except for explicitly agreed station-specific mechanics. Its deployed force emerges from legal card plays, rather than automatic escalation spawning.

**LOCKED (v3.5) — station activation exception:** **Every deployed hostile Process may activate once during each Station phase.** This is an explicit exception to the player rule limiting a player to one activated Process per turn. Each hostile Process still performs only **one legal activation action** (Move, Shoot, ability or Pass), subject to its movement restrictions, costs and card text. The station uses the same card legality, costs and board rules as players otherwise. A station Process not deployed on the board cannot activate. The exact order in which hostile Processes activate remains an AI/priority implementation detail to define.

**LOCKED (v3.4):** Escalation increases the station's **draw-to hand-size target**, not its automatically deployed Process count. The station begins at **4**, increases to **5** at the first board threshold, **6** at the second, and has an intended later target of **7** at a **not-yet-defined third escalation trigger**. With the currently approved two dotted zones, the standard board supports 4 → 5 → 6; **do not invent a third dotted boundary** for 7.

At its regular **pre-Station-Phase** draw step, the station draws **up to its current target**; existing cards remain in hand, and hand size has no hard maximum. An escalation increases future draw capacity, **not an immediate draw or an automatic Process deployment**. The former suggested fixed hostile force-size tiers **3 → 5 → 6+** are **superseded as station spawning/force requirements**; actual deployment depends on legal cards, resources, locations and station decisions.

**Escalation is irreversible.** Once a tier is triggered, the station never decreases its escalation level because players retreat, change position or are eliminated. Store the highest escalation tier reached in game state; do not derive the current tier solely from current Process positions.

**LOCKED (v3.2) — inward progression, visually marked:** Escalation triggers as player-controlled Processes advance **inward from the player starting edges toward the station core**, not by measuring a remaining-radius value outward from the center. Show **dotted boundary lines on the board** separating escalation zones so players can anticipate when their advance will increase the station's alert tier. Once a zone triggers, its escalation remains active even if the triggering Process retreats or dies. Dotted zone markers are board UI guidance; do not confuse their appearance with traversable board-graph connections.

**LOCKED (v3.3) — precise zones on the standard 9×9 board:** Use zero-based coordinates **(row, column) = 0…8**, center **(4,4)**. The approved diagram defines two nested square boundaries, **not circular radii or graph-distance rings**:

- **Tier 0, initial:** outer region, station draw-to hand-size target **4**.
- **Tier 1:** when **any player-controlled Process reaches** a cell with **row 2…6 and column 2…6** (inclusive), i.e. the central **5×5 square**, escalation rises to Tier 1, station hand-size target **5**. Draw a dotted perimeter around this 5×5 zone.
- **Tier 2:** when any player-controlled Process reaches a cell with **row 3…5 and column 3…5** (inclusive), i.e. the central **3×3 square**, escalation rises to Tier 2, station hand-size target **6**. Draw a second, distinct dotted perimeter around this 3×3 zone.

Thus the boundaries are **2** and **3** squares inward from the closest outer board edge, respectively. The approved 5×5 / 3×3 diagram is authoritative for zone geometry; **a boundary cell counts as inside**. Escalation is global and monotonic: entering the inner zone implies both thresholds have been attained; retreat, death or changes in ownership never lower an attained tier. For nonstandard boards, use explicitly configured equivalent escalation regions rather than hardcoding 9×9 coordinates.

**LOCKED (§14):** Third escalation to draw target **7** triggers at the first successful player damage to the central core; no third dotted boundary. Individual preparation occurs just before each faction's turn or phase (v3.33). Detailed card-driven AI strategies are partly defined in §14; further balance and planning sophistication may still evolve.

**Important:** 'play Updates/Modules during your own turn' does **not** imply unlimited plays or free timing in violation of the ordered actions. Their owner-turn permission and their once-per-turn fixed action slots both apply. Commands can be played between those steps and as responses without altering the fixed order.

**OPEN:** detailed timing of phase boundaries, priority passing, residual same-location/off-board trigger ties, incomparable same-name abilities, start/end hooks and what happens if a player is eliminated mid-round.

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

**LOCKED (v3.19) — simultaneous triggers across factions:** When multiple abilities trigger from the same event, resolve them **automatically**, one at a time, in faction priority order; no player chooses the faction ordering. The **active faction resolves first**. During a player turn, the remaining player factions follow the **fixed initiative order**, wrapping around after the active player, and **the station resolves last**. During the Station Phase, the station is active and resolves its triggers first; player factions then follow their established initiative order. Each trigger completes before the next is processed. This is a triggered-effect ordering rule, **not** a change to the last-in-first-out Command response stack.

**LOCKED (v3.20):** The station always resolves its simultaneous triggered abilities **last during any player's turn**, after all player factions in wrapped initiative order. During the Station Phase, the station resolves **first**, followed by players in fixed initiative order.

**LOCKED (v3.21) — same-faction board order:** When two or more board-associated abilities controlled by the same faction trigger from the same event, resolve them in **board order: top-left to bottom-right**, using ascending row, then ascending column (zero-based on the standard board). For a Process or its attached Update, use that Process's location; for a Module, use its installed location. Apply this ordering *within* the faction's slot in the established cross-faction priority. No player chooses between different board locations.

**LOCKED (v3.22) — same-location and Operator ordering:** Within one faction, for simultaneous triggers associated with the same board location, resolve the **Process's printed trigger first**, then its attached **Update trigger**, then **other location-associated effects** (including an installed Module). Apply this tie-break at each location while continuing top-left-to-bottom-right board ordering. After **all board-associated triggers of that faction** have resolved, resolve that faction's **off-board Operator trigger(s)**. These rules apply inside the existing active-faction/initiative ordering and do not override the Command response stack.

**LOCKED (v3.23) — chained triggers:** If resolving a triggered ability causes one or more further abilities to trigger, resolve the **newly caused trigger(s) immediately**, before returning to the other pending triggers from the original event. Apply the established faction, board, and same-location priority rules when several new triggers arise simultaneously. This applies recursively: a trigger caused by a triggered effect interrupts the remaining pending triggers until that chain completes. Then resume the original pending triggers in their established order. Example: Process A's trigger deals damage that triggers Process B's defensive effect; resolve B's response before processing other triggers from A's original event. This is a trigger-resolution rule and does **not** replace the separate LIFO Command response stack.

**LOCKED (v3.24) — per-event trigger safeguard:** For a given **original event**, the **same triggered ability instance may fire at most once** throughout the full chain of effects caused by that event. Track the original event's identity across recursively generated triggers; an ability that has already fired for that event cannot fire again for it, preventing ping-pong loops between abilities. This restriction is **per original event**, not once per turn/round: the ability may trigger again from a genuinely independent later event. Distinct instances of the same-named ability remain distinct for loop tracking, subject to the separate same-name non-stacking rule for Updates/Modules.

**OPEN:** tie-break if multiple *other* effects occupy the same ordering category/location, ordering of off-board effects other than the Operator, exact interaction with pending Commands/response windows, and precedence for same-named abilities with incomparable effect text.

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
1. Station behavior: refine card catalog and AI evaluation, balance central core's configurable starting Integrity and the now-locked station income formula, and implement the agreed station Process activation order/behavior. Tier-3 escalation (first core damage) and per-faction pre-action income/preparation are LOCKED.
2. Boss contribution and scoring tie-breaks; interaction with eliminated contributors and overkill.
3. Exact multiplayer priority/Command stack response windows, remaining same-location/off-board trigger ties, incomparable same-name effect text, and trigger/Command timing interactions.
4. Line of sight and range on boards with removed/added connections.
5. Additional spawn locations, and any board obstacles beyond the five initial Injection Points.
6. Full Operator/Process/Command/Update/Module card catalog, costs, icons and keyword glossary.
7. Detailed rules for optional activation, card-specific exceptions, duration of effects and damage/heal timing.
8. AI strategy, difficulty, multiplayer networking and UI/UX.

## 12. Guiding examples / tests

- A newly deployed **Fast** Process can be chosen in the fourth action and Move one step; a newly deployed **Normal** or **Slow** Process cannot Move, but either can Shoot/use an activated ability if legal.
- A player does **not** carry 9 ordinary resources into their next turn. At their next preparation, previous unspent ordinary resources expire and are replaced by income from their currently controlled locations (up to 5). A special storage card may explicitly override this.
- A Process moves onto neutral space but is destroyed before end-turn capture: the location **does not change owner**.
- A Slow Process attacks via attempted entry, fails to displace defender, and remains; its attempted Move still triggers next-round movement cooldown.
- A Process discarded from hand enters Discard, while a deployed Process destroyed in combat enters Graveyard.
- A new Process may be Updated in the second fixed action of the same turn and Activated in the fourth.
- A response Command can stop a pending Command; global targeting is valid unless printed text restricts it.
- If two Processes kill one another on melee entry, the destination remains empty and its prior ownership remains.


## 13. Bulk rules review — LOCKED (v3.30, 2026-11-05)

The owner approved **68 rules together in v3.30** and finalized the remaining **S04 in v3.31**; **all 69 cases are now LOCKED**. These cases supplement the earlier locked decisions and should be implemented/tested together. Card text can override defaults only as specified.

### Stats and damage

- **S01 — Apply all signed stat modifiers to the printed/base stat after same-name deduplication.**
- **S02 — Clamp effective stats to a minimum of 0 unless card text explicitly permits a lower value.**
- **S03 — No universal maximum exists for Attack, Range, or Integrity.**
- **S05 — Track persistent damage as accumulated damage rather than mutable current-health totals.**
- **S06 — Healing removes accumulated damage down to a minimum of 0; it does not increase maximum Integrity.**
- **S07 — Excess damage does not spill onto another Process unless specified by an effect.**
- **S08 — Damage reduced to 0 counts as no damage dealt and does not trigger 'takes damage' effects.**
- **S09 — For scoring, damage to a target with limited remaining Integrity only counts up to that actual remaining Integrity (no overkill scoring).**

### Combat and movement

- **C01 — An attack with effective Attack 0 deals 0 damage but still counts as an attack for attack-based triggers.**
- **C02 — Determine attack damage from effective Attack at resolution.**
- **C03 — Retaliation uses the defender's current effective Attack and Range.**
- **C04 — Determine both parties' simultaneous combat damage before destroying either.**
- **C05 — After destruction, ordinary healing cannot save a Process; explicit prevention/resurrection can override.**
- **C06 — A Process cannot attack itself without explicit permission.**
- **C07 — Ranged attack targets must be within graph distance; Processes between attacker and target do not obstruct baseline shots.**
- **C08 — Removed board edges prevent traversal and do not count as adjacency; range follows the current connection graph.**
- **C09 — An invalid movement declaration does not use an action; a legal melee entry attempt uses the Move action even if displacement fails.**
- **C10 — Entering a friendly-occupied location is illegal; friendly position swaps require explicit card text.**
- **C11 — Forced movement ignores Slow/Normal/Fast movement cooldowns but still requires a legal destination unless overridden.**
- **C12 — Forced or effect-driven movement can cause entry/location triggers but never grants another activation by itself.**

### Abilities and event timing

- **A01 — Give every event a stable identity so chained triggers can obey the once-per-original-event safeguard.**
- **A02 — A triggered ability that destroys its own source during resolution still completes unless its text requires the source to remain in play.**
- **A03 — Trigger sources must be active when their event occurs, except when explicitly active from another zone.**
- **A04 — Recalculate continuous effects when sources, ownership, placement, attachments, or relevant stats change.**
- **A05 — For tied simultaneous triggers in the same priority category and board location, use a stable card ID to order them.**
- **A06 — Resolve as much of an effect as remains possible unless explicitly all-or-nothing.**
- **A07 — An ability with an illegal mandatory target cannot be declared; if a target later becomes illegal, its targeted portion fails.**
- **A08 — Optional triggers can be declined; they still count as having triggered for that original event.**
- **A09 — Same-named abilities with incompatible nonnumeric text require distinct IDs or explicit precedence, not silent combination.**
- **A10 — Discard a destroyed Process's attached Update as part of destruction; apply subsequent triggers through normal event timing.**

### Commands and responses

- **R01 — Respond to a Command with Commands until all eligible factions pass consecutively.**
- **R02 — After the top Command resolves, reopen responses before resolving the next pending Command.**
- **R03 — Response priority: active faction first, then wrapped player initiative; station last on player turns and first in Station Phase.**
- **R04 — Pay Command costs on declaration; failed/canceled effects do not refund costs.**
- **R05 — Choose Command targets when played, then recheck legality at resolution.**
- **R06 — Commands cannot undo completed events retroactively; prevention must resolve before the event.**
- **R07 — Stat changes affect unresolved attacks, not damage already applied.**
- **R08 — Triggered chains produced during Command resolution finish before moving to the next pending Command.**

### Cards and attachments

- **D01 — Card names and ability names are separate identifiers; different cards can grant the same named ability.**
- **D02 — A disabled Update still occupies the Process's one Update slot.**
- **D03 — On replacement, discard the previous Update/Module before the new card becomes active.**
- **D04 — On Process controller change, its Update stays attached and changes controller with it unless card text says otherwise.**
- **D05 — A Module functions only on a valid location; remove it if its location becomes ineligible.**
- **D06 — Attachment effects stop immediately when that attachment leaves play.**
- **D07 — Explicit extra Start/Update/Mod/Activate effects override ordinary action counts only to their printed extent.**
- **D08 — Overlapping same-named abilities use the highest currently eligible strength; fall back to the next strongest when necessary.**
- **D09 — Same printed card name with different artwork is the same card for deckbuilding and stacking.**
- **D10 — Card text overrides only directly contradictory general rules, leaving all other general rules intact.**

### Turns, resources and board

- **T01 — Optional action slots may be skipped, but earlier slots cannot be revisited after progressing.**
- **T02 — A faction without legal actions can end its turn; normal end-of-turn triggers/scoring still happen.**
- **T03 — Pay costs fully before declaring legal actions; resource totals cannot become negative.**
- **T04 — Superseded by v3.33:** income uses the board snapshot immediately before **each faction's own turn/Station Phase**, not at universal round start. Once that preparation sets the faction's resources, later changes do not retroactively change that preparation.
- **T05 — A player eliminated before their turn skips that turn and gains no additional income.**
- **T06 — Apply ownership changes before recalculating Module legality and continuous effects.**
- **T07 — A location cannot have two owners; the most recent legal capture controls it.**
- **T08 — A Process removed before owner end-turn capture cannot capture a location.**
- **T09 — On elimination, deactivate Operator and remove deployed cards immediately, even mid-opponent-turn.**
- **T10 — Elimination never reverses completed effects or already awarded points.**

### Station, escalation and completion

- **E01 — Check escalation immediately when a player Process enters a qualifying cell, including forced movement.**
- **E02 (timing updated v3.33) — Escalation changes the station's draw-to target at its next **pre-Station-Phase preparation**, not its present hand immediately.**
- **E03 — The station chooses activation order for deployed station Processes; each activates at most once per Station Phase.**
- **E04 — Station uses standard card legality and costs except where its cards explicitly override.**
- **E05 — When unable to pay for legal actions, station can pass instead of gaining free substitute actions.**
- **E06 — Station reclaims only cells occupied by surviving station Processes at end of Station Phase.**
- **E07 — Once the central enemy is defeated, finish the damage event and immediate trigger chain, then end before further normal actions.**
- **E08 — Final score counts earned points and explicitly defined endgame bonuses; spare cards/resources confer no points automatically.**
- **E09 — Tie surviving players in final score at equal rank unless specific card/scenario tie-break applies.**
- **E10 — Being the sole remaining player does not end the game; defeating the station is still required.**

### S04 — Maximum Integrity bonuses and lethal reductions — LOCKED (v3.31)

- **A bonus increasing maximum Integrity by N also removes N accumulated damage**, with damage floored at **0**. This damage reduction is part of the maximum-Integrity increase effect, not an additional action. Example: a Process at maximum Integrity 5 with 2 accumulated damage gains +2 maximum Integrity; its new maximum is **7** and its damage drops to **0** (remaining Integrity 7). A bonus never creates negative damage or heals above the new maximum.
- **A decrease in maximum Integrity does not itself add damage.** Once the entire Integrity-changing effect finishes resolving, compare the Process's accumulated damage with its **resulting effective maximum Integrity**. If accumulated damage is **greater than or equal to** the resulting maximum, destroy the Process. Do not destroy it midway through a multi-step effect before its final state is established. Example: maximum Integrity falls from 5 to 2 while accumulated damage is 2 → the Process dies after the effect resolves.
- **Distinct from proportional scaling:** 'same ratio' here is implemented as the owner's explicit **one-for-one reduction of accumulated damage on positive maximum-Integrity bonuses**, not as preserving an exact mathematical health percentage. This is the canonical numeric rule.
- If an ongoing maximum-Integrity bonus is later removed, the maximum falls accordingly, but **do not restore damage previously removed when the bonus was gained**; check for death only against the resulting maximum after the removal effect. Explicit card effects may override this default.


## 14. Station-focused bulk review — LOCKED (v3.32, 2026-11-07)

The owner approved **48 of 52** station-specific proposals in v3.32 and resolved the four flagged items in **v3.33**. All 52 station cases now have a recorded decision (some originally approved defaults are superseded where the v3.33 no-hoarding economy requires it). These defaults supplement earlier LOCKED rules. Balance values expressly called configurable (e.g. core Integrity 30) remain adjustable without silently changing core mechanics. Where this section resolves an earlier OPEN topic, it is authoritative. **SE01** remains the locked formula; SE02/SE03 now establish each faction's own just-before-action preparation and current-board income. **SE06** rejects ordinary resource carryover for **all** factions. **SP03** applies the existing Slow/Normal/Fast deployment-round action rules to station Processes.

### Station economy and resources

- **SE01:** Station income is `max(2, min(5, ceil(Open locations / 15)))`, with a 2-resource minimum and a 5-resource income cap per preparation. Ordinary resource banking is superseded by SE06/v3.33.
- **SE04:** Factions prepare in their established acting order: each player immediately before their turn, and station immediately before Station Phase. SE03/v3.33 defines their preparation steps.
- **SE05 (partly superseded by SE06/v3.33):** Station cannot spend resources it does not have; it **cannot accumulate unspent ordinary resources between phases**. Only explicitly printed storage effects can retain resources.
- **SE07:** When several legal purchases compete for a limited budget, station AI evaluates their combined tactical value, not simply their individual printed costs.
- **SE08 (timing updated v3.33):** Ownership changes affect the **next preparation of the affected faction**, potentially later in the same round; never retroactively change resources already allocated in a completed preparation.

### Station spawning and Process activation

- **SP01:** The station starts without deployed Processes and must legally Start its first from hand with resources earned in round 1.
- **SP02:** The central core's station spawn remains legal despite nearby enemy Processes when the spawn cell is empty and Open.
- **SP04:** Station AI may choose the order in which its deployed Processes activate; there is no fixed board-cell activation order.
- **SP05:** After each station Process activation, AI reevaluates tactical state before selecting the next Process.
- **SP06:** Each station Process receives no more than one ordinary activation in a Station Phase, no matter how many are deployed.
- **SP07:** When all legal station spawn cells are occupied, skip Start rather than automatically displacing or deleting a Process.
- **SP08:** With multiple legal spawns, station AI chooses based on defense, nearby threats, and strategic territory.

### Station AI and tactical behavior

- **AI01:** Station AI uses the same authoritative action legality/combat rules as players and gets no secret exceptions.
- **AI02:** Station Processes evaluate legal Move, Shoot, Ability, and Pass options; illegal actions cannot be chosen.
- **AI03:** Prefer a guaranteed enemy Process kill to equal-value nonlethal damage unless that would substantially increase danger to the core.
- **AI04:** Defending the central core generally has higher value than reclaiming distant territory.
- **AI05:** Prefer reclaiming an economically valuable player location over an otherwise equivalent low-value location.
- **AI06:** A station Process may hold position to guard the core rather than moving outward.
- **AI07:** Target evaluation considers remaining Integrity, threat to core, attack range, retaliation, and expected follow-up attacks.
- **AI08:** Station may play defensive/reactive Commands on player turns when tactically beneficial.
- **AI09:** Station AI cannot inspect hidden player hand or deck identities/order; it uses public information and previously revealed events only.
- **AI10:** Station may plan coordinated multi-Process activations and reevaluate after unexpected Commands/events.
- **AI11:** Configurable AI difficulty changes planning depth/evaluation quality, not printed card stats, costs, resources, or action allowances.

### Escalation

- **ES01:** Existing inclusive 5×5 and 3×3 dotted square escalation zones are unchanged.
- **ES02:** Reaching a deeper zone may advance multiple previously untriggered tiers simultaneously.
- **ES03 (timing updated v3.33):** Tier 1/2 change station draw-to-hand target to 5/6 at its next **pre-Station-Phase draw**.
- **ES04:** Tier 3 triggers when a player first successfully damages the central core; station draw-to-hand target becomes 7.
- **ES05:** Tier 3 uses the first-core-damage event, not a third dotted board boundary.
- **ES06:** Escalation never decreases from retreat, healing, loss of territory, or Process destruction.
- **ES07:** Escalation never gives an immediate free draw, Process deployment, or resources.
- **ES08:** Forced entry by a player Process into an escalation zone also triggers it.
- **ES09:** Escalation announcements publicly reveal the current tier and its station draw target.

### Central core and combat

- **SC01:** The central core is an immobile objective tied to the central Injection Point, not a normal station Process card in the station's deck.
- **SC02:** Starting core maximum Integrity defaults to 30 for the first prototype and is configurable.
- **SC03:** The core takes persistent damage and dies when accumulated damage reaches its effective maximum Integrity.
- **SC04:** The core has no default Attack, movement, or automatic retaliation; defense comes from station Processes, Modules, Commands, or explicit core abilities.
- **SC05:** Attack the core through legal attacks targeting its center location, using effective Range and current board graph.
- **SC06:** A station Process occupying the center does not inherently make the core untargetable or invulnerable.
- **SC07:** Core and station Process at the center are distinct selectable targets; attacks only affect the chosen one absent area text.
- **SC08:** Only cards expressly capable of targeting the core can heal/repair it; Process-only healing cannot.
- **SC09:** When the core dies, finish the present damage event and immediate trigger chain before ending play, consistent with existing E07.

### Core scoring and endgame

- **SV01:** Initial scoring defaults: +1 point per owned location at player-turn end, +1 per actual point of core damage, and +50 to the surviving player with highest cumulative core-damage contribution.
- **SV02:** Boss contribution counts actual core damage after prevention/mitigation, never overkill.
- **SV03:** Damage to station Processes does not count toward central-core contribution bonus without explicit card text.
- **SV04:** All surviving players tied for the highest cumulative core damage receive the full +50 bonus.
- **SV05:** Players eliminated before core destruction are ineligible for the +50 bonus regardless of earlier contribution.
- **SV06:** Rank survivors by final accumulated score, with equal ranks on score ties.
- **SV07:** After the completed core-destruction event, no further player turns or Station Phase occur.

### Formerly flagged cases — LOCKED (v3.33)

- **SE02:** Station income reads the board **immediately before Station Phase**, not at universal round start. Each player likewise computes income immediately before their own turn; territorial disruption before preparation matters immediately.
- **SE03:** Each faction performs its own preparation **immediately before acting**: optionally discard hand cards, draw up to its target, then replace ordinary resources with income from the current board (plus first-turn setup allocation where applicable). Players follow initiative and station prepares after player turns.
- **SE06:** **No faction, including the station, may hoard ordinary resources between turns/phases.** Unspent ordinary resources remain available for off-turn Commands until the faction's next preparation, when they expire and are replaced. Future Modules or explicit card effects may introduce special location-based storage. The station may pass/decline buys, but cannot bank the remainder for a future Station Phase.
- **SP03:** A station Process Started in the current Station Phase **may use one ordinary activation during that same phase**, exactly like a newly deployed player Process: **Fast** may Move (one edge); **Normal** and **Slow** may not Move that deployment round; all three may Shoot, use an eligible activated Ability, or Pass. Slow's later-round movement cooldown applies normally. This is not bonus activation or automatic free movement.

---

**Change policy:** When an agreed rule changes, update this document, note the change in the commit/PR, update tests, and do not silently rewrite other LOCKED rules. This document describes the *current* approved design, not a license to fill gaps by assumption.
