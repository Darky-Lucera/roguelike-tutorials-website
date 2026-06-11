# Part 12: Procedural Difficulty

## What You Will Build

By the end of this part, deeper dungeon floors will feel harder in a way the player can notice: tougher enemies appear more often, and better items become available to compensate. The spawn system driving this is small but reusable for any future content.

## Learning goals

- Replace fixed per-room spawn limits with floor-keyed tables
- Turn the flat weighted spawn table from Part 5 into a depth-aware encounter table
- Make both monster variety and item variety scale with dungeon depth
- Write a reusable generic helper with Python 3.12 type parameter syntax

---

## The problem with flat spawn rates

Right now, every floor draws from the same spawn tables with the same weights. The weighted table from Part 5 Exercise 2 already made some monsters rarer than others, but the weights never change: floor 10 rolls on exactly the same table as floor 1. This makes the game feel flat: depth adds no tension.

!!! info "Where you stand"
    Two Part 5 exercises built the current spawn system: Exercise 1 added minimum and maximum counts per room, and Exercise 2 replaced the hardcoded monster split with a `monster_chances` table and `random.choices`. This chapter absorbs both: the counts become floor-keyed tables in `config.py`, and the weight tables gain a floor dimension. If you skipped them, don't worry: Part 8 made the weighted tables part of the main path anyway, and Part 11's listings already carried the per-room limit parameters, so the diffs below match your code. Where a deleted line covers something you never added (an exercise item, for example), there is simply nothing to delete.

The fix is an **encounter table**: a mapping from dungeon floor to spawn probabilities. Trolls appear rarely on shallow floors and frequently on deep ones. New items only appear once the player has had time to learn the basics.

!!! info "Out-of-depth monsters"
    Depth-based spawn tables are as old as the genre. Rogue (1980) already picked monsters from bands tied to the dungeon level, so the alphabet got meaner as you went down. Angband turned the exception into a feature: its tables allow a small chance of an "out-of-depth" monster, a creature from deeper levels appearing early, producing the rare but memorable moment when something far too dangerous shows up on floor 2. The tables you are about to write are the same idea, expressed as data.

---

## Weighted selection, floor by floor

You already used weighted random selection in Part 5 Exercise 2: given a list of `(item, weight)` pairs, `random.choices` picks an item at random, where higher-weight items are more likely. As a refresher, here is how the item weights from this chapter behave on a deep floor, keeping only the four base items to make the example short:

```txt
Item              Weight   Cumulative
─────────────────────────────────────
Health Potion       35         35
Confusion Scroll    10         45
Lightning Scroll    25         70
Fireball Scroll     25         95

Random 0-94:   0-34  → Health Potion
              35-44  → Confusion Scroll
              45-69  → Lightning Scroll
              70-94  → Fireball Scroll
```

Note that "spawn nothing" is not an entry in the table. The number of items per room is rolled first (and can be zero); the table only decides *which* items spawn once that count is known.

What is new in this part is the floor dimension: the weight of each entry now depends on how deep the player is.

---

## Floor-keyed weight tables

A **floor-keyed table** is a list of `(floor, value)` pairs meaning "from this floor onward, the value is this". To find the value for floor N, take the last entry whose floor is less than or equal to N. Entries must be sorted by floor.

```python
# Troll: absent on floors 1-2, rare from floor 3, common from floor 7
troll: [(3, 15), (5, 30), (7, 60)]
```

On floor 1: troll has weight 0 (no entry with floor ≤ 1).
On floor 4: troll has weight 15 (entry `(3, 15)` is the last one ≤ 4).
On floor 7: troll has weight 60.

The same format works for things that are not weights at all. A table like `[(1, 2), (4, 3), (6, 5)]` can describe "at most 2 monsters per room on floors 1-3, at most 3 on floors 4-5, at most 5 from floor 6". One format, one helper function, many uses.

---

## config.py: spawn limits become tables

The four per-room constants from Part 5 turn into floor-keyed tables. Update `game/constants/config.py`:

```diff
 # Map generation
 MAP_WIDTH             = 80
 MAP_HEIGHT            = 44
 MAX_ROOMS             = 30
 ROOM_MIN_SIZE         = 6
 ROOM_MAX_SIZE         = 10
-MIN_MONSTERS_PER_ROOM = 0
-MAX_MONSTERS_PER_ROOM = 2
-MIN_ITEMS_PER_ROOM    = 0
-MAX_ITEMS_PER_ROOM    = 2
+
+# Procedural spawn limits: (floor_minimum, value), sorted by floor
+MIN_MONSTERS_BY_FLOOR = [(1, 0)]
+MAX_MONSTERS_BY_FLOOR = [(1, 2), (4, 3), (6, 5)]
+MIN_ITEMS_BY_FLOOR    = [(1, 0)]
+MAX_ITEMS_BY_FLOOR    = [(1, 1), (4, 2)]
```

The minimum tables keep their old behavior (always 0), but now they can scale too: change `MIN_MONSTERS_BY_FLOOR` to `[(1, 0), (6, 1)]` and every room from floor 6 onward is guaranteed at least one monster. Empty rooms stop being a place to catch your breath exactly when the dungeon turns hostile.

---

## factories.py: weights become floor-keyed

The entity weight tables stay where Part 5 Exercise 2 put them: in `game/entities/factories.py`, right next to the templates they reference. They could not move to `config.py` even if we wanted: the tables need the templates, and `factories.py` already imports `config.py`, so importing back would create a circular import. Only the plain numeric limits belong in `config.py`; the weight tables live with the data they describe. The format changes from a flat list to a dict of floor-keyed weights. Update `monster_chances`:

```diff
-# Part-5. Ex 2: Weighted monster table
-monster_chances = [
-    (orc,   25),
-    (troll, 75),
-]
+# Spawn weights by floor: (floor_minimum, weight), sorted by floor
+monster_chances = {
+    orc:   [(1, 80)],
+    troll: [(3, 15), (5, 30), (7, 60)],
+}
```

And `item_chances`:

```diff
-item_chances = [
-    (health_potion,    40),
-    (chest,            60),
-    # Part-8. Ex 2: Backpack growing scroll
-    (backpack_scroll,  20),
-    (confusion_scroll, 15),
-    (fireball_scroll,  15),
-    (lightning_scroll, 15),
-    (mapping_scroll,   15),
-    (drain_scroll,     15),
-    (teleport_scroll,  15),
-]
+# Spawn weights by floor: (floor_minimum, weight), sorted by floor
+item_chances = {
+    health_potion:    [(1, 35)],
+    chest:            [(1, 25)],   # Part-5. Ex 3: Chest
+    backpack_scroll:  [(2, 10)],   # Part-8. Ex 2: Backpack growing scroll
+    confusion_scroll: [(2, 10)],
+    mapping_scroll:   [(2, 10)],   # Part-9. Ex 1: Scroll of mapping
+    lightning_scroll: [(4, 25)],
+    drain_scroll:     [(4, 10)],   # Part-9. Ex 2: Drain scroll
+    teleport_scroll:  [(5, 10)],   # Part-9. Ex 3: Teleport scroll
+    fireball_scroll:  [(6, 25)],
+}
```

Floor 1 now offers only potions and chests. Utility scrolls trickle in from floor 2, attack scrolls from floor 4, and fireballs arrive on floor 6, just in time for the crowds that the new monster limits allow. Some weights shift along the way (the chest drops from 60 to 25, the potion from 40 to 35): with scrolls joining the pool floor by floor, the early-game entries no longer need to dominate the table.

!!! info "Items from earlier exercises"
    The chest comes from Part 5 Exercise 3 (via Part 8), the backpack scroll from Part 8 Exercise 2, and the mapping, drain and teleport scrolls from Part 9 Exercises 1-3. If you skipped any of them, leave that line out: the table works with any subset of entries.

!!! question "Can entities be dict keys?"
    Yes. Python objects are hashable by identity unless a class says otherwise, and the templates in `factories.py` are module-level singletons: there is exactly one `orc` object, so it works perfectly as a key. Keying the table by template also keeps it honest: a typo like `trol` is an immediate `NameError`, while a misspelled string key would fail silently by never spawning.

---

## map_generator.py: two helpers

The selection logic lives in `game/map/map_generator.py`, next to its only caller. First add the import for the new config tables:

```diff
+from game.constants import config as constants
 from game.entities import factories
```

Then add the two helpers, above `place_entities`:

```python
def get_value_for_floor(
    values_by_floor: list[tuple[int, int]],
    floor: int,
) -> int:
    """Return the value of the last entry at or below the given floor."""
    current_value = 0
    for floor_minimum, value in values_by_floor:
        if floor_minimum > floor:
            break
        current_value = value
    return current_value


def get_entities_at_random[T](
    weighted_chances_by_floor: dict[T, list[tuple[int, int]]],
    number_of_entities: int,
    floor: int,
) -> list[T]:
    """Pick random entities using the weights for the given floor."""
    entity_weighted_chances: dict[T, int] = {}
    for entity, values in weighted_chances_by_floor.items():
        weight = get_value_for_floor(values, floor)
        if weight > 0:
            entity_weighted_chances[entity] = weight

    if not entity_weighted_chances:
        return []

    entities = list(entity_weighted_chances.keys())
    weights  = list(entity_weighted_chances.values())

    return random.choices(entities, weights=weights, k=number_of_entities)
```

`get_value_for_floor` walks the table in order and remembers the last entry that applies. Older tutorials call this function `get_max_value_for_floor`, but ours also reads the *minimum* tables, so that name would lie; the helper returns whichever value is in force on the given floor, whatever it represents.

`get_entities_at_random` builds the weight list for the requested floor, drops every entry whose weight is 0 (not available yet), and lets `random.choices` do the rest. Both keys and values come from the same dict, so the two lists stay aligned: dicts preserve insertion order.

!!! tip "Generic functions: `def f[T](...)`"
    The `[T]` after the function name declares a **type parameter** (Python 3.12, PEP 695). It ties the input to the output: pass a `dict[Actor, ...]` and the checker knows you get a `list[Actor]` back; pass a `dict[Item, ...]` and you get `list[Item]`. Without it we would have to type the function with a common base class and lose precision, or repeat ourselves with two nearly identical functions. Before Python 3.12 this required declaring a separate `TypeVar` object from the `typing` module; the new syntax does the same job inline.

---

## A smaller place_entities()

With the tables and helpers in place, `place_entities` shrinks. Replace it entirely:

```python
def place_entities(
    room: RectangularRoom,
    dungeon: GameMap,
    current_floor: int,
) -> None:
    number_of_monsters = random.randint(
        get_value_for_floor(constants.MIN_MONSTERS_BY_FLOOR, current_floor),
        get_value_for_floor(constants.MAX_MONSTERS_BY_FLOOR, current_floor),
    )
    number_of_items = random.randint(
        get_value_for_floor(constants.MIN_ITEMS_BY_FLOOR, current_floor),
        get_value_for_floor(constants.MAX_ITEMS_BY_FLOOR, current_floor),
    )

    monsters = get_entities_at_random(
        factories.monster_chances, number_of_monsters, current_floor,
    )
    items = get_entities_at_random(
        factories.item_chances, number_of_items, current_floor,
    )

    for entity in monsters + items:
        x = random.randint(room.x1 + 1, room.x2 - 1)
        y = random.randint(room.y1 + 1, room.y2 - 1)

        if not any(e.x == x and e.y == y for e in dungeon.entities):
            entity.spawn(dungeon, x, y)
```

The min/max parameters are gone: counts now come from the config tables, scaled by floor. The selection produces template objects directly, so spawning is a single method call, and monsters and items share one placement loop. The collision rule is unchanged: if the rolled tile is occupied, that spawn is skipped.

---

## generate_dungeon() loses four parameters

`generate_dungeon` no longer needs to carry the per-room limits around. Note what does *not* change: `current_floor` already arrives here since Part 11 (the up stairs need it), the first room still spawns nothing (your starting room stays safe), and the stairs placement at the end is untouched.

```diff
 def generate_dungeon(
     max_rooms: int,
     room_min_size: int,
     room_max_size: int,
     map_width: int,
     map_height: int,
-    # Part-5. Ex 1: Minimum monsters per room
-    min_monsters_per_room: int,
-    max_monsters_per_room: int,
-    min_items_per_room: int,
-    max_items_per_room: int,
     player: Entity,
     seed: int,
     current_floor: int,
 ) -> GameMap:
```

And the call inside the room loop:

```diff
-            place_entities(
-                new_room,
-                dungeon,
-                min_monsters_per_room,
-                max_monsters_per_room,
-                min_items_per_room,
-                max_items_per_room,
-            )
+            place_entities(new_room, dungeon, current_floor)
```

This is the architectural payoff of the chapter: adding new floor-scaled content (a new monster, a new item, a different difficulty curve) now means editing a data table. The generator's signature never changes again for spawn reasons.

---

## GameWorld and setup_game slim down

`GameWorld` stored the four limits only to forward them. Remove them from `game/game_world.py`:

```diff
     def __init__(
         self,
         *,
         engine: Engine,
         map_width: int,
         map_height: int,
         max_rooms: int,
         room_min_size: int,
         room_max_size: int,
-        min_monsters_per_room: int,
-        max_monsters_per_room: int,
-        min_items_per_room: int,
-        max_items_per_room: int,
         seed: int,
         current_floor: int = 0,
     ) -> None:
         self.engine                = engine
         self.map_width             = map_width
         self.map_height            = map_height
         self.max_rooms             = max_rooms
         self.room_min_size         = room_min_size
         self.room_max_size         = room_max_size
-        self.min_monsters_per_room = min_monsters_per_room
-        self.max_monsters_per_room = max_monsters_per_room
-        self.min_items_per_room    = min_items_per_room
-        self.max_items_per_room    = max_items_per_room
         self.seed                  = seed
         self.current_floor         = current_floor
         self.floors: list[GameMap] = []
```

```diff
     def generate_floor(self) -> None:
         from game.map.map_generator import generate_dungeon

         self.current_floor += 1
         self.engine.game_map = generate_dungeon(
             max_rooms             = self.max_rooms,
             room_min_size         = self.room_min_size,
             room_max_size         = self.room_max_size,
             map_width             = self.map_width,
             map_height            = self.map_height,
-            min_monsters_per_room = self.min_monsters_per_room,
-            max_monsters_per_room = self.max_monsters_per_room,
-            min_items_per_room    = self.min_items_per_room,
-            max_items_per_room    = self.max_items_per_room,
             player                = self.engine.player,
             seed                  = self.seed + self.current_floor,
             current_floor         = self.current_floor,
         )
         self.floors.append(self.engine.game_map)
```

The `self.floors.append(...)` line stays exactly where it is: floor persistence from Part 11 is not affected by any of this.

Finally, remove the four arguments from `new_game()` in `game/setup_game.py`:

```diff
     engine.game_world = GameWorld(
         engine                = engine,
         max_rooms             = constants.MAX_ROOMS,
         room_min_size         = constants.ROOM_MIN_SIZE,
         room_max_size         = constants.ROOM_MAX_SIZE,
         map_width             = constants.MAP_WIDTH,
         map_height            = constants.MAP_HEIGHT,
-        min_monsters_per_room = constants.MIN_MONSTERS_PER_ROOM,
-        max_monsters_per_room = constants.MAX_MONSTERS_PER_ROOM,
-        min_items_per_room    = constants.MIN_ITEMS_PER_ROOM,
-        max_items_per_room    = constants.MAX_ITEMS_PER_ROOM,
         seed                  = seed,
     )
```

!!! info "Old saves keep working this time"
    Part 11 warned that adding the Level component broke old save files. This chapter does the opposite kind of change, and it is harmless: pickle restores whatever attributes the saved object had, so an old `GameWorld` loads with its now-unused `min_monsters_per_room` and friends still attached. The new code simply never reads them. As a general rule: *adding* required state breaks old saves, *removing* reads does not.

---

## Visualizing the difficulty curve

Here is how the monster mix changes across floors with the tables above:

| Floor | Max monsters | Orc weight | Troll weight | Effective troll % |
| --- | --- | --- | --- | --- |
| 1 | 2 | 80 | 0 | 0% |
| 3 | 2 | 80 | 15 | ~16% |
| 5 | 3 | 80 | 30 | ~27% |
| 7 | 5 | 80 | 60 | ~43% |

By floor 7 you face up to 5 monsters per room, nearly half of which are trolls. Items scale in step: attack scrolls appear from floor 4, fireballs from floor 6, just in time for the harder encounters.

!!! example "Tuning the tables"
    Adjust the numbers until the game feels balanced. Run the game 10 times at each floor depth. If you consistently clear floor 6 without taking damage, trolls need more weight or higher stats. If you die on floor 2, tone down the monster limits. The table format makes this iteration fast: every knob is one number in one place.

---

## Testing your work

Run `python main.py` multiple times and descend to different floors:

- [ ] Floor 1: only orcs; at most 2 monsters and 1 item per room; only potions and chests on the ground
- [ ] Floor 2: confusion, backpack and mapping scrolls join the item pool
- [ ] Floor 3: the occasional troll appears
- [ ] Floor 4: up to 2 items per room; lightning and drain scrolls appear
- [ ] Floor 6: fireball scrolls appear; up to 5 monsters per room
- [ ] Floor 7+: trolls are common; combat is noticeably harder
- [ ] Revisiting a floor through the stairs still restores it exactly as you left it

---

## Summary

Spawn rates now scale with dungeon depth. Key additions:

- **`get_value_for_floor`**: reads a floor-keyed table and returns the value currently in force
- **`get_entities_at_random`**: weighted random selection using the weights of the current floor
- **Floor-keyed tables**: `MIN/MAX_MONSTERS_BY_FLOOR` and `MIN/MAX_ITEMS_BY_FLOOR` in `config.py`; `monster_chances` and `item_chances` in `factories.py`

**Current architecture**:

- `game/entities/factories.py`: owns the spawn data (templates and their floor-keyed weights)
- `game/map/map_generator.py`: owns the selection algorithm and entity placement
- `GameWorld`: passes `current_floor` into generation (since Part 11); spawn limits no longer travel through it
- Adding new floor-scaled content means editing data tables, not changing signatures

**File structure**:

```txt
main.py
game/
├── __init__.py
├── actions.py
├── engine.py
├── exceptions.py
├── game_world.py               ← modified
├── hud.py
├── game_states.py
├── message_log.py
├── setup_game.py               ← modified
├── constants/
│   ├── __init__.py
│   ├── colors.py
│   ├── config.py               ← modified
│   ├── keys.py
│   └── sprites.py
├── entities/
│   ├── __init__.py
│   ├── entity.py
│   ├── factories.py            ← modified
│   ├── render_order.py
│   └── components/
│       ├── __init__.py
│       ├── ai.py
│       ├── base_component.py
│       ├── consumable.py
│       ├── fighter.py
│       ├── inventory.py
│       └── level.py
└── map/
    ├── __init__.py
    ├── game_map.py
    ├── tile_types.py
    └── map_generator.py        ← modified
```

---

## Exercises

1. **Stairs guardian**:

    Every 5th floor, guarantee one troll next to the down stairs regardless of the RNG roll. In `generate_dungeon`, after the stairs spawn, check `current_floor % 5 == 0` and spawn a troll on a free tile of the last room. The `free` list built for the stairs placement is a good starting point, but remember it was computed before the stairs spawned: remove the stairs tile from it first.

2. **New monster: vampire**:

    Add a `vampire` entry to `monster_chances` that only appears from floor 8 onward (weight 20). Create the factory in `game/entities/factories.py` with high HP but low defense. For its signature move, healing when it hits, remember that attacks resolve in `Fighter.melee_attack`, not in the AI: give `Fighter` an optional lifesteal amount and heal the attacker there whenever the attack deals damage.

3. **Item drought**:

    Change `health_potion` to `[(1, 35), (2, 0), (3, 35)]`: no potions spawn on floor 2, then they return. This forces players to ration their healing early; observe how it changes risk-taking behavior. Careful with the third entry: table entries mean "from this floor onward", so `(2, 0)` alone would remove potions for the entire rest of the game.
