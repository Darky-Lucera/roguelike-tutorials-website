# Part 10: Save and Load

## What You Will Build

By the end of this part, your roguelike will have a main menu and a save system, allowing the player to quit the game and continue later from the saved state.

## Learning goals

- Serialize the entire game state to disk with `pickle` + `lzma`
- Add a main menu with New Game / Continue options
- Refactor game states to return states (not just actions)
- Delete the save file on player death
- Verify: save, quit, reload; map, entities, inventory, and message history are restored

---

## The serialization problem

Saving a game means converting the entire live object graph (engine, game map, entities, components, AI instances) into bytes that can be written to disk and reconstructed later.

```text
Engine
  ├── player (Actor)
  │     ├── fighter (Fighter)   ← component back-reference to player
  │     └── inventory (Inventory)
  ├── game_map (GameMap)
  │     └── entities: set[Entity]
  │           └── orc (Actor)
  │                 └── ai (HostileEnemy)  ← back-reference to the orc
Static state
  └── MessageLog.messages
```

The challenge: `Fighter.entity` points back to the `Actor` that owns it, `HostileEnemy.entity` points to the orc, and in Part 9 `ConfusedEnemy.previous_ai` holds another AI. This is a circular reference graph. The message log is a separate static list, so we save it next to the engine.

**JSON** cannot handle circular references without custom encoding, every class needs a `to_dict()` / `from_dict()` pair. That is a lot of boilerplate.

**`pickle`** handles circular references natively. Python's pickle module serializes the entire object graph, following references and deduplicating them automatically. It is the right tool for internal game state that you fully control.

!!! danger "pickle is not safe with untrusted data"
    Never load a pickle file from an unknown source, pickle can execute arbitrary code during deserialization. For a single-player game saving its own state, this is fine.

!!! warning "Save files are not stable between code changes"
    `pickle` stores class names and attribute names. If you rename a class, move a module, or add a required attribute between parts, old save files will fail with `AttributeError` or `ModuleNotFoundError`. Delete your save file whenever you change the code.

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

    def save_as(self, filename: str, active_state: object = None) -> None:
        save_data = lzma.compress(
            pickle.dumps(
                {
                    "state": active_state,
                    "message_log": MessageLog.messages,
                }
            )
        )

        Path(filename).write_bytes(save_data)
```

`pickle.dumps(...)` serializes `active_state` (which transitively includes the engine and everything it references) plus the static message log. `lzma.compress` shrinks it. `Path.write_bytes` writes the compressed bytes to disk.

Loading is the inverse:

```python
    @staticmethod
    def load(filename: str):
        save_data = Path(filename).read_bytes()
        data = pickle.loads(lzma.decompress(save_data))
        MessageLog.messages = data["message_log"]

        return data["state"]
```

`MessageLog.messages` is stored explicitly because it is class-level state, not an instance attribute of `Engine`. Without this extra field, loading would restore the map and entities but lose the visible message history.

The active state is now the root of the saved object graph. Any state that inherits from `GameState` stores the engine on `self.engine`, so code that needs the engine can access it through the current state after checking that the state is engine-backed.

---

## setup_game.py

We need a function that creates a fresh game (used by "New Game") and one that loads an existing save (used by "Continue"). Extract them into a dedicated module so `main.py` and the main menu state stay clean.

Create `game/setup_game.py`:

```python
from __future__ import annotations

import copy
import os
import secrets
from pathlib import Path

from game.constants import colors
from game.engine import Engine
from game.entities import factories
from game.map.map_generator import generate_dungeon
from game.message_log import MessageLog

SAVE_PATH             = "savegame.sav"

MAP_WIDTH             = 80
MAP_HEIGHT            = 44
MAX_ROOMS             = 30
ROOM_MIN_SIZE         = 6
ROOM_MAX_SIZE         = 10
MIN_MONSTERS_PER_ROOM = 0
MAX_MONSTERS_PER_ROOM = 2
MIN_ITEMS_PER_ROOM    = 0
MAX_ITEMS_PER_ROOM    = 2


def new_game() -> Engine:
    """Return a fresh engine for a brand-new game."""
    MessageLog.clear()

    # Part-3. Ex 1: Reproducible dungeons
    seed = int(os.environ.get("GAME_SEED", secrets.randbits(64)))
    #seed = 12345 # Write here the game seed to reproduce a map
    print(f"Game seed: {seed}")

    player = copy.deepcopy(factories.player)

    game_map = generate_dungeon(
        max_rooms             = MAX_ROOMS,
        room_min_size         = ROOM_MIN_SIZE,
        room_max_size         = ROOM_MAX_SIZE,
        map_width             = MAP_WIDTH,
        map_height            = MAP_HEIGHT,
        min_monsters_per_room = MIN_MONSTERS_PER_ROOM,
        max_monsters_per_room = MAX_MONSTERS_PER_ROOM,
        min_items_per_room    = MIN_ITEMS_PER_ROOM,
        max_items_per_room    = MAX_ITEMS_PER_ROOM,
        player                = player,
        seed                  = seed,
    )

    engine = Engine(game_map=game_map, player=player)
    engine.update_fov()

    MessageLog.add_message(
        "Hello and welcome, adventurer, to yet another dungeon!",
        colors.WELCOME_TEXT,
    )

    return engine


def load_game(filename: str):
    """Load a save file and return the saved game state."""
    if not Path(filename).exists():
        raise FileNotFoundError(f"No save file found at {filename!r}")

    return Engine.load(filename)
```

---

## BaseGameState and PopupMessageState

Before adding the main menu, we need one more abstraction: a state that does *not* delegate to `Engine`. The main menu runs before any engine exists, so it cannot inherit from the current `GameState`.

We split the hierarchy:

```text
BaseGameState
  ├── GameState (has engine, handles actions)
  │     ├── MainGameState
  │     ├── GameOverState
  │     └── InventoryState (and subclasses)
  │           ├── InventoryUseState
  │           └── InventoryDropState
  ├── MainMenuState  (no engine yet)
  └── PopupMessageState (renders a message overlay over the parent state)
```

Add to `game/game_states.py`, immediately before `GameState`:

```python
class BaseGameState:

    def handle_events(self, event: tcod.event.Event) -> BaseGameState:
        state = self.dispatch(event)
        if isinstance(state, BaseGameState):
            return state

        return self

    def dispatch(self, event: tcod.event.Event) -> Action | BaseGameState | None:
        match event:
            case tcod.event.Quit():
                raise SystemExit()

            case tcod.event.KeyDown():
                return self.event_keydown(event)

            case tcod.event.MouseButtonDown():
                return self.event_mousebuttondown(event)

        return None

    def event_keydown(self, _event: tcod.event.KeyDown) -> Action | BaseGameState | None:
        return None

    def event_mousebuttondown(self, _event: tcod.event.MouseButtonDown) -> Action | None:
        return None

    def on_render(self, console: tcod.console.Console) -> None:
        raise NotImplementedError()


class PopupMessageState(BaseGameState):
    """Display a message rendered over the parent state with a darkened overlay."""

    def __init__(self, parent_state: BaseGameState, text: str) -> None:
        self.parent = parent_state
        self.text   = text

    def on_render(self, console: tcod.console.Console) -> None:
        self.parent.on_render(console)
        console.fg[:] = console.fg // 8
        console.bg[:] = console.bg // 8
        console.print(
            console.width // 2,
            console.height // 2,
            self.text,
            fg = colors.WHITE,
            bg = colors.BLACK,
            alignment = tcod.constants.CENTER,
        )

    def event_keydown(self, _event: tcod.event.KeyDown) -> Action | BaseGameState | None:
        return self.parent
```

`PopupMessageState` darkens the existing frame by dividing all color channels by 8, then overlays centered text. Any keypress dismisses it and returns to the parent state.

---

## MainMenuState

The main menu needs two new key constants. Add them to `game/constants/keys.py`:

```python
KEY_NEW_GAME     = tcod.event.KeySym.N
KEY_CONTINUE     = tcod.event.KeySym.C
```

`KEY_QUIT_GAME` (already defined as `ESCAPE`) covers the quit option. With these three constants in place, the menu handler is fully decoupled from raw `KeySym` values.

`MainMenuState` is the first state the game enters. Unlike every other state, it holds no engine reference: the engine does not exist until the player makes a choice.

`on_render` draws the title and the three menu options centered on screen. `event_keydown` handles three transitions:

- `keys.KEY_NEW_GAME` (`N`): calls `new_game()`, wraps the fresh engine in a `MainGameState`, and returns it.
- `keys.KEY_CONTINUE` (`C`): calls `load_game()`. If no save file exists it falls back to a `PopupMessageState`; if the file is corrupt it shows the exception message.
- `keys.KEY_QUIT_GAME` (`Esc`): raises `SystemExit`.

Add to `game/game_states.py` after `PopupMessageState`:

```python
class MainMenuState(BaseGameState):
    """Renders the main menu and handles New / Continue / Quit."""

    def on_render(self, console: tcod.console.Console) -> None:
        console.print(
            console.width  // 2,
            console.height // 2 - 4,
            "ROGUELIKE TUTORIAL",
            fg = colors.MENU_TITLE,
            alignment = tcod.constants.CENTER,
        )
        for i, text in enumerate(
            [
                "[N] Play a new game",
                "[C] Continue last game",
                "[Esc] Quit"
            ]
        ):
            console.print(
                console.width  // 2,
                console.height // 2 - 2 + i,
                text.ljust(30),
                fg = colors.MENU_TEXT,
                bg = colors.BLACK,
                alignment = tcod.constants.CENTER,
                bg_blend  = tcod.libtcodpy.BKGND_ALPHA(64),
            )

    def event_keydown(self, event: tcod.event.KeyDown) -> Action | BaseGameState | None:
        from game.setup_game import SAVE_PATH, load_game

        match event.sym:
            case keys.KEY_NEW_GAME:
                return new_game_state()

            case keys.KEY_QUIT_GAME:
                raise SystemExit()

            case keys.KEY_CONTINUE:
                try:
                    return load_game(SAVE_PATH)

                except FileNotFoundError:
                    return PopupMessageState(self, "No saved game to load.")

                except Exception as ex:  # pylint: disable=broad-exception-caught
                    return PopupMessageState(self, f"Failed to load save:\n{ex}")

        return None


def new_game_state() -> MainGameState:
    from game.setup_game import new_game

    engine = new_game()
    return MainGameState(engine)
```

`setup_game` is imported locally inside each method rather than at the top of the file. A module-level import would create the circular chain `game_states → setup_game → engine → game_states`.

The `except Exception` that catches load failures is intentionally broad at the menu boundary: a corrupt or incompatible save file should show a user-facing popup, not crash the program.

Add to `game/constants/colors.py`:

```python
MENU_TITLE             = Color(255, 255, 63)
MENU_TEXT              = WHITE
```

---

## Refactor: states return states

!!! warning "Complete all steps and 'Delete save on death' before running type checks"
    After adding `BaseGameState`, mypy will report that `GameState` subclasses are not assignable to `BaseGameState` until Step 2 is done. Step 2 also introduces a call to `GameOverState.on_enter()`, which is not defined until the "Delete save on death" section below. Complete Steps 1, 2, 3 **and** "Delete save on death" before checking types.

The current `GameState.handle_events()` returns `Action | None` and mutates `engine.game_state` as a side effect. The new design returns the next state directly. The caller (`run()` in `main.py`) replaces its local `state` variable with the return value; no shared mutable attribute needed.

### Step 1: remove `game_state` from `Engine`

`engine.game_state` was only needed so states could mutate the active state from inside the engine. With state transitions now expressed as return values, that attribute is no longer needed.

In `game/engine.py`, remove the three imports that are no longer needed, drop `self.game_state` from `__init__`, and delete `handle_events` and `run` entirely (the main loop moves to `main.py`):

```diff
-from collections.abc import Iterable
+import lzma
+import pickle
+from pathlib import Path

 import tcod.constants
-import tcod.event
 import tcod.map
 from tcod.console import Console
-from tcod.context import Context

 from game import hud
 from game.entities.entity import Actor
-from game.game_states import GameState, MainGameState
 from game.map.game_map import GameMap
 from game.message_log import MessageLog

 class Engine:

     def __init__(self,
                  game_map: GameMap,
                  player: Actor,
                  # Part-4. Ex 1: Variable torch radius
                  fov_radius: int = 8,
                  # Part-4. Ex 4: Fading memory
                  fading_memory: bool = False,
                  memory_duration: int = 10) -> None:
         ...
-        self.game_state: GameState = MainGameState(self)
         self.update_fov()

-    def handle_events(self, events: Iterable[tcod.event.Event]) -> None:
-        for event in events:
-            self.game_state.handle_events(event)
-
-    def run(self, context: Context, console: Console) -> None:
-        while True:
-            console.clear()
-            self.game_state.on_render(console=console)
-            context.present(console)
-            for event in tcod.event.wait():
-                event = context.convert_event(event)
-                self.handle_events([event])
```

Removing `from game.game_states import GameState, MainGameState` breaks the circular chain `engine → game_states → setup_game → engine`.

### Step 2: replace `GameState` with the new version

**Delete the entire existing `GameState` class** and replace it with:

```python
class GameState(BaseGameState):

    def __init__(self, engine: Engine) -> None:
        self.engine = engine

    def handle_events(self, event: tcod.event.Event) -> BaseGameState:
        action: Action | BaseGameState | None = None
        match event:
            case tcod.event.Quit():
                action = EscapeAction()

            case tcod.event.MouseMotion():
                self.engine.mouse_location = event.integer_position
                return self

            case tcod.event.MouseButtonDown():
                action = self.event_mousebuttondown(event)  # pylint: disable=assignment-from-none

            case tcod.event.KeyDown():
                action = self.event_keydown(event)  # pylint: disable=assignment-from-none

        if isinstance(action, BaseGameState):
            return action

        if action is not None:
            try:
                action.perform(self.engine, self.engine.player)

            except Impossible as exc:
                MessageLog.add_message(str(exc), colors.INVALID)
                return self

            if isinstance(action, TargetingAction):
                MessageLog.add_message(action.prompt, colors.NEEDS_TARGET)
                if isinstance(action, SingleRangedTargetingAction):
                    return SingleRangedAttackState(self.engine, callback=action.callback)

                if isinstance(action, AreaRangedTargetingAction):
                    return AreaRangedAttackState(
                        self.engine,
                        radius=action.radius,
                        color=action.color,
                        callback=action.callback,
                    )

            if self.engine.player.is_alive:
                self.engine.handle_enemy_turns()

            if not self.engine.player.is_alive:
                new_state = GameOverState(self.engine)
                new_state.on_enter()
                return new_state

            self.engine.update_fov()

            if isinstance(self, ActionModalState):
                return MainGameState(self.engine)

        return self

    def event_keydown(self, _event: tcod.event.KeyDown) -> Action | BaseGameState | None:
        return None

    def event_mousebuttondown(self, _event: tcod.event.MouseButtonDown) -> Action | None:
        return None

    def on_render(self, console: tcod.console.Console) -> None:
        self.engine.render(console)
```

`handle_events` calls `new_state.on_enter()` when the player dies. That method will be filled in the "Delete save on death" section; for now, add a stub to `GameOverState` so the code compiles:

```diff
 class GameOverState(GameState):
+    def on_enter(self) -> None:
+        pass
```

`handle_events` now checks `isinstance(action, BaseGameState)` before treating it as an `Action`. If a subclass's `event_keydown` returns a state (e.g. `InventoryUseState`), the method returns it immediately without executing any game logic. The base `event_keydown` declaration uses `Action | BaseGameState | None` so subclasses can return either without a type mismatch.

### Step 3: replace `engine.game_state` mutations with return values

There are three places in `game_states.py` that assign to `self.engine.game_state`. Replace each one with a return value.

**`SelectIndexState.event_keydown`**:

```diff
-    def event_keydown(self, event: tcod.event.KeyDown) -> Action | None:
+    def event_keydown(self, event: tcod.event.KeyDown) -> Action | BaseGameState | None:
         key = event.sym
@@
         if key == keys.KEY_EXIT:
-            self.engine.game_state = MainGameState(self.engine)
-            return None
+            return MainGameState(self.engine)
```

**`MainGameState.event_keydown`**:

```diff
-    def event_keydown(self, event: tcod.event.KeyDown) -> Action | None:
+    def event_keydown(self, event: tcod.event.KeyDown) -> Action | BaseGameState | None:
         key = event.sym
@@
         if key == keys.KEY_INVENTORY:
-            self.engine.game_state = InventoryUseState(self.engine)
+            return InventoryUseState(self.engine)

         if key == keys.KEY_DROP:
-            self.engine.game_state = InventoryDropState(self.engine)
+            return InventoryDropState(self.engine)
```

**`InventoryState.event_keydown`**:

```diff
-    def event_keydown(self, event: tcod.event.KeyDown) -> Action | None:
+    def event_keydown(self, event: tcod.event.KeyDown) -> Action | BaseGameState | None:
         key = event.sym
@@
         if key == keys.KEY_EXIT:
-            self.engine.game_state = MainGameState(self.engine)
-            return None
+            return MainGameState(self.engine)
```

After these three changes, `self.engine.game_state` no longer appears anywhere in `game_states.py`.

Part 9 already wired this correctly: `ConfusionConsumable` and `FireballDamageConsumable` return `TargetingAction` data classes from `get_action()`, and `handle_events()` dispatches on them. The only migration here is changing the mutation (`engine.game_state = ...`) to a return value, which the updated `handle_events()` above already reflects. No changes to `consumable.py` are needed.

---

## main.py: save on quit, main menu start

```python
from __future__ import annotations

from pathlib import Path

import tcod

from game.game_states import BaseGameState, MainMenuState
from game.setup_game import SAVE_PATH


def run(
    state: BaseGameState,
    context: tcod.context.Context,
    console: tcod.console.Console,
    on_exit = None,
) -> None:
    """Drive the game state machine until SystemExit. Calls on_exit(state) before re-raising."""
    try:
        while True:
            console.clear()
            state.on_render(console = console)
            context.present(console)

            for event in tcod.event.wait():
                event = context.convert_event(event)
                state = state.handle_events(event)

    except SystemExit:
        if on_exit is not None:
            on_exit(state)
        raise


def save_game(state: BaseGameState) -> None:
    from game.game_states import GameOverState, GameState
    if isinstance(state, GameState) and not isinstance(state, GameOverState):
        state.engine.save_as(SAVE_PATH, state)
        print("Game saved.")


def main() -> None:
    screen_width  = 80
    screen_height = 50

    state: BaseGameState = MainMenuState()

    tileset = tcod.tileset.load_tilesheet(
        Path(__file__).parent / "res" / "dejavu12x12_gs_tc.png",
        32,
        8,
        tcod.tileset.CHARMAP_TCOD,
    )

    title   = "Roguelike Tutorial"
    version = "0.1.0"
    app_id  = "com.tutorial.roguelike"

    tcod.lib.SDL_SetAppMetadata(
        title.encode("utf-8"),
        version.encode("utf-8"),
        app_id.encode("utf-8")
    )
    tcod.lib.SDL_SetHint(
        b"SDL_RENDER_SCALE_QUALITY",
        b"0" # Nearest pixel sampling
    )

    with tcod.context.new(
        columns          = screen_width,
        rows             = screen_height,
        tileset          = tileset,
        title            = title,
        vsync            = True,
        sdl_window_flags = tcod.context.SDL_WINDOW_ALLOW_HIGHDPI | tcod.context.SDL_WINDOW_RESIZABLE,
    ) as context:
        root_console = tcod.console.Console(screen_width, screen_height, order="F")
        run(state, context, root_console, on_exit=save_game)


if __name__ == "__main__":
    main()
```

`Engine.run()` from earlier chapters no longer fits: the main menu runs *before* any engine exists, so the loop has to belong somewhere outside `Engine`. We move it back to a free `run()` function in `main.py`, similar in shape to the `game_loop()` from Part 1 but driving a game state machine instead of a single update.

`main.py` no longer generates a seed or adds the welcome message. Both belong in `new_game()`: the seed decides the map layout, and the welcome message is part of the initial game state, not app setup.

`save_game()` checks whether the current state has an engine. If the player quits from the main menu (before starting a game), there is nothing to save. If they quit mid-game, `state.engine.save_as(SAVE_PATH, state)` serializes the active state, which includes the engine through `state.engine`, so loading restores exactly the state the player was in when they quit. If they quit from the game-over screen, no save is written because death already deleted it. The `on_exit` callback is invoked with the *current* state (after any state-machine transitions), not the initial one.

The `try/except SystemExit` inside `run()` catches the quit signal raised by any state, runs `save_game`, then re-raises so Python exits normally.

---

## Delete save on death

If the player dies, the save file is stale (it would reload a dead character). Delete it in `GameOverState`.

Add `Path` to the imports in `game/game_states.py`:

```diff
+from pathlib import Path
```

Do **not** add a module-level import of `setup_game` here: `game_states.py` already participates in the import graph through `engine.py`, and a top-level `from game.setup_game import ...` would create a circular chain. Use a local import inside `on_enter()` instead.

Replace the stub `on_enter()` added in Step 2 with the real implementation. Also update `GameOverState.event_keydown` to use the wider return type:

```diff
 class GameOverState(GameState):

-    def event_keydown(self, event: tcod.event.KeyDown) -> Action | None:
+    def event_keydown(self, event: tcod.event.KeyDown) -> Action | BaseGameState | None:
         if event.sym == keys.KEY_QUIT_GAME:
             return EscapeAction()

         return None

-    def on_enter(self) -> None:
-        pass
+    def on_enter(self) -> None:
+        from game.setup_game import SAVE_PATH
+
+        save_path = Path(SAVE_PATH)
+        if save_path.exists():
+            save_path.unlink()
```

`GameOverState` keeps its own `event_keydown` so Escape still quits from the game-over screen. The save file is deleted when the state is entered, before the player has a chance to quit.

`new_state.on_enter()` is already called from `GameState.handle_events()` (added in Step 2):

```python
            if not self.engine.player.is_alive:
                new_state = GameOverState(self.engine)
                new_state.on_enter()
                return new_state
```

---

## Complete file listing for Part 10

After this part, the major files look like this:

**`game/engine.py`**: additions (`save_as`, `load`) and one removal:

```python
import lzma
import pickle
from pathlib import Path

from game.message_log import MessageLog


class Engine:

    def save_as(self, filename: str, active_state: object = None) -> None:
        save_data = lzma.compress(
            pickle.dumps(
                {
                    "state": active_state,
                    "message_log": MessageLog.messages,
                }
            )
        )

        Path(filename).write_bytes(save_data)

    @staticmethod
    def load(filename: str):
        save_data = Path(filename).read_bytes()
        data = pickle.loads(lzma.decompress(save_data))
        MessageLog.messages = data["message_log"]

        return data["state"]
```

The removals (`self.game_state`, `handle_events`, `run`) are covered in the Refactor section above.

`handle_events` and `run` both referenced `self.game_state`; without it they would crash if called. The free `run()` in `main.py` replaces them entirely.

This also breaks the potential circular import introduced when `game_states.py` imports from `setup_game.py`: once `engine.py` no longer imports from `game_states.py`, the chain `game_states → setup_game → engine` is not circular.

**`game/setup_game.py`**: new file (full content above)

**`game/game_states.py`**: additions: `BaseGameState`, `PopupMessageState`, `MainMenuState`, updated `GameState`

**`main.py`**: starts with `MainMenuState`, saves on `SystemExit`

---

## Testing your work

Run `python main.py`:

- [ ] The main menu appears with three options
- [ ] `N` starts a new game
- [ ] Play for a few turns (pick up items, fight enemies)
- [ ] Press `Esc` or close the window, `"Game saved."` prints in the terminal
- [ ] Run `python main.py` again, `C` loads the game with the same map, entities, and message log
- [ ] Die in combat, the game-over screen appears
- [ ] Press `Esc` to quit, no new save is written
- [ ] Run `python main.py` again, `C` shows `"No saved game to load."` because death deleted it

!!! tip "Add a return-to-menu option"
    Consider binding `Escape` in `GameOverState` to return to `MainMenuState` instead of quitting. This is a one-line change in `event_keydown`: return `MainMenuState()` instead of raising `SystemExit`.

---

## Summary

Save and load is complete. The verification milestone is met:

> **save, quit, reload → map, entities, inventory, and message history restored**

Key additions:

- **`pickle` + `lzma`**: serialize/deserialize the active state plus static message log state
- **`game/setup_game.py`**: `new_game()` and `load_game()` functions
- **`BaseGameState`**: state base that works without an engine (main menu, popups)
- **`PopupMessageState`**: dismissable overlay with darkened background
- **`MainMenuState`**: New / Continue / Quit at startup
- **States return states**: clean state machine transitions
- **Save on quit, delete on death**: the save file is always valid

**Current architecture**:

- `main.py`: owns the outer app loop and active `BaseGameState`
- `setup_game.py`: creates new games and loads saved ones
- `BaseGameState`: common interface for menu, popup, and engine-backed states
- `MainMenuState`: starts before an `Engine` exists
- `Engine.save_as(filename, state)` / `Engine.load()`: serialize and restore the active state (engine included transitively) plus `MessageLog.messages`

**File structure**:

```text
main.py                         ← modified
game/
├── __init__.py
├── actions.py
├── engine.py                   ← modified
├── exceptions.py
├── hud.py
├── game_states.py              ← modified
├── message_log.py
├── setup_game.py               ← new
├── constants/
│   ├── __init__.py
│   ├── colors.py               ← modified
│   ├── keys.py                 ← modified
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

    Change `GameOverState.event_keydown` so `Escape` returns `MainMenuState()` instead of quitting. The player can start a new run without restarting the program.

2. **Multiple save slots**:

    Add a save-slot selection screen before `new_game()` and `load_game()`. Use a list `["slot1.sav", "slot2.sav", "slot3.sav"]` and show which slots are occupied (file exists) vs empty.

3. **Autosave**:

    Call `self.engine.save_as(SAVE_PATH, self)` after every `handle_enemy_turns()` in `GameState.handle_events()`. The game is now crash-proof, a power outage only loses the current turn. Measure whether the save is fast enough to be imperceptible (it should be, at under 1 ms for a small game state).
