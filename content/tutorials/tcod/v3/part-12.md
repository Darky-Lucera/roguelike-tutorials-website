# Part 12: Procedural Difficulty

## What You Will Build

By the end of this part, deeper dungeon floors will feel harder in a way the player can notice: tougher enemies appear more often, and better items become available to compensate. The spawn system driving this is small but reusable for any future content.

## Learning goals

- Replace hard-coded spawn counts with floor-keyed weighted tables
- Understand weighted random selection as a reusable algorithm
- Make both monster variety and item variety scale with dungeon depth
- Eliminate magic numbers from the map generator

---

## The problem with flat spawn rates

Right now, every floor spawns the same monster types in the same proportions. A troll on floor 1 is just as likely as a troll on floor 10. This makes the game feel flat, depth has no tension.

The fix is an **encounter table**: a mapping from dungeon floor to spawn probabilities. Trolls appear rarely on shallow floors and frequently on deep ones. New items only appear once the player has had time to learn the basics.

---

## Weighted random selection

Before writing the tables, understand the algorithm. Given a list of `(item, weight)` pairs, pick one item at random where higher-weight items are more likely.

```txt
Item          Weight   Cumulative
─────────────────────────────────
Health Potion   35         35
Lightning Scroll 25        60
Fireball Scroll  25        85
Confusion Scroll 10        95
(nothing)         5       100

Random 0–99:  0–34 → Health Potion
              35–59 → Lightning Scroll
              60–84 → Fireball Scroll
              85–94 → Confusion Scroll
              95–99 → nothing
```

Python's `random.choices` does this directly with a `weights` argument. We just need to build the weight list for the current floor.

---

## Floor-keyed weight tables

A **floor-keyed table** is a list of `(floor, weight)` pairs meaning "from this floor onward, the weight is this value". To find the weight for floor N, find the last entry whose floor ≤ N.

```python
# Troll: not on floor 1-2, rare on 3-4, common by floor 6
"troll": [(3, 15), (5, 30), (7, 60)]
```

On floor 1: troll has weight 0 (no entry with floor ≤ 1).
On floor 4: troll has weight 15 (entry `(3, 15)` is the last one ≤ 4).
On floor 7: troll has weight 60.

---

## The map generator: rewrite spawn logic

Replace the static `place_entities` with a table-driven version. Update `game/map/map_generator.py`:

```python
from __future__ import annotations

import random
from collections.abc import Iterator
from typing import TYPE_CHECKING

import tcod

from game.entities import factories
from game.map.game_map import GameMap
from game.map import tile_types

if TYPE_CHECKING:
    from game.entities.entity import Entity

# ── Spawn count tables ────────────────────────────────────────────────────────

max_items_by_floor = [
    (1, 1),
    (4, 2),
]

max_monsters_by_floor = [
    (1, 2),
    (4, 3),
    (6, 5),
]

# ── Weighted entity tables ────────────────────────────────────────────────────

item_chances: dict[str, list[tuple[int, int]]] = {
    "health_potion":    [(1, 35)],
    "confusion_scroll": [(2, 10)],
    "lightning_scroll": [(4, 25)],
    "fireball_scroll":  [(6, 25)],
}

enemy_chances: dict[str, list[tuple[int, int]]] = {
    "orc":   [(1, 80)],
    "troll": [(3, 15), (5, 30), (7, 60)],
}

# ── Helpers ───────────────────────────────────────────────────────────────────

def get_max_value_for_floor(
    weighted_chances_by_floor: list[tuple[int, int]],
    floor: int,
) -> int:
    current_value = 0
    for floor_minimum, value in weighted_chances_by_floor:
        if floor_minimum > floor:
            break
        current_value = value
    return current_value


def get_entities_at_random(
    weighted_chances_by_entity: dict[str, list[tuple[int, int]]],
    number_of_entities: int,
    floor: int,
) -> list[str]:
    entity_weighted_chances = {}
    for key, values in weighted_chances_by_entity.items():
        chance = get_max_value_for_floor(values, floor)
        if chance:
            entity_weighted_chances[key] = chance

    if not entity_weighted_chances:
        return []

    entities = list(entity_weighted_chances.keys())
    weights = list(entity_weighted_chances.values())

    chosen = random.choices(entities, weights=weights, k=number_of_entities)
    return chosen


# ── Entity name → factory lookup ──────────────────────────────────────────────

ITEM_FACTORIES = {
    "health_potion":    factories.health_potion,
    "confusion_scroll": factories.confusion_scroll,
    "lightning_scroll": factories.lightning_scroll,
    "fireball_scroll":  factories.fireball_scroll,
}

ENEMY_FACTORIES = {
    "orc":   factories.orc,
    "troll": factories.troll,
}
```

!!! question "Why string keys?"
    We map string names to factories instead of using factory objects as dict keys. This makes the tables readable, `"troll": [(3, 15)]` is clear, while `factories.troll: [(3, 15)]` requires knowing what that object is. The lookup at the end is one extra line.

---

## Updated place_entities()

```python
class RectangularRoom:
    ...  # unchanged


def place_entities(room: RectangularRoom, dungeon: GameMap, floor_number: int) -> None:
    number_of_monsters = random.randint(
        0, get_max_value_for_floor(max_monsters_by_floor, floor_number)
    )
    number_of_items = random.randint(
        0, get_max_value_for_floor(max_items_by_floor, floor_number)
    )

    monsters = get_entities_at_random(enemy_chances, number_of_monsters, floor_number)
    items = get_entities_at_random(item_chances, number_of_items, floor_number)

    for entity_name in monsters + items:
        factories = ENEMY_FACTORIES if entity_name in ENEMY_FACTORIES else ITEM_FACTORIES
        factory = factories[entity_name]

        x = random.randint(room.x1 + 1, room.x2 - 1)
        y = random.randint(room.y1 + 1, room.y2 - 1)

        if not any(entity.x == x and entity.y == y for entity in dungeon.entities):
            factory.spawn(dungeon, x, y)
```

---

## Updated generate_dungeon()

`generate_dungeon` now receives the current floor from `GameWorld` and passes it to `place_entities`:

```python
def generate_dungeon(
    max_rooms: int,
    room_min_size: int,
    room_max_size: int,
    map_width: int,
    map_height: int,
    player: Entity,
    floor_number: int,
) -> GameMap:
    dungeon = GameMap(map_width, map_height, entities={player})

    rooms: list[RectangularRoom] = []

    center_of_last_room = (0, 0)

    for _ in range(max_rooms):
        room_width = random.randint(room_min_size, room_max_size)
        room_height = random.randint(room_min_size, room_max_size)

        x = random.randint(0, dungeon.width - room_width - 1)
        y = random.randint(0, dungeon.height - room_height - 1)

        new_room = RectangularRoom(x, y, room_width, room_height)

        if any(new_room.intersects(other_room) for other_room in rooms):
            continue

        dungeon.tiles[new_room.inner] = tile_types.floor

        if not rooms:
            player.place(*new_room.center, dungeon)
        else:
            for x, y in tunnel_between(rooms[-1].center, new_room.center):
                dungeon.tiles[x, y] = tile_types.floor

        center_of_last_room = new_room.center

        place_entities(new_room, dungeon, floor_number)

        rooms.append(new_room)

    dungeon.tiles[center_of_last_room] = tile_types.floor
    dungeon.downstairs_location = center_of_last_room
    factories.down_stairs.spawn(dungeon, *center_of_last_room)

    return dungeon
```

The signature no longer takes `min/max_monsters_per_room` or `min/max_items_per_room`, those came from static values. Now `place_entities` reads floor-scaled values from the tables.

Update `GameWorld.generate_floor()` to match the new signature:

```python
def generate_floor(self) -> None:
    from game.map.map_generator import generate_dungeon

    self.current_floor += 1
    self.engine.game_map = generate_dungeon(
        max_rooms=self.max_rooms,
        room_min_size=self.room_min_size,
        room_max_size=self.room_max_size,
        map_width=self.map_width,
        map_height=self.map_height,
        player=self.engine.player,
        floor_number=self.current_floor,
    )
```

---

## Visualizing the difficulty curve

Here is how monster mix changes across floors with the tables above:

| Floor | Max monsters | Orc weight | Troll weight | Effective troll % |
| --- | --- | --- | --- | --- |
| 1 | 2 | 80 | 0 | 0% |
| 3 | 2 | 80 | 15 | ~16% |
| 5 | 3 | 80 | 30 | ~27% |
| 7 | 5 | 80 | 60 | ~43% |

By floor 7 you face up to 5 monsters per room, nearly half of which are trolls. Items also scale: lightning scrolls appear from floor 4, fireballs from floor 6, just in time for the harder encounters.

!!! example "Tuning the tables"
    Adjust the numbers until the game feels balanced. Run the game 10 times at each floor depth. If you consistently clear floor 6 without taking damage, trolls need more weight or higher stats. If you die on floor 2, tone down orc spawn counts. The table format makes this iteration fast.

---

## Testing your work

Run `python main.py` multiple times and descend to different floors:

- [ ] Floor 1: only orcs, at most 2 per room, at most 1 item
- [ ] Floor 3: occasional troll appears
- [ ] Floor 4: up to 2 items per room, lightning scrolls appear
- [ ] Floor 6: fireball scrolls appear, up to 5 monsters per room
- [ ] Floor 7+: trolls are common; combat is noticeably harder

---

## Summary

Spawn rates now scale with dungeon depth. Key additions:

- **`get_max_value_for_floor`**: reads a floor-keyed table and returns the current value
- **`get_entities_at_random`**: weighted random selection for any number of entities
- **Floor-keyed tables**: `max_monsters_by_floor`, `max_items_by_floor`, `item_chances`, `enemy_chances`

**Current architecture**:

- `game/map/map_generator.py`: owns floor-scaled spawn tables and weighted selection
- `GameWorld`: passes the current floor into dungeon generation
- Spawn limits and entity chances are data tables instead of fixed parameters
- Adding new floor-scaled content no longer changes `generate_dungeon()`'s signature

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
├── input_handlers.py
├── message_log.py
├── setup_game.py
├── constants/
│   ├── __init__.py
│   ├── colors.py
│   └── sprites.py
├── entities/
│   ├── __init__.py
│   ├── entity.py
│   ├── factories.py
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

1. **Boss floor**:

    Every 5th floor, guarantee one troll regardless of the RNG roll. Add a post-processing step in `place_entities` that checks `floor_number % 5 == 0` and forces a troll spawn in the first available position.

2. **New monster: vampire**:

    Add a `vampire` entry to `enemy_chances` that only appears from floor 8 onward (weight 20). Create the factory in `game/entities/factories.py` with high HP but low defense, and an AI that heals 2 HP whenever it successfully attacks the player.

3. **Item drought**:

    Add a `(2, 0)` entry to `health_potion` in `item_chances`, meaning no potions on floor 2. This forces players to ration their healing early. Observe how this changes risk-taking behavior.

**Next**: [Part 13: Equipment](part-13.md)
