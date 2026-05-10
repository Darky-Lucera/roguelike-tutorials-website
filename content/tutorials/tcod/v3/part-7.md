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

For now, this tutorial dedicates 45 rows to the map and 5 rows to a UI panel at the bottom. That is enough for a health bar and a short message history. Later, you can decide where to place the panel and which size makes the most sense for each part of your UI.

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

```python
# Generic colors
WHITE = (0xFF, 0xFF, 0xFF)
BLACK = (0x0, 0x0, 0x0)

# Combat message colors
PLAYER_ATTACK = (0xE0, 0xE0, 0xE0)
ENEMY_ATTACK  = (0xFF, 0xC0, 0xC0)
PLAYER_DEATH  = (0xFF, 0x30, 0x30)
ENEMY_DEATH   = (0xFF, 0xA0, 0x30)

# UI colors
WELCOME_TEXT  = (0x20, 0xA0, 0xFF)
BAR_TEXT      = WHITE
HP_BAR_FILLED    = (0x0, 0x60, 0x0)
HP_BAR_EMPTY     = (0x40, 0x10, 0x10)
```

We split the new constants into three sections (`Generic colors`, `Combat message colors`, `UI colors`) to make scanning the file easier as it grows. Notice we name the death colors `PLAYER_DEATH` and `ENEMY_DEATH`, not `_die`: full words read better at every call site.

!!! note "If you kept earlier exercises"
    Extra exercise messages can get their own colors here as well. For example, the reference implementation uses optional colors such as `ENEMY_FLEE`, `ATTACK_MISS`, and `BLOCK_MOVEMENT` for Part 5/6 exercise feedback. They are useful, but not required for the base tutorial path.

!!! question "Why this lives in `constants/`, not in `engine.py`"
    `game/constants/colors.py` is imported by `hud`, `message_log`, `fighter`, and eventually every UI component. Putting colors in `game/engine.py` would force circular imports, those modules would need to import `engine` just for a tuple.

---

## message_log.py

The message log stores recent events and renders them in the panel. Two features make it feel polished: **color** (attacks are different from deaths) and **stacking** (if the same message repeats, it shows `(×3)` instead of three identical lines).

Create `game/message_log.py`:

```python
from __future__ import annotations

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
    messages: list[Message] = []

    @classmethod
    def add_message(
        cls,
        text: str,
        fg: tuple[int, int, int] = colors.WHITE,
        *,
        stack: bool = True,
    ) -> None:
        if stack and cls.messages and cls.messages[-1].plain_text == text:
            cls.messages[-1].count += 1
        else:
            cls.messages.append(Message(text, fg))

    @classmethod
    def clear(cls) -> None:
        cls.messages.clear()

    @staticmethod
    def render(
        console: tcod.Console,
        x: int,
        y: int,
        width: int,
        height: int,
    ) -> None:
        wrapped_lines: list[tuple[str, tuple[int, int, int]]] = []
        for message in MessageLog.messages:
            for line in textwrap.wrap(message.full_text, width):
                wrapped_lines.append((line, message.fg))

        total = len(wrapped_lines)
        start = max(0, total - height)
        end   = start + height

        for y_offset, (line, color) in enumerate(wrapped_lines[start:end]):
            console.print(x=x, y=y + y_offset, text=line, fg=color)
```

`MessageLog` has no `__init__`: `messages` is a class variable shared by everyone, and the methods operate on it without needing an instance. `render` first flattens all messages into a list of `(line, color)` pairs, then takes a slice of `height` lines from the end (the most recent ones) and renders them top to bottom. `clear()` empties the list; it is called in Part 10 when starting a new game.

!!! info "@classmethod vs @staticmethod"
    A **`@classmethod`** receives the class itself as its first argument (`cls`). That is what `add_message` needs: it reads and writes `cls.messages`, the shared list. A **`@staticmethod`** receives nothing implicit (no `self`, no `cls`). `render` qualifies because it only reads `MessageLog.messages` by name; it does not need to be overridden in a subclass, and it does not modify class state.

!!! question "Why a static class instead of an instance on Engine (as in the 2019 and v2 tutorials)"
    The 2019 and v2 tutorials store a `MessageLog` instance on `Engine`. That works, but it means every component that wants to log (a `Fighter`, an AI, a consumable) must receive `engine` as a parameter just to reach `engine.message_log`. With a static `MessageLog`, any module can call `MessageLog.add_message(...)` after a one-line import, with no extra dependency on `Engine`.

---

## hud.py

Two standalone HUD helpers that the engine calls each frame.

Create `game/hud.py`:

```python
from __future__ import annotations

from typing import TYPE_CHECKING

from game.constants import colors

if TYPE_CHECKING:
    from tcod import Console
    from game.map.game_map import GameMap


def render_bar(
    console: Console,
    current_value: int,
    maximum_value: int,
    total_width: int,
    y: int = 45,
) -> None:
    bar_width = int(float(current_value) / maximum_value * total_width)

    console.draw_rect(x=0, y=y, width=total_width, height=1, ch=1, bg=colors.HP_BAR_EMPTY)

    if bar_width > 0:
        console.draw_rect(
            x=0, y=y, width=bar_width, height=1, ch=1, bg=colors.HP_BAR_FILLED
        )

    console.print(
        x=1,
        y=y,
        text=f"HP: {current_value}/{maximum_value}",
        fg=colors.BAR_TEXT,
    )


def render_names_at_mouse_location(
    console: Console,
    x: int,
    y: int,
    mouse_location: tuple[int, int],
    game_map: GameMap,
) -> None:
    mouse_x, mouse_y = mouse_location

    if game_map.in_bounds(mouse_x, mouse_y) and game_map.visible[mouse_x, mouse_y]:
        names = ", ".join(
            entity.name
            for entity in game_map.entities
            if entity.x == mouse_x and entity.y == mouse_y
        )
        console.print(x=x, y=y, text=names)
```

`render_bar` draws a filled rectangle for the filled portion and an empty rectangle for the background, then overlays the text `"HP: N/M"`. `render_names_at_mouse_location` collects all entity names at the cursor position and joins them with commas. Both functions take only what they need: no `Engine` reference, no hidden dependencies.

---

## Update engine.py

Five changes: new imports, updated `__init__`, updated `handle_events`, updated `render()`, and updated `run()`. All come from switching to the new `EventHandler` API (next section) and adding the UI.

Add the imports at the top of `game/engine.py`:

```diff
+from game.message_log import MessageLog
+from game import hud
-from game.input_handlers import EventHandler, GameOverEventHandler
+from game.input_handlers import EventHandler, MainGameEventHandler, GameOverEventHandler
```

Update `__init__` to use the new handler class and track mouse position:

```diff
     self.game_map = game_map
+    self.mouse_location = tcod.event.Point(0, 0)
     self.player = player
-    self.event_handler = EventHandler()
+    self.event_handler: EventHandler = MainGameEventHandler(self)
```

Update `handle_events` to call the new `handle_events()` method (replacing the old `dispatch()`), guard enemy turns so they only run while the player is alive, and pass `self` to `GameOverEventHandler`:

```diff
    def handle_events(self, events: Iterable[Any]) -> None:
        for event in events:
-            action = self.event_handler.dispatch(event)
+            action = self.event_handler.handle_events(event)
            if action is None:
                continue

            action.perform(self, self.player)
-            self.handle_enemy_turns()
-            self.update_fov()  # recompute after every action
-
-            if not self.player.is_alive:
-                self.event_handler = GameOverEventHandler()
+
+            if self.player.is_alive:
+                self.handle_enemy_turns()
+
+            if not self.player.is_alive:
+                self.event_handler = GameOverEventHandler(self)
+
+            self.update_fov()  # recompute after every action
```

Replace the existing `render()` method. It no longer receives `context` or controls the frame cycle; `run()` owns those steps now.

```python
    def render(self, console: Console) -> None:
        self.game_map.render(console)

        MessageLog.render(
            console = console,
            x       = 21,
            y       = 45,
            width   = 40,
            height  = 5
        )

        hud.render_bar(
            console       = console,
            current_value = self.player.fighter.hp,
            maximum_value = self.player.fighter.max_hp,
            total_width   = 20,
        )

        hud.render_names_at_mouse_location(
            console        = console,
            x              = 21,
            y              = 44,
            mouse_location = self.mouse_location,
            game_map       = self.game_map,
        )
```

Update `run()` to delegate rendering to the active event handler and to capture the result of `context.convert_event`:

```diff
    def run(self, context: Context, console: Console) -> None:
         while True:
-            self.render(console=console, context=context)
-            self.handle_events(tcod.event.wait())
+            console.clear()
+            self.event_handler.on_render(console=console)
+            context.present(console)
+            for event in tcod.event.wait():
+                event = context.convert_event(event)
+                self.handle_events([event])
```

`context.convert_event(event)` returns a new event object with tile-space coordinates set. The return value must be captured; the original event object is not modified in place.

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

    # Numpad
    tcod.event.KeySym.KP_1:     (-1,  1), # LEFT  - DOWN
    tcod.event.KeySym.KP_2:     ( 0,  1), #         DOWN
    tcod.event.KeySym.KP_3:     ( 1,  1), # RIGHT - DOWN
    tcod.event.KeySym.KP_4:     (-1,  0), # LEFT
    tcod.event.KeySym.KP_6:     ( 1,  0), # RIGHT
    tcod.event.KeySym.KP_7:     (-1, -1), # LEFT  - UP
    tcod.event.KeySym.KP_8:     ( 0, -1), #         UP
    tcod.event.KeySym.KP_9:     ( 1, -1), # RIGHT - UP

    # Vi keys
    tcod.event.KeySym.B:        (-1,  1), # LEFT  - DOWN
    tcod.event.KeySym.J:        ( 0,  1), #         DOWN
    tcod.event.KeySym.N:        ( 1,  1), # RIGHT - DOWN
    tcod.event.KeySym.H:        (-1,  0), # LEFT
    tcod.event.KeySym.L:        ( 1,  0), # RIGHT
    tcod.event.KeySym.Y:        (-1, -1), # LEFT  - UP
    tcod.event.KeySym.K:        ( 0, -1), #         UP
    tcod.event.KeySym.U:        ( 1, -1), # RIGHT - UP
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
                self.engine.mouse_location = event.integer_position

            case tcod.event.KeyDown():
                return self.event_keydown(event)

        return None

    def event_keydown(self, _event: tcod.event.KeyDown) -> Action | None:
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

The `MouseMotion` case stores the cursor tile position in `engine.mouse_location` so `render_names_at_mouse_location` always has current data. `integer_position` is the tile-space coordinate set by `context.convert_event`; the older `event.tile` attribute is deprecated.

---

## Update main.py

Three small changes: two new imports, posting the welcome message, and renaming `console` to `root_console` for clarity now that the engine owns the frame loop.

Add the imports:

```diff
+from game.constants import colors
 from game import entity_factories
 from game.engine import Engine
+from game.message_log import MessageLog
 from game.map.map_generator import generate_dungeon
```

Post the welcome message after creating the engine:

```diff
 engine = Engine(game_map=game_map, player=player)
+
+MessageLog.add_message(
+    "Hello and welcome, adventurer, to yet another dungeon!",
+    colors.WELCOME_TEXT,
+)
```

Rename the console variable in the context block:

```diff
-        console = tcod.console.Console(screen_width, screen_height, order="F")
-        engine.run(context, console)
+        root_console = tcod.Console(screen_width, screen_height, order="F")
+        engine.run(context, root_console)
```

---

## Replace print() with message_log in fighter.py

All the `print()` calls from Parts 5 and 6 now route through `MessageLog.add_message()`. Because `MessageLog` is a static class, `melee_attack()` and `die()` can call it directly with no extra parameters. `die()` returns to the `hp` setter, where it always belonged.

Update `Fighter.melee_attack()` in `game/components/fighter.py`:

```diff
 def melee_attack(self, target: Actor) -> None:
     damage = self.attack - target.fighter.defense
     attack_msg = f"{self.entity.name.capitalize()} attacks {target.name}"
+    attack_color = colors.PLAYER_ATTACK if self.entity.ai is None else colors.ENEMY_ATTACK
+
     if damage > 0:
-        print(f"{attack_msg} for {damage} hit points.")
+        MessageLog.add_message(f"{attack_msg} for {damage} hit points.", attack_color)
         target.fighter.hp -= damage
     else:
-        print(f"{attack_msg} but does no damage.")
+        MessageLog.add_message(f"{attack_msg} but does no damage.", attack_color)
```

`self.entity.ai is None` identifies the player: the player never has an AI component, enemies always do. Player attacks use a lighter color (`PLAYER_ATTACK`) and enemy attacks a red tint (`ENEMY_ATTACK`), so the player can scan the log quickly. `die()` is triggered by the `hp` setter as before: no call site change needed in `melee_attack`.

Update `Fighter.die()` in `game/components/fighter.py` to write to the message log:

```diff
-def die(self) -> None:
-    if self.entity.ai:
-        death_message = f"The {self.entity.name} is dead!"
-    else:
-        death_message = "You died!"
-
-    print(death_message)
+def die(self) -> None:
+    if self.entity.ai is None:
+        death_message = "You died!"
+        death_message_color = colors.PLAYER_DEATH
+    else:
+        death_message = f"The {self.entity.name} is dead!"
+        death_message_color = colors.ENEMY_DEATH
+
+    MessageLog.add_message(death_message, death_message_color)
```

Also add `heal()` and `take_damage()` to `Fighter` in `game/components/fighter.py`:

```python
    def heal(self, amount: int) -> int:
        if self.hp == self.max_hp:
            return 0

        new_hp_value = self.hp + amount
        new_hp_value = min(new_hp_value, self.max_hp)

        recovered = new_hp_value - self.hp
        self.hp = new_hp_value

        return recovered

    def take_damage(self, amount: int) -> None:
        self.hp -= amount
```

`heal()` returns the amount actually recovered; the `hp` setter clamps to `max_hp`, so you cannot overheal. It is used in Part 8 by healing potions. `take_damage()` is a thin wrapper over `self.hp -= amount` that gives call sites a readable name.

!!! note "If you kept Part 5/6 exercise code"
    Convert those messages to `MessageLog.add_message(...)` too. For example, non-combat blockers in `actions.py` should log instead of printing, and optional flee/critical-hit logic in `fighter.py` should keep the same behavior while routing its feedback through the message log.

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
- **`MessageLog`**: static class that stores and renders recent events with stacking and color
- **`hud`**: stateless HUD helpers for the bar and mouse names
- **`on_render()`**: each event handler controls its own frame rendering
- **`mouse_location`**: engine tracks the cursor for hover tooltips

**Current architecture**:

- `Engine.render()`: composes the full frame: map, message log, HP bar, and hover text
- `GameMap.render()`: still draws only terrain and entities
- `MessageLog`: static class; any module can call `MessageLog.add_message()` without touching `Engine`
- `hud.py`: stateless HUD drawing helpers; each function takes only what it needs
- `EventHandler.on_render()`: lets each handler control what gets drawn for its state

**Files created**: `game/message_log.py`, `game/hud.py`

**Files modified**: `game/constants/colors.py`, `game/engine.py`, `game/input_handlers.py`, `main.py`, `game/components/fighter.py`

---

## Exercises

1. **Colored HP bar**:

    Change `HP_BAR_FILLED` to green when HP > 70%, yellow/orange when > 30%, and red when ≤ 30%. You'll need to compute the percentage and pick the color before calling `draw_rect`.

2. **Scroll the message panel**:

    The five-row panel shows only the most recent messages. Add `Page Up` / `Page Down` bindings that shift which portion of the log is visible. Store the current scroll value in `MessageLog`. When rendering, wrap all messages first, clamp `scroll` between `0` and `max(0, total - height)`, then use it to offset the visible slice: `start = max(0, total - height - scroll)` and `end = start + height`. Reset `scroll` to `0` when a new message is added.

3. **Entity details**:

    When hovering over an entity, show full combat stats if it is an `Actor`: `"Player (HP: 30/30, ATK: 5, DEF: 2)"`. Plain entities (items, corpses) still show just their name. Modify `hud.render_names_at_mouse_location` to iterate with an explicit loop, check `isinstance(entity, Actor)`, and format the stats line accordingly.

**Next**: Part 8: Items and Inventory
