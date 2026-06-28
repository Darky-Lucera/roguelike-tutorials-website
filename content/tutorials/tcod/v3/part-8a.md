# Part 8a: The Item System

Part 8 is the longest and densest chapter in this tutorial. It introduces more moving parts at once than any earlier chapter: a new ownership model, a structured rejection pattern, two new entity subclasses, two consumable components, and a new entity type.

Part 8a builds the data model: items exist in the dungeon, can be seen, but cannot yet be touched. Part 8b adds the player-facing side: actions, keyboard bindings, and the inventory overlay.

---

## What You Will Build

By the end of Part 8a, health potions and chests appear in the dungeon. Hovering the mouse over either shows its name. Walking over a chest does nothing yet: the auto-collect mechanic will be added in Part 8b. There are no other keyboard interactions with items yet either.

## Learning goals

- Add `Item` as a new entity subclass with an `owner` field
- Introduce `Impossible` as a structured rejection pattern
- Implement `Inventory`, `HealingConsumable`, and `TreasureConsumable` as components
- Narrow base component types so annotations match the actual runtime type
- Spawn health potions and chests using a weighted item table

---

## Where does an item live?

Before writing any code, consider the central design question: a health potion needs to exist in two places.

When it is on the dungeon floor, the map owns it. When the player picks it up, the inventory owns it. It is the same Python object in both cases; only its *owner* changes. How does the game track which owner currently holds it?

The answer is an `owner` field on `Entity`. Every entity stores a reference to its current owner:

```text
On the dungeon floor:
  Item
  └── owner = GameMap

In the player's inventory:
  Item
  └── owner = Inventory
```

In both cases the item is the same Python object, only `owner` changes. The field is the single source of truth for where an entity is: on the floor or in an inventory.

This chapter also introduces a second architectural change: **important action rejections should say why**. The old approach was to return silently (or return `None`). The new approach raises an `Impossible` exception for cases where the player needs feedback, with the reason as its message.

---

## `game/exceptions.py`

Create a new file:

```python
from __future__ import annotations


class Impossible(Exception):
    """Exception raised when an action is impossible. The reason is the exception message."""
```

!!! info "Exceptions as control flow"
    Python uses exceptions for unexpected bugs, but also for *expected* failures: requesting an impossible action, reading past the end of a file, looking up a missing key. Raising `Impossible` is not a bug, it is a structured way to carry a rejection reason out of any call depth without threading a return value through every intermediate function.

Compare the alternatives. Returning `None` is silent: the caller has to check for it and decide what to tell the player. Returning a `bool` is only marginally better: the caller knows the action failed, but not why. Raising `Impossible("Your inventory is full.")` gives the reason for free.

---

## New constants

Part 8a introduces the visual constants for both item types: the health potion, the chest, and the gold color. The colors used for action rejections and the inventory overlay will be introduced in Part 8b, at the point where they are first used.

In `game/constants/sprites.py`, add both item sprites. `CHEST` belongs in the entity section (before `CORPSE`); `HEALTH_POTION` opens a new items section below:

```diff
+CHEST   = "$"
 CORPSE  = "%"
+
+# Items
+HEALTH_POTION = "!"
```

In `game/constants/colors.py`, add `CHEST` in the entity colors section (between `TROLL` and `CORPSE`) and `HEALTH_POTION` in a new item colors section right below:

```diff
 TROLL            = Color(  0, 127,   0)
+CHEST            = Color(255, 240,   0)
 CORPSE           = Color(191,   0,   0)
+
+# Item colors
+HEALTH_POTION    = Color(127, 0, 255)
```

Then add `HEALTH_RECOVERED` and `GOLD` alongside the other message colors:

```diff
 ENEMY_DEATH      = Color(0xFF, 0xA0, 0x30)
+HEALTH_RECOVERED = Color(0x00, 0xFF, 0x00)
+GOLD             = Color(0xFF, 0xD7, 0x00)
```

`HEALTH_RECOVERED` is bright green for HP-restore messages. `GOLD` is used by `TreasureConsumable.activate()` and later by the gold counter in the HUD.

!!! note "If you completed the Part 5 chest exercise"
    `sprites.CHEST` and `colors.CHEST` may already be defined. Keep the existing definitions.

---

## Narrowing component types

Every component in `game/entities/components/base_component.py` declares `entity: Entity`. That annotation is incorrect. A `Fighter` component's entity is always an `Actor`, it will never be a plain `Entity` or an `Item`. Once this chapter moves `ai` off `Entity`, `entity: Entity` no longer describes the design accurately: `self.entity.ai` and `self.entity.is_alive` are `Actor` attributes, and `Entity` does not declare them.

!!! question "What do we gain by narrowing?"
    After `ai` moves to `Actor`, the annotation `entity: Entity` is incomplete: `Entity` does not have `ai`, but `Fighter` accesses `self.entity.ai`. The attribute exists at runtime because `Fighter` is always attached to an `Actor`, but the declared type does not capture that invariant. Narrowing to `entity: Actor` makes the annotation accurate, and a type checker can verify the access correctly as a result.

By introducing `ActorComponent` (where `entity: Actor`) and `ItemComponent` (where `entity: Item`), we make the annotations correct: certain components are *always* actor components, others are *always* item components. The annotation narrows from `Entity` to the actual runtime type, so the declared type finally matches the design.

Update `game/entities/components/base_component.py`:

```diff
 from __future__ import annotations

 from typing import TYPE_CHECKING

 if TYPE_CHECKING:
-    from game.entities.entity import Entity
+    from game.entities.entity import Actor, Item


 class BaseComponent:
-    entity: Entity  # set by the owning Entity after creation
+    pass


+class ActorComponent(BaseComponent):
+    entity: Actor
+
+
+class ItemComponent(BaseComponent):
+    entity: Item
```

!!! info "Why keep `BaseComponent` at all?"
    `BaseComponent` has no `entity` annotation because the type of `entity` depends on whether the component belongs to an `Actor` or an `Item`: it is not meaningful at this level. `ActorComponent` and `ItemComponent` carry the correct declarations. `BaseComponent` is kept as their common ancestor in case future code needs to refer to "any component" generically (a debug inspector, a serialization layer, or a plugin system). If your project never needs that, you can remove `BaseComponent` entirely and make `ActorComponent` and `ItemComponent` independent classes.

Now update each existing component to use the correct base class. In `game/entities/components/fighter.py`:

```diff
-from game.entities.components.base_component import BaseComponent
+from game.entities.components.base_component import ActorComponent

 ...

-class Fighter(BaseComponent):
+class Fighter(ActorComponent):
```

In `game/entities/components/ai.py`:

```diff
-from game.entities.components.base_component import BaseComponent
+from game.entities.components.base_component import ActorComponent

 ...

-class BaseAI(BaseComponent):
+class BaseAI(ActorComponent):
```

`ItemComponent` declares `entity: Item`, but `Item` does not exist yet in `entity.py`. Add a one-line stub now so the annotation refers to a real class from the start:

```diff
+class Item(Entity):
+    pass
```

**Note**: Step 6 in the next section will replace this stub with the full implementation.

The two new components introduced in this chapter (`Inventory` and `Consumable`) will use the correct bases from the start.

---

## Updating `game/entities/entity.py`

`entity.py` changes in several steps. Each step introduces one concept before the next one depends on it.

### Step 1: Remove `ai` from `Entity` and expand TYPE_CHECKING imports

`ai` is an `Actor` concern, not an `Entity` concern. Plain entities (passive blockers, map decorations) never have AI. Keeping it on the base class was a holdover from before `Actor` existed.

At the same time, add the imports that the new classes and annotations in this chapter require:

```diff
 if TYPE_CHECKING:
     from game.entities.components.ai import BaseAI
+    from game.entities.components.consumable import Consumable
+    from game.entities.components.inventory import Inventory
     from game.map.game_map import GameMap


 class Entity:

     def __init__(
         self,
         x: int = 0,
         ...
         render_order: RenderOrder = RenderOrder.UNKNOWN,
-        ai: BaseAI | None = None,
     ) -> None:
         self.x               = x
         ...
         self.render_order    = render_order
-        self.ai              = ai
```

### Step 2: Add the `owner` field

Add `owner` as the first parameter of `__init__`, and auto-register the entity when an owner is provided:

```diff
 class Entity:
+    owner: GameMap | Inventory | None
+
     def __init__(
         self,
+        owner: GameMap | None = None,
         x: int = 0,
         y: int = 0,
         ...
         render_order: RenderOrder = RenderOrder.UNKNOWN,
     ) -> None:
+        self.owner           = owner
         self.x               = x
         ...
         self.render_order    = render_order
+        if owner is not None:
+            owner.entities.add(self)
```

Every entity tracks its owner. The class-level annotation covers all values the field holds over its lifetime, while the `__init__` parameter is intentionally narrower.

!!! tip "Class-level annotation vs `__init__` parameter"
    Declare the field's full type at class level and use a narrower `__init__` parameter for the valid *initial* states only. The class-level annotation is authoritative: a type checker respects it and will not flag later reassignments to `Inventory`. This pattern is useful whenever an attribute can legitimately change type over its lifetime but only starts in a subset of those states.

The constructor parameter is `GameMap | None`: entities start on the map or unowned. Item *templates* (defined in `game/entities/factories.py`) are created with no owner. When `spawn()` places a clone on the floor, the clone gets `owner = dungeon`. When `PickupAction` picks it up, `item.owner` changes to `inventory`. The class-level annotation covers all three runtime states.

!!! info "The three ownership states"
    - **Template**: `entity.owner = None`

        `None` means the entity currently has no owner. At construction time, this is how factory templates are represented.

    - **On the map**: `entity.owner = GameMap`

        `GameMap` means it has just been spawned.

    - **In inventory**: `entity.owner = Inventory`

        `Inventory` is only set later, on pickup, so accepting it at construction time would be misleading.

### Step 3: Update `spawn()`

`spawn()` existed since Part 5. It creates a deep copy and adds it to the map's entity set. Now it also sets `owner` on the clone:

```diff
     def spawn(self, game_map: GameMap, x: int, y: int) -> Entity:
         clone = copy.deepcopy(self)
         clone.x     = x
         clone.y     = y
+        clone.owner = game_map
         game_map.entities.add(clone)
         return clone
```

### Step 4: Add `place()`

`place()` moves an entity to a new position and optionally transfers it to a new owner. The inventory `drop()` method calls it to return an item to the dungeon floor:

```diff
+    def place(self, x: int, y: int, game_map: GameMap | None = None) -> None:
+        from game.map.game_map import GameMap
+
+        self.x = x
+        self.y = y
+        if game_map is not None:
+            if self.owner is not None:
+                if isinstance(self.owner, GameMap):
+                    self.owner.entities.discard(self)
+
+            self.owner = game_map
+            game_map.entities.add(self)
+
     def set_position(self, x: int, y: int) -> None:
```

The local import makes `GameMap` available at runtime without introducing a module-level circular import. `self.owner is not None` guards against placing a fresh template entity for the first time. Once that check passes, `isinstance(self.owner, GameMap)` determines whether the entity is registered directly with the map and needs to be removed from it: items being dropped from inventory have `owner = Inventory`, so the isinstance check is `False` and the discard is skipped correctly.

### Step 5: Add `inventory` to `Actor` and fix the `ai` annotation

`Actor` now requires an `Inventory` component. This step also cleans up the `ai` wiring: since `Entity` no longer accepts `ai`, `Actor` must own the attribute directly.

Update `Actor.__init__`:

```diff
     def __init__(
         self,
         *,
         x: int              = 0,
         y: int              = 0,
         char: str           = sprites.UNKNOWN,
         color: Color        = colors.DEFAULT_FG,
         name: str           = "<unnamed>",
         ai: BaseAI | None   = None,
         fighter: Fighter,
+        inventory: Inventory,
     ) -> None:
         super().__init__(
             x               = x,
             y               = y,
             char            = char,
             color           = color,
             name            = name,
             blocks_movement = True,
             render_order    = RenderOrder.ACTOR,
-            ai              = ai,
         )
         self.fighter = fighter
         self.fighter.entity = self

-        if self.ai:
-            self.ai.entity = self
+        self.inventory = inventory
+        self.inventory.entity = self
+
+        self.ai = ai
+        if self.ai is not None:
+            self.ai.entity = self
```

`ai=ai` is no longer passed to `super().__init__()` because `Entity` no longer accepts it.

### Step 6: Add the `Item` class

`Item` is parallel to `Actor`: a specialised entity with its own required component. Replace the stub with the full class:

```diff
-class Item(Entity):
-    pass
+class Item(Entity):
+
+    def __init__(
+        self,
+        *,
+        x: int      = 0,
+        y: int      = 0,
+        char: str   = sprites.UNKNOWN,
+        color: Color = colors.DEFAULT_FG,
+        name: str   = "<unnamed>",
+        consumable: Consumable,
+    ) -> None:
+        super().__init__(
+            x               = x,
+            y               = y,
+            char            = char,
+            color           = color,
+            name            = name,
+            blocks_movement = False,
+            render_order    = RenderOrder.ITEM,
+        )
+        self.consumable = consumable
+        self.consumable.entity = self
```

Items do not usually block movement (you can stand on top of a potion) and render at `RenderOrder.ITEM`, below actors and above the floor.

---

## Add `items` to `GameMap`

Items on the map need a way to be found. Add a filtered property to `game/map/game_map.py` that yields only `Item` instances:

```diff
 if TYPE_CHECKING:
-    from game.entities.entity import Actor, Entity
+    from game.entities.entity import Actor, Entity, Item

 ...

     @property
     def actors(self) -> Iterator[Actor]:
         ...

+    @property
+    def items(self) -> Iterator[Item]:
+        from game.entities.entity import Item
+
+        yield from (e for e in self.entities if isinstance(e, Item))
```

---

## Create `game/entities/components/inventory.py`

```python
from __future__ import annotations

from typing import TYPE_CHECKING

from game.entities.components.base_component import ActorComponent

if TYPE_CHECKING:
    from game.entities.entity import Item
    from game.map.game_map import GameMap


class Inventory(ActorComponent):

    def __init__(self, capacity: int) -> None:
        self.capacity = capacity
        self.items: list[Item] = []
        self.gold: int = 0

    def add_item(self, item: Item, game_map: GameMap | None) -> bool:
        if len(self.items) >= self.capacity:
            return False

        if game_map is not None:
            game_map.entities.discard(item)

        item.owner = self
        self.items.append(item)

        return True

    def drop_item(self, item: Item, game_map: GameMap) -> None:
        self.items.remove(item)
        item.place(self.entity.x, self.entity.y, game_map)
```

`self.gold: int = 0` stores the player's accumulated treasure. Gold is something the player carries, so it lives here alongside the item list rather than as a bare attribute on `Actor`.

`add_item()` checks capacity, transfers ownership to the inventory, and appends the item to `items`. When the item comes from the dungeon floor, the caller passes the current `game_map`, and `add_item()` removes it from the map's entity set via `game_map.entities.discard()`. When the item was never placed on a map, for example a starting item, the caller can pass `None`.

`game_map` accepts `None`, but it does not have a default value. This makes each call state its intent explicitly: `inventory.add_item(item, engine.game_map)` means "take this item from the map", while `inventory.add_item(item, None)` means "add an item that is not on any map". That explicit argument helps avoid accidentally leaving a floor item both in the map and in the inventory. The method returns `False` if the inventory is full so that the caller can raise `Impossible` with a reason.

`drop_item()` removes the item from `items`, then calls `item.place()` to move it back to the dungeon floor at the actor's current position. The caller passes `game_map` explicitly. In Part 8b, `DropItem.perform` will already have `engine` and can supply `engine.game_map` directly, so `Inventory` does not need to navigate the ownership chain itself.

---

## Create `game/entities/components/consumable.py`

```python
from __future__ import annotations

from typing import TYPE_CHECKING

from game.constants import colors
from game.entities.components.base_component import ItemComponent
from game.exceptions import Impossible
from game.message_log import MessageLog

if TYPE_CHECKING:
    from game.actions import Action
    from game.engine import Engine
    from game.entities.entity import Actor


class Consumable(ItemComponent):
    auto_activate: bool = False

    def activate(self, _action: Action, _engine: Engine, _consumer: Actor) -> None:
        """Apply this consumable's effect. Must be overridden."""
        raise NotImplementedError()

    def consume(self) -> None:
        """Remove the item from the holder's inventory."""
        from game.entities.components.inventory import Inventory

        entity = self.entity
        inventory = entity.owner
        if isinstance(inventory, Inventory) and entity in inventory.items:
            inventory.items.remove(entity)
            entity.owner = None


class HealingConsumable(Consumable):

    def __init__(self, amount: int) -> None:
        self.amount = amount

    def activate(self, _action: Action, _engine: Engine, consumer: Actor) -> None:
        amount_recovered = consumer.fighter.heal(self.amount)

        if amount_recovered > 0:
            MessageLog.add_message(
                f"You consume the {self.entity.name}, and recover {amount_recovered} HP!",
                colors.HEALTH_RECOVERED,
            )
            self.consume()

        else:
            raise Impossible("Your health is already full.")


class TreasureConsumable(Consumable):
    auto_activate = True

    def __init__(self, value: int) -> None:
        self.value = value

    def activate(self, _action: Action, engine: Engine, consumer: Actor) -> None:
        consumer.inventory.gold += self.value
        MessageLog.add_message(
            f"You found {self.value} gold!",
            colors.GOLD,
        )
        engine.game_map.entities.discard(self.entity)
        self.entity.owner = None
```

The file defines three classes: `Consumable` as the base for all item effects, `HealingConsumable` for HP recovery, and `TreasureConsumable` for treasure collected automatically on contact.

`activate()` is declared with `_action: Action` rather than the more specific `ItemAction`, because `ItemAction` does not exist yet. Using the base type keeps this file free of forward references.

`auto_activate: bool = False` is a class-level flag. `TreasureConsumable` overrides it to `True`. The flag exists but nothing reads it yet: chests spawn on the floor and stay there until Part 8b adds the collection logic.

`consume()` imports `Inventory` locally to avoid a circular import: `consumable.py` and `inventory.py` would otherwise form a cycle at module level. The `isinstance` check serves double duty: it guards against consuming an unowned item and narrows the declared type from `GameMap | Inventory | None` to `Inventory`, making the subsequent `inventory.items` access well-typed. After removing the item, `entity.owner = None` clears the stale reference so the consumed item no longer points at the inventory it came from.

`HealingConsumable.activate()` calls `fighter.heal()`, which you wrote in Part 7. If the player is already at full health, `heal()` returns `0` and `activate()` raises `Impossible`.

`TreasureConsumable.activate()` adds `value` to `consumer.inventory.gold`, logs the find, and removes the chest from the map in the same call. There is no `self.consume()` here because `consume()` removes an item from an inventory; the chest was never in one.

---

## Update `game/entities/factories.py`

Every `Actor` now requires an `Inventory`. Add the component to the existing templates, and also import both consumable classes and `Item`:

```diff
 from game.constants import colors, sprites
 from game.entities.components.ai import HostileEnemy
+from game.entities.components.consumable import HealingConsumable, TreasureConsumable
 from game.entities.components.fighter import Fighter
+from game.entities.components.inventory import Inventory
-from game.entities.entity import Actor, Entity
+from game.entities.entity import Actor, Item
```

```diff
 player = Actor(
     char      = sprites.PLAYER,
     color     = colors.PLAYER,
     name      = "Player",
     ai        = None,
     fighter   = Fighter(hp=30, defense=2, attack=5),
+    inventory = Inventory(capacity=10),
 )

 orc = Actor(
     char      = sprites.ORC,
     color     = colors.ORC,
     name      = "Orc",
     ai        = HostileEnemy(),
     fighter   = Fighter(hp=18, defense=1, attack=4),
+    inventory = Inventory(capacity=0),
 )

 troll = Actor(
     char      = sprites.TROLL,
     color     = colors.TROLL,
     name      = "Troll",
     ai        = HostileEnemy(),
     fighter   = Fighter(hp=12, defense=0, attack=3),
+    inventory = Inventory(capacity=0),
 )
```

The player starts with `capacity=10`. 10 makes a full inventory a real constraint worth managing. Enemies get `capacity=0`: their inventory exists (so the type is satisfied) but they cannot hold any items. An orc that walks over a potion will not pick it up.

Then add both item templates and the spawn table at the bottom of the file:

```python
# Items
health_potion = Item(
    char       = sprites.HEALTH_POTION,
    color      = colors.HEALTH_POTION,
    name       = "Health Potion",
    consumable = HealingConsumable(amount=4),
)

chest = Item(
    char       = sprites.CHEST,
    color      = colors.CHEST,
    name       = "Chest",
    consumable = TreasureConsumable(value=10),
)

item_chances = [
    (health_potion, 40),
    (chest,         60),
]
```

`item_chances` mirrors the `monster_chances` pattern from Part 5: a list of `(template, weight)` pairs that `map_generator.py` uses to select which item to spawn. Weights are relative: a chest (60) is created more often than a potion (40). Exercise 2 will add `backpack_scroll` to this list.

!!! note "If you completed the Part 5 chest exercise"
    Replace the passive `Entity` definition with this `Item` definition, and remove any old spawn logic or spawn table entry you added for it. The `item_chances` table above supersedes that.

---

## Update `game/map/map_generator.py`

In Part 5, `place_entities` accepted a single monster limit. Part 8 adds **item** spawning, so the function receives both monster and item limits.

!!! tip "If you completed Exercise 1 from Part 5"
    That exercise added `min_monsters` to `place_entities`. The diffs below assume it is already there. If you skipped it, add `min_monsters: int` alongside `max_monsters` and replace `max_monsters` with `random.randint(min_monsters, max_monsters)` while you are here.

!!! tip "If you skipped Exercise 2 from Part 5"
    The monster loop below assumes the weighted table from that exercise: `factories.monster_chances`, split into `monster_templates` and `monster_weights`, then selected with `random.choices`. If your code still has a hardcoded orc/troll `if/else`, replace it with the `random.choices` version here. It is the version future spawn tables build on.

Expand the function signature:

```diff
 def place_entities(
     room: RectangularRoom,
     dungeon: GameMap,
     min_monsters: int,
     max_monsters: int,
+    min_items: int,
+    max_items: int,
 ) -> None:
     number_of_monsters = random.randint(min_monsters, max_monsters)
+    number_of_items    = random.randint(min_items, max_items)
```

Then add the item spawning loop at the end of the function body. The diff also renames the `monster` variable to `monsters` and moves the `[0]` index onto its own line so the comment reads on its own line:

```diff
        if not any(entity.x == x and entity.y == y for entity in dungeon.entities):
-            monster = random.choices(
+            monsters = random.choices(
                 monster_templates,
                 weights = monster_weights,
-                k       = 1,
-            )[0]
-            monster.spawn(dungeon, x, y)
+                k       = 1,
+            )
+            # First element (because random.choices returns a list)
+            monsters[0].spawn(dungeon, x, y)
+
+    item_templates, item_weights = zip(*factories.item_chances)
+    for _ in range(number_of_items):
+        x = random.randint(room.x1 + 1, room.x2 - 1)
+        y = random.randint(room.y1 + 1, room.y2 - 1)
+        if not any(entity.x == x and entity.y == y for entity in dungeon.entities):
+            items = random.choices(
+                item_templates,
+                weights=item_weights,
+                k       = 1,
+            )
+            # First element (because random.choices returns a list)
+            items[0].spawn(dungeon, x, y)
```

Also expand the signature of `generate_dungeon` itself to accept the item parameters:

```diff
 def generate_dungeon(
     max_rooms: int,
     room_min_size: int,
     room_max_size: int,
     map_width: int,
     map_height: int,
     min_monsters_per_room: int,
     max_monsters_per_room: int,
+    min_items_per_room: int,
+    max_items_per_room: int,
     player: Entity,
     seed: int,
 ) -> GameMap:
```

While in `generate_dungeon`, update the first-room branch to use `place()` instead of `set_position()`:

```diff
         if not rooms:
-            player.set_position(*new_room.center)
+            player.place(*new_room.center, dungeon)
```

`place()` sets `player.owner = dungeon`. Without this, the player would be present in `dungeon.entities` but still have `owner = None`, which would make the ownership state inconsistent from the first map onward.

Update the call site in `generate_dungeon`:

```diff
             place_entities(
                 new_room,
                 dungeon,
                 min_monsters_per_room,
                 max_monsters_per_room,
+                min_items_per_room,
+                max_items_per_room,
             )
```

---

## Update `main.py`

Add the item density parameters alongside the monster parameters:

```diff
     min_monsters_per_room = 0
     max_monsters_per_room = 2
+    min_items_per_room    = 0
+    max_items_per_room    = 2
```

Pass them to `generate_dungeon`:

```diff
     game_map = generate_dungeon(
         ...
         min_monsters_per_room = min_monsters_per_room,
         max_monsters_per_room = max_monsters_per_room,
+        min_items_per_room    = min_items_per_room,
+        max_items_per_room    = max_items_per_room,
         player                = player,
         seed                  = seed,
     )
```

---

## Testing Part 8a

Run the game and verify the following:

- Health potions (`!`) and chests (`$`) appear on the dungeon floor.
- Moving the mouse over a potion shows "Health Potion" in the status bar; moving it over a chest shows "Chest".
- Walking over a chest does nothing yet. Auto-collect will be added in Part 8b.
- No keyboard interactions with items exist yet. The `G`, `I`, and `D` keys will be added in Part 8b.

If both item types appear and hover names work, Part 8a is complete.

---
