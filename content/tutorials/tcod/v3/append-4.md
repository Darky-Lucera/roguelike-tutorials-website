# Appendix 4 Draft: Advanced Dungeon Generation Ideas

!!! note "Draft notes"

    This document is a parking place for future ideas. It is not part of the
    main tutorial flow yet, and the details below should be revisited before
    being turned into a finished appendix.

The tutorial currently uses rectangular rooms connected by corridors. This is a
good starting point because it is easy to explain, easy to debug, and produces
maps where the player can understand the layout quickly.

Later, it could be useful to add an appendix about more expressive dungeon
generation techniques. The goal would not be to replace the simple generator in
the main tutorial, but to show how the same codebase could grow into something
with more variety and stronger level design constraints.

## Composite Rooms

One possible extension is to add special rooms made from more than one rectangle.
For example, two overlapping rectangles can produce L-shaped, T-shaped, or
cross-shaped rooms.

This should probably not be implemented as two fully random rectangles. A small
set of heuristics would make the result easier to control:

- Pick a room shape first: rectangle, L, T, or cross.
- Generate the main rectangle.
- Generate one or more secondary rectangles relative to the main rectangle.
- Require a minimum overlap so the final room is a single connected space.
- Avoid very thin arms unless they are intentional.
- Keep the overall room bounds within the map.
- Reject shapes whose floor area is too small or too large for the current depth.

A useful implementation direction would be to introduce a more general room
interface instead of assuming that every room is a single `RectangularRoom`.

For example:

```python
class Room:

    @property
    def center(self) -> tuple[int, int]:
        ...

    def roughly_center(self) -> tuple[int, int]:
        ...

    def intersects(self, other: "Room") -> bool:
        ...

    def carve(self, dungeon: GameMap) -> None:
        ...
```

Then `RectangularRoom` would be one implementation, and a `CompositeRoom` could
contain several rectangles internally:

```python
class CompositeRoom:

    def __init__(self, parts: list[RectangularRoom]):
        self.parts = parts

    def intersects(self, other: Room) -> bool:
        return any(part.intersects(other) for part in self.parts)

    def carve(self, dungeon: GameMap) -> None:
        for part in self.parts:
            part.carve(dungeon)
```

The exact method names may need to change to match the codebase at that point.
The important idea is that the map generator should ask a room what it does,
instead of depending on the room being a single rectangle.

## Dungeon Graphs

Another useful appendix topic would be treating the dungeon as a graph:

- Rooms are nodes.
- Corridors are edges.
- The first room is the start node.
- The exit room can be selected from rooms far away from the start.
- Side branches can be measured by their distance from the main path.

This would make it easier to reason about level structure after generation.
Instead of only carving rooms and corridors, the generator could keep a graph of
what was connected to what.

That graph could answer questions such as:

- Which room is farthest from the start?
- Which rooms are dead ends?
- Which rooms are on the main path?
- Which rooms are optional branches?
- How many steps does the player need to reach the exit?

This information is useful for placing stairs, treasure, keys, monsters, and
special encounters.

## Areas, Keys, and Locks

A larger extension would be to divide a level into areas. Each area could contain
several rooms, and the generator could keep an adjacency matrix or graph
describing which areas connect to each other.

This makes it possible to support key and lock design. For example:

- Area 1 is the starting area.
- Area 5 is locked behind a red door.
- The red key must be placed somewhere in areas 1 to 4.
- The key should preferably be placed away from the shortest path.
- The key room could be guarded by a stronger monster or encounter.

The important constraint is that the generator must never place a key behind the
same lock that requires it. This means the generator needs to understand
reachability, not just room placement.

A possible workflow:

1. Generate the area graph.
2. Decide which connections are locked.
3. For each lock, find the set of areas reachable before opening that lock.
4. Place the key in one of those reachable areas.
5. Prefer rooms far from the main path, dead ends, or optional branches.
6. Validate that the final dungeon can still be completed.

This would be more advanced than the main tutorial, but it connects naturally to
the idea of using graph information after the basic dungeon has been generated.

## Validation

As the generator becomes more expressive, validation becomes more important.
Instead of trusting every generated map, the code should check the final result.

Useful validation rules could include:

- Every room is reachable from the start.
- The exit is reachable.
- No required key is placed behind its own lock.
- Locked areas become reachable after collecting the correct key.
- There is enough walkable space for the player and entities.
- Special rooms do not overlap in invalid ways.

If validation fails, the generator could either repair the map or discard it and
try again. For the tutorial, retrying is easier to explain. Repairing maps could
be a later refinement.

## Other Generation Techniques

Several other techniques could fit in the same appendix or in nearby appendices:

- Binary space partitioning, useful for more structured layouts.
- Cellular automata, useful for caves and organic areas.
- Drunkard walk generation, useful for winding natural caves.
- Prefab rooms or vaults, useful for authored encounters.
- Themed areas, where room contents and tile types vary by region.
- Corridor styles, such as straight, winding, wide, or decorated corridors.
- Post-processing passes, such as adding loops, secret rooms, or shortcuts.

These techniques should be introduced as optional tools. The main tutorial should
keep the core generator understandable, while appendices can show how to build
more ambitious systems on top of it.

## Possible Appendix Shape

A future finished appendix could be organized like this:

1. Explain why the basic rectangular-room generator is limited.
2. Introduce a generic room interface.
3. Add composite rooms using overlapping rectangles.
4. Update intersection and carving logic.
5. Keep a dungeon graph while connecting rooms.
6. Use graph distance to place stairs and optional rewards.
7. Introduce areas and locked connections.
8. Place keys using reachability checks.
9. Validate the generated level.

This should remain optional content. It is valuable because it shows how dungeon
generation becomes a design tool, not only a random map algorithm.
