# Part 10: Save and Load

## What You Will Build

By the end of this part, your roguelike will have a main menu and a save system, allowing the player to quit the game and continue later from the saved state.

## Learning goals

- Serialize the entire game state to disk with `pickle` + `lzma`
- Add a main menu with New Game / Continue options
- Refactor event handlers to return handlers (not just actions)
- Delete the save file on player death
- Verify: save, quit, reload, game state is exactly as left

---

## The serialization problem

Saving a game means converting the entire live object graph (engine, game map, entities, components, AI instances) into bytes that can be written to disk and reconstructed later.

```txt
Engine
  ├── player (Actor)
  │     ├── fighter (Fighter)   ← component back-reference to player
  │     └── inventory (Inventory)
  ├── game_map (GameMap)
  │     └── entities: set[Entity]
  │           └── orc (Actor)
  │                 └── ai (HostileEnemy)  ← back-reference to the orc
Static state
  â””â”€â”€ MessageLog.messages
```

The challenge: `Fighter.entity` points back to the `Actor` that owns it, `HostileEnemy.entity` points to the orc, and in Part 9 `ConfusedEnemy.previous_ai` holds another AI. This is a circular reference graph. The message log is a separate static list, so we save it next to the engine.

**JSON** cannot handle circular references without custom encoding, every class needs a `to_dict()` / `from_dict()` pair. That is a lot of boilerplate.

**`pickle`** handles circular references natively. Python's pickle module serializes the entire object graph, following references and deduplicating them automatically. It is the right tool for internal game state that you fully control.

!!! danger "pickle is not safe with untrusted data"
    Never load a pickle file from an unknown source, pickle can execute arbitrary code during deserialization. For a single-player game saving its own state, this is fine.

We add `lzma` compression on top to shrink the file. A typical save is a few kilobytes compressed, negligible, but it is good practice.

---

## Engine.save_as()

Add to `game/engine.py`:

```python
import lzma
import pickle
from pathlib import Path

from game.message_log import MessageLog


class Engine:
    ...

    def save_as(self, filename: str) -> None:
        save_data = lzma.compress(
            pickle.dumps(
                {
                    "engine": self,
                    "message_log": MessageLog.messages,
                }
            )
        )
        Path(filename).write_bytes(save_data)
```

That is the entire save implementation. `pickle.dumps(...)` serializes the engine (and everything it references) plus the static message log into bytes. `lzma.compress` shrinks it. `Path.write_bytes` writes the compressed bytes to disk.

Loading is the inverse:

```python
    @staticmethod
    def load(filename: str) -> Engine:
        save_data = Path(filename).read_bytes()
        data = pickle.loads(lzma.decompress(save_data))
        MessageLog.messages = data["message_log"]
        return data["engine"]
```

`MessageLog.messages` is stored explicitly because it is class-level state, not an instance attribute of `Engine`. Without this extra field, loading would restore the map and entities but lose the visible message history.

---

## setup_game.py

We need a function that creates a fresh game (used by "New Game") and one that loads an existing save (used by "Continue"). Extract them into a dedicated module so `main.py` and the main menu handler stay clean.

Create `game/setup_game.py`:

```python
from __future__ import annotations

import copy
from pathlib import Path

from game.constants import colors
from game.entities import factories
from game.engine import Engine
from game.message_log import MessageLog
from game.map.map_generator import generate_dungeon

SAVE_PATH = "savegame.sav"

MAP_WIDTH = 80
MAP_HEIGHT = 45
MAX_ROOMS = 30
ROOM_MIN_SIZE = 6
ROOM_MAX_SIZE = 10
MIN_MONSTERS_PER_ROOM = 0
MAX_MONSTERS_PER_ROOM = 2
MIN_ITEMS_PER_ROOM = 0
MAX_ITEMS_PER_ROOM = 2


def new_game() -> Engine:
    """Return a fresh engine for a brand-new game."""
    MessageLog.clear()

    player = copy.deepcopy(factories.player)

    game_map = generate_dungeon(
        max_rooms=MAX_ROOMS,
        room_min_size=ROOM_MIN_SIZE,
        room_max_size=ROOM_MAX_SIZE,
        map_width=MAP_WIDTH,
        map_height=MAP_HEIGHT,
        min_monsters_per_room=MIN_MONSTERS_PER_ROOM,
        max_monsters_per_room=MAX_MONSTERS_PER_ROOM,
        min_items_per_room=MIN_ITEMS_PER_ROOM,
        max_items_per_room=MAX_ITEMS_PER_ROOM,
        player=player,
    )
    engine = Engine(game_map=game_map, player=player)
    engine.update_fov()

    MessageLog.add_message(
        "Hello and welcome, adventurer, to yet another dungeon!",
        colors.WELCOME_TEXT,
    )
    return engine


def load_game(filename: str) -> Engine:
    """Load a save file and return the engine."""
    if not Path(filename).exists():
        raise FileNotFoundError(f"No save file found at {filename!r}")
    return Engine.load(filename)
```

---

## BaseEventHandler and PopupMessage

Before adding the main menu, we need one more abstraction: a handler that does *not* delegate to `Engine`. The main menu runs before any engine exists, so it cannot inherit from the current `EventHandler`.

We split the hierarchy:

```txt
BaseEventHandler
  ├── EventHandler (has engine, handles actions)
  │     ├── MainGameEventHandler
  │     ├── GameOverEventHandler
  │     └── InventoryEventHandler (and subclasses)
  │           ├── InventoryActivateHandler
  │           └── InventoryDropHandler
  ├── MainMenu  (no engine yet)
  └── PopupMessage (shows a message over a frozen background)
```

Add to `game/input_handlers.py`:

```python
class BaseEventHandler:
    def handle_events(self, event: tcod.event.Event):
        state = self.dispatch(event)
        if isinstance(state, BaseEventHandler):
            return state
        return self

    def dispatch(self, event: tcod.event.Event):
        match event:
            case tcod.event.Quit():
                raise SystemExit()
            case tcod.event.KeyDown():
                return self.event_keydown(event)
        return None

    def event_keydown(self, event: tcod.event.KeyDown):
        return None

    def on_render(self, console: tcod.Console) -> None:
        raise NotImplementedError()


class PopupMessage(BaseEventHandler):
    """Display a message over a screenshot of the current state."""

    def __init__(self, parent_handler: BaseEventHandler, text: str) -> None:
        self.parent = parent_handler
        self.text = text

    def on_render(self, console: tcod.Console) -> None:
        self.parent.on_render(console)
        console.rgb["fg"] //= 8
        console.rgb["bg"] //= 8
        console.print(
            console.width // 2,
            console.height // 2,
            self.text,
            fg=colors.WHITE,
            bg=colors.BLACK,
            alignment=tcod.constants.CENTER,
        )

    def event_keydown(self, event: tcod.event.KeyDown):
        return self.parent
```

`PopupMessage` darkens the existing frame by right-shifting all color channels by 3 (dividing by 8), then overlays centered text. Any keypress dismisses it and returns to the parent handler.

---

## MainMenu handler

```python
from game.setup_game import SAVE_PATH, load_game, new_game


class MainMenu(BaseEventHandler):
    """Renders the main menu and handles New / Continue / Quit."""

    def on_render(self, console: tcod.Console) -> None:
        console.print(
            console.width // 2,
            console.height // 2 - 4,
            "ROGUELIKE TUTORIAL",
            fg=colors.MENU_TITLE,
            alignment=tcod.constants.CENTER,
        )
        for i, text in enumerate(
            ["[N] Play a new game", "[C] Continue last game", "[Q] Quit"]
        ):
            console.print(
                console.width // 2,
                console.height // 2 - 2 + i,
                text.ljust(30),
                fg=colors.MENU_TEXT,
                bg=colors.BLACK,
                alignment=tcod.constants.CENTER,
                bg_blend=tcod.constants.BKGND_ALPHA(64),
            )

    def event_keydown(self, event: tcod.event.KeyDown):
        match event.sym:
            case tcod.event.KeySym.Q | tcod.event.KeySym.ESCAPE:
                raise SystemExit()

            case tcod.event.KeySym.C:
                try:
                    engine = load_game(SAVE_PATH)
                    return MainGameEventHandler(engine)

                except FileNotFoundError:
                    return PopupMessage(self, "No saved game to load.")

                except Exception as exc:
                    return PopupMessage(self, f"Failed to load save:\n{exc}")

            case tcod.event.KeySym.N:
                return new_game_handler()

        return None


def new_game_handler() -> MainGameEventHandler:
    engine = new_game()
    return MainGameEventHandler(engine)
```

Add to `game/constants/colors.py`:

```python
MENU_TITLE = (255, 255, 63)
MENU_TEXT = WHITE
```

---

## Refactor: handlers return handlers

The current `EventHandler.handle_events()` returns `Action | None`. The new `BaseEventHandler.handle_events()` returns a handler. We need to unify these.

Update `EventHandler` to extend `BaseEventHandler` and return the correct type:

```python
class EventHandler(BaseEventHandler):
    def __init__(self, engine: Engine) -> None:
        self.engine = engine

    def handle_events(self, event: tcod.event.Event) -> BaseEventHandler:
        action: Action | None = None
        match event:
            case tcod.event.Quit():
                action = EscapeAction()
            case tcod.event.MouseMotion():
                self.engine.mouse_location = event.integer_position
                return self
            case tcod.event.MouseButtonDown():
                action = self.event_mousebuttondown(event)
            case tcod.event.KeyDown():
                action = self.event_keydown(event)

        if isinstance(action, BaseEventHandler):
            return action

        if action is not None:
            try:
                action.perform(self.engine, self.engine.player)
            except Impossible as exc:
                MessageLog.add_message(str(exc), colors.INVALID)
                return self

            if self.engine.player.is_alive:
                self.engine.handle_enemy_turns()

            if not self.engine.player.is_alive:
                new_handler = GameOverEventHandler(self.engine)
                new_handler.on_enter()
                return new_handler

            self.engine.update_fov()

            if isinstance(
                self,
                (InventoryActivateHandler, InventoryDropHandler, SelectIndexHandler),
            ):
                return MainGameEventHandler(self.engine)

        return self

    def event_mousebuttondown(self, event: tcod.event.MouseButtonDown) -> Action | None:
        return None

    def on_render(self, console: tcod.Console) -> None:
        self.engine.render(console)
```

The key change: `handle_events` now **returns a handler**. The caller replaces its current handler with the returned one. This makes handler transitions explicit and self-contained.

Update temporary handlers so they return handlers instead of mutating `engine.event_handler`:

```python
class InventoryEventHandler(EventHandler):
    ...

    def event_keydown(self, event: tcod.event.KeyDown) -> Action | BaseEventHandler | None:
        ...
        if event.sym == tcod.event.KeySym.ESCAPE:
            return MainGameEventHandler(self.engine)
        ...


class MainGameEventHandler(EventHandler):
    def event_keydown(self, event: tcod.event.KeyDown) -> Action | BaseEventHandler | None:
        ...
        if key == tcod.event.KeySym.I:
            return InventoryActivateHandler(self.engine)
        if key == tcod.event.KeySym.D:
            return InventoryDropHandler(self.engine)
        ...


class SelectIndexHandler(EventHandler):
    ...

    def event_keydown(self, event: tcod.event.KeyDown) -> Action | BaseEventHandler | None:
        ...
        if key == tcod.event.KeySym.ESCAPE:
            return MainGameEventHandler(self.engine)
        ...
```

`BaseEventHandler.handle_events()` already returns handler objects directly, so `InventoryActivateHandler` and `InventoryDropHandler` can now transition without touching `engine.event_handler`.

Targeting consumables also need to be migrated. In Part 9, `ConfusionConsumable.get_action()` and `FireballDamageConsumable.get_action()` installed their targeting handler by mutating `engine.event_handler` and returning `None`. That worked when the engine owned the active handler, but now the main loop drives state through `handle_events`'s return value. A `None` return leaves the inventory handler active and the targeting cursor never appears.

Rather than mixing handler transitions into `get_action()`, which is supposed to return an action, we give `Consumable` a dedicated method. Add `get_targeting_handler()` to the base class:

```python
class Consumable:
    def get_targeting_handler(self, engine: Engine) -> BaseEventHandler | None:
        return None
```

Then override it in the targeting consumables. Remove the old `get_action()` overrides from Part 9 at the same time, since that code is now dead:

```diff
 class ConfusionConsumable(Consumable):
-    def get_action(self, consumer: Actor, engine: Engine):
-        MessageLog.add_message("Select a target location.", colors.NEEDS_TARGET)
-        from game.input_handlers import SingleRangedAttackHandler
-        engine.event_handler = SingleRangedAttackHandler(
-            engine,
-            callback=lambda xy: ItemAction(item=self.entity, target_xy=xy),
-        )
-        return None
-
+    def get_targeting_handler(self, engine: Engine) -> BaseEventHandler | None:
+        MessageLog.add_message("Select a target location.", colors.NEEDS_TARGET)
+        from game.input_handlers import SingleRangedAttackHandler
+        return SingleRangedAttackHandler(
+            engine,
+            callback=lambda xy: ItemAction(item=self.entity, target_xy=xy),
+        )
+

 class FireballDamageConsumable(Consumable):
-    def get_action(self, consumer: Actor, engine: Engine):
-        MessageLog.add_message("Select a target location.", colors.NEEDS_TARGET)
-        from game.input_handlers import AreaRangedAttackHandler
-        engine.event_handler = AreaRangedAttackHandler(
-            engine,
-            radius=self.radius,
-            callback=lambda xy: ItemAction(item=self.entity, target_xy=xy),
-        )
-        return None
-
+    def get_targeting_handler(self, engine: Engine) -> BaseEventHandler | None:
+        MessageLog.add_message("Select a target location.", colors.NEEDS_TARGET)
+        from game.input_handlers import AreaRangedAttackHandler
+        return AreaRangedAttackHandler(
+            engine,
+            radius=self.radius,
+            callback=lambda xy: ItemAction(item=self.entity, target_xy=xy),
+        )
```

Finally, update `InventoryActivateHandler.on_item_selected` to check for a targeting handler before falling back to `get_action()`:

```python
class InventoryActivateHandler(InventoryEventHandler):
    TITLE = "Select an item to use"

    def on_item_selected(self, item: Item) -> Action | BaseEventHandler | None:
        handler = item.consumable.get_targeting_handler(self.engine)
        if handler is not None:
            return handler
        return item.consumable.get_action(self.engine.player, self.engine)
```

`get_action()` itself is unchanged: it still returns `Action | None`. The wider return type on `on_item_selected` comes from the fact that it can now also return a `BaseEventHandler` from `get_targeting_handler`. `handle_events` already handles this with `if isinstance(action, BaseEventHandler): return action`.

---

## main.py: save on quit, main menu start

```python
from __future__ import annotations

from pathlib import Path

import tcod

from game.input_handlers import BaseEventHandler, MainMenu
from game.setup_game import SAVE_PATH


def run(
    handler: BaseEventHandler,
    context: tcod.context.Context,
    console: tcod.Console,
    on_exit=None,
) -> None:
    """Drive the handler state machine until SystemExit. Calls on_exit(handler) before re-raising."""
    try:
        while True:
            console.clear()
            handler.on_render(console=console)
            context.present(console)

            for event in tcod.event.wait():
                context.convert_event(event)
                handler = handler.handle_events(event)

    except SystemExit:
        if on_exit is not None:
            on_exit(handler)
        raise


def save_game(handler: BaseEventHandler) -> None:
    from game.input_handlers import EventHandler, GameOverEventHandler
    if isinstance(handler, EventHandler) and not isinstance(handler, GameOverEventHandler):
        handler.engine.save_as(SAVE_PATH)
        print("Game saved.")


def main() -> None:
    screen_width = 80
    screen_height = 50

    tileset = tcod.tileset.load_tilesheet(
        Path(__file__).parent / "res" / "dejavu10x10_gs_tc.png",
        32,
        8,
        tcod.tileset.CHARMAP_TCOD,
    )

    handler: BaseEventHandler = MainMenu()

    with tcod.context.new(
        columns=screen_width,
        rows=screen_height,
        tileset=tileset,
        title="Roguelike Tutorial",
        vsync=True,
    ) as context:
        root_console = tcod.Console(screen_width, screen_height, order="F")
        run(handler, context, root_console, on_exit=save_game)


if __name__ == "__main__":
    main()
```

`Engine.run()` from earlier chapters no longer fits: the main menu runs *before* any engine exists, so the loop has to belong somewhere outside `Engine`. We move it back to a free `run()` function in `main.py`, similar in shape to the `game_loop()` from Part 1 but driving a handler state machine instead of a single update.

`save_game()` checks whether the current handler has an engine. If the player quits from the main menu (before starting a game), there is nothing to save. If they quit mid-game, `engine.save_as()` writes the file. If they quit from the game-over screen, no save is written because death already deleted it. The `on_exit` callback is invoked with the *current* handler (after any state-machine transitions), not the initial one.

The `try/except SystemExit` inside `run()` catches the quit signal raised by any handler, runs `save_game`, then re-raises so Python exits normally.

---

## Delete save on death

If the player dies, the save file is stale (it would reload a dead character). Delete it in `GameOverEventHandler`.

First, add `Path` and `SAVE_PATH` to the imports in `game/input_handlers.py`:

```diff
+from pathlib import Path
 ...
+from game.setup_game import SAVE_PATH, load_game, new_game
```

Then add the class:

```python
class GameOverEventHandler(EventHandler):
    def on_render(self, console: tcod.Console) -> None:
        super().on_render(console)

    def handle_events(self, event: tcod.event.Event) -> BaseEventHandler:
        match event:
            case tcod.event.Quit():
                raise SystemExit()
            case tcod.event.KeyDown():
                self.event_keydown(event)
        return self

    def event_keydown(self, event: tcod.event.KeyDown):
        if event.sym == tcod.event.KeySym.ESCAPE:
            raise SystemExit()
        return None

    def on_enter(self) -> None:
        save_path = Path(SAVE_PATH)
        if save_path.exists():
            save_path.unlink()
```

Call `handler.on_enter()` from `EventHandler.handle_events()` when transitioning to `GameOverEventHandler`:

```python
            if not self.engine.player.is_alive:
                new_handler = GameOverEventHandler(self.engine)
                new_handler.on_enter()
                return new_handler
```

---

## Complete file listing for Part 10

After this part, the major files look like this:

**`game/engine.py`** (additions only):

```python
import lzma
import pickle
from pathlib import Path

from game.message_log import MessageLog

class Engine:
    def save_as(self, filename: str) -> None:
        save_data = lzma.compress(
            pickle.dumps(
                {
                    "engine": self,
                    "message_log": MessageLog.messages,
                }
            )
        )
        Path(filename).write_bytes(save_data)

    @staticmethod
    def load(filename: str) -> Engine:
        data = pickle.loads(lzma.decompress(Path(filename).read_bytes()))
        MessageLog.messages = data["message_log"]
        return data["engine"]
```

**`game/setup_game.py`**: new file (full content above)

**`game/input_handlers.py`**: additions: `BaseEventHandler`, `PopupMessage`, `MainMenu`, updated `EventHandler`

**`main.py`**: starts with `MainMenu`, saves on `SystemExit`

---

## Testing your work

Run `python main.py`:

- [ ] The main menu appears with three options
- [ ] `N` starts a new game
- [ ] Play for a few turns (pick up items, fight enemies)
- [ ] Press `Q` or close the window, `"Game saved."` prints in the terminal
- [ ] Run `python main.py` again, `C` loads the game with the same map, entities, and message log
- [ ] Die in combat, the game returns to... nothing (you'd need to press Q to quit, which saves nothing, or add a "return to main menu" key)
- [ ] Run `python main.py` again, `C` shows `"No saved game to load."` because death deleted it

!!! tip "Add a return-to-menu option"
    Consider binding `Escape` in `GameOverEventHandler` to return to `MainMenu` instead of quitting. This is a one-line change in `event_keydown`: return `MainMenu()` instead of raising `SystemExit`.

---

## Summary

Save and load is complete. The verification milestone is met:

> **save, quit, reload → game state restored exactly**

Key additions:

- **`pickle` + `lzma`**: serialize/deserialize the engine plus static message log state
- **`game/setup_game.py`**: `new_game()` and `load_game()` functions
- **`BaseEventHandler`**: handler base that works without an engine (main menu, popups)
- **`PopupMessage`**: dismissable overlay with darkened background
- **`MainMenu`**: New / Continue / Quit at startup
- **Handlers return handlers**: clean state machine transitions
- **Save on quit, delete on death**: the save file is always valid

**Current architecture**:

- `main.py`: owns the outer app loop and active `BaseEventHandler`
- `setup_game.py`: creates new games and loads saved ones
- `BaseEventHandler`: common interface for menu, popup, and engine-backed handlers
- `MainMenu`: starts before an `Engine` exists
- `Engine.save_as()` / `Engine.load()`: serialize and restore the live object graph plus `MessageLog.messages`

**File structure**:

```txt
main.py                         ← modified
game/
├── __init__.py
├── actions.py
├── engine.py                   ← modified
├── exceptions.py
├── hud.py
├── input_handlers.py           ← modified
├── message_log.py
├── setup_game.py               ← new
├── constants/
│   ├── __init__.py
│   ├── colors.py               ← modified
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
│       └── inventory.py
└── map/
    ├── __init__.py
    ├── game_map.py
    ├── tile_types.py
    └── map_generator.py
```

---

## Exercises

1. **Return to main menu on death**:

    Change `GameOverEventHandler.event_keydown` so `Escape` returns `MainMenu()` instead of quitting. The player can start a new run without restarting the program.

2. **Multiple save slots**:

    Add a save-slot selection screen before `new_game()` and `load_game()`. Use a list `["slot1.sav", "slot2.sav", "slot3.sav"]` and show which slots are occupied (file exists) vs empty.

3. **Autosave**:

    Call `engine.save_as(SAVE_PATH)` after every `handle_enemy_turns()`. The game is now crash-proof, a power outage only loses the current turn. Measure whether the save is fast enough to be imperceptible (it should be, at under 1 ms for a small game state).

**Next**: [Part 11: Dungeon Levels and Experience](part-11.md)
