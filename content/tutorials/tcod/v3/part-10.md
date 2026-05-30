# Part 10: Save and Load

## What You Will Build

By the end of this part, your roguelike will have a main menu and a save system, allowing the player to quit the game and continue later from the saved state.

## Learning goals

- Serialize the entire game state to disk with `pickle` + `lzma`
- Add a main menu with New Game / Continue options
- Refactor game states to return states (not just actions)
- Delete the save file on player death
- Verify: save, quit, reload, game state is exactly as left

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
from pathlib import Path
import secrets

from game.constants import colors
from game.entities import factories
from game.engine import Engine
from game.message_log import MessageLog
from game.map.map_generator import generate_dungeon

SAVE_PATH = "savegame.sav"

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

    seed = int(os.environ.get("GAME_SEED", secrets.randbits(64)))
    #seed = 12345 # Write here the game seed to reproduce a map
    print(f"Game seed: {seed}")

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
        seed=seed,
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
  └── PopupMessageState (shows a message over a frozen background)
```

Add to `game/game_states.py`:

```python
class BaseGameState:

    def handle_events(self, event: tcod.event.Event):
        state = self.dispatch(event)
        if isinstance(state, BaseGameState):
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

    def on_render(self, console: tcod.console.Console) -> None:
        raise NotImplementedError()


class PopupMessageState(BaseGameState):
    """Display a message over a screenshot of the current state."""

    def __init__(self, parent_state: BaseGameState, text: str) -> None:
        self.parent = parent_state
        self.text = text

    def on_render(self, console: tcod.console.Console) -> None:
        self.parent.on_render(console)
        console.fg //= 8
        console.bg //= 8
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

`PopupMessageState` darkens the existing frame by right-shifting all color channels by 3 (dividing by 8), then overlays centered text. Any keypress dismisses it and returns to the parent state.

---

## MainMenuState

```python
from game.setup_game import SAVE_PATH, load_game, new_game


class MainMenuState(BaseGameState):
    """Renders the main menu and handles New / Continue / Quit."""

    def on_render(self, console: tcod.console.Console) -> None:
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
                    return load_game(SAVE_PATH)

                except FileNotFoundError:
                    return PopupMessageState(self, "No saved game to load.")

                except Exception as exc:
                    return PopupMessageState(self, f"Failed to load save:\n{exc}")

            case tcod.event.KeySym.N:
                return new_game_state()

        return None


def new_game_state() -> MainGameState:
    engine = new_game()
    return MainGameState(engine)
```

Add to `game/constants/colors.py`:

```python
MENU_TITLE = (255, 255, 63)
MENU_TEXT = WHITE
```

---

## Refactor: states return states

The current `GameState.handle_events()` returns `Action | None`. The new `BaseGameState.handle_events()` returns a state. We need to unify these.

Update `GameState` to extend `BaseGameState` and return the correct type:

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
                action = self.event_mousebuttondown(event)
            case tcod.event.KeyDown():
                action = self.event_keydown(event)

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
                elif isinstance(action, AreaRangedTargetingAction):
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

    def event_mousebuttondown(self, event: tcod.event.MouseButtonDown) -> Action | None:
        return None

    def on_render(self, console: tcod.console.Console) -> None:
        self.engine.render(console)
```

The key change: `handle_events` now **returns a state**. The caller replaces its current state with the returned one. This makes state transitions explicit and self-contained.

Update states so they return states instead of mutating `engine.game_state`:

```python
class InventoryState(ActionModalState):
    ...

    def event_keydown(self, event: tcod.event.KeyDown) -> Action | BaseGameState | None:
        ...
        if event.sym == tcod.event.KeySym.ESCAPE:
            return MainGameState(self.engine)
        ...


class MainGameState(GameState):

    def event_keydown(self, event: tcod.event.KeyDown) -> Action | BaseGameState | None:
        ...
        if key == tcod.event.KeySym.I:
            return InventoryUseState(self.engine)
        if key == tcod.event.KeySym.D:
            return InventoryDropState(self.engine)
        ...


class SelectIndexState(ActionModalState):
    ...

    def event_keydown(self, event: tcod.event.KeyDown) -> Action | BaseGameState | None:
        ...
        if key == tcod.event.KeySym.ESCAPE:
            return MainGameState(self.engine)
        ...
```

`BaseGameState.handle_events()` already returns state objects directly, so `InventoryUseState` and `InventoryDropState` can now transition without touching `engine.game_state`.

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
    on_exit=None,
) -> None:
    """Drive the game state machine until SystemExit. Calls on_exit(state) before re-raising."""
    try:
        while True:
            console.clear()
            state.on_render(console=console)
            context.present(console)

            for event in tcod.event.wait():
                event = context.convert_event(event)
                state = state.handle_events(event)

    except SystemExit:
        if on_exit is not None:
            on_exit(state)
        raise


def save_game(state: BaseGameState) -> None:
    from game.game_states import GameState, GameOverState
    if isinstance(state, GameState) and not isinstance(state, GameOverState):
        state.engine.save_as(SAVE_PATH, state)
        print("Game saved.")


def main() -> None:
    screen_width  = 80
    screen_height = 50

    tileset = tcod.tileset.load_tilesheet(
        Path(__file__).parent / "res" / "dejavu12x12_gs_tc.png",
        32,
        8,
        tcod.tileset.CHARMAP_TCOD,
    )

    state: BaseGameState = MainMenuState()

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

`save_game()` checks whether the current state has an engine. If the player quits from the main menu (before starting a game), there is nothing to save. If they quit mid-game, `state.engine.save_as(SAVE_PATH, state)` serializes the active state, which includes the engine through `state.engine`, so loading restores exactly the state the player was in when they quit. If they quit from the game-over screen, no save is written because death already deleted it. The `on_exit` callback is invoked with the *current* state (after any state-machine transitions), not the initial one.

The `try/except SystemExit` inside `run()` catches the quit signal raised by any state, runs `save_game`, then re-raises so Python exits normally.

---

## Delete save on death

If the player dies, the save file is stale (it would reload a dead character). Delete it in `GameOverState`.

First, add `Path` and `SAVE_PATH` to the imports in `game/game_states.py`:

```diff
+from pathlib import Path
 ...
+from game.setup_game import SAVE_PATH, load_game, new_game
```

Then add the class:

```python
class GameOverState(GameState):

    def on_render(self, console: tcod.console.Console) -> None:
        super().on_render(console)

    def handle_events(self, event: tcod.event.Event) -> BaseGameState:
        match event:
            case tcod.event.Quit():
                raise SystemExit()
            case tcod.event.KeyDown():
                if (result := self.event_keydown(event)) is not None:
                    return result
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

Call `new_state.on_enter()` from `GameState.handle_events()` when transitioning to `GameOverState`:

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
        data = pickle.loads(lzma.decompress(Path(filename).read_bytes()))
        MessageLog.messages = data["message_log"]
        return data["state"]
```

Also remove `self.game_state`, `Engine.handle_events()`, and `Engine.run()`. State ownership and the main loop both move to `main.py`:

```diff
-from game.game_states import GameState, MainGameState
 ...
 class Engine:

     def __init__(self, ...) -> None:
-        self.game_state: GameState = MainGameState(self)
         ...

-    def handle_events(self, events: Iterable[Any]) -> None:
-        for event in events:
-            action = self.game_state.handle_events(event)
-            ...
-
-    def run(self, context: Context, console: Console) -> None:
-        while True:
-            console.clear()
-            self.game_state.on_render(console=console)
-            context.present(console)
-            ...
```

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
- [ ] Press `Q` or close the window, `"Game saved."` prints in the terminal
- [ ] Run `python main.py` again, `C` loads the game with the same map, entities, and message log
- [ ] Die in combat, the game returns to... nothing (you'd need to press Q to quit, which saves nothing, or add a "return to main menu" key)
- [ ] Run `python main.py` again, `C` shows `"No saved game to load."` because death deleted it

!!! tip "Add a return-to-menu option"
    Consider binding `Escape` in `GameOverState` to return to `MainMenuState` instead of quitting. This is a one-line change in `event_keydown`: return `MainMenuState()` instead of raising `SystemExit`.

---

## Summary

Save and load is complete. The verification milestone is met:

> **save, quit, reload → game state restored exactly**

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
