# Part 3: Generating a Dungeon

## Learning goals

- Understand what procedural generation means and why it matters
- Implement the rooms-and-corridors dungeon algorithm
- Place the player in the first room
- Use Bresenham lines to dig L-shaped tunnels

---

## Procedural generation

A hand-crafted dungeon is fixed: every player sees the same thing. A procedurally generated dungeon is created algorithmically at runtime: different every time, potentially infinite in variety.

The classic roguelike dungeon uses the **rooms-and-corridors** approach:

1. Pick a random position and size for a room
2. Check if it overlaps with any existing room
3. If not, place it and dig a tunnel to the previous room
4. Repeat `max_rooms` times. Some attempts get skipped (overlap), so the final dungeon usually has fewer than `max_rooms` rooms.

This gives dungeons that feel hand-designed (recognizable rooms connected by corridors) but with endless variety.

!!! example "Other algorithms"
    Many alternatives exist:

    | Algorithm | Feel | Complexity |
    |---|---|---|
    | Rooms-and-corridors | Classic dungeon | Low |
    | BSP (Binary Space Partitioning) | Structured, no wasted space | Medium |
    | Cellular automata | Organic caves | Medium |
    | Wave Function Collapse | Highly varied, tile-pattern aware | High |

    We use rooms-and-corridors because it is the most intuitive to understand and produces instantly recognizable roguelike dungeons. The architecture we build here can be swapped for another generator later without changing anything else.

---

## The map package

Part 2 created `game/game_map.py` and `game/tile_types.py` directly inside `game/`. Now that we are adding dungeon generation, it is worth grouping the map-related files before the project grows further:

```txt
game/
  map/
    __init__.py
    game_map.py
    tile_types.py
    map_generator.py
```

Create `game/map/__init__.py` as an empty file. Then move:

- `game/game_map.py` to `game/map/game_map.py`
- `game/tile_types.py` to `game/map/tile_types.py`

Update the import in `game/map/game_map.py`:

```diff
-from game import tile_types
+from game.map import tile_types
```

Update the type-checking import in `game/engine.py`:

```diff
-    from game.game_map import GameMap
+    from game.map.game_map import GameMap
```

---

## The dungeon generator module

The original libtcod tutorial put dungeon generation inside the `GameMap` class. That works for simple cases, but it couples the map *data structure* to one specific *algorithm*. If you ever want a second generator (say, a cave level that uses cellular automata), you end up with a cluttered `GameMap`.

The better approach: a separate `game/map/map_generator.py` module that takes configuration parameters and returns a `GameMap`.

!!! info "Design decision: Separate map generator module"
    `GameMap` is responsible for *storing* and *rendering* the tile grid. `game/map/map_generator.py` is responsible for *filling* it. This is the Single Responsibility Principle: each module does one thing. Swapping algorithms later is then a one-line change in `main.py`.

---

## The RectangularRoom class

Create `game/map/map_generator.py` with our first class:

```python
from __future__ import annotations

import random
from collections.abc import Iterator
from typing import TYPE_CHECKING

import tcod

from game.map.game_map import GameMap
from game.map import tile_types

if TYPE_CHECKING:
    from game.entity import Entity


class RectangularRoom:
    def __init__(self, x: int, y: int, width: int, height: int) -> None:
        self.x1 = x
        self.y1 = y
        self.x2 = x + width
        self.y2 = y + height

    @property
    def center(self) -> tuple[int, int]:
        return (self.x1 + self.x2) // 2, (self.y1 + self.y2) // 2

    @property
    def inner(self) -> tuple[slice, slice]:
        """The interior of the room as numpy slices."""
        return slice(self.x1 + 1, self.x2), slice(self.y1 + 1, self.y2)

    def intersects(self, other: RectangularRoom) -> bool:
        """True if this room overlaps with another."""
        return (
            self.x1 <= other.x2
            and self.x2 >= other.x1
            and self.y1 <= other.y2
            and self.y2 >= other.y1
        )
```

!!! question "What does `@property` do?"
    `@property` lets us call a method like an attribute. Instead of writing `room.center()`, we write `room.center`. This is useful for values that are computed from the object's current state but feel like data: a room's center is derived from `x1`, `y1`, `x2`, and `y2`.

Two rectangles overlap if and only if their projections overlap on **both** the X and Y axes. The four conditions check exactly that: each axis must have at least one common point.

### Why `inner` adds 1

Consider a room placed at `(1, 1)` to `(6, 6)`. If we dig out exactly that rectangle, and then place a room at `(6, 1)` to `(11, 6)`, the two rooms share the column at `x=6`; they merge with no wall between them.

By starting the interior at `x1 + 1` and `y1 + 1`, we always leave at least one tile of wall around each room:

```txt
Without +1:                  With +1 (what we do):

  0 1 2 3 4 5 6 7            0 1 2 3 4 5 6 7
0 # # # # # # # #          0 # # # # # # # #
1 # . . . . . . #          1 # + + + + + + #
2 # . . . . . . #    ->    2 # + . . . . + #
3 # . . . . . . #          3 # + . . . . + #
4 # . . . . . . #          4 # + . . . . + #
5 # . . . . . . #          5 # + . . . . + #
6 # . . . . . . #          6 # + + + + + + #
7 # # # # # # # #          7 # # # # # # # #
```

!!! info "About `+` and `#`"
    Here `+` is only used to highlight the wall tiles preserved by the `+1` offset. In the actual game, these are still normal wall tiles.

Two rooms next to each other will always have a wall between them.

---

## L-shaped tunnels

Rooms need corridors between them. We connect room centers with an L-shaped tunnel: first move horizontally (or vertically), then turn.

Add this function to `game/map/map_generator.py`:

```python
def tunnel_between(
    start: tuple[int, int],
    end: tuple[int, int],
) -> Iterator[tuple[int, int]]:
    """Yield the (x, y) coordinates of an L-shaped tunnel."""
    x1, y1 = start
    x2, y2 = end

    if random.random() < 0.5:
        corner_x, corner_y = x2, y1  # Move right, then down
    else:
        corner_x, corner_y = x1, y2  # Move down, then right

    for x, y in tcod.los.bresenham((x1, y1), (corner_x, corner_y)).tolist():
        yield x, y

    for x, y in tcod.los.bresenham((corner_x, corner_y), (x2, y2)).tolist():
        yield x, y
```

The two options look like this:

```txt
Option A (right then down):      Option B (down then right):

  ┌──────┐                    ┌──────┐
  │      │                    │      │
  │  ●───────────────┐        │  ●   │
  └──────┘           │        └──│───┘
                     │           │
                     │           │
                  ┌──│───┐       │            ┌──────┐
                  │  ●   │       └───────────────●   │
                  └──────┘                    └──────┘
```

The 50/50 random choice means the dungeon gets a mix of both shapes, which looks more natural than always turning the same way.

`tcod.los.bresenham` returns the coordinates of a straight line between two points using [Bresenham's line algorithm](https://en.wikipedia.org/wiki/Bresenham%27s_line_algorithm). We use it from tcod's line-of-sight module: it happens to be exactly what we need for drawing grid lines.

The `yield` keyword makes `tunnel_between` a *generator*: it produces coordinates one at a time instead of building a full list. The caller (`for x, y in tunnel_between(...)`) receives them as it iterates.

---

## The dungeon generator

Now add the main function:

```python
def generate_dungeon(
    max_rooms: int,
    room_min_size: int,
    room_max_size: int,
    map_width: int,
    map_height: int,
    player: Entity,
) -> GameMap:
    """Generate a new dungeon map and place the player."""
    dungeon = GameMap(map_width, map_height)

    rooms: list[RectangularRoom] = []

    for _ in range(max_rooms):
        room_width = random.randint(room_min_size, room_max_size)
        room_height = random.randint(room_min_size, room_max_size)

        x = random.randint(0, dungeon.width - room_width - 1)
        y = random.randint(0, dungeon.height - room_height - 1)

        new_room = RectangularRoom(x, y, room_width, room_height)

        # Skip this room if it overlaps with any existing room.
        if any(new_room.intersects(other) for other in rooms):
            continue

        # Dig out the interior.
        dungeon.tiles[new_room.inner] = tile_types.floor

        if not rooms:
            # First room: place the player here.
            player.set_position(*new_room.center)
        else:
            # All subsequent rooms: dig a tunnel to the previous room.
            for x, y in tunnel_between(rooms[-1].center, new_room.center):
                dungeon.tiles[x, y] = tile_types.floor

        rooms.append(new_room)

    return dungeon
```

The algorithm in plain language:

1. Attempt to place up to `max_rooms` rooms
2. For each attempt, pick a random size and position
3. If it overlaps an existing room, skip it (try again next iteration)
4. Otherwise, dig it out and connect it to the last room with a tunnel
5. Put the player in the first room that was successfully placed

!!! tip "Tweaking the dungeon"
    The three parameters `max_rooms`, `room_min_size`, and `room_max_size` control the dungeon character. More rooms = denser dungeon. Smaller rooms = tighter corridors. Experiment after Part 3 is working.

---

## The GameMap needs walls to start

The generator digs floors out of a solid wall map. Update `game/map/game_map.py` to start with walls:

```diff
-        self.tiles = np.full((width, height), fill_value=tile_types.floor, order="F")
-
-        # A small wall for testing, we will remove it in Part 3.
-        self.tiles[30:33, 22] = tile_types.wall
+        self.tiles = np.full((width, height), fill_value=tile_types.wall, order="F")
```

---

## Wiring it into main.py

Update `main.py` to call `generate_dungeon`:

```python
from __future__ import annotations

from pathlib import Path

import tcod

from game.engine import Engine
from game.entity import Entity
from game.input_handlers import EventHandler
from game.map.map_generator import generate_dungeon


def main() -> None:
    screen_width = 80
    screen_height = 50

    map_width = 80
    map_height = 45

    room_max_size = 10
    room_min_size = 6
    max_rooms = 30

    tileset = tcod.tileset.load_tilesheet(
        Path(__file__).parent / "res" / "dejavu10x10_gs_tc.png",
        32,
        8,
        tcod.tileset.CHARMAP_TCOD,
    )

    event_handler = EventHandler()

    player = Entity(x=0, y=0, char="@", color=(255, 255, 255))

    game_map = generate_dungeon(
        max_rooms=max_rooms,
        room_min_size=room_min_size,
        room_max_size=room_max_size,
        map_width=map_width,
        map_height=map_height,
        player=player,
    )

    engine = Engine(
        entities={player},
        event_handler=event_handler,
        game_map=game_map,
        player=player,
    )

    with tcod.context.new(
        columns=screen_width,
        rows=screen_height,
        tileset=tileset,
        title="Roguelike Tutorial",
        vsync=True,
    ) as context:
        console = tcod.console.Console(screen_width, screen_height, order="F")
        engine.run(context, console)


if __name__ == "__main__":
    main()
```

Notice that the player starts at `(0, 0)`, a wall. `generate_dungeon` will move the player to the center of the first room before the game begins. We also removed the `npc` entity since it was just for demonstration.

---

## Testing your work

Run `python main.py` a few times:

- [ ] A different dungeon layout appears every run
- [ ] The player starts inside a room, not inside a wall
- [ ] All rooms are connected by corridors (you can reach every room)
- [ ] Walls block movement as expected
- [ ] The dungeon fits within the map bounds
- [ ] There is no exit yet; descending stairs come in Part 11

!!! bug "What if some rooms are unreachable?"
    Our algorithm connects each room to the *previous* room in the list. As long as no room overlaps (which we check), every placed room is reachable. If you see a floating room with no connection, it is a bug in the intersection check; verify that `intersects` is comparing `x1/x2/y1/y2` correctly.

---

## Summary

We built a rooms-and-corridors dungeon generator in a dedicated `game/map/map_generator.py` module. Key ideas:

- **Separation of concerns**: `GameMap` stores tiles, `map_generator` fills them
- **Wall-first approach**: start with all walls and dig out rooms
- **Intersection testing**: reject rooms that overlap, guaranteeing clean separation
- **L-shaped tunnels**: connecting room centers with a random horizontal-or-vertical-first choice
- **Player placement**: the first valid room becomes the player's starting position

**Current architecture**:

- `main.py`: chooses generation parameters and starts the engine
- `game/map/`: groups tile definitions, map storage, and map generation
- `game/map/map_generator.py`: builds a dungeon and places the player
- `GameMap`: stores the generated tile grid
- `RectangularRoom`: local helper for the rooms-and-corridors algorithm
- `Engine`: runs the already-generated map

**Class Diagram**:

![classes](images/part3_classes.png)

**File structure**:

```txt
main.py                     ← modified
game/
├── __init__.py
├── actions.py
├── engine.py               ← modified
├── entity.py
├── input_handlers.py
└── map/
    ├── __init__.py         ← new
    ├── game_map.py         ← moved (game/game_map.py), modified
    ├── tile_types.py       ← moved (game/tile_types.py)
    └── map_generator.py    ← new
```

---

## Exercises

1. **Reproducible dungeons**:

    Add a `seed` parameter to `generate_dungeon` and call `random.seed(seed)` at the top of the function. With a fixed seed, the dungeon is always the same. This is useful for debugging: if you find a problematic layout, record its seed to reproduce it.

2. **Connect to the *nearest* room instead of the *previous* one**:

    Our current algorithm connects each room to the one placed before it. This sometimes creates long diagonal tunnels. Instead, find the already-placed room whose center is closest to the new room's center and connect to that. The dungeon will look more compact.

3. **Mark visited rooms**:

    Draw a `+` at the center of each room after it is placed (then clear it once the dungeon is done). This lets you see the generation order while debugging. Remove it before moving on.

**Next**: [Part 4: Field of View](part-4.md)
