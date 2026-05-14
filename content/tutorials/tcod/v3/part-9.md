# Part 9: Spells and Targeting

## Learning goals

- Add a targeting cursor the player moves with keyboard or mouse
- Implement three spell scrolls: lightning bolt (auto-target), confusion (cursor), fireball (area of effect)
- Show a visual radius ring for AoE targeting
- Add a `ConfusedEnemy` AI that wanders randomly

---

## Targeting as a game state

So far, every keypress either moves the player or triggers an action. Spells that require targeting need a different mode: the player moves a cursor around the map and confirms a target, or presses Escape to cancel.

This fits naturally into the handler-as-state-machine pattern from Part 8. When the player uses a targeting scroll, we push a new handler. That handler:

1. Intercepts all keyboard and mouse input
2. Draws the cursor (and for AoE, the radius ring) on every frame
3. On confirm, runs the action and returns to `MainGameEventHandler`
4. On Escape, cancels and returns to `MainGameEventHandler`

```txt
MainGameEventHandler
  │  player presses i → use scroll
  ▼
InventoryActivateHandler
  │  scroll.consumable.get_action() returns targeting handler
  ▼
SingleRangedAttackHandler  (or AreaRangedAttackHandler)
  │  player confirms target
  ▼
MainGameEventHandler  ← back to normal play
```

---

## Add entity.distance()

Fireball needs to measure distance from the explosion center to each entity. Add to `Entity`:

```python
def distance(self, x: int, y: int) -> float:
    return ((self.x - x) ** 2 + (self.y - y) ** 2) ** 0.5
```

---

## SelectIndexHandler: cursor base class

Add to `game/input_handlers.py`:

```python
class SelectIndexHandler(EventHandler):
    """Base for handlers that ask the player to select a map tile."""

    def __init__(self, engine: Engine) -> None:
        super().__init__(engine)
        player = self.engine.player
        engine.mouse_location = player.x, player.y

    def on_render(self, console: tcod.Console) -> None:
        super().on_render(console)
        x, y = self.engine.mouse_location
        if self.engine.game_map.in_bounds(x, y):
            console.rgb["bg"][x, y] = colors.WHITE
            console.rgb["fg"][x, y] = colors.BLACK

    def event_keydown(self, event: tcod.event.KeyDown) -> Action | None:
        key = event.sym
        if key in MOVE_KEYS:
            modifier = 1
            if event.mod & (tcod.event.Modifier.LSHIFT | tcod.event.Modifier.RSHIFT):
                modifier *= 5
            if event.mod & (tcod.event.Modifier.LCTRL | tcod.event.Modifier.RCTRL):
                modifier *= 10
            x, y = self.engine.mouse_location
            dx, dy = MOVE_KEYS[key]
            x = max(0, min(x + dx * modifier, self.engine.game_map.width - 1))
            y = max(0, min(y + dy * modifier, self.engine.game_map.height - 1))
            self.engine.mouse_location = x, y
            return None
        if key in (tcod.event.KeySym.RETURN, tcod.event.KeySym.KP_ENTER):
            return self.on_index_selected(*self.engine.mouse_location)
        if key == tcod.event.KeySym.ESCAPE:
            from game.input_handlers import MainGameEventHandler
            self.engine.event_handler = MainGameEventHandler(self.engine)
            return None
        return super().event_keydown(event)

    def event_mousebuttondown(self, event: tcod.event.MouseButtonDown) -> Action | None:
        if self.engine.game_map.in_bounds(*event.integer_position):
            if event.button == 1:
                return self.on_index_selected(*event.integer_position)
        return None

    def on_index_selected(self, x: int, y: int) -> Action | None:
        raise NotImplementedError()
```

The cursor starts at the player's position. Holding Shift multiplies movement speed by 5; holding Ctrl multiplies by 10, handy on large maps. Enter or left-click confirms the selection.

Escape cancels targeting and returns to normal gameplay.

---

## SingleRangedAttackHandler

```python
class SingleRangedAttackHandler(SelectIndexHandler):
    """Asks the player to select a single target tile."""

    def __init__(
        self,
        engine: Engine,
        callback,
    ) -> None:
        super().__init__(engine)
        self.callback = callback

    def on_index_selected(self, x: int, y: int) -> Action | None:
        return self.callback((x, y))
```

The `callback` is a function that accepts `(x, y)` and returns an `Action`. The consumable provides the callback when it creates the handler.

---

## AreaRangedAttackHandler

```python
class AreaRangedAttackHandler(SelectIndexHandler):
    """Shows an AoE radius ring and asks the player to confirm."""

    def __init__(
        self,
        engine: Engine,
        radius: int,
        callback,
    ) -> None:
        super().__init__(engine)
        self.radius = radius
        self.callback = callback

    def on_render(self, console: tcod.Console) -> None:
        super().on_render(console)
        x, y = self.engine.mouse_location
        diameter = self.radius * 2 + 1
        console.draw_frame(
            x=x - self.radius,
            y=y - self.radius,
            width=diameter,
            height=diameter,
            fg=colors.RED,
            bg=None,
            clear=False,
        )

    def on_index_selected(self, x: int, y: int) -> Action | None:
        return self.callback((x, y))
```

Add `RED` to `game/constants/colors.py`:

```python
RED = (0xFF, 0x0, 0x0)
```

`draw_frame` draws a rectangle outline. We pass `clear=False` so the tiles inside are not blanked, the frame is a visual indicator only.

!!! info "The radius ring is approximate"
    `draw_frame` draws a square, not a circle. A true circular highlight requires iterating every tile within radius. The square is a good-enough approximation for a tutorial, and is what the reference implementation uses.

---

## Three new consumables

### LightningDamageConsumable (auto-target nearest)

```python
class LightningDamageConsumable(Consumable):
    def __init__(self, damage: int, maximum_range: int) -> None:
        self.damage = damage
        self.maximum_range = maximum_range

    def activate(self, action, engine: Engine, consumer: Actor) -> None:
        target = None
        closest_distance = self.maximum_range + 1.0

        for actor in engine.game_map.actors:
            if actor is not consumer and engine.game_map.visible[actor.x, actor.y]:
                distance = consumer.distance(actor.x, actor.y)
                if distance < closest_distance:
                    target = actor
                    closest_distance = distance

        if target:
            MessageLog.add_message(
                f"A lightning bolt strikes the {target.name} for {self.damage} damage!",
                colors.PLAYER_ATTACK,
            )
            target.fighter.take_damage(self.damage)
            self.consume()
        else:
            raise Impossible("No enemy is close enough to strike.")
```

### ConfusionConsumable (cursor targeting)

```python
class ConfusionConsumable(Consumable):
    def __init__(self, number_of_turns: int) -> None:
        self.number_of_turns = number_of_turns

    def get_action(self, consumer: Actor, engine: Engine):
        MessageLog.add_message(
            "Select a target location.", colors.NEEDS_TARGET
        )
        from game.input_handlers import SingleRangedAttackHandler
        engine.event_handler = SingleRangedAttackHandler(
            engine,
            callback=lambda xy: ItemAction(item=self.entity, target_xy=xy),
        )
        return None

    def activate(self, action, engine: Engine, consumer: Actor) -> None:
        target = engine.game_map.get_actor_at_location(*action.target_xy)

        if not target:
            raise Impossible("You must select an enemy to target.")
        if not engine.game_map.visible[target.x, target.y]:
            raise Impossible("You cannot target an area you cannot see.")
        if target is consumer:
            raise Impossible("You cannot confuse yourself!")

        MessageLog.add_message(
            f"The eyes of the {target.name} look vacant, as it starts to stumble around!",
            colors.STATUS_EFFECT_APPLIED,
        )
        target.ai = ConfusedEnemy(
            entity=target,
            previous_ai=target.ai,
            turns_remaining=self.number_of_turns,
        )
        self.consume()
```

`InventoryActivateHandler.on_item_selected` (from Part 8) calls `consumable.get_action()` when the player selects an item. Targeting consumables override it to install the cursor handler on the engine and return `None` — no action is performed yet. The actual `ItemAction` is built by the callback once the player confirms a target.

Add colors to `game/constants/colors.py`:

```python
NEEDS_TARGET = (0x3F, 0xFF, 0xFF)
STATUS_EFFECT_APPLIED = (0x3F, 0xFF, 0x3F)
```

`ItemAction` needs a `target_xy` parameter. Update `game/actions.py`:

```python
class ItemAction(Action):
    def __init__(self, item: Item, target_xy=None) -> None:
        super().__init__()
        self.item = item
        self.target_xy = target_xy

    def perform(self, engine: Engine, entity: Entity) -> None:
        self.item.consumable.activate(self, engine, entity)
```

### FireballDamageConsumable (AoE cursor)

```python
class FireballDamageConsumable(Consumable):
    def __init__(self, damage: int, radius: int) -> None:
        self.damage = damage
        self.radius = radius

    def get_action(self, consumer: Actor, engine: Engine):
        MessageLog.add_message(
            "Select a target location.", colors.NEEDS_TARGET
        )
        from game.input_handlers import AreaRangedAttackHandler
        engine.event_handler = AreaRangedAttackHandler(
            engine,
            radius=self.radius,
            callback=lambda xy: ItemAction(item=self.entity, target_xy=xy),
        )
        return None

    def activate(self, action, engine: Engine, consumer: Actor) -> None:
        target_xy = action.target_xy

        if not engine.game_map.visible[target_xy]:
            raise Impossible("You cannot target an area you cannot see.")

        targets_hit = False
        for actor in engine.game_map.actors:
            if actor.distance(*target_xy) <= self.radius:
                MessageLog.add_message(
                    f"The {actor.name} is engulfed in a fiery explosion,"
                    f" taking {self.damage} damage!"
                )
                actor.fighter.take_damage(self.damage)
                targets_hit = True

        if not targets_hit:
            raise Impossible("There are no targets in the radius.")
        self.consume()
```

---

## ConfusedEnemy AI

Add to `game/entities/components/ai.py`:

```python
import random

from game.message_log import MessageLog

class ConfusedEnemy(BaseAI):
    def __init__(
        self,
        entity: Actor,
        previous_ai: BaseAI | None,
        turns_remaining: int,
    ) -> None:
        super().__init__()
        self.entity = entity
        self.previous_ai = previous_ai
        self.turns_remaining = turns_remaining

    def perform(self, engine: Engine, entity: Actor) -> None:
        if self.turns_remaining <= 0:
            MessageLog.add_message(
                f"The {entity.name} is no longer confused."
            )
            entity.ai = self.previous_ai
        else:
            direction_x, direction_y = random.choice(
                [
                    (-1, -1), (0, -1), (1, -1),
                    (-1,  0),          (1,  0),
                    (-1,  1), (0,  1), (1,  1),
                ]
            )
            self.turns_remaining -= 1
            from game.actions import BumpAction
            BumpAction(direction_x, direction_y).perform(engine, entity)
```

A confused enemy picks a random direction each turn. It can still accidentally attack the player if it bumps into them, this is a feature, not a bug. When the confusion expires it restores its previous AI.

---

## Adding scroll constants

Three new sprites and three new colors. All scrolls share the same `~` glyph but each gets its own named constant so they can be retextured independently.

Extend `game/constants/sprites.py`:

```diff
 PLAYER = "@"
 ORC = "o"
 TROLL = "T"

 CORPSE = "%"

 HEALTH_POTION = "!"
+
+CONFUSION_SCROLL = "~"
+FIREBALL_SCROLL = "~"
+LIGHTNING_SCROLL = "~"
```

Extend `game/constants/colors.py`:

```diff
 PLAYER = (255, 255, 255)
 ORC = (63, 127, 63)
 TROLL = (0, 127, 0)

 CORPSE = (191, 0, 0)

 HEALTH_POTION = (127, 0, 255)
+
+CONFUSION_SCROLL = (207, 63, 255)
+FIREBALL_SCROLL = (255, 0, 0)
+LIGHTNING_SCROLL = (255, 255, 0)
```

---

## game/entities/factories.py: add scrolls

```python
from game.entities.components.consumable import (
    ConfusionConsumable,
    FireballDamageConsumable,
    HealingConsumable,
    LightningDamageConsumable,
)
from game.constants import colors, sprites

confusion_scroll = Item(
    char=sprites.CONFUSION_SCROLL,
    color=colors.CONFUSION_SCROLL,
    name="Confusion Scroll",
    consumable=ConfusionConsumable(number_of_turns=10),
)

fireball_scroll = Item(
    char=sprites.FIREBALL_SCROLL,
    color=colors.FIREBALL_SCROLL,
    name="Fireball Scroll",
    consumable=FireballDamageConsumable(damage=12, radius=3),
)

lightning_scroll = Item(
    char=sprites.LIGHTNING_SCROLL,
    color=colors.LIGHTNING_SCROLL,
    name="Lightning Scroll",
    consumable=LightningDamageConsumable(damage=20, maximum_range=5),
)
```

---

## Update the map generator to spawn scrolls

Update `place_entities()` to pick randomly from several item types:

```python
item_chances = [
    (factories.health_potion, 35),
    (factories.confusion_scroll, 10),
    (factories.lightning_scroll, 25),
    (factories.fireball_scroll, 25),
]

def place_entities(room, dungeon, min_monsters, max_monsters, min_items, max_items):
    ...
    for _ in range(number_of_items):
        x = random.randint(room.x1 + 1, room.x2 - 1)
        y = random.randint(room.y1 + 1, room.y2 - 1)
        if not any(entity.x == x and entity.y == y for entity in dungeon.entities):
            chosen = random.choices(
                [item for item, _ in item_chances],
                weights=[w for _, w in item_chances],
            )
            chosen[0].spawn(dungeon, x, y)
```

`random.choices` with weights handles the probability table in one line.

---

## Update EventHandler.handle_events

The targeting handlers respond to mouse input. Update `EventHandler.handle_events()` to dispatch mouse button clicks, and to restore the main game handler after an inventory or targeting action resolves:

```python
class EventHandler:
    def handle_events(self, event: tcod.event.Event) -> None:
        action: Action | None = None
        match event:
            case tcod.event.Quit():
                action = EscapeAction()
            case tcod.event.MouseMotion():
                self.engine.mouse_location = event.integer_position
            case tcod.event.MouseButtonDown():
                action = self.event_mousebuttondown(event)
            case tcod.event.KeyDown():
                action = self.event_keydown(event)

        if action is not None:
            try:
                action.perform(self.engine, self.engine.player)
            except Impossible as exc:
                MessageLog.add_message(str(exc), colors.INVALID)
                return

            if self.engine.player.is_alive:
                self.engine.handle_enemy_turns()

            if not self.engine.player.is_alive:
                from game.input_handlers import GameOverEventHandler
                self.engine.event_handler = GameOverEventHandler(self.engine)

            elif isinstance(
                self.engine.event_handler,
                (InventoryActivateHandler, InventoryDropHandler, SelectIndexHandler),
            ):
                self.engine.event_handler = MainGameEventHandler(self.engine)

            self.engine.update_fov()

    def event_mousebuttondown(self, event: tcod.event.MouseButtonDown) -> Action | None:
        return None
```

---

## Testing your work

Run `python main.py`:

- [ ] `~` scrolls appear on the floor in different colors (purple, red, yellow)
- [ ] Picking up a lightning scroll and pressing `i → a` strikes the nearest enemy
- [ ] Picking up a confusion scroll opens a targeting cursor (blue highlight on player tile)
- [ ] Moving the cursor to an enemy and pressing Enter confuses it; it wanders randomly for ~10 turns
- [ ] A fireball scroll opens AoE targeting with a red square border
- [ ] The fireball damages all actors (including the player!) within the radius
- [ ] Pressing Escape during targeting cancels and returns to normal play
- [ ] Using a scroll removes it from the inventory

!!! danger "Fireball can hit you"
    The fireball does not exempt the player from its AoE, this is intentional. Don't stand in the blast radius.

---

## Summary

The targeting system is now in place. Key additions:

- **`SelectIndexHandler`**: cursor movement + confirm/cancel
- **`SingleRangedAttackHandler`** and **`AreaRangedAttackHandler`**: targeting modes
- **`get_action()`**: consumables can push their own handler instead of returning an action immediately
- **`ConfusedEnemy`**: temporary AI swap with countdown
- Three scroll types covering auto-target, single-target, and AoE

**Current architecture**:

- Targeting handlers are temporary input states layered on top of normal gameplay
- Consumables can return an action immediately or push a targeting handler first
- `ItemAction` carries both the selected item and optional target coordinates
- AI can be swapped at runtime, as with `ConfusedEnemy`
- Existing inventory and action systems now support targeted effects

**File structure**:

```txt
main.py
game/
├── __init__.py
├── actions.py                  ← modified
├── engine.py
├── exceptions.py
├── hud.py
├── input_handlers.py           ← modified
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
│       ├── base_component.py
│       ├── consumable.py       ← modified
│       ├── fighter.py
│       └── inventory.py
└── map/
    ├── __init__.py
    ├── game_map.py
    ├── tile_types.py
    └── map_generator.py        ← modified
```

---

## Exercises

1. **Teleport scroll**:

    Add a `TeleportConsumable` that uses `SingleRangedAttackHandler` to let the player pick any visible tile and teleport to it. The player should not be able to teleport into walls.

2. **Scroll of mapping**:

    Add a consumable that sets `game_map.explored` to `True` for every tile, revealing the whole floor. No targeting needed, use the base `get_action()` directly.

3. **Confusion self-damage**:

    Modify `ConfusedEnemy` so that on each wandering move, there is a 20% chance the entity also takes 1 point of damage (it's stumbling into walls). Add a message: `"The Orc stumbles into a wall!"`.

**Next**: Part 10: Save and Load
