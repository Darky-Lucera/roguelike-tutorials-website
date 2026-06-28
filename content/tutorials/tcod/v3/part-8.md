---
hide:
  - footer
---

# Part 8: Items and Inventory

## What You Will Build

By the end of this part, the dungeon will contain items the player can pick up, carry in an inventory, use for healing, and drop back onto the map. Treasure chests will also become auto-collected items that reward the player with gold.

## Learning goals

- Add `Item` as a new entity class and `HealingConsumable` as its first component
- Give every entity an `owner` field so items know whether they are on the floor or in an inventory
- Add an `Inventory` component to actors and implement pickup, use, and drop actions
- Add `TreasureConsumable` for chests collected automatically on contact
- Raise `Impossible` for action rejections that need player-facing feedback
- Build a letter-keyed inventory overlay using the modal state pattern
- Spawn health potions and treasure chests in the dungeon

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

This chapter also introduces a second architectural change: **important action rejections should say why**. The old approach was to return silently (or return `None`). The new approach raises an `Impossible` exception for cases where the player needs feedback, with the reason as its message. A single `try/except` in the game state intercepts every `Impossible` rejection and shows it in the message log.

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

Compare the alternatives. Returning `None` is silent: the caller has to check for it and decide what to tell the player. Returning a `bool` is only marginally better: the caller knows the action failed, but not why. Raising `Impossible("Your inventory is full.")` gives the reason for free, and a single `except Impossible as ex:` in the event handler catches every `Impossible` case with `str(ex)`.

---

## New constants

This chapter uses visuals for health potions and chests, plus colors for item messages and the inventory overlays. Add them to the constants files.

In `game/constants/sprites.py`, make sure `CHEST` exists in the entity section, then add `HEALTH_POTION` below it:

```diff
+CHEST   = "$"
 CORPSE  = "%"
+
+# Items
+HEALTH_POTION = "!"
```

In `game/constants/colors.py`, make sure `CHEST` exists in the entity colors section, then add `HEALTH_POTION` below the entity colors:

```diff
 TROLL             = Color(  0, 127,   0)
+CHEST             = Color(255, 240,   0)
 CORPSE            = Color(191,   0,   0)
+
+# Item colors
+HEALTH_POTION     = Color(127, 0, 255)
```

Then add the HP-restore message color near the other UI colors:

```diff
 ENEMY_DEATH       = Color(0xFF, 0xA0, 0x30)
+HEALTH_RECOVERED  = Color(0x00, 0xFF, 0x00)
```

`HEALTH_RECOVERED` is bright green for HP-restore messages. The action-rejection color (`INVALID`) and the inventory overlay palette are added later in this chapter, each at the point where it is first used.

!!! note "If you completed the Part 5 chest exercise"
    `sprites.CHEST` and `colors.CHEST` may already be defined. Keep the existing definitions instead of adding duplicates.

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

Step 6 in the next section replaces this stub with the full implementation.

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

`PickupAction` needs to iterate over items on the map. Add a filtered property to `game/map/game_map.py`:

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

`add_item()` checks capacity, transfers ownership to the inventory, and appends the item to `items`. When the item comes from the dungeon floor, the caller passes the current `game_map`, and `add_item()` removes it from the map's entity set via `game_map.entities.discard()`. When the item was never placed on a map, for example a debug starting item, the caller can pass `None`.

`game_map` accepts `None`, but it does not have a default value. This makes each call state its intent explicitly: `inventory.add_item(item, engine.game_map)` means "take this item from the map", while `inventory.add_item(item, None)` means "add an item that is not on any map". That explicit argument helps avoid accidentally leaving a floor item both in the map and in the inventory. The method returns `False` if the inventory is full so that the caller can raise `Impossible` with a reason.

`drop_item()` removes the item from `items`, then calls `item.place()` to move it back to the dungeon floor at the actor's current position. The caller passes `game_map` explicitly: `DropItem.perform` already has `engine` and can supply `engine.game_map` directly, so `Inventory` does not need to navigate the ownership chain itself.

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
    from game.actions import Action, ItemAction
    from game.engine import Engine
    from game.entities.entity import Actor


class Consumable(ItemComponent):

    def get_action(self, _consumer: Actor, _engine: Engine) -> Action | None:
        """Return the action this consumable produces when used."""
        from game.actions import ItemAction

        return ItemAction(item=self.entity)

    def activate(self, _action: ItemAction, _engine: Engine, _consumer: Actor) -> None:
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

    def activate(self, _action: ItemAction, _engine: Engine, consumer: Actor) -> None:
        amount_recovered = consumer.fighter.heal(self.amount)

        if amount_recovered > 0:
            MessageLog.add_message(
                f"You consume the {self.entity.name}, and recover {amount_recovered} HP!",
                colors.HEALTH_RECOVERED,
            )
            self.consume()

        else:
            raise Impossible("Your health is already full.")
```

The file defines `Consumable` as the base class for all item effects and `HealingConsumable` as its first concrete subclass.

`Consumable.get_action()` returns the action produced by using this item. In this chapter it returns a plain `ItemAction`, but future consumable types can override it to request additional input before acting, for example a targeting scroll that needs a destination tile. The local import inside `get_action()` avoids a module-level circular import between `consumable.py` and `actions.py`.

`consume()` imports `Inventory` locally to avoid a circular import: `consumable.py` and `inventory.py` would otherwise form a cycle at module level. The `isinstance` check serves double duty: it guards against consuming an unowned item and narrows the declared type from `GameMap | Inventory | None` to `Inventory`, making the subsequent `inventory.items` access well-typed. After removing the item, `entity.owner = None` clears the stale reference so the consumed item no longer points at the inventory it came from.

`HealingConsumable.activate()` calls `fighter.heal()`, which you wrote in Part 7. If the player is already at full health, `heal()` returns `0` and `activate()` raises `Impossible`. The event handler will catch it and show the reason as a yellow message.

---

## Update `game/entities/factories.py`

Every `Actor` now requires an `Inventory`. Add the component to the existing templates:

```diff
 from game.constants import colors, sprites
 from game.entities.components.ai import HostileEnemy
+from game.entities.components.consumable import HealingConsumable
 from game.entities.components.fighter import Fighter
+from game.entities.components.inventory import Inventory
-from game.entities.entity import Actor, Entity
+from game.entities.entity import Actor, Entity, Item
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

Then add the health potion template and the spawn table at the bottom of the file:

```python
# Items
health_potion = Item(
    char       = sprites.HEALTH_POTION,
    color      = colors.HEALTH_POTION,
    name       = "Health Potion",
    consumable = HealingConsumable(amount=4),
)

chest = Entity(
    char            = sprites.CHEST,
    color           = colors.CHEST,
    name            = "Chest",
    blocks_movement = True,
)

item_chances = [
    (health_potion, 40),
    (chest,         60),
]
```

`item_chances` mirrors the `monster_chances` pattern from Part 5: a list of `(template, weight)` pairs that `map_generator.py` will use to pick which item to place in each room. Weights are relative: a chest (60) is created more often than a potion (40). Exercise 2 adds `backpack_scroll` to this list.

!!! note "If you completed the Part 5 chest exercise"
    You may already have a passive `chest` template or custom chest spawn logic. Keep a single `chest` template, include it in `item_chances`, and remove any older separate spawn entry you added for it. Later in this chapter we will convert this template from `Entity` to `Item`.

---

## Update `game/actions.py`

Add the three new action classes and two new imports at the top of the file:

```diff
+from game.entities.entity import Actor, Item
+from game.exceptions import Impossible
 from game.message_log import MessageLog
```

```python
class PickupAction(Action):

    def perform(self, engine: Engine, entity: Entity) -> None:
        assert isinstance(entity, Actor)
        inventory = entity.inventory

        for item in engine.game_map.items:
            if entity.x == item.x and entity.y == item.y:
                if not inventory.add_item(item, game_map=engine.game_map):
                    raise Impossible("Your inventory is full.")

                MessageLog.add_message(f"You picked up the {item.name}!")
                return

        raise Impossible("There is nothing here to pick up.")


class ItemAction(Action):

    def __init__(self, item: Item) -> None:
        super().__init__()
        self.item = item

    def perform(self, engine: Engine, entity: Entity) -> None:
        assert isinstance(entity, Actor)
        self.item.consumable.activate(self, engine, entity)


class DropItem(ItemAction):

    def perform(self, engine: Engine, entity: Entity) -> None:
        assert isinstance(entity, Actor)
        entity.inventory.drop_item(self.item, engine.game_map)
        MessageLog.add_message(f"You dropped the {self.item.name}.")
```

`PickupAction` iterates `engine.game_map.items` (the new property) looking for an item at the player's position. If found, it delegates to `inventory.add_item()`, which removes the item from the map's entity set, transfers ownership to the inventory, and appends it to `inventory.items`. Both trailing `raise Impossible` statements ensure the player always gets feedback.

`ItemAction` delegates to `consumable.activate()`. `DropItem` extends `ItemAction` because dropping also needs the item reference, but calls `inventory.drop_item()` instead.

The `assert isinstance(entity, Actor)` calls enforce a design contract: `Action.perform` is declared with `entity: Entity`, but these three actions require an `Actor` (only actors have `inventory`). The asserts make that constraint explicit at runtime and narrow the declared type, so a type checker can verify the subsequent attribute accesses without casts.

Since `Actor` is now imported at the top of the file, remove the local import that was inside `MeleeAction.perform()`:

```diff
-        from game.entities.entity import Actor
-
         if not isinstance(entity, Actor):
```

Also add `PickupAction` to the import list at the top of `game/game_states.py`:

```diff
-from game.actions import (
-    Action,
-    BumpAction,
-    EscapeAction,
-    WaitAction,
-)
+from game.actions import (
+    Action,
+    BumpAction,
+    EscapeAction,
+    PickupAction,
+    WaitAction,
+)
```

---

## Move action execution into `GameState`

So far `Engine.handle_events()` ran the action returned by the active game state. That worked with only one state, but modal states (the inventory overlay) need to switch the active state *after* an action completes. Only the state knows which state it should return to; the engine does not.

!!! info "Why move execution into the state?"
    `Engine.handle_events()` currently does: get action from state, perform it, run enemy turns, update FOV. That is fine with one state. But when the inventory overlay is open, pressing `a` should use the item *and then close the overlay* (returning to `MainGameState`). The engine cannot make that switch because it does not know which state to return to. The state does: it opened the overlay, so it knows to close it.

    Moving the execution loop into `GameState.handle_events()` gives each state control over what happens after an action.

Update `game/engine.py`. The `handle_events` signature changes from `Iterable[Any]` to `Iterable[tcod.event.Event]`, so `Any` is no longer needed. `GameOverState` also moves out of `engine.py` (it is now referenced from inside `GameState.handle_events` in `game_states.py`):

```diff
-from typing import Any

 import tcod.event
 ...
-from game.game_states import (
-    GameState,
-    GameOverState,
-    MainGameState
-)
+from game.game_states import GameState, MainGameState
```

Then simplify `handle_events()` to a one-line dispatch:

```diff
-def handle_events(self, events: Iterable[Any]) -> None:
+def handle_events(self, events: Iterable[tcod.event.Event]) -> None:
     for event in events:
-        action = self.game_state.handle_events(event)
-        if action is not None:
-            action.perform(self, self.player)
-            self.handle_enemy_turns()
-            self.update_fov()
+        self.game_state.handle_events(event)
```

Also add `INVALID` to `game/constants/colors.py`, since the rewritten `handle_events` uses it for rejection messages:

```diff
 WELCOME_TEXT = Color(0x20, 0xA0, 0xFF)
 ...
+INVALID      = Color(0xFF, 0xFF, 0x00)
```

`INVALID` is yellow: it is used for every rejection the player should read, from a full inventory to an already-healed condition.

Now rewrite `GameState.handle_events()` in `game/game_states.py` to own the full execution cycle. `from game.constants import colors` was already imported in Part 7; only `Impossible` is new:

```diff
+from game.exceptions import Impossible
 from game.message_log import MessageLog
```

```python
def handle_events(self, event: tcod.event.Event) -> None:
    action: Action | None = None

    match event:
        case tcod.event.Quit():
            action = EscapeAction()

        case tcod.event.MouseMotion():
            self.engine.mouse_location = event.integer_position

        case tcod.event.KeyDown():
            action = self.event_keydown(event)

    if action is not None:
        try:
            action.perform(self.engine, self.engine.player)

        except Impossible as ex:
            MessageLog.add_message(str(ex), colors.INVALID)
            return

        if self.engine.player.is_alive:
            self.engine.handle_enemy_turns()

        if not self.engine.player.is_alive:
            self.engine.game_state = GameOverState(self.engine)

        elif isinstance(self.engine.game_state, (InventoryUseState, InventoryDropState)):
            self.engine.game_state = MainGameState(self.engine)

        self.engine.update_fov()
```

Key differences from the old engine code:

- `except Impossible as ex:` catches any rejection, logs it with `colors.INVALID`, and returns early so enemies do not take their turn on a failed action.
- After a successful action, if the current state is an inventory state, it switches back to `MainGameState`. Opening the inventory does not advance time; using or dropping an item does.

!!! note "What Impossible is for"
    `Impossible` is reserved for rejections that are worth telling the player about: a full inventory, an already-healed condition, nothing to pick up. Low-level movement failures (bumping into a wall, trying to walk off the map) are not raised as `Impossible`. Those actions still cost a turn (the player pressed a key and something was attempted), but generating a log message for every wall collision would be noise. The convention is: if the player needs to read why an action failed, raise `Impossible`; if the failure is self-evident from the game state, return silently.

The return type of `handle_events` changes from `Action | None` to `None`. Update the method signature line accordingly.

---

## Trim movement keys

Vi keys (`b`, `h`, `j`, `k`, `l`, `n`, `u`, `y`) have been part of roguelikes since the original *Rogue* (1980), which ran on VT100 terminals with no arrow keys. On a modern keyboard, arrow keys and the numpad cover the same directions with less ambiguity, and freeing those letters matters here: this chapter adds `G`, `I`, and `D` as action keys, and Exercise 3 assigns the remaining letters to items for direct use from the map.

Remove the vi keys block from `MOVE_KEYS` in `game/game_states.py`:

```diff
-    # Vi keys
-    tcod.event.KeySym.B:     (-1,  1),
-    tcod.event.KeySym.J:     ( 0,  1),
-    tcod.event.KeySym.N:     ( 1,  1),
-    tcod.event.KeySym.H:     (-1,  0),
-    tcod.event.KeySym.L:     ( 1,  0),
-    tcod.event.KeySym.Y:     (-1, -1),
-    tcod.event.KeySym.K:     ( 0, -1),
-    tcod.event.KeySym.U:     ( 1, -1),
```

!!! info "Numpad vs. regular number keys"
    `tcod.event.KeySym.KP_1`–`KP_9` are distinct key codes from `tcod.event.KeySym.N1`–`N9`. Numpad keys continue to work for movement. Regular number keys (`1`–`9`, `0`) remain free for future use, such as equipment slots in Part 13.

---

## Update `MainGameState`

Add three key bindings at the end of `event_keydown`:

```diff
         if key == tcod.event.KeySym.ESCAPE:
             return EscapeAction()

+        if key == tcod.event.KeySym.G:
+            return PickupAction()
+
+        if key == tcod.event.KeySym.I:
+            self.engine.game_state = InventoryUseState(self.engine)
+
+        if key == tcod.event.KeySym.D:
+            self.engine.game_state = InventoryDropState(self.engine)
+
         return None
```

`G` returns a `PickupAction`; the action system handles the rest. `I` and `D` do not return actions; they switch the active state immediately. The overlay then handles the next key press.

---

## Inventory states

Three new state classes go at the bottom of `game/game_states.py`.

Add the inventory overlay colors to `game/constants/colors.py`:

```diff
 INVALID = Color(0xFF, 0xFF, 0x00)
+
+# Inventory overlay colors
+INVENTORY_MENU_TITLE = Color(255, 245, 160)
+INVENTORY_MENU_TEXT  = Color(232, 255, 255)
+INVENTORY_MENU_DIM   = Color(168, 216, 216)
+INVENTORY_MENU_KEY   = BLACK
+
+INVENTORY_USE_FG     = Color( 80, 255, 184)
+INVENTORY_USE_BG     = Color(  5,  36,  30)
+INVENTORY_USE_ACCENT = Color( 32, 168, 112)
+INVENTORY_USE_ROW_BG = Color( 10,  58,  44)
+
+INVENTORY_DROP_FG     = Color(224, 128, 255)
+INVENTORY_DROP_BG     = Color( 38,  14,  58)
+INVENTORY_DROP_ACCENT = Color(160,  80, 232)
+INVENTORY_DROP_ROW_BG = Color( 58,  22,  82)
```

The inventory overlay uses two distinct color schemes: green tones for item use (activation) and purple tones for item dropping. At a glance, the player always knows which overlay is open. The `INVENTORY_MENU_*` constants are shared by both schemes: title, text, and dim shades that look the same in either overlay, plus the key badge foreground.

!!! tip "Modal states"
    An inventory state follows exactly the same pattern as `GameOverState` from Part 7: it overrides `on_render()` to draw an overlay on top of the map, and `event_keydown()` to handle its own key set. The overlay closes when the player selects a valid item (an action executes, then `GameState.handle_events` switches back to `MainGameState`) or presses `Escape` (handled explicitly in `event_keydown`, which sets the state directly without returning an action). Any other unrecognised key does nothing. This pattern composes cleanly: any state can open any other state, and the "stack" is simply `self.engine.game_state` with no state stack to maintain.

Item names come from data, so a very long name could overflow the panel. A small helper trims text to a maximum width and marks the cut with `...`. Add it to `game/game_states.py`, next to `_draw_panel` from Part 7:

```python
def _trim_text(text: str, max_width: int) -> str:
    if len(text) <= max_width:
        return text

    if max_width <= 3:
        return text[:max_width]

    return f"{text[:max_width - 3]}..."
```

Now the three state classes:

```python
class InventoryState(GameState):
    """Base class for inventory screens (use and drop share the same UI)."""

    TITLE        = "<missing title>"
    PROMPT       = "<missing prompt>"
    EMPTY_TEXT   = "Your pack is empty."
    FG_COLOR     = colors.WHITE
    BG_COLOR     = colors.BLACK
    ACCENT_COLOR = colors.WHITE
    ROW_BG_COLOR = colors.BLACK

    def on_render(self, console: tcod.console.Console) -> None:
        super().on_render(console)  # draws the map behind the overlay

        # Dim the map background to highlight the inventory.
        console.fg[:] = console.fg // 2
        console.bg[:] = console.bg // 2

        inventory = self.engine.player.inventory
        number_of_items_in_inventory = len(inventory.items)

        slot_count   = f"({len(inventory.items)} / {inventory.capacity} slots)"
        max_height   = console.height - 4
        height       = min(max(8, number_of_items_in_inventory + 7), max_height)
        visible_rows = min(number_of_items_in_inventory, max(0, height - 7))

        item_width = max((len(item.name) for item in inventory.items), default=0) + 14
        width = min(
            console.width - 4,
            max(
                46,
                len(self.TITLE) + 4,
                len(self.PROMPT) + 6,
                len(slot_count) + 6,
                item_width,
            ),
        )

        x = (console.width  - width)  // 2
        y = (console.height - height) // 2

        # Draw the inventory box.
        _draw_panel(console, x, y, width, height, self.FG_COLOR, self.BG_COLOR)

        title = f" {self.TITLE} "
        # Draw the inventory title over the frame.
        console.print(
            x    = x + (width - len(title)) // 2,
            y    = y,
            text = title,
            fg   = colors.INVENTORY_MENU_TITLE,
            bg   = self.BG_COLOR,
        )

        # Draw the main help text.
        console.print(
            x         = console.width // 2,
            y         = y + 2,
            text      = self.PROMPT,
            fg        = colors.INVENTORY_MENU_TEXT,
            alignment = tcod.constants.CENTER,
        )

        row_x = x + 3
        row_width = width - 6
        if visible_rows > 0:
            name_width = max(1, row_width - 12)
            for i, item in enumerate(inventory.items[:visible_rows]):
                item_key = chr(ord("a") + i)
                row_y = y + 4 + i

                # Draw the background for one item row.
                console.draw_rect(
                    x      = row_x,
                    y      = row_y,
                    width  = row_width,
                    height = 1,
                    ch     = ord(" "),
                    bg     = self.ROW_BG_COLOR,
                )

                # Draw the key that selects this item.
                console.print(
                    row_x + 2,
                    row_y,
                    f"[ {item_key} ]",
                    fg = colors.INVENTORY_MENU_KEY,
                    bg = self.ACCENT_COLOR,
                )

                # Draw the item glyph using its own color.
                console.print(
                    row_x + 8,
                    row_y,
                    item.char,
                    fg = item.color,
                    bg = self.ROW_BG_COLOR
                )

                # Draw the item name, trimmed if it does not fit.
                console.print(
                    row_x + 10,
                    row_y,
                    _trim_text(item.name, name_width),
                    fg = colors.INVENTORY_MENU_TEXT,
                    bg = self.ROW_BG_COLOR,
                )

        else:
            row_y = y + 4

            # Draw the empty row when there are no items.
            console.draw_rect(
                x      = row_x,
                y      = row_y,
                width  = row_width,
                height = 1,
                ch     = ord(" "),
                bg     = self.ROW_BG_COLOR,
            )

            # Draw the empty-inventory message.
            console.print(
                x         = console.width // 2,
                y         = row_y,
                text      = self.EMPTY_TEXT,
                fg        = colors.INVENTORY_MENU_DIM,
                bg        = self.ROW_BG_COLOR,
                alignment = tcod.constants.CENTER,
            )

        # Draw the used-slots counter after the item list.
        console.print(
            x         = console.width // 2,
            y         = y + height - 2,
            text      = slot_count,
            fg        = colors.INVENTORY_MENU_DIM,
            bg        = self.BG_COLOR,
            alignment = tcod.constants.CENTER,
        )

    def event_keydown(self, event: tcod.event.KeyDown) -> Action | None:
        player = self.engine.player
        key = event.sym
        index = key - tcod.event.KeySym.A

        if 0 <= index <= 25:
            try:
                selected_item = player.inventory.items[index]
            except IndexError:
                MessageLog.add_message("Invalid entry.", colors.INVALID)
                return None

            return self.on_item_selected(selected_item)

        if key == tcod.event.KeySym.ESCAPE:
            self.engine.game_state = MainGameState(self.engine)
            return None

        return super().event_keydown(event)

    def on_item_selected(self, item: Item) -> Action | None:
        raise NotImplementedError()
```

```python
class InventoryUseState(InventoryState):
    TITLE        = "Use Item"
    PROMPT       = "Select an item to use:"
    FG_COLOR     = colors.INVENTORY_USE_FG
    BG_COLOR     = colors.INVENTORY_USE_BG
    ACCENT_COLOR = colors.INVENTORY_USE_ACCENT
    ROW_BG_COLOR = colors.INVENTORY_USE_ROW_BG

    def on_item_selected(self, item: Item) -> Action | None:
        return item.consumable.get_action(self.engine.player, self.engine)
```

**The use inventory overlay looks like this**:

![Inventory: Use Item](images/window_inventory_use1.png)

```python
class InventoryDropState(InventoryState):
    TITLE        = "Drop Item"
    PROMPT       = "Select an item to drop:"
    FG_COLOR     = colors.INVENTORY_DROP_FG
    BG_COLOR     = colors.INVENTORY_DROP_BG
    ACCENT_COLOR = colors.INVENTORY_DROP_ACCENT
    ROW_BG_COLOR = colors.INVENTORY_DROP_ROW_BG

    def on_item_selected(self, item: Item) -> Action | None:
        from game.actions import DropItem

        return DropItem(item=item)
```

**The drop overlay shares the same layout**:

![Inventory: Drop Item](images/window_inventory_drop1.png)

!!! info "Pattern: Template Method"
    `InventoryState` defines the complete algorithm (render the overlay, map keys to items, call `on_item_selected`) but leaves the final step as an abstract hook that each concrete subclass fills in. `InventoryUseState` uses the item; `InventoryDropState` drops it. The skeleton of the algorithm lives in the base class; the variation lives in the subclasses.

    The same structure appears with `on_index_selected` in `SelectIndexState` (Part 9).

`on_render()` renders the map first via `super()`, dims it with the same `// 2` trick introduced in Part 7 for the game-over screen, and then draws the box with `_draw_panel()` (shadow, fill, frame). Nothing new there: the overlay reuses the exact machinery the game-over panel taught us.

The panel layout is fixed at the top and grows downward with the item list. Row `y` carries the frame and the centered title; `y+2` prints the `PROMPT` help text; item rows start at `y+4`. The used-slots counter (`(3 / 10 slots)`) sits on the row above the bottom border, like a status line, and doubles as capacity feedback now that the inventory can fill up.

Each item row is drawn in four pieces: a one-row `draw_rect` with `ROW_BG_COLOR` that makes the row read as a single unit, the selection key as a badge (`[ a ]`, dark text over `ACCENT_COLOR`), the item sprite in its own color at `row_x + 8`, and the name at `row_x + 10`, passed through `_trim_text()` so a long name cannot break the layout. The badge-over-accent style makes the actionable keys pop out from the text around them.

`width` adapts to the longest content line (title, prompt, counter, or item name plus its prefixes) with a floor of 46 cells, and is clamped to `console.width - 4` so the panel always leaves a margin. `height` follows the item count, with `visible_rows` recomputed from the clamped height so the loop never draws past the bottom border. When the inventory is empty, a single dim row prints `EMPTY_TEXT` instead of the item list.

`TITLE`, `PROMPT`, and the four color class variables follow the same class-variable pattern introduced in Part 7 for `GameOverState`. Each subclass overrides them: green tones for activation, purple tones for dropping, so the player always knows which overlay is open at a glance.

`event_keydown()` converts the pressed key to an index: `a → 0`, `b → 1`, and so on. The range `0 <= index <= 25` covers exactly the 26 letters `a`-`z`. If the index falls outside the item list, it logs "Invalid entry." and returns `None`. Otherwise it calls `on_item_selected()`, which the two subclasses implement differently. `Escape` closes the overlay immediately by switching back to `MainGameState` without returning an action (so enemies do not take a turn).

`InventoryUseState` asks the item's consumable for an action; `InventoryDropState` returns a `DropItem`. The action is then executed by `GameState.handle_events()` and, because the current state is an inventory state, it automatically switches back to `MainGameState` after the action completes.

You will notice that `InventoryUseState` and `InventoryDropState` are referenced in `GameState.handle_events()`, which is defined earlier in the same file. This is fine in Python: method bodies are only executed when called, at which point all classes in the module are already defined.

Also add `Item` to the imports at the top of `game_states.py` so that `on_item_selected` can use it as a type annotation:

```diff
 if TYPE_CHECKING:
     from game.engine import Engine
+    from game.entities.entity import Item
```

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

## Treasure chests

The chest introduced in Part 5 has been blocking movement without doing anything useful. Now that `Item` and `Consumable` exist, the chest can become a collectible that rewards the player with gold.

### New color constant

In `game/constants/colors.py`:

```diff
 INVALID = Color(0xFF, 0xFF, 0x00)
+GOLD    = Color(0xFF, 0xD7, 0x00)
```

### `TreasureConsumable`

Add a new consumable class in `game/entities/components/consumable.py`. It differs from `HealingConsumable` in one important way: the item is never added to the inventory. It is collected directly from the floor and disappears immediately. Add a class-level flag to mark this behavior, plus the base `on_contact` hook that activates the item when the flag is set (the `MovementAction` wiring that calls `on_contact` is added later in this part):

```diff
 class Consumable(ItemComponent):
+    auto_activate: bool = False
+
+    def on_contact(self, engine: Engine, consumer: Actor) -> None:
+        from game.actions import ItemAction
+
+        if self.auto_activate:
+            ItemAction(item=self.entity).perform(engine, consumer)
```

```python
class TreasureConsumable(Consumable):
    auto_activate = True

    def __init__(self, value: int) -> None:
        self.value = value

    def on_contact(self, engine: Engine, consumer: Actor) -> None:
        if consumer is engine.player:
            super().on_contact(engine, consumer)

    def activate(self, _action: ItemAction, engine: Engine, consumer: Actor) -> None:
        consumer.inventory.gold += self.value
        MessageLog.add_message(
            f"You found {self.value} gold!",
            colors.GOLD,
        )
        engine.game_map.entities.discard(self.entity)
        self.entity.owner = None
```

`activate()` adds `value` to `consumer.inventory.gold`, logs the find, and removes the chest from the map in the same call. There is no `self.consume()` here because `consume()` removes an item from an inventory; the chest was never in one.

### `gold` field on `Inventory`

Gold is something the player carries, not a combat stat, so it belongs in `Inventory` alongside the item list. Add `self.gold: int = 0` to `Inventory.__init__`:

```diff
     def __init__(self, capacity: int) -> None:
         self.capacity = capacity
         self.items: list[Item] = []
+        self.gold: int = 0
```

### Convert `chest` in `factories.py`

First update the imports in `game/entities/factories.py`. `chest` will no longer be a plain `Entity`, and it now needs `TreasureConsumable`:

```diff
-from game.entities.components.consumable import HealingConsumable
+from game.entities.components.consumable import HealingConsumable, TreasureConsumable
 from game.entities.components.fighter import Fighter
 from game.entities.components.inventory import Inventory
-from game.entities.entity import Actor, Entity, Item
+from game.entities.entity import Actor, Item
```

Then replace the passive `Entity` with an `Item`:

```diff
-chest = Entity(
-    char            = sprites.CHEST,
-    color           = colors.CHEST,
-    name            = "Chest",
-    blocks_movement = True,
-)
+chest = Item(
+    char       = sprites.CHEST,
+    color      = colors.CHEST,
+    name       = "Chest",
+    consumable = TreasureConsumable(value=10),
+)
```

The chest no longer blocks movement (items never block movement). The player walks over it to collect it.

### Auto-collect in `MovementAction`

`PickupAction` is triggered by the `G` key. Treasure should also be collected automatically when the player steps on it. The `Consumable.on_contact` hook is already in place (added alongside `TreasureConsumable` above); two pieces remain to wire it up.

First, add a `GameMap.items_at(x, y)` helper that returns all items at a given position:

```python
def items_at(self, x: int, y: int) -> list[Item]:
    from game.entities.entity import Item
    return [e for e in self.entities if isinstance(e, Item) and e.x == x and e.y == y]
```

Second, after `entity.move()` in `MovementAction.perform()`, trigger contact for every item on the new tile:

```diff
         entity.move(self.dx, self.dy)
+
+        if isinstance(entity, Actor):
+            for item in engine.game_map.items_at(entity.x, entity.y):
+                item.consumable.on_contact(engine=engine, consumer=entity)
```

The `isinstance(entity, Actor)` check satisfies the type checker: `on_contact` expects an `Actor`, and `entity` in `MovementAction` is annotated as the wider `Entity` type. In practice any entity that moves will be an actor.

`items_at()` returns a fresh list, so `activate()` can safely modify `game_map.entities` during iteration. The `MovementAction` wiring has no player check: it fires `on_contact` for every actor that steps on the tile. The decision to react lives in the consumable: `TreasureConsumable` only acts when the consumer is the player (so a wandering monster cannot scoop up a chest), while a different consumable could choose to react to anyone.

### `render_gold` in `hud.py`

Add a one-line gold display below the HP bar:

```python
def render_gold(
    console: Console,
    gold: int,
    y: int = 44,
) -> None:
    console.print(
        x    = 0,
        y    = y,
        text = f"$ {gold}",
        fg   = colors.GOLD,
    )
```

Call it from `Engine.render()`:

```diff
     hud.render_bar(...)
+
+    hud.render_gold(
+        console = console,
+        gold    = self.player.inventory.gold,
+    )
```

---

## Testing your work

Run the game and verify the following:

- Health potions (`!`) appear on the dungeon floor.
- Walking over a potion and pressing `G` adds it to the inventory and shows "You picked up the Health Potion!".
- Pressing `G` on an empty tile shows "There is nothing here to pick up." in yellow.
- Pressing `I` opens the "Use Item" overlay: the map dims, a framed panel with a drop shadow appears, and carried items are listed as rows with `[ a ]`-style key badges.
- The overlay shows the used-slots counter (for example `(2 / 10 slots)`) at the bottom of the panel.
- Pressing the letter for a potion when injured heals the player and shows a green message.
- Pressing the letter for a potion at full health shows "Your health is already full." in yellow.
- Pressing `D` opens the "Drop Item" overlay in purple tones; selecting an item drops it at the player's feet.
- Pressing an out-of-range letter in the inventory overlay shows "Invalid entry." in yellow.
- Enemies still act on their turn after the player successfully uses or drops an item.
- Enemies do *not* act when the player tries to pick up from an empty tile (a failed action costs no turn).
- Chests (`$`) appear on the dungeon floor. Walking over one shows "You found 10 gold!" and the `$` symbol disappears.
- The gold counter below the HP bar increments each time a chest is collected.

---

## Summary

Items are now a first-class part of the game. Key additions:

- **`game/exceptions.py`**: `Impossible` exception carries a rejection reason; one `except` in `handle_events` covers every case
- **`ActorComponent` / `ItemComponent`**: narrowed base classes so type annotations match the actual runtime type
- **`Entity.owner`**: single source of truth for whether an entity is on the floor or in an inventory
- **`Item` / `Inventory`**: new entity subclass and actor component; pickup, use, and drop are modelled as actions
- **`HealingConsumable`**: first consumable component; knows how to apply its effect independently of the action layer
- **`TreasureConsumable`**: second consumable; collected on contact rather than through the inventory; `auto_activate = True` triggers pickup on walk
- **`InventoryState`**: modal overlay base class; subclasses override `TITLE`, `PROMPT`, the four color class variables, and `on_item_selected()`
- **`Inventory.gold`**: running treasure total stored in the `Inventory` component; read as `player.inventory.gold`; keeping all player-held state in one place simplifies future serialization

**Current architecture**:

- `Entity.owner`: set to `GameMap` on spawn, changed to `Inventory` on pickup, back to `GameMap` on drop; never `None` while the entity is active on the map or carried in an inventory
- `Inventory`: component on `Actor`; holds items up to `capacity`, enforces the limit, and handles drop logic
- `Consumable.auto_activate`: class-level flag; `Consumable.on_contact()` checks it and activates the item via `ItemAction` when the player steps on the tile
- `HealingConsumable.activate()`: applies healing and calls `self.consume()` to remove the item from inventory
- `TreasureConsumable.activate()`: adds gold, logs the message, and removes the item from the map directly; it never enters inventory
- `Impossible`: raised anywhere in the action chain; `handle_events` catches it and shows the message in the log
- `InventoryState`: subclasses provide `on_item_selected()` and override the title, prompt, and color class variables; `GameState.handle_events()` automatically switches back to `MainGameState` after an inventory action
- `ActorComponent` / `ItemComponent`: `entity` annotation narrows from `Entity` to the actual holder type, so type checkers can verify component attribute access correctly

**Class Diagram**:

![classes](images/part8_classes.png)

**File structure**:

```text
main.py                         ← modified
game/
├── __init__.py
├── actions.py                  ← modified
├── engine.py                   ← modified
├── exceptions.py               ← new
├── hud.py                      ← modified
├── game_states.py              ← modified
├── message_log.py
├── constants/
│   ├── __init__.py
│   ├── colors.py               ← modified
│   └── sprites.py              ← modified
├── entities/
│   ├── __init__.py
│   ├── entity.py               ← modified
│   ├── factories.py            ← modified
│   ├── render_order.py
│   └── components/
│       ├── __init__.py
│       ├── ai.py               ← modified
│       ├── base_component.py   ← modified
│       ├── consumable.py       ← new
│       ├── fighter.py          ← modified
│       └── inventory.py        ← new
└── map/
    ├── __init__.py
    ├── game_map.py             ← modified
    ├── tile_types.py
    └── map_generator.py        ← modified
```

---

## Exercises

!!! tip "Exercises 1 and 2 together make the game noticeably more fun"
    Stacking keeps the inventory list clean, and the backpack gives the player a reason to seek scrolls. Exercises 3 and 4 polish the user interface.

1. **Item stacking**:

    When the inventory shows items, group identical ones into a single row with a count, `[ h ] Health Potion (x3)`, instead of three separate lines. Items with the same `name` form a stack. This is a display-layer change: the underlying `Inventory.items` list is untouched, so dropping still removes one instance at a time and the stack just shrinks by one.

    ??? note "Reference implementation"
        Add two static helpers to `InventoryState`. `stack_items(items) -> list[list[Item]]` groups items by name (`dict.setdefault`); each stack is a `list[Item]` whose first element is used for display and selection (after Exercise 3, also sort the stacks here: `stacks.sort(key=lambda s: s[0].key or 0)`). `stack_name(stack) -> str` returns `"Health Potion (x3)"` when `len(stack) > 1`, else just the name.

        Then `on_render()` calls `stack_items()` and iterates over stacks, computing the panel width from `stack_name()` so the frame fits the longest entry. The alphabetical letter-to-index system is replaced by Exercise 3's key-based selection. (`Inventory.drop_item()` already removes the first matching item, so dropping needs no extra logic.)

2. **Backpack growing scroll**:

    Add a scroll that permanently grows the player's backpack, up to a ceiling. Make `max_capacity: int` a **required** parameter of `Inventory.__init__` (no default): that forces every `Actor` in `factories.py` to declare its ceiling (monsters get `max_capacity=0`, so they never expand), and a forgotten actor fails loudly at startup instead of silently inheriting 26. Each scroll is a permanent, irreversible upgrade (the player starts at `capacity=10`, gains `+4` per scroll, up to 26), so finding one is meaningful.

    ??? note "Reference implementation"
        `BackpackConsumable(amount: int)` raises `capacity` by `amount`, capped at `max_capacity`. Use `min()` for the actual gain, and raise `Impossible` if already at the cap before touching anything:

        ```python
        actual = min(self.amount, consumer.inventory.max_capacity - consumer.inventory.capacity)
        consumer.inventory.capacity += actual
        ```

        Add the visual constants: `BACKPACK_SCROLL = "?"` in `sprites.py` and a parchment `BACKPACK_SCROLL = Color(255, 224, 160)` in `colors.py`. After growing the capacity, log the gain in that color and `self.consume()`:

        ```python
        MessageLog.add_message(
            f"Your backpack grows by {actual} slots.",
            colors.BACKPACK_SCROLL,
        )
        self.consume()
        ```

        Finally wire a `backpack_scroll` item in `factories.py` (`sprites.BACKPACK_SCROLL`, `colors.BACKPACK_SCROLL`, name `"Backpack Growing Scroll"`) and add it to `item_chances` alongside the health potion and chest.

3. **Persistent item keys**:

    Right now the inventory letter for each item shifts whenever a preceding item is used or dropped: after consuming the first potion, what was `b` becomes `a`. Give each item type a **fixed hotkey**, assigned by the programmer in `factories.py`, that never changes regardless of inventory order (and `None` for auto-collected items like the chest, which need no shortcut). Because the key lives on the template object, `spawn()` copies it for free: no per-instance bookkeeping on pickup, consume or drop. The wiring touches a handful of places: the `Item` itself, the factories, the inventory render, the inventory's key handling, the main-game key handling (so a hotkey works without opening the inventory), and the pickup message.

    ??? note "Reference implementation"
        `Item.__init__` (`entity.py`): add `key: tcod.event.KeySym | None` as a required parameter, alongside `consumable`. The `| None` supports auto-collected items (the chest) that need no shortcut. Import `tcod.event` under `TYPE_CHECKING` (the annotation is a string at runtime thanks to `from __future__ import annotations`).

        `factories.py`: assign a mnemonic key per keyboard-interactive item, `None` for auto-collected ones. After Exercise 4 these raw values become `keys.*` constants:

        ```python
        health_potion   = Item(..., key=tcod.event.KeySym.H)   # becomes keys.HEALTH_POTION after Ex 4
        backpack_scroll = Item(..., key=tcod.event.KeySym.B)   # becomes keys.BACKPACK_SCROLL after Ex 4
        chest           = Item(..., key=None)
        ```

        `InventoryState.on_render()`: show the letter in the badge with `chr(item.key) if item.key is not None else "-"` (`KeySym` is an `IntEnum` whose letter values equal their ASCII codes, so `chr(KeySym.H)` is `'h'`). Add the key-based sort (`stacks.sort(key=lambda s: s[0].key or 0)`) inside `stack_items`, not here, so the order stays stable everywhere the stacks are used.

        `InventoryState.event_keydown()`: replace the index computation with a loop checking `stack[0].key is not None and stack[0].key == key`. After it (and the escape check), add a fallback for stray letters: if `ord("a") <= int(key) <= ord("z")`, log `"Invalid entry."` in `colors.INVALID` and return `None`.

        `MainGameState.event_keydown()`: add a fallback at the end that scans the inventory for an item whose `key` matches `event.sym` and calls `item.consumable.get_action()`. Same effect as opening the inventory and selecting it; if it needs targeting (Part 9), the targeting UI opens just the same.

        `PickupAction`: after the `"You picked up the ..."` message, if `item.key is not None`, log a second line. Import `from game.constants.keys import key_label` and write `MessageLog.add_message(f"Press {key_label(item.key)} to use it.")` (`key_label` makes a badge like `[ H ]`; you add it in Exercise 4).

        With vi keys removed, `a`–`z` minus `g`, `i`, `d` gives 23 conflict-free hotkey slots.

4. **Centralise keybindings in `game/constants/keys.py`**:

    Right now your keybindings are scattered, and as raw `tcod.event.KeySym` values: movement, wait and action keys in `game/game_states.py`, item hotkeys in `game/entities/factories.py`. Centralize them into one new `game/constants/keys.py`: the `MOVE_KEYS` / `WAIT_KEYS` dictionaries, every `KEY_*` constant, and a small `key_label` helper that formats a key as a badge like `[ H ]` or `[ Esc ]` (the pickup hint from Exercise 3 and the Part 10 main menu both render through it). Then make every file reference `keys.*` instead of a raw `KeySym`. The payoff is real: a player can remap every control by editing one file, without touching any state or factory.

    !!! tip "How `key_label` picks a name"
        Three rules, in order: an entry in `_SPECIAL_KEY_NAMES` wins first (`ESCAPE` becomes `Esc`); otherwise, a printable ASCII code (32 to 126) becomes `chr(v).upper()`, so the period key reads `.` instead of the verbose `"PERIOD"`; everything else falls back to `sym.name` (`"F1"`, `"KP_8"`, `"UP"`).

    ??? note "Reference implementation"
        The whole file, `game/constants/keys.py`. `KEY_QUIT_GAME` and `KEY_EXIT` both map to `ESCAPE` but carry different names to express intent (quit the game vs close an overlay):

        ```python
        from __future__ import annotations

        import tcod.event


        _SPECIAL_KEY_NAMES = {
            tcod.event.KeySym.ESCAPE: "Esc",
        }

        def key_label(sym: tcod.event.KeySym) -> str:
            v    = int(sym)
            name = _SPECIAL_KEY_NAMES.get(sym) or (chr(v).upper() if 32 <= v <= 126 else sym.name)
            return f"[ {name} ]"


        # Part-8. Exercise 4: Centralise keybindings in game/constants/keys.py
        MOVE_KEYS = {
            # Arrow keys
            tcod.event.KeySym.UP:    ( 0, -1),
            tcod.event.KeySym.DOWN:  ( 0,  1),
            tcod.event.KeySym.LEFT:  (-1,  0),
            tcod.event.KeySym.RIGHT: ( 1,  0),

            # Numpad
            tcod.event.KeySym.KP_8:  ( 0, -1),
            tcod.event.KeySym.KP_2:  ( 0,  1),
            tcod.event.KeySym.KP_4:  (-1,  0),
            tcod.event.KeySym.KP_6:  ( 1,  0),

            # Part-1. Exercise 1: Add diagonal movement
            tcod.event.KeySym.KP_7:  (-1, -1),
            tcod.event.KeySym.KP_9:  ( 1, -1),
            tcod.event.KeySym.KP_1:  (-1,  1),
            tcod.event.KeySym.KP_3:  ( 1,  1),
        }

        WAIT_KEYS = {
            tcod.event.KeySym.PERIOD,
            tcod.event.KeySym.KP_5,
            tcod.event.KeySym.CLEAR,
        }

        KEY_PICKUP    = tcod.event.KeySym.G
        KEY_INVENTORY = tcod.event.KeySym.I
        KEY_DROP      = tcod.event.KeySym.D
        KEY_QUIT_GAME = tcod.event.KeySym.ESCAPE
        KEY_EXIT      = tcod.event.KeySym.ESCAPE

        # Part-7. Exercise 2: Scroll the message panel
        SCROLL_UP   = tcod.event.KeySym.PAGEUP
        SCROLL_DOWN = tcod.event.KeySym.PAGEDOWN

        # Part-8. Exercise 3: Persistent item keys
        HEALTH_POTION   = tcod.event.KeySym.H
        BACKPACK_SCROLL = tcod.event.KeySym.B
        ```

        In `game/entities/factories.py`, import `keys`, drop the now-unused `import tcod.event`, and point the item hotkeys at the constants:

        ```diff
        -from game.constants import colors, sprites
        +from game.constants import colors, keys, sprites
        -import tcod.event
        ```

        ```diff
        -    key        = tcod.event.KeySym.H,
        +    key        = keys.HEALTH_POTION,
        ...
        -    key        = tcod.event.KeySym.B,
        +    key        = keys.BACKPACK_SCROLL,
        ```

        In `game/game_states.py`, move the `MOVE_KEYS` and `WAIT_KEYS` dicts out to `keys.py`, import `keys`, and reference everything through it:

        ```diff
        -from game.constants import colors
        +from game.constants import colors, keys
        ```

        ```diff
        -if key in MOVE_KEYS:
        -    dx, dy = MOVE_KEYS[key]
        +if key in keys.MOVE_KEYS:
        +    dx, dy = keys.MOVE_KEYS[key]
        ```

        Then do the same for `keys.WAIT_KEYS` and every `keys.KEY_*` / `keys.SCROLL_*` check through the file.

---

*Finished with Part 8. Continue to [Part 9: Spells and Targeting](part-9.md).*
