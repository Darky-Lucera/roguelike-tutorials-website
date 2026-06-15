# Part 1: Drawing the @ and Moving It Around

## What You Will Build

By the end of this part, you will have the first playable version of your roguelike: a player character, `@`, moving with the arrow keys inside a small room bounded by walls, `#`.

## Learning goals

- Understand the game loop and why roguelikes structure it the way they do
- Open a tcod window using three key objects: tileset, console, and context
- Draw a character on screen
- Read keyboard input and move the character

---

## The game loop

Every game, from Pong to Elden Ring, runs a loop. The loop repeats until the player quits, doing the same three things each iteration:

```text
┌──────────┐     ┌──────────┐     ┌──────────┐
│  INPUT   │────▶│  UPDATE  │────▶│  RENDER  │
└──────────┘     └──────────┘     └──────────┘
     ▲                                 │
     └─────────────────────────────────┘
```

- **Input**: read what the player did (key pressed, mouse moved)
- **Update**: apply the consequences (move the player, resolve combat)
- **Render**: draw the current state to the screen

In a real-time game, this loop runs 30 or 60 times per second regardless of whether the player does anything. In a **turn-based roguelike**, the loop is different: we *wait* for input before updating. Nothing happens until the player presses a key. This is what makes the "time only passes when you act" feeling.

Our loop will look like this:

```text
Render ──▶ Wait for input ──▶ Process action ──▶ (back to Render)
```

We render first so the player sees the current state, then block until they act.

---

## Three tcod objects

Before writing code, understand what tcod gives us:

**Tileset**: the font sheet. A grid image where each cell is one character tile. tcod uses it to draw characters on screen. We load it once at startup.

**Console**: an in-memory buffer of cells. Each cell holds a character, a foreground color, and a background color. Drawing to the console does not show anything on screen yet; it is just updating memory.

**Context**: the actual window. When we call `context.present(console)`, it takes the console buffer and displays it. The context also gives us events (key presses, quit signal).

The workflow every frame:

```text
console.clear() ──▶ draw to console ──▶ context.present(console) ──▶ wait for events
```

---

## Opening a window

Replace `main.py` with:

```python
from __future__ import annotations

from pathlib import Path

import tcod


def main() -> None:
    screen_width  = 80
    screen_height = 50

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
        console = tcod.console.Console(screen_width, screen_height, order="F")

        while True:
            console.print(x=1, y=1, text="@")
            context.present(console)

            for event in tcod.event.wait():
                match event:
                    case tcod.event.Quit():
                        raise SystemExit()


if __name__ == "__main__":
    main()
```

Run it with `python main.py`. You should see a window with an `@` in the top-left corner. Close it by clicking the X.

The 80 × 50 dimensions follow the classic 80-column terminal convention going back to the VT100; 50 rows leaves comfortable space for the map plus a UI strip we will add in Part 7.

Let's go through each part.

### The tileset

```python
tileset = tcod.tileset.load_tilesheet(
    Path(__file__).parent / "res" / "dejavu12x12_gs_tc.png",
    32,           # columns in the sheet
    8,            # rows in the sheet
    tcod.tileset.CHARMAP_TCOD,
)
```

`Path(__file__).parent` is the folder where `main.py` lives, so `Path(__file__).parent / "res" / "dejavu12x12_gs_tc.png"` resolves the image path relative to `main.py` itself, not the terminal's current directory. For example, if `main.py` lives in `/home/user/my-game/`, the path evaluates to `/home/user/my-game/res/dejavu12x12_gs_tc.png`, regardless of which folder you launched Python from. This means `python main.py` works the same way from anywhere.

The sheet has 32 columns and 8 rows = 256 tiles, one for each character in the TCOD character map.

### SDL metadata and scaling hints

```python
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
```

`tcod.lib` exposes low-level SDL functions. `SDL_SetAppMetadata` gives SDL the application name, version, and identifier as UTF-8 bytes. This lets the operating system and window manager identify the application more consistently.

`SDL_SetHint` with `SDL_RENDER_SCALE_QUALITY` set to `0` asks SDL to use nearest pixel sampling. That keeps the bitmap font sharp when the window is resized instead of smoothing the tiles.

### The context (window)

```python
with tcod.context.new(
    columns          = screen_width,
    rows             = screen_height,
    tileset          = tileset,
    title            = title,
    vsync            = True,
    sdl_window_flags = tcod.context.SDL_WINDOW_ALLOW_HIGHDPI | tcod.context.SDL_WINDOW_RESIZABLE,
) as context:
```

`tcod.context.new` returns a context manager. The window lives only inside the `with` block; when the block exits, the window closes. `title` uses the variable we defined earlier, so the SDL metadata and the window title stay in sync. `vsync=True` synchronizes rendering to the monitor's refresh rate to avoid screen tearing.

`sdl_window_flags` lets us pass SDL window options through tcod. `SDL_WINDOW_ALLOW_HIGHDPI` makes the window behave better on high-DPI displays, and `SDL_WINDOW_RESIZABLE` lets the player resize the window. The `|` operator combines both flags into one value.

### The console

```python
console = tcod.console.Console(screen_width, screen_height, order="F")
```

`order="F"` changes the array layout so we can index the console as `console[x, y]` instead of the row-first order many grid libraries use. This is more natural for a 2D game.

### The event loop

```python
for event in tcod.event.wait():
    match event:
        case tcod.event.Quit():
            raise SystemExit()
```

`tcod.event.wait()` blocks until at least one event arrives, then returns all pending events. The `match` statement (Python 3.10+) is the cleanest way to handle tcod's typed events.

!!! question "About `match` / `case`"
    `match` is Python's pattern matching, introduced in 3.10. Each `case` matches against the *type* (and optionally the contents) of the value. Here it lets us branch on event types like `tcod.event.Quit` and `tcod.event.KeyDown` without writing a chain of `isinstance` calls. If you have not used `match` before, treat it for now as a more readable `if/elif` over types.

---

## Moving the @

A static `@` is not very interesting. Let's track the player's position and handle movement.

### Player position

```diff
 def main() -> None:
     screen_width  = 80
     screen_height = 50
+
+    player_x = screen_width // 2
+    player_y = screen_height // 2
+
     tileset = tcod.tileset.load_tilesheet(
```

`//` is integer division: it gives a whole number result, so the player starts exactly in the center.

Update the drawing call to use the new variables:

```diff
-        console.print(x=1, y=1, text="@")
+        console.print(x=player_x, y=player_y, text="@")
```

### Setting up the `game/` package

We are about to add our first module beyond `main.py`. The convention in this tutorial is that **everything except the entry point lives inside a `game/` folder**, which Python will see as a *package*. Create the folder with an empty `__init__.py` inside:

```text
roguelike-tutorial/
  main.py
  res/
    dejavu12x12_gs_tc.png
  game/
    __init__.py     ← empty file, marks `game/` as a Python package
```

!!! question "What is `__init__.py`?"
    The presence of an `__init__.py` file tells Python that the folder it lives in is a *package*: a collection of modules that can be imported with dotted notation, like `from game.actions import MovementAction`. The file itself can be empty; its existence is what matters. (Python 3.3+ technically allows packages without it, but we keep it explicit so the structure is obvious at a glance.)

### Actions and input handlers

We could handle key presses directly in `main.py`, but as the game grows we will have dozens of possible commands. It is cleaner to separate the *intent* (move left) from the *trigger* (left arrow key).

Create `game/actions.py`:

```python
from __future__ import annotations


class Action:
    pass


class EscapeAction(Action):
    pass


class MovementAction(Action):

    def __init__(self, dx: int, dy: int) -> None:
        self.dx = dx
        self.dy = dy
```

`Action` is a base class. It is empty for now, but it gives these commands a shared type that later chapters can build on. `EscapeAction` means "the player wants to quit". `MovementAction` carries a direction as `dx` (delta-x) and `dy` (delta-y).

!!! question "Why separate actions from key presses?"
    Later, enemies will also perform actions. An `Orc` moving toward the player will use the same `MovementAction` as the player pressing the arrow key. By decoupling action *type* from action *trigger*, the same logic handles both.

Create `game/input_handlers.py`:

```python
from __future__ import annotations

import tcod.event

from game.actions import Action, EscapeAction, MovementAction


class EventHandler:

    def dispatch(self, event: tcod.event.Event) -> Action | None:
        match event:
            case tcod.event.Quit():
                return self.event_quit(event)

            case tcod.event.KeyDown():
                return self.event_keydown(event)

            case _:
                return None

    def event_quit(self, _event: tcod.event.Quit) -> Action | None:
        return EscapeAction()

    def event_keydown(self, event: tcod.event.KeyDown) -> Action | None:
        key = event.sym

        match key:
            case tcod.event.KeySym.UP:
                return MovementAction(dx=0, dy=-1)

            case tcod.event.KeySym.DOWN:
                return MovementAction(dx=0, dy=1)

            case tcod.event.KeySym.LEFT:
                return MovementAction(dx=-1, dy=0)

            case tcod.event.KeySym.RIGHT:
                return MovementAction(dx=1, dy=0)

            case tcod.event.KeySym.ESCAPE:
                return EscapeAction()

            case _:
                return None
```

`EventHandler.dispatch` receives a raw tcod event and routes it to the right method. `event_keydown` translates key symbols into `Action` objects, and `event_quit` (the OS-level "close window" signal) returns an `EscapeAction`, the same one as pressing the Escape key. The dispatcher itself never decides what to do; it just translates events into intent. Any unrecognized key returns `None`.

!!! tip "Why `_event`?"
    `_event` is still a normal Python parameter. The leading underscore is a convention that tells readers and linters "this value is required by the function signature, but this function does not use it." If you later need the event data, you can rename it back to `event` and use it normally.

!!! question "Why not `tcod.event.EventDispatch`?"
    In older tcod code you will see `class EventHandler(tcod.event.EventDispatch[Action])`. That base class still exists but is marked as deprecated in recent versions of tcod. Writing the dispatch by hand keeps us in control of the routing and makes it straightforward to add subclasses of `EventHandler` later (one per game state: main menu, inventory, targeting, etc.).

!!! tip "Vim editor keys and numpad"
    To support `hjkl` (Vim editor movement) or the numpad, add more `case` branches in `event_keydown`. For example: `case tcod.event.KeySym.H: return MovementAction(dx=-1, dy=0)`.

### Wiring it together

Update `main.py` to use the new modules. We also extract the `while True` block into its own function, `game_loop`, so `main` is left as a focused setup orchestrator:

```python
from __future__ import annotations

from pathlib import Path

import tcod

from game.actions import EscapeAction, MovementAction
from game.input_handlers import EventHandler


def game_loop(
    context: tcod.context.Context,
    console: tcod.console.Console,
    event_handler: EventHandler,
    player_x: int,
    player_y: int,
) -> None:
    while True:
        console.print(x=player_x, y=player_y, text="@")
        context.present(console)

        for event in tcod.event.wait():
            action = event_handler.dispatch(event)

            if action is None:
                continue

            match action:
                case MovementAction(dx=dx, dy=dy):
                    player_x += dx
                    player_y += dy

                case EscapeAction():
                    raise SystemExit()


def main() -> None:
    screen_width  = 80
    screen_height = 50

    player_x = screen_width // 2
    player_y = screen_height // 2

    tileset = tcod.tileset.load_tilesheet(
        Path(__file__).parent / "res" / "dejavu12x12_gs_tc.png",
        32,
        8,
        tcod.tileset.CHARMAP_TCOD,
    )

    event_handler = EventHandler()

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
        console = tcod.console.Console(screen_width, screen_height, order="F")
        game_loop(context, console, event_handler, player_x, player_y)


if __name__ == "__main__":
    main()
```

Splitting `main` and `game_loop` is the standard pattern: setup goes in one place, the per-frame work goes in another. You will see this everywhere in game code.

!!! abstract "A quick preview of Part 2"
    For now, the player's position lives in local variables inside the loop, which keeps Part 1 small and easy to follow. In Part 2, we will group this kind of game state into an `Engine` object so the loop can manage it more cleanly.

Run the game. The `@` moves! But it leaves a trail behind it.

### Clearing the console

The trail appears because we never erase the old position. Add `console.clear()` at the start of each frame in `game_loop`, before drawing:

```diff
     while True:
+        console.clear()
         console.print(x=player_x, y=player_y, text="@")
         context.present(console)

         for event in tcod.event.wait():
```

The order is now `clear → draw → present → wait`, the standard game-loop pattern: every iteration starts with a blank buffer, draws the current state from scratch, displays it, and then blocks until the next event.

---

## Testing your work

Run `python main.py`:

- [ ] A window opens with a white `@` in the center
- [ ] Arrow keys move the `@` in all four directions
- [ ] The `@` does not leave a trail
- [ ] Pressing `Escape` or clicking the X closes the game
- [ ] The `@` can move off-screen (we will fix this in Part 2)

---

## Summary

We built a game loop that renders, then waits for input. tcod's three key objects (**tileset**, **console**, and **context**) work together: the tileset defines how characters look, the console is the drawing buffer, and the context is the window that displays it.

We separated **actions** (what to do) from **input handlers** (what key triggers it). This separation will pay off as soon as enemies need to take turns.

We also adopted the project layout we will use for the rest of the tutorial: `main.py` at the root as the entry point, the `game/` package for our modules, and `res/` for static assets like the font sheet. The per-frame work lives in `game_loop`, separate from the setup in `main`.

**Current architecture**:

- `main.py`: creates the window, loads the tileset, and starts the loop
- `game_loop`: clears, draws, presents, waits for input, and updates player coordinates
- `game/actions.py`: defines action objects such as movement and escape
- `game/input_handlers.py`: translates tcod events into actions

**Class Diagram**:

![classes](images/part1_classes.png)

**File structure**:

```text
main.py                 ← modified
game/
├── __init__.py         ← new
├── actions.py          ← new
└── input_handlers.py   ← new
```

---

## Exercises

1. **Add diagonal movement**:

    The numpad keys `1`, `3`, `7`, `9` (and vim editor keys `y`, `u`, `b`, `n`) conventionally move diagonally in roguelikes. Add cases for them in `event_keydown`. A diagonal move has both `dx` and `dy` set to non-zero (e.g. `MovementAction(dx=1, dy=-1)` for up-right).

2. **Keep the player on screen**:

    Right now the player can move beyond the console boundaries. Before applying a `MovementAction`, compute the destination position and only update `player_x` and `player_y` if the destination is inside `0 <= x < screen_width` and `0 <= y < screen_height`. You will need `screen_width` and `screen_height` inside `game_loop`, so pass them in from `main` along with the starting coordinates. Try holding a movement key at each edge of the window.

3. **Add a wait action**:

    In many roguelikes, pressing `.` or `5` (numpad) passes a turn without moving. Create a `WaitAction(Action)` class and handle it in both `game/input_handlers.py` and `main.py`.
