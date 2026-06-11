# Appendix 3: Consumable Scaling

## Concept

Many consumables can become more interesting when they scale with a `level` parameter rather than fixed stats. This appendix covers the general pattern and works through two concrete examples: the Mapping Scroll and the Lightning Scroll.

!!! question "Why not a separate class for each tier?"
    You could create `WeakLightningScroll`, `StrongLightningScroll`, and so on. That works for two or three tiers, but becomes unwieldy as the number of scrolls and tiers grows. A single class with a `level` parameter keeps all the logic in one place and lets the item factory express difficulty through data rather than code. The trade-off: `activate()` grows slightly more complex. The payoff: adding a new tier means changing one number in the factory, not writing a new class.

---

## The level parameter pattern

Instead of separate consumable classes for each power tier, pass a `level: int` to `__init__` and let it control behaviour:

```python
mapping_scroll_1 = Item(consumable=MappingConsumable(level=1), ...)
mapping_scroll_4 = Item(consumable=MappingConsumable(level=4), ...)
```

Each level adds or enhances something. The factory defines how rare each tier is via `item_chances`.

A minimal skeleton looks like this:

```python
class LightningDamageConsumable(Consumable):

    def __init__(self, level: int = 1) -> None:
        self.level = level

    def activate(self, action: ItemAction) -> None:
        # behaviour branches on self.level
        ...
```

!!! tip "Python: default parameter values"
    `level: int = 1` means the argument is optional. `LightningDamageConsumable()` and `LightningDamageConsumable(level=1)` are equivalent. Using a default of `1` keeps the common case short while still letting the factory pass higher tiers explicitly.

!!! info "Connecting level to dungeon depth"
    A `level` parameter is most useful when the item factory controls which tier appears on which dungeon floor. Part 12 turns `item_chances` into a floor-keyed table: each item template maps to `(floor, weight)` pairs. A `MappingConsumable(level=4)` would be configured to appear only from floor 6 onward, while `level=1` is available from the start. This way, the difficulty curve in the factory data automatically selects which tier the player finds, with no extra logic inside the consumable itself.

---

## Mapping Scroll levels

| Level | Reveals | Duration |
| ----- | ------- | -------- |
| 1 | Floor tiles only | Permanent |
| 2 | Floor + surrounding walls (`mapped_tiles`) | Permanent |
| 3 | + Items (`stays_visible = True` on all items) | Permanent |
| 4 | + Monsters (clairvoyance) | N turns |

A level-checking `activate()` reads naturally as a series of `if self.level >= N` guards. Each guard adds the next tier of effect on top of all previous ones:

```python
def activate(self, action: ItemAction) -> None:
    game_map = action.engine.game_map

    # Level 1: reveal all walkable (floor) tiles permanently
    game_map.explored |= game_map.tiles["walkable"]

    if self.level >= 2:
        # Level 2: also reveal the walls that border those floor tiles
        for x, y in mapped_tiles(game_map):
            game_map.explored[x, y] = True

    if self.level >= 3:
        # Level 3: items remain visible even outside the player's FOV
        for item in game_map.items:
            item.stays_visible = True

    if self.level >= 4:
        # Level 4: monsters are also visible, but only for a limited time
        turns = 10  # move to a named constant when you implement this
        for actor in game_map.actors:
            if actor is not action.engine.player:
                actor.stays_visible = True
                actor.stays_visible_turns = turns
```

The `>=` pattern means you never repeat lower-level code: level 4 implicitly includes all of levels 1 through 3.

### Temporary visibility: `stays_visible_turns`

For level 4, monster visibility expires after N turns. The mechanism:

- Add `stays_visible_turns: int = -1` to `Entity` (`-1` = permanent, `N` = countdown).
- Each full turn cycle, the engine decrements all entities with `stays_visible_turns > 0`. When it reaches `0`, set `entity.stays_visible = False`.
- `MappingConsumable` sets `stays_visible = True` and `stays_visible_turns = N` on the relevant entities.

The engine-side tick that drives the countdown looks like this:

```python
# called once per full turn cycle, after all actors have moved
for entity in self.game_map.entities:
    if entity.stays_visible_turns > 0:
        entity.stays_visible_turns -= 1
        if entity.stays_visible_turns == 0:
            entity.stays_visible = False
```

This two-dimensional scaling (what is revealed x how long) can be reused by other spells.

---

## Lightning Scroll levels: chain lightning

At level 1 the scroll hits the nearest visible enemy. Higher levels add chain jumps:

- Each jump targets the nearest actor to the previous target (excluding already-hit actors).
- Each jump deals a fraction of the previous hit (e.g. 75%), making it strong against groups but weaker against a single tough enemy: a meaningful tactical trade-off.

| Level | Targets | Damage per jump    |
| ----- | ------- | ------------------ |
| 1     | 1       | 100%               |
| 2     | 2       | 100% → 75%         |
| 3     | 3       | 100% → 75% → 56%  |

The inner loop mirrors this table directly:

```python
def activate(self, action: ItemAction) -> None:
    hit: list[Actor] = []
    current = find_nearest_visible_actor(action.engine, exclude=hit)
    damage: float = self.base_damage  # e.g. 20.0

    for _ in range(self.level):
        if current is None:
            break
        current.fighter.take_damage(damage)
        hit.append(current)
        damage *= 0.75  # each jump carries 75% of the previous hit
        current = find_nearest_to(current, action.engine, exclude=hit)
```

!!! tip "Why keep damage as `float`?"
    Storing damage as `float` lets the 75% factor propagate without rounding errors between jumps. If you truncated to `int` at each step, a 20-damage bolt would jump as 20 -> 15 -> 11 -> 8, losing a total of 2 damage to accumulated rounding. With floats the chain stays precise: 20.0 -> 15.0 -> 11.25 -> 8.4375. The display layer or the hit resolution (e.g. `hp -= int(damage)`) can round once at the point of application, not repeatedly through the chain.

!!! tip "Geometric decay"
    Multiplying by `0.75` repeatedly is a **geometric series**: the jump sequence is 100%, 75%, 56.25%, 42.2%, ... It converges toward zero quickly, which means even a very high `level` scroll will not chain forever in a useful way. You can tune the feel by choosing a different factor: `0.5` (fast decay, good for burst) or `0.9` (slow decay, dangerous against large groups). The factor becomes a good named constant once you decide on a value.

The jump range can be a fixed constant (e.g. 5 tiles) or scale with level as well.

---

## Current consumables

| Item | Class | Introduced |
| --- | --- | --- |
| Health Potion | `HealingConsumable` | Part 7 |
| Backpack Growing Scroll | `BackpackConsumable` | Part 8, Ex. 2 |
| Lightning Scroll | `LightningDamageConsumable` | Part 9 |
| Confusion Scroll | `ConfusionConsumable` | Part 9 |
| Fireball Scroll | `FireballDamageConsumable` | Part 9 |
| Mapping Scroll | `MappingConsumable` | Part 9, Ex. 1 |
| Drain Scroll | `DrainConsumable` | Part 9, Ex. 2 |
| Teleport Scroll | `TeleportConsumable` | Part 9, Ex. 3 |

The Chest (`TreasureConsumable`) is auto-collected and has no active effect, so it is excluded from scaling considerations.

---

## Scaling ideas per consumable

- **Health Potion**: more HP healed per level.
- **Backpack Scroll**: more slots added per level.
- **Lightning Scroll**: chain jumps per level (see above).
- **Confusion Scroll**: more turns of confusion per level.
- **Fireball Scroll**: more damage and/or larger radius per level.
- **Mapping Scroll**: broader reveal + temporary clairvoyance per level (see above).
- **Drain Scroll**: more damage/heal per level; at higher levels could drain without range restriction.
- **Teleport Scroll**: at higher levels could teleport any visible actor, not just the player.

---

## Teleport Scroll: two-phase targeting (state chaining)

Instead of always teleporting the player, a higher-level teleport scroll could add a two-phase targeting flow. In the first phase the player picks an actor (themselves or any visible enemy). In the second phase they pick the destination tile. The selected actor is then moved there.

This demonstrates state chaining: one `SelectIndexState` can hand off to another before the action fires. The pattern generalises to any multi-step targeting flow.

!!! tip "State chaining as a general pattern"
    Any time a consumable needs more than one piece of input (pick a target, then pick a tile; confirm a spell, then confirm the area), the same chain pattern applies. Each state gathers one input and, instead of returning an action immediately, returns a new state with the partial result. The final state in the chain produces the action. Keeping each state responsible for exactly one decision makes the logic easy to follow and straightforward to test in isolation.

Implementation hints:

- Create a `SelectActorState(SelectIndexState)` that validates the chosen tile has an actor. On confirm, instead of returning an action, it returns a new `SelectDestinationState` with the chosen actor.
- `SelectDestinationState(SelectIndexState)` receives the actor as a constructor parameter. On confirm it returns a `TeleportAction(actor, x, y)`.

---

## Summary

Scaling consumables with a `level` parameter keeps your item classes small while letting the factory express dungeon difficulty through data. The two patterns shown here, multi-dimensional reveal (what is shown x for how long) and chain damage with geometric decay, are reusable templates for any consumable that needs graduated strength. State chaining in the Teleport example generalises to any multi-step targeting flow across the entire spell system.
