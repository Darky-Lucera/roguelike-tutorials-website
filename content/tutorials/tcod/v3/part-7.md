# Part 7: The User Interface

## Learning goals

- Divide the screen into a map area and a UI panel
- Add a health bar that updates in real time
- Introduce a message log that replaces `print()` calls
- Show entity names under the mouse cursor
- Refactor event handlers so each one owns its rendering logic

---

## What information does a player need?

A roguelike UI has one design constraint: every piece of information the player needs to make a decision must be visible *without* leaving the main screen. That means:

- **Health**: how close am I to death?
- **Recent events**: what just happened? (especially damage numbers)
- **Context**: what is this tile or entity I'm hovering over?

We dedicate 45 rows to the map and 5 rows to a UI panel at the bottom. That is enough for a health bar and a short message history.

```txt
┌──────────────────────────────────────────────────────────────────────────────┐
│                                                                              │
│                           MAP AREA  (80 × 45)                                │
│                                                                              │
├──────────────────────────────────────────────────────────────────────────────┤
│ HP: ████████░░  14/30    You attack the Orc for 3 hit points.                │
│                          The Orc attacks you for 2 hit points.               │
│                          The Troll is dead!                                  │
└──────────────────────────────────────────────────────────────────────────────┘
```

---

## Extending `game/constants/colors.py` with UI colors

`game/constants/colors.py` already holds entity colors (`PLAYER`, `ORC`, `TROLL`, `CORPSE` from Parts 5 and 6). The UI we are about to build needs more constants: combat message colors, the welcome banner, and the health bar's filled and empty states.

Extend `game/constants/colors.py`:

```diff
 PLAYER = (255, 255, 255)
 ORC = (63, 127, 63)
 TROLL = (0, 127, 0)

 CORPSE = (191, 0, 0)
+
+# Generic colors
+WHITE = (0xFF, 0xFF, 0xFF)
+BLACK = (0x0, 0x0, 0x0)
+
+# Combat message colors
+PLAYER_ATTACK = (0xE0, 0xE0, 0xE0)
+ENEMY_ATTACK = (0xFF, 0xC0, 0xC0)
+PLAYER_DEATH = (0xFF, 0x30, 0x30)
+ENEMY_DEATH = (0xFF, 0xA0, 0x30)
+
+# UI colors
+WELCOME_TEXT = (0x20, 0xA0, 0xFF)
+BAR_TEXT = WHITE
+BAR_FILLED = (0x0, 0x60, 0x0)
+BAR_EMPTY = (0x40, 0x10, 0x10)
```

We split the new constants into three sections (`Generic colors`, `Combat message colors`, `UI colors`) to make scanning the file easier as it grows. Notice we name the death colors `PLAYER_DEATH` and `ENEMY_DEATH`, not `_die`: full words read better at every call site.

!!! question "Why this lives in `constants/`, not in `engine.py`"
    `game/constants/colors.py` is imported by `render_functions`, `message_log`, `fighter`, and eventually every UI component. Putting colors in `game/engine.py` would force circular imports, those modules would need to import `engine` just for a tuple.

---

## message_log.py

The message log stores recent events and renders them in the panel. Two features make it feel polished: **color** (attacks are different from deaths) and **stacking** (if the same message repeats, it shows `(×3)` instead of three identical lines).

Create `game/message_log.py`:

```python
from __future__ import annotations

from collections.abc import Iterable, Reversible
import textwrap

import tcod

from game.constants import colors


class Message:
    def __init__(self, text: str, fg: tuple[int, int, int]) -> None:
        self.plain_text = text
        self.fg = fg
        self.count = 1

    @property
    def full_text(self) -> str:
        if self.count > 1:
            return f"{self.plain_text} (x{self.count})"
        return self.plain_text


class MessageLog:
    def __init__(self) -> None:
        self.messages: list[Message] = []

    def add_message(
        self,
        text: str,
        fg: tuple[int, int, int] = colors.WHITE,
        *,
        stack: bool = True,
    ) -> None:
        if stack and self.messages and self.messages[-1].plain_text == text:
            self.messages[-1].count += 1
        else:
            self.messages.append(Message(text, fg))

    def render(
        self,
        console: tcod.Console,
        x: int,
        y: int,
        width: int,
        height: int,
    ) -> None:
        self.render_messages(console, x, y, width, height, self.messages)

    @staticmethod
    def render_messages(
        console: tcod.Console,
        x: int,
        y: int,
        width: int,
        height: int,
        messages: Reversible[Message],
    ) -> None:
        y_offset = height - 1

        for message in reversed(messages):
            for line in reversed(textwrap.wrap(message.full_text, width)):
                console.print(x=x, y=y + y_offset, text=line, fg=message.fg)
                y_offset -= 1
                if y_offset < 0:
                    return
```

The render loop walks messages from newest to oldest (via `reversed`), and wraps each one to fit the panel width. It fills the panel from the bottom up (`y_offset` counts down). When the panel is full it returns early.

!!! tip "Why render bottom-up?"
    The most recent message should appear at the bottom of the log, like a terminal. Rendering from the bottom up naturally fills the visible area with the newest messages, and older messages scroll off the top.

---

## render_functions.py

Two standalone rendering helpers that the engine calls each frame.

Create `game/render_functions.py`:

```python
from __future__ import annotations

from typing import TYPE_CHECKING

import tcod

from game.constants import colors

if TYPE_CHECKING:
    from tcod import Console
    from game.engine import Engine


def render_bar(
    console: Console,
    current_value: int,
    maximum_value: int,
    total_width: int,
    y: int = 45,
) -> None:
    bar_width = int(float(current_value) / maximum_value * total_width)

    console.draw_rect(x=0, y=y, width=total_width, height=1, ch=1, bg=colors.BAR_EMPTY)

    if bar_width > 0:
        console.draw_rect(
            x=0, y=y, width=bar_width, height=1, ch=1, bg=colors.BAR_FILLED
        )

    console.print(
        x=1,
        y=y,
        text=f"HP: {current_value}/{maximum_value}",
        fg=colors.BAR_TEXT,
    )


def render_names_at_mouse_location(
    console: Console, x: int, y: int, engine: Engine
) -> None:
    mouse_x, mouse_y = engine.mouse_location

    if engine.game_map.in_bounds(mouse_x, mouse_y) and engine.game_map.visible[
        mouse_x, mouse_y
    ]:
        names = ", ".join(
            entity.name
            for entity in engine.game_map.entities
            if entity.x == mouse_x and entity.y == mouse_y
        )
        console.print(x=x, y=y, text=names)
```

`render_bar` draws a filled rectangle for the filled portion and an empty rectangle for the background, then overlays the text `"HP: N/M"`. `render_names_at_mouse_location` collects all entity names at the cursor position and joins them with commas.

---

## Update engine.py

The engine needs:

1. A `message_log` attribute
2. A `mouse_location` attribute to track the cursor
3. A `render()` method that calls the new helpers

```python
from __future__ import annotations

from typing import TYPE_CHECKING

import tcod

from game.constants import colors
from game.message_log import MessageLog
from game import render_functions

if TYPE_CHECKING:
    from tcod.console import Console
    from game.entity import Actor
    from game.map.game_map import GameMap
    from game.input_handlers import EventHandler


class Engine:
    def __init__(self, game_map: GameMap, player: Actor) -> None:
        from game.input_handlers import MainGameEventHandler
        self.event_handler: EventHandler = MainGameEventHandler(self)
        self.game_map = game_map
        self.message_log = MessageLog()
        self.mouse_location = (0, 0)
        self.player = player

    def handle_events(self, events: tcod.event.EventLike) -> None:
        for event in events:
            action = self.event_handler.handle_events(event)
            if action is None:
                continue
            action.perform(self, self.player)

            if self.player.is_alive:
                self.handle_enemy_turns()

            if not self.player.is_alive:
                from game.input_handlers import GameOverEventHandler
                self.event_handler = GameOverEventHandler(self)

            self.update_fov()

    def handle_enemy_turns(self) -> None:
        for entity in set(self.game_map.actors) - {self.player}:
            if entity.ai:
                entity.ai.perform(self, entity)

    def update_fov(self) -> None:
        self.game_map.visible[:] = tcod.map.compute_fov(
            self.game_map.tiles["transparent"],
            (self.player.x, self.player.y),
            radius=8,
        )
        self.game_map.explored |= self.game_map.visible

    def render(self, console: Console) -> None:
        self.game_map.render(console)

        self.message_log.render(console=console, x=21, y=45, width=40, height=5)

        render_functions.render_bar(
            console=console,
            current_value=self.player.fighter.hp,
            maximum_value=self.player.fighter.max_hp,
            total_width=20,
        )

        render_functions.render_names_at_mouse_location(
            console=console, x=21, y=44, engine=self
        )

    def run(self, context: tcod.context.Context, console: Console) -> None:
        while True:
            console.clear()
            self.event_handler.on_render(console=console)
            context.present(console)
            for event in tcod.event.wait():
                context.convert_event(event)
                self.handle_events([event])
```

`Engine.run()` evolves: rendering now delegates to `event_handler.on_render()` instead of calling `Engine.render()` directly. Event processing still goes through `Engine.handle_events()` so actions are executed in one place.

!!! question "Why no `algorithm=` in `update_fov`?"
    Part 4 passed `algorithm=tcod.constants.FOV_SHADOW` explicitly. It is omitted here because `FOV_SHADOW` is tcod 21.x's default, so the call is equivalent. Removing it keeps the signature cleaner; add it back if you want to try a different algorithm.

!!! info "render() vs game_map.render()"
    `GameMap.render()` draws tiles and entity sprites. `Engine.render()` composes the full frame: map first, then the UI panel on top. Keeping these separate means the map never needs to know about the UI layout.

---

## Refactor input_handlers.py

Event handlers need a bigger change. Currently they return `Action` objects. In Part 10 we will need handlers that can return *other handlers* (for menus and targeting screens). We prepare for this now.

Each handler:
- Takes an `engine` in its constructor
- Has a `handle_events(event)` method
- Has an `on_render(console)` method that draws anything the handler needs

Replace `game/input_handlers.py`:

```python
from __future__ import annotations

from typing import TYPE_CHECKING

import tcod

from game.actions import (
    Action,
    BumpAction,
    EscapeAction,
    WaitAction,
)

if TYPE_CHECKING:
    from game.engine import Engine

MOVE_KEYS = {
    # Arrow keys
    tcod.event.KeySym.UP:       ( 0, -1),
    tcod.event.KeySym.DOWN:     ( 0,  1),
    tcod.event.KeySym.LEFT:     (-1,  0),
    tcod.event.KeySym.RIGHT:    ( 1,  0),

    tcod.event.KeySym.HOME:     (-1, -1),
    tcod.event.KeySym.END:      (-1,  1),
    tcod.event.KeySym.PAGEUP:   ( 1, -1),
    tcod.event.KeySym.PAGEDOWN: ( 1,  1),

    # Numpad
    tcod.event.KeySym.KP_1:     (-1,  1),
    tcod.event.KeySym.KP_2:     ( 0,  1),
    tcod.event.KeySym.KP_3:     ( 1,  1),
    tcod.event.KeySym.KP_4:     (-1,  0),
    tcod.event.KeySym.KP_6:     ( 1,  0),
    tcod.event.KeySym.KP_7:     (-1, -1),
    tcod.event.KeySym.KP_8:     ( 0, -1),
    tcod.event.KeySym.KP_9:     ( 1, -1),

    # Vi keys
    tcod.event.KeySym.K:        ( 0, -1),
    tcod.event.KeySym.J:        ( 0,  1),
    tcod.event.KeySym.H:        (-1,  0),
    tcod.event.KeySym.L:        ( 1,  0),
    tcod.event.KeySym.Y:        (-1, -1),
    tcod.event.KeySym.U:        ( 1, -1),
    tcod.event.KeySym.B:        (-1,  1),
    tcod.event.KeySym.N:        ( 1,  1),
}

WAIT_KEYS = {
    tcod.event.KeySym.PERIOD,
    tcod.event.KeySym.KP_5,
    tcod.event.KeySym.CLEAR,
}


class EventHandler:
    def __init__(self, engine: Engine) -> None:
        self.engine = engine

    def handle_events(self, event: tcod.event.Event) -> Action | None:
        match event:
            case tcod.event.Quit():
                return EscapeAction()
            case tcod.event.MouseMotion():
                self.engine.mouse_location = event.tile.x, event.tile.y
            case tcod.event.KeyDown():
                return self.event_keydown(event)
        return None

    def event_keydown(self, event: tcod.event.KeyDown) -> Action | None:
        return None

    def on_render(self, console: tcod.Console) -> None:
        self.engine.render(console)


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

        return None


class GameOverEventHandler(EventHandler):
    def event_keydown(self, event: tcod.event.KeyDown) -> Action | None:
        if event.sym == tcod.event.KeySym.ESCAPE:
            return EscapeAction()
        return None
```

The `MouseMotion` case stores the cursor tile position in `engine.mouse_location` so `render_names_at_mouse_location` always has current data.

---

## Update main.py

The main loop now calls `handler.on_render()` instead of `engine.render()` directly, and we no longer need to import event handler classes, the engine creates the initial one.

```python
from __future__ import annotations

import copy
from pathlib import Path

import tcod

from game.constants import colors
from game import entity_factories
from game.engine import Engine
from game.map.map_generator import generate_dungeon


def main() -> None:
    screen_width = 80
    screen_height = 50

    map_width = 80
    map_height = 45

    room_max_size = 10
    room_min_size = 6
    max_rooms = 30
    max_monsters_per_room = 2

    tileset = tcod.tileset.load_tilesheet(
        Path(__file__).parent / "res" / "dejavu10x10_gs_tc.png",
        32,
        8,
        tcod.tileset.CHARMAP_TCOD,
    )

    player = copy.deepcopy(entity_factories.player)

    game_map = generate_dungeon(
        max_rooms=max_rooms,
        room_min_size=room_min_size,
        room_max_size=room_max_size,
        map_width=map_width,
        map_height=map_height,
        max_monsters_per_room=max_monsters_per_room,
        player=player,
    )

    engine = Engine(game_map=game_map, player=player)
    engine.update_fov()

    engine.message_log.add_message(
        "Hello and welcome, adventurer, to yet another dungeon!",
        colors.WELCOME_TEXT,
    )

    with tcod.context.new(
        columns=screen_width,
        rows=screen_height,
        tileset=tileset,
        title="Roguelike Tutorial",
        vsync=True,
    ) as context:
        root_console = tcod.Console(screen_width, screen_height, order="F")
        engine.run(context, root_console)


if __name__ == "__main__":
    main()
```

The `context.convert_event(event)` call inside `Engine.run()` translates pixel coordinates to tile coordinates so `MouseMotion.tile` is populated correctly.

---

## Replace print() with message_log in fighter.py

All the `print()` calls from Parts 5 and 6 now route through `engine.message_log.add_message()`. Because `die()` now needs `engine` to write to the log, `melee_attack()` also receives it: it calls `die(engine)` explicitly instead of relying on the `hp` setter.

Update `Fighter.melee_attack()` in `game/components/fighter.py`:

```diff
-def melee_attack(self, target: Actor) -> None:
+def melee_attack(self, target: Actor, engine: Engine) -> None:
     damage = self.attack - target.fighter.defense
     attack_msg = f"{self.entity.name.capitalize()} attacks {target.name}"
+    attack_color = colors.PLAYER_ATTACK if self.entity is engine.player else colors.ENEMY_ATTACK
+
     if damage > 0:
-        print(f"{attack_msg} for {damage} hit points.")
-        target.fighter.hp -= damage
+        engine.message_log.add_message(
+            f"{attack_msg} for {damage} hit points.", attack_color
+        )
+        target.fighter.hp -= damage
+        if not target.is_alive:
+            target.fighter.die(engine)
     else:
-        print(f"{attack_msg} but does no damage.")
+        engine.message_log.add_message(
+            f"{attack_msg} but does no damage.", attack_color
+        )
```

Player attacks use a lighter color (`PLAYER_ATTACK`) and enemy attacks a red tint (`ENEMY_ATTACK`), so the player can scan the log quickly.

Update `MeleeAction` in `game/actions.py` to pass `engine`:

```diff
-        entity.fighter.melee_attack(target)
+        entity.fighter.melee_attack(target, engine)
```

Update `Fighter.die()` in `game/components/fighter.py` to accept `engine` and write to the message log:

```python
    def die(self, engine: Engine) -> None:
        if self.entity is engine.player:
            death_message = "You died!"
            death_message_color = colors.PLAYER_DEATH
        else:
            death_message = f"The {self.entity.name} is dead!"
            death_message_color = colors.ENEMY_DEATH

        self.entity.char = sprites.CORPSE
        self.entity.color = colors.CORPSE
        self.entity.blocks_movement = False
        self.entity.ai = None
        self.entity.name = f"remains of {self.entity.name}"
        self.entity.render_order = RenderOrder.CORPSE

        engine.message_log.add_message(death_message, death_message_color)
```

And simplify the `hp.setter`, it no longer auto-triggers `die()`:

```python
    @hp.setter
    def hp(self, value: int) -> None:
        self._hp = max(0, min(value, self.max_hp))
```

!!! question "Why not keep die() in the setter?"
    The setter does not have access to `engine`, which `die()` now needs for the message log. Passing `engine` down through the setter would require changing the property interface (setters cannot take extra arguments in Python). Moving the `die()` call to the damage site (`melee_attack`) keeps the `Fighter` API clean.

---

## complete fighter.py

```python
from __future__ import annotations

from typing import TYPE_CHECKING

from game.components.base_component import BaseComponent
from game.constants import colors, sprites
from game.render_order import RenderOrder

if TYPE_CHECKING:
    from game.engine import Engine
    from game.entity import Actor


class Fighter(BaseComponent):
    def __init__(self, hp: int, defense: int, attack: int) -> None:
        self.max_hp = hp
        self._hp = hp
        self.defense = defense
        self.attack = attack

    @property
    def hp(self) -> int:
        return self._hp

    @hp.setter
    def hp(self, value: int) -> None:
        self._hp = max(0, min(value, self.max_hp))

    def heal(self, amount: int) -> int:
        if self.hp == self.max_hp:
            return 0
        new_hp_value = self.hp + amount
        if new_hp_value > self.max_hp:
            new_hp_value = self.max_hp
        recovered = new_hp_value - self.hp
        self.hp = new_hp_value
        return recovered

    def take_damage(self, amount: int) -> None:
        self.hp -= amount

    def melee_attack(self, target: Actor, engine: Engine) -> None:
        damage = self.attack - target.fighter.defense
        attack_msg = f"{self.entity.name.capitalize()} attacks {target.name}"
        attack_color = colors.PLAYER_ATTACK if self.entity is engine.player else colors.ENEMY_ATTACK

        if damage > 0:
            engine.message_log.add_message(
                f"{attack_msg} for {damage} hit points.", attack_color
            )
            target.fighter.hp -= damage
            if not target.is_alive:
                target.fighter.die(engine)
        else:
            engine.message_log.add_message(
                f"{attack_msg} but does no damage.", attack_color
            )

    def die(self, engine: Engine) -> None:
        if self.entity is engine.player:
            death_message = "You died!"
            death_message_color = colors.PLAYER_DEATH
        else:
            death_message = f"The {self.entity.name} is dead!"
            death_message_color = colors.ENEMY_DEATH

        self.entity.char = sprites.CORPSE
        self.entity.color = colors.CORPSE
        self.entity.blocks_movement = False
        self.entity.ai = None
        self.entity.name = f"remains of {self.entity.name}"
        self.entity.render_order = RenderOrder.CORPSE

        engine.message_log.add_message(death_message, death_message_color)
```

We also added `heal()` (used in Part 8 by healing potions) and `take_damage()` as a named wrapper for clarity.

---

## Testing your work

Run `python main.py`:

- [ ] A health bar appears in the bottom-left corner showing `HP: 30/30`
- [ ] The welcome message appears in the message log panel
- [ ] Attacking an enemy adds a colored line to the message log
- [ ] When enemies attack you, the message appears in a different color
- [ ] Hovering the mouse over a visible entity shows its name above the panel
- [ ] On death, `"You died!"` appears in the log and the bar shows 0 HP
- [ ] Repeated identical messages stack: `"Orc attacks Player for 2 hit points. (x3)"`

---

## Summary

The UI panel is now live. Key additions:

- **`game/constants/colors.py`**: centralized color constants for the whole project
- **`MessageLog`**: stores and renders recent events with stacking and color
- **`render_functions`**: stateless rendering helpers for the bar and mouse names
- **`on_render()`**: each event handler controls its own frame rendering
- **`mouse_location`**: engine tracks the cursor for hover tooltips

**Current architecture**:

- `Engine.render()`: composes the full frame: map, message log, HP bar, and hover text
- `GameMap.render()`: still draws only terrain and entities
- `MessageLog`: owns message storage and message rendering
- `render_functions.py`: contains stateless UI drawing helpers
- `EventHandler.on_render()`: lets each handler control what gets drawn for its state

**Files created**: `game/message_log.py`, `game/render_functions.py`

**Files modified**: `game/constants/colors.py`, `game/engine.py`, `game/input_handlers.py`, `main.py`, `game/actions.py`, `game/components/fighter.py`

---

## Exercises

1. **Colored HP bar.** Change `bar_filled` to green when HP > 66%, yellow when > 33%, and red when ≤ 33%. You'll need to compute the percentage and pick the color before calling `draw_rect`.

2. **Message history screen.** Add a keybinding (e.g. `V`) that opens a full-screen overlay showing all messages in the log, not just the last five visible in the panel. You will need a new handler class for this.

3. **Entity details.** When hovering over an enemy, show its HP alongside its name: `"Orc (8/10 HP)"`. Modify `render_names_at_mouse_location` to check if the entity is an `Actor` and append its HP.

**Next**: Part 8: Items and Inventory
