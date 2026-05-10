# Part 8: Items and Inventory

## Learning goals

- Create an `Item` entity class and a `HealingConsumable` component
- Add an `Inventory` component to the player
- Implement pickup, drop, and use actions
- Build an inventory menu the player can navigate with letter keys
- Spawn health potions in the dungeon

---

## The item data model

Before writing code, sketch the data model:

```txt
Actor (player)
  └── Inventory (component)
        └── items: list[Item]
              ├── Item ("Health Potion")
              │     └── HealingConsumable (component)
              └── Item ("Health Potion")
                    └── HealingConsumable (component)
```

An `Item` is just an `Entity` with a `Consumable` component attached. The `Inventory` component on the player holds the list of carried items. This keeps the data flat and consistent with how `Fighter` and `AI` components work.

---

## exceptions.py

Actions can fail, the player tries to pick up when there is nothing there, or tries to drink a potion at full HP. Instead of silently ignoring these, we raise an `Impossible` exception that the event handler catches and adds to the message log.

Create `game/exceptions.py`:

```python
class Impossible(Exception):
    """Exception raised when an action is impossible.

    The reason is the exception message.
    """
```

That is the entire file. The exception message *is* the UI message, callers write `raise Impossible("Your inventory is full.")` and the handler shows it to the player.

---

## components/consumable.py

A consumable component defines what happens when an item is *used*. We start with a single type: healing.

Create `game/components/consumable.py`:

```python
from __future__ import annotations

from typing import TYPE_CHECKING

from game.components.base_component import BaseComponent
from game.exceptions import Impossible
from game.constants import colors
from game.message_log import MessageLog

if TYPE_CHECKING:
    from game.actions import Action, ItemAction
    from game.engine import Engine
    from game.entity import Actor, Item


class Consumable(BaseComponent):
    def get_action(self, consumer: Actor, engine: Engine) -> Action | None:
        """Return the action this consumable produces when used."""
        from game.actions import ItemAction
        return ItemAction(item=self.entity)

    def activate(self, action: ItemAction, engine: Engine, consumer: Actor) -> None:
        """Apply this consumable's effect. Must be overridden."""
        raise NotImplementedError()

    def consume(self) -> None:
        """Remove the item from the holder's inventory."""
        entity = self.entity
        inventory = entity.parent
        if inventory is not None and entity in inventory.items:
            inventory.items.remove(entity)


class HealingConsumable(Consumable):
    def __init__(self, amount: int) -> None:
        self.amount = amount

    def activate(self, action, engine: Engine, consumer: Actor) -> None:
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

`consume()` removes the item from the inventory after use. `activate()` raises `Impossible` if the item has no effect, drinking a potion at full HP is wasted, and we tell the player.

Add `HEALTH_RECOVERED` to `game/constants/colors.py` in the combat message colors section:

```python
HEALTH_RECOVERED = (0x0, 0xFF, 0x0)
```

---

## components/inventory.py

Create `game/components/inventory.py`:

```python
from __future__ import annotations

from typing import TYPE_CHECKING

from game.components.base_component import BaseComponent

if TYPE_CHECKING:
    from game.entity import Actor, Item


class Inventory(BaseComponent):
    def __init__(self, capacity: int) -> None:
        self.capacity = capacity
        self.items: list[Item] = []

    def drop(self, item: Item) -> None:
        self.items.remove(item)
        item.place(self.entity.x, self.entity.y, self.entity.game_map)
```

We will finish `drop` once we add `place()` to `Entity`.

---

## Item entity class and Entity.place()

Items need a `parent` reference so `consume()` can find the inventory they belong to. We also add `place()` for moves that change ownership.

`set_position()` only changes coordinates. `place()` changes coordinates and also says where the entity now lives. That distinction matters for items: when an item is dropped, it moves from an `Inventory` back to the `GameMap`.

Update `game/entity.py`:

```python
from __future__ import annotations

import copy
from typing import TYPE_CHECKING

from game.render_order import RenderOrder

if TYPE_CHECKING:
    from game.components.ai import BaseAI
    from game.components.consumable import Consumable
    from game.components.fighter import Fighter
    from game.components.inventory import Inventory
    from game.map.game_map import GameMap

from game.constants import colors, sprites


class Entity:
    parent: GameMap  # set when the entity is placed on a map

    def __init__(
        self,
        parent: GameMap | None = None,
        x: int = 0,
        y: int = 0,
        char: str = sprites.UNKNOWN,
        color: tuple[int, int, int] = colors.DEFAULT_FG,
        name: str = "<unnamed>",
        blocks_movement: bool = False,
        stays_visible: bool = False,
        render_order: RenderOrder = RenderOrder.CORPSE,
    ) -> None:
        self.parent = parent
        self.x = x
        self.y = y
        self.char = char
        self.color = color
        self.name = name
        self.blocks_movement = blocks_movement
        self.stays_visible = stays_visible
        self.render_order = render_order
        if parent:
            parent.entities.add(self)

    @property
    def game_map(self) -> GameMap:
        if isinstance(self.parent, GameMap):
            return self.parent
        return self.parent.game_map

    def spawn(self, game_map: GameMap, x: int, y: int) -> Entity:
        clone = copy.deepcopy(self)
        clone.x = x
        clone.y = y
        clone.parent = game_map
        game_map.entities.add(clone)
        return clone

    def place(self, x: int, y: int, game_map: GameMap | None = None) -> None:
        self.x = x
        self.y = y
        if game_map:
            if hasattr(self, "parent"):
                if self.parent is self.game_map:
                    self.game_map.entities.discard(self)
            self.parent = game_map
            game_map.entities.add(self)

    def set_position(self, x: int, y: int) -> None:
        self.x = x
        self.y = y

    def move(self, dx: int, dy: int) -> None:
        self.x += dx
        self.y += dy


class Actor(Entity):
    def __init__(
        self,
        *,
        x: int = 0,
        y: int = 0,
        char: str = sprites.UNKNOWN,
        color: tuple[int, int, int] = colors.DEFAULT_FG,
        name: str = "<unnamed>",
        ai: BaseAI | None = None,
        fighter: Fighter,
        inventory: Inventory,
    ) -> None:
        super().__init__(
            x=x,
            y=y,
            char=char,
            color=color,
            name=name,
            blocks_movement=True,
            render_order=RenderOrder.ACTOR,
        )
        self.fighter = fighter
        self.fighter.entity = self

        self.inventory = inventory
        self.inventory.entity = self

        if ai:
            self.ai: BaseAI | None = ai
            self.ai.entity = self
        else:
            self.ai = None

    @property
    def is_alive(self) -> bool:
        return self.fighter.hp > 0


class Item(Entity):
    def __init__(
        self,
        *,
        x: int = 0,
        y: int = 0,
        char: str = sprites.UNKNOWN,
        color: tuple[int, int, int] = colors.DEFAULT_FG,
        name: str = "<unnamed>",
        consumable: Consumable,
    ) -> None:
        super().__init__(
            x=x,
            y=y,
            char=char,
            color=color,
            name=name,
            blocks_movement=False,
            render_order=RenderOrder.ITEM,
        )
        self.consumable = consumable
        self.consumable.entity = self
```

!!! info "parent vs game_map"
    For entities on the map, `parent` *is* the `GameMap`. For items in an inventory, `parent` will be the `Inventory` component. The `game_map` property navigates up: `entity.game_map` works whether the entity is on the floor or in a bag, because `Inventory.game_map` delegates to its owning actor's game_map.

---

## Complete components/inventory.py

```python
from __future__ import annotations

from typing import TYPE_CHECKING

from game.components.base_component import BaseComponent
from game.exceptions import Impossible

from game.constants import colors

if TYPE_CHECKING:
    from game.entity import Actor, Item


class Inventory(BaseComponent):
    def __init__(self, capacity: int) -> None:
        self.capacity = capacity
        self.items: list[Item] = []

    @property
    def game_map(self):
        return self.entity.game_map

    def drop(self, item: Item) -> None:
        self.items.remove(item)
        item.place(self.entity.x, self.entity.y, self.entity.parent)
```

---

## New actions in actions.py

Three new actions:

- **PickupAction**: take an item from the floor
- **ItemAction**: use an item from the inventory
- **DropItem**: remove an item from the inventory and drop it

Add to `game/actions.py`:

```python
from game.exceptions import Impossible
from game.message_log import MessageLog

class PickupAction(Action):
    def perform(self, engine: Engine, entity: Entity) -> None:
        actor_location_x = entity.x
        actor_location_y = entity.y
        inventory = entity.inventory

        for item in engine.game_map.items:
            if actor_location_x == item.x and actor_location_y == item.y:
                if len(inventory.items) >= inventory.capacity:
                    raise Impossible("Your inventory is full.")

                engine.game_map.entities.discard(item)
                item.parent = inventory
                inventory.items.append(item)

                MessageLog.add_message(f"You picked up the {item.name}!")
                return

        raise Impossible("There is nothing here to pick up.")


class ItemAction(Action):
    def __init__(self, item: Item) -> None:
        super().__init__()
        self.item = item

    def perform(self, engine: Engine, entity: Entity) -> None:
        self.item.consumable.activate(self, engine, entity)


class DropItem(ItemAction):
    def perform(self, engine: Engine, entity: Entity) -> None:
        entity.inventory.drop(self.item)
        MessageLog.add_message(f"You dropped the {self.item.name}.")
```

`GameMap` needs an `items` property (parallel to `actors`). Add to `game/map/game_map.py`:

```python
from game.entity import Actor, Item

@property
def items(self) -> Iterator[Item]:
    yield from (e for e in self.entities if isinstance(e, Item))
```

---

## Inventory menu in input_handlers.py

The inventory UI is a modal overlay. When the player presses `i`, a new handler takes over, shows a letter-keyed list of items, and returns to `MainGameEventHandler` after a selection.

Add to `game/input_handlers.py`:

```python
from game.constants import colors
from game.exceptions import Impossible
from game.message_log import MessageLog


class InventoryEventHandler(EventHandler):
    """Base class for inventory screens (use and drop share the same UI)."""

    TITLE = "<missing title>"

    def on_render(self, console: tcod.Console) -> None:
        super().on_render(console)  # draws the map behind the overlay

        number_of_items_in_inventory = len(self.engine.player.inventory.items)

        height = number_of_items_in_inventory + 2
        if height <= 3:
            height = 3

        x = 5
        y = 0

        width = len(self.TITLE) + 4

        console.draw_frame(
            x=x,
            y=y,
            width=width,
            height=height,
            title=self.TITLE,
            clear=True,
            fg=colors.WHITE,
            bg=colors.BLACK,
        )

        if number_of_items_in_inventory > 0:
            for i, item in enumerate(self.engine.player.inventory.items):
                item_key = chr(ord("a") + i)
                console.print(x + 1, y + i + 1, f"({item_key}) {item.name}")
        else:
            console.print(x + 1, y + 1, "(Empty)")

    def event_keydown(self, event: tcod.event.KeyDown) -> Action | None:
        player = self.engine.player
        key = event.sym
        index = key - tcod.event.KeySym.a

        if 0 <= index <= 26:
            try:
                selected_item = player.inventory.items[index]
            except IndexError:
                MessageLog.add_message("Invalid entry.", colors.INVALID)
                return None
            return self.on_item_selected(selected_item)
        return super().event_keydown(event)

    def on_item_selected(self, item) -> Action | None:
        raise NotImplementedError()


class InventoryActivateHandler(InventoryEventHandler):
    TITLE = "Select an item to use"

    def on_item_selected(self, item) -> Action | None:
        from game.actions import ItemAction
        return ItemAction(item=item)


class InventoryDropHandler(InventoryEventHandler):
    TITLE = "Select an item to drop"

    def on_item_selected(self, item) -> Action | None:
        from game.actions import DropItem
        return DropItem(item=item)
```

Add `INVALID` to `game/constants/colors.py` in the UI colors section:

```python
INVALID = (0xFF, 0xFF, 0x00)
```

Add keybindings to `MainGameEventHandler`:

```python
from game.actions import PickupAction

class MainGameEventHandler(EventHandler):
    def event_keydown(self, event: tcod.event.KeyDown) -> Action | None:
        key = event.sym

        if key in MOVE_KEYS:
            dx, dy = MOVE_KEYS[key]
            return BumpAction(dx, dy)
        if key in WAIT_KEYS:
            return WaitAction()
        if key == tcod.event.KeySym.ESCAPE:
            return EscapeAction()
        if key == tcod.event.KeySym.g:
            return PickupAction()
        if key == tcod.event.KeySym.i:
            self.engine.event_handler = InventoryActivateHandler(self.engine)
        if key == tcod.event.KeySym.d:
            self.engine.event_handler = InventoryDropHandler(self.engine)

        return None
```

!!! info "Handlers switch handlers"
    When the player presses `i`, we don't return an `Action`, we swap `engine.event_handler` to the inventory screen. The next frame renders the inventory overlay. After the player picks an item (or presses Escape), the handler switches back to `MainGameEventHandler`. This is the handler-as-state-machine pattern we will expand in Parts 9 and 10.

After an inventory action resolves, the handler must return to the main game. Update `EventHandler.handle_events` to restore the main handler after an action is performed:

```python
class EventHandler:
    def handle_events(self, event: tcod.event.Event) -> Action | None:
        action = None
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
            except Impossible as exc:
                MessageLog.add_message(str(exc), colors.INVALID)
                return None

            if self.engine.player.is_alive:
                self.engine.handle_enemy_turns()

            if not self.engine.player.is_alive:
                from game.input_handlers import GameOverEventHandler
                self.engine.event_handler = GameOverEventHandler(self.engine)
            elif isinstance(self.engine.event_handler, (InventoryActivateHandler, InventoryDropHandler)):
                self.engine.event_handler = MainGameEventHandler(self.engine)

            self.engine.update_fov()

        return None
```

!!! info "Moving perform() into the handler"
    We moved `action.perform()` into the event handler so we can catch `Impossible` exceptions in one place.

Update `Engine.handle_events()` so it only passes events to the active handler:

```python
def handle_events(self, events: tcod.event.EventLike) -> None:
    for event in events:
        self.event_handler.handle_events(event)
```

---

## Adding the health potion to constants

Following our convention, register the new sprite and color before using them.

Extend `game/constants/sprites.py`:

```diff
 PLAYER = "@"
 ORC = "o"
 TROLL = "T"

 CORPSE = "%"
+
+HEALTH_POTION = "!"
```

Extend `game/constants/colors.py` in the entity colors section:

```diff
 PLAYER = (255, 255, 255)
 ORC = (63, 127, 63)
 TROLL = (0, 127, 0)

 CORPSE = (191, 0, 0)
+
+HEALTH_POTION = (127, 0, 255)
```

---

## entity_factories.py: add health potion

Update `game/entity_factories.py`:

```python
from __future__ import annotations

from game.components.consumable import HealingConsumable
from game.components.inventory import Inventory
from game.constants import colors, sprites
from game.entity import Actor, Item

player = Actor(
    char=sprites.PLAYER,
    color=colors.PLAYER,
    name="Player",
    ai=None,
    fighter=Fighter(hp=30, defense=2, attack=5),
    inventory=Inventory(capacity=26),
)

orc = Actor(
    char=sprites.ORC,
    color=colors.ORC,
    name="Orc",
    ai=HostileEnemy(),
    fighter=Fighter(hp=10, defense=0, attack=3),
    inventory=Inventory(capacity=0),
)

troll = Actor(
    char=sprites.TROLL,
    color=colors.TROLL,
    name="Troll",
    ai=HostileEnemy(),
    fighter=Fighter(hp=16, defense=1, attack=4),
    inventory=Inventory(capacity=0),
)

health_potion = Item(
    char=sprites.HEALTH_POTION,
    color=colors.HEALTH_POTION,
    name="Health Potion",
    consumable=HealingConsumable(amount=4),
)
```

Enemies get `Inventory(capacity=0)`, they carry nothing, but the attribute must exist since `Actor` requires it.

---

## Spawn potions in the map generator

Update `place_entities()` in `game/map/map_generator.py`:

```python
from game import entity_factories

def place_entities(
    room: RectangularRoom,
    dungeon: GameMap,
    maximum_monsters: int,
    maximum_items: int,
) -> None:
    number_of_monsters = random.randint(0, maximum_monsters)
    number_of_items = random.randint(0, maximum_items)

    for _ in range(number_of_monsters):
        x = random.randint(room.x1 + 1, room.x2 - 1)
        y = random.randint(room.y1 + 1, room.y2 - 1)
        if not any(entity.x == x and entity.y == y for entity in dungeon.entities):
            if random.random() < 0.8:
                entity_factories.orc.spawn(dungeon, x, y)
            else:
                entity_factories.troll.spawn(dungeon, x, y)

    for _ in range(number_of_items):
        x = random.randint(room.x1 + 1, room.x2 - 1)
        y = random.randint(room.y1 + 1, room.y2 - 1)
        if not any(entity.x == x and entity.y == y for entity in dungeon.entities):
            entity_factories.health_potion.spawn(dungeon, x, y)
```

Update `generate_dungeon()` to pass `maximum_items`. While we are here, use `place()` for the player's initial map placement. The player is already in `dungeon.entities`, but `place()` also sets `player.parent` to the dungeon. That ownership link is what lets `Inventory.drop()` find the correct map later.

```diff
 def generate_dungeon(
     max_rooms: int,
     room_min_size: int,
     room_max_size: int,
     map_width: int,
     map_height: int,
     max_monsters_per_room: int,
+    max_items_per_room: int,
     player: Entity,
 ) -> GameMap:
     ...
         if not rooms:
-            player.set_position(*new_room.center)
+            player.place(*new_room.center, dungeon)
         ...
         place_entities(room, dungeon, max_monsters_per_room, max_items_per_room)
```

In `main.py`, add `max_items_per_room = 2` and pass it to `generate_dungeon`.

---

## Testing your work

Run `python main.py`:

- [ ] Health potions (`!` in purple) appear on the dungeon floor
- [ ] Walking over a potion and pressing `g` picks it up: `"You picked up the Health Potion!"`
- [ ] Pressing `i` opens an inventory overlay: `"(a) Health Potion"`
- [ ] Pressing `a` in the inventory consumes the potion and heals you
- [ ] Drinking a potion at full HP shows `"Your health is already full."`
- [ ] Pressing `d` then `a` drops the potion back on the floor
- [ ] Pressing an invalid key in the inventory shows `"Invalid entry."` in yellow

---

## Summary

Items and inventory are now complete. Key additions:

- **`Impossible` exception**: clean way to reject invalid actions with a message
- **`Consumable` + `HealingConsumable`**: component-based item behavior
- **`Inventory`**: component on the player storing carried items
- **`Item`**: Entity subclass with a consumable component
- **`PickupAction`, `ItemAction`, `DropItem`**: the three item verbs
- **Inventory overlay**: modal screen with letter-key item selection

**Current architecture**:

- `Item`: specialized entity for objects that can be picked up or used
- `Consumable`: component that defines what an item does when activated
- `Inventory`: component that owns carried items
- Entity ownership now matters: map entities live on `GameMap`, carried items live in `Inventory`
- Inventory handlers temporarily replace normal gameplay input

**Files created**: `game/exceptions.py`, `game/components/consumable.py`, `game/components/inventory.py`

**Files modified**: `game/entity.py`, `game/entity_factories.py`, `game/actions.py`, `game/map/game_map.py`, `game/input_handlers.py`, `game/map/map_generator.py`, `game/constants/sprites.py`, `game/constants/colors.py`, `main.py`

---

## Exercises

1. **Inventory full message.** The capacity is 26 (one slot per letter). Try to pick up a 27th item, you should see `"Your inventory is full."` Verify this works.

2. **Rename on pickup.** Some roguelikes call potions `"Red Potion"` until you identify them. After pickup, rename the item to `"Health Potion (identified)"`. Modify `PickupAction` to append `" (identified)"` to the item name.

3. **Stack display.** If the player has two health potions, the inventory shows them as separate lines `(a)` and `(b)`. Modify `InventoryEventHandler.on_render` to group identical items and show `"Health Potion ×2"` instead.

**Next**: Part 9: Spells and Targeting
