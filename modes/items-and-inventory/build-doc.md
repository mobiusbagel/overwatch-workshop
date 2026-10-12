# Items & Inventory — Build Doc

Overwatch Workshop custom mode. Health-pack gameplay replaced with an item/power-up
system: spawners dispense held items, players use them on demand, deaths drop them
as contestable loot.

## Locked design (2026-10-06)

- **Health packs**: removed via lobby setting `Spawn health packs → Disabled`.
- **Spawners**: positions baked once per map with the Placer tool (below), hardcoded
  as vectors in the init rule.
- **Inventory**: 1 slot. Pick up only with empty slot (no swapping). Use on demand
  via Interact press.
- **Death drop**: held item drops at death position. Teammates-only pickup for 5s
  (team-colored visuals), then free-for-all until 25s (neutral visuals), then it
  expires with a poof.
- **Item pool** (6): Damage Surge, Swift Boots, Overshield, Second Wind, Ghost, Reset.
- **Distribution**: host picks **Fixed** or **Random** in the Workshop Settings menu.
  Fixed rolls each spawner's item once at match start and keeps it all match;
  Random re-rolls each spawner's item on *every respawn*.
- **Perks**: disabled via the per-mode lobby setting (2026-10-06).

## Lobby setup

- Base mode: Team Deathmatch (script is mode-agnostic; TDM is the sane default).
- Settings → General: `Spawn health packs` → **Disabled**.
- Perks: **disabled** via the per-mode lobby setting (Workshop has no perk actions,
  so this is handled at the lobby level, not in script).
- Workshop Settings tab: add dropdown **`Item Distribution`** with options
  `Fixed` / `Random` (default: Random). Branch on its selected value in the init rule.

## Item pool

| ID | Name         | Effect on use                          | Duration | Implementation                              |
|----|--------------|----------------------------------------|----------|---------------------------------------------|
| 0  | Damage Surge | +50% damage dealt                       | 8s       | Start/Stop Damage Modification              |
| 1  | Swift Boots  | +40% move speed                         | 8s       | Set Move Speed 140% → restore 100%          |
| 2  | Overshield   | +150 temporary health                   | 12s      | Add/Remove Health Pool From Player          |
| 3  | Second Wind  | Instant 200 heal                        | —        | Heal(self, 200); no expiry tracking         |
| 4  | Ghost        | Invisible to enemies                    | 4s       | Set Invisible → restore                     |
| 5  | Reset        | All ability cooldowns refunded          | —        | Set Ability Cooldown(0); no expiry tracking |

## Tuning constants

Set these up as the first four actions of R1 (I&I: Init). Each assigns a
**Workshop Setting Real** value to a global variable. The setting auto-generates
a slider in the lobby's Workshop Settings card, so hosts can tune without
opening the script.

| Var | Category | Name | Default | Min | Max | Sort |
|-----|----------|------|---------|-----|-----|------|
| `PickupRadius` | Items & Inventory | Pickup Radius | 2.0 | 0.5 | 10 | 0 |
| `RespawnTime` | Items & Inventory | Item Respawn Time | 12.0 | 1 | 60 | 1 |
| `TeamWindow` | Items & Inventory | Teammate Pickup Window | 5.0 | 0 | 30 | 2 |
| `DropExpiry` | Items & Inventory | Drop Expiry Time | 25.0 | 5 | 120 | 3 |

How to enter one: in R1, add action **Set Global Variable** → create the variable
with the exact name shown (e.g. `PickupRadius`) → click the value field → choose
**Workshop Setting Real** from the value list → fill in the six parameters above
(Category and Name are text, the rest numbers). No spaces allowed in variable names.

## Variables

The editor supports custom variable names (no spaces — underscores are fine).
Global and player variables live in separate sections of the variable menu, so
the names below are used as-is — no prefixes needed. Create them with the exact
names used in the rules.

**Mode preset — Global:** `PickupRadius`, `RespawnTime`, `TeamWindow`, `DropExpiry`
(the four Workshop Setting values, assigned in R1), `InitDone`, `Mode`,
`SpawnerPos` (array of vectors, hardcoded from Placer export), `SpawnerType`,
`SpawnerReadyAt`, `SpawnerFX` (parallel arrays; FX = 0 means on cooldown),
`DropPos`, `DropType`, `DropTime`, `DropTeam`, `DropFX`, `DropIcon`,
`DropFFA` (parallel arrays for live drops).

**Mode preset — Player:** `Item` (-1 = empty, else 0–5), `ActiveBuff`,
`BuffExpiresAt`, `UseWas` (edge detection), `HUD`.

**Placer preset — Global:** `InitDone`, `Placer`, `SpawnerPos`, `PinFX`,
`PinIcons`, `PinTexts`, `Exporting`, `ExportTexts`.

**Placer preset — Player:** `InteractWas`, `UndoWas`, `UltWas`, `ClearWas`.

(The two presets are separate scripts, so reused names like `InitDone` don't
collide.)

## Mode rules

**R1: I&I Init**
- Rule Name: I&I: Init
- Event Type: Ongoing - Global
- Conditions: `InitDone == false`
- Actions:
  - Set `InitDone` = true.
  - Set Global Variable `PickupRadius` = Workshop Setting Real("Items & Inventory", "Pickup Radius",
    2.0, 0.5, 10, 0).
  - Set Global Variable `RespawnTime` = Workshop Setting Real("Items & Inventory", "Item Respawn Time",
    12.0, 1, 60, 1).
  - Set Global Variable `TeamWindow` = Workshop Setting Real("Items & Inventory",
    "Teammate Pickup Window", 5.0, 0, 30, 2).
  - Set Global Variable `DropExpiry` = Workshop Setting Real("Items & Inventory", "Drop Expiry Time",
    25.0, 5, 120, 3).
  - Read `Item Distribution` setting → `Mode` (0 = Fixed, 1 = Random).
  - For each spawner index `i`: `SpawnerType[i]` = random integer 0–5 (both modes
    roll at init; they differ only on respawn — see R2). Set `SpawnerReadyAt[i]` = 0.
  - Create spawner visuals: ring effect color-coded by item type (optional in-world
    text with item name — costs an extra entity per spawner, see budget note).
    Store effect ID in `SpawnerFX[i]`.
  - For each player: `Item` = -1, `ActiveBuff` = -1. Create inventory HUD text,
    store ID in `HUD`. Call `UpdateHUD`.

**R2: I&I Pickup + Respawn Loop**
- Rule Name: I&I: Pickup + Respawn Loop
- Event Type: Ongoing - Global
- Conditions: (none)
- Actions:
  - `Wait(0.1)` — throttles the whole rule to 10 Hz.
  - For each spawner `i`: if `SpawnerReadyAt[i] <= now` and `SpawnerFX[i] == 0`
    → spawner came back online. If `Mode` == Random, re-roll `SpawnerType[i]`
    = random integer 0–5 (Fixed mode keeps the init roll all match). Recreate
    visuals color-coded to the (possibly new) type; store new effect ID.
  - For each living player `p`:
    - For each spawner `i` where `SpawnerReadyAt[i] <= now`:
      - If `Item(p) == -1` and Distance(Position Of(`p`), `SpawnerPos[i]`)
        ≤ `PickupRadius`:
        - `Item(p)` = `SpawnerType[i]`.
        - Destroy `SpawnerFX[i]`; set to 0.
        - `SpawnerReadyAt[i]` = now + `RespawnTime`.
        - Small message: `"Picked up <name>"`. Call `UpdateHUD(p)`.

**R3: I&I Use Item**
- Rule Name: I&I: Use Item
- Event Type: Ongoing - Each Player
- Conditions: Is Alive(Event Player); Is Button Held(Event Player, Interact);
  `UseWas` == false; `Item` >= 0
- Actions:
  - If / Else-If chain on `Item`:
    - 0 (Damage Surge): Start Damage Modification (+50% dealt); `ActiveBuff` = 0;
      `BuffExpiresAt` = now + 8.
    - 1 (Swift Boots): Set Move Speed 140%; `ActiveBuff` = 1; expires now + 8.
    - 2 (Overshield): Add Health Pool 150; `ActiveBuff` = 2; expires now + 12.
    - 3 (Second Wind): Heal(self, 200); no expiry tracking.
    - 4 (Ghost): Set Invisible(self); `ActiveBuff` = 4; expires now + 4.
    - 5 (Reset): Set Ability Cooldown (all abilities, 0); no expiry tracking.
  - `Item` = -1.
  - Small message: `"<name> used"`. Call `UpdateHUD`. Set `UseWas` = true.

**R4: I&I Expire Buffs**
- Rule Name: I&I: Expire Buffs
- Event Type: Ongoing - Each Player
- Conditions: `BuffExpiresAt` > 0; now >= `BuffExpiresAt`
- Actions:
  - Call `ClearBuff(Event Player)`.

**R5: I&I Death Drop**
- Rule Name: I&I: Death Drop
- Event Type: Player Died
- Conditions: (none — the item check is in the actions)
- Actions:
  - Call `ClearBuff(Victim)` — no buff survives death.
  - If `Item(Victim)` >= 0:
    - Append drop: pos = Position Of(Victim); type = `Item(Victim)`; time = now;
      team = Team Of(Victim); FFA flag = false.
    - Create team-colored ring effect + icon; append IDs to the drop arrays.
    - `Item(Victim)` = -1. Call `UpdateHUD(Victim)`.
    - Small message to victim's team: `"Item dropped!"`.

**R6: I&I Drop Pickup**
- Rule Name: I&I: Drop Pickup
- Event Type: Ongoing - Global
- Conditions: (none)
- Actions:
  - `Wait(0.1)` — same 10 Hz throttle as R2.
  - For each drop `i` (skip expired ones — R7 handles those):
    - `teamPhase` = (now < `DropTime[i]` + `TeamWindow`).
    - For each living player `p` with `Item(p)` == -1:
      - If (`teamPhase` == false OR Team Of(`p`) == `DropTeam[i]`) and
        Distance(Position Of(`p`), `DropPos[i]`) ≤ `PickupRadius`:
        - `Item(p)` = `DropType[i]`.
        - Destroy drop visuals; remove index `i` from all drop arrays.
        - Small message: `"Picked up <name>"`. Call `UpdateHUD(p)`.

**R7: I&I Drop Expiry + Phase Flip**
- Rule Name: I&I: Drop Expiry + Phase Flip
- Event Type: Ongoing - Global
- Conditions: (none)
- Actions:
  - `Wait(0.5)`.
  - For each drop `i`:
    - If `DropFFA[i]` == false and now >= `DropTime[i]` + `TeamWindow`:
      swap the team-colored icon for a neutral white/gold one;
      set `DropFFA[i]` = true.
    - If now >= `DropTime[i]` + `DropExpiry`: play poof effect at
      `DropPos[i]`; destroy visuals; remove index `i` from all drop arrays.

**R8: I&I Edge Tracker**
- Rule Name: I&I: Edge Tracker
- Event Type: Ongoing - Each Player
- Conditions: (none)
- Actions:
  - If Interact is NOT held by Event Player → `UseWas(Event Player)` = false.
  - (Add lines here for any future edge-detected binds.)

**Subroutines**
- `ClearBuff(p)`: If/Else-If on `ActiveBuff(p)` — 0: Stop Damage Modification;
  1: Set Move Speed 100%; 2: Remove Health Pool; 4: clear invisibility.
  Then `ActiveBuff(p)` = -1, `BuffExpiresAt(p)` = 0, `UpdateHUD(p)`.
- `UpdateHUD(p)`: set HUD text to `"Item: <name>"` (name resolved by If-chain on
  `Item(p)`) or `"Item: —"` when empty. Append active-buff countdown when a
  buff is running (nice-to-have).

**Entity budget**: effects + icons share one pool (historically 64). 20 spawners ×
(ring + optional name text) + live drops must fit. If spawner count grows, drop the
name texts and keep rings only — color coding carries the info.

---

## Placer tool (separate preset — "I&I Placer")

**Lobby**: Skirmish. The placer is auto-assigned Jetpack Cat by the script (P1) —
no hero select needed. Keep Jetpack Cat enabled in the lobby hero pool, or the
force has nothing to assign.

**Variables** — Global: `InitDone`, `Placer`, `SpawnerPos` (vectors),
`PinFX` / `PinIcons` / `PinTexts` (parallel ID arrays), `Exporting`,
`ExportTexts`. Player: `InteractWas`, `UndoWas`, `UltWas`, `ClearWas`.

**Binds**: Interact = drop pin · Crouch+Interact = undo last · Ult = toggle export
view · Crouch+Ult = clear all.

**P1: Placer Init**
- Rule Name: Placer: Init
- Event Type: Ongoing - Global
- Conditions: `InitDone` == false
- Actions:
  - Set `Placer` = first of All Players. Set `InitDone` = true.
  - Start Forcing Player To Be Hero(`Placer`, Hero(Jetpack Cat)) — auto-assigns
    the placer; the force persists for the session, so no manual hero pick needed.
  - Big message to `Placer`: `"Fly with Jetpack Cat. Interact = drop pin, Crouch+Interact = undo, Ult = export view, Crouch+Ult = clear all."`

**P2: Placer Drop Pin**
- Rule Name: Placer: Drop Pin
- Event Type: Ongoing - Each Player
- Conditions: Event Player == `Placer`; Is Alive(Event Player);
  Is Button Held(Event Player, Interact); `InteractWas` == false
- Actions:
  - `pos` = Position Of(Event Player) + (0, 0.5, 0).
  - Append `pos` to `SpawnerPos`.
  - Create ring effect at `pos`; create icon at `pos` + (0, 1.5, 0); create
    in-world text `"P<n>"` (`n` = array count) at `pos` + (0, 2.5, 0).
    Append each ID to `PinFX` / `PinIcons` / `PinTexts`.
  - Small message: `"Pin <n> placed"`. Set `InteractWas` = true.

**P3: Placer Undo**
- Rule Name: Placer: Undo
- Event Type: Ongoing - Each Player
- Conditions: Event Player == `Placer`; Is Button Held(Event Player, Interact);
  Is Button Held(Event Player, Crouch); `UndoWas` == false;
  Count Of(`SpawnerPos`) > 0
- Actions:
  - Destroy last of `PinFX`, `PinIcons`, `PinTexts`.
  - Pop the last entry from all four arrays.
  - Small message: `"Pin removed"`. Set `UndoWas` = true.

**P4: Placer Export Toggle**
- Rule Name: Placer: Export Toggle
- Event Type: Ongoing - Each Player
- Conditions: Event Player == `Placer`; Is Button Held(Event Player, Ultimate);
  `UltWas` == false
- Actions:
  - Flip `Exporting`.
  - If `Exporting` == true: for each spawner `i`, create HUD text
    `"P01: (12.4, 0.8, -33.1)"` (components via X Of / Y Of / Z Of, rounded to
    1 decimal); store IDs in `ExportTexts`.
  - Else: destroy all `ExportTexts`; empty the array.
  - Set `UltWas` = true.

**P5: Placer Clear All**
- Rule Name: Placer: Clear All
- Event Type: Ongoing - Each Player
- Conditions: Event Player == `Placer`; Is Button Held(Event Player, Crouch);
  Is Button Held(Event Player, Ultimate); `ClearWas` == false;
  Count Of(`SpawnerPos`) > 0
- Actions:
  - Destroy all pin visuals; empty all four arrays.
  - Big message: `"All pins cleared"`. Set `ClearWas` = true.

**P6: Placer Edge Tracker**
- Rule Name: Placer: Edge Tracker
- Event Type: Ongoing - Each Player
- Conditions: Event Player == `Placer`
- Actions:
  - If Interact NOT held → `InteractWas` = false.
  - If NOT (Interact held AND Crouch held) → `UndoWas` = false.
  - If Ultimate NOT held → `UltWas` = false.
  - If NOT (Crouch held AND Ultimate held) → `ClearWas` = false.

**Transcription**: toggle export view, screenshot it, type vectors into the real
mode's `SpawnerPos` init. One fly-through per map, done forever.

**Per-map presets (recommended)**: spawner positions are baked per map, so save one
preset per mapped map (e.g. "I&I — King's Row") with *only that map enabled* in the
Maps setting. Never enable a map you haven't run the Placer on — its spawner array
would be empty/mismatched and the mode breaks silently.

## Testing checklist

1. Solo Skirmish: pins drop, undo/export/clear behave, vectors look sane.
2. Solo TDM: spawner grants item, HUD updates, Interact uses it, buff expires cleanly.
3. Two players: death drop → teammate picks up in 5s window; enemy blocked, then
   allowed after flip; drop expires at 25s.
4. Full lobby: spawner respawn cadence feels right; entity budget holds (no
   shutdowns); Fixed vs Random setting flips behavior on restart.

## Future options (parked)

- Minor/major spawner tiers (10s/15s respawn, mirroring old pack cadence).
- Multi-slot inventory. (R3/R5/R6 assume single slot — would need arrays.)
- Per-lobby live placement as a game feature instead of bake-once.
