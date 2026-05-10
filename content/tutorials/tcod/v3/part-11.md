# Part 11: Dungeon Levels and Experience

## Learning goals

- Track experience points and character level on a `Level` component
- Show a level-up modal when the player gains enough XP
- Add descending stairs that generate a new dungeon floor
- Introduce `GameWorld` to separate "how to generate maps" from the current map
- Award XP to the killer when an enemy dies

---

## Experience and character progression

Classic roguelikes use an XP curve where each level requires significantly more XP than the last. We use a simple formula:

| Level | XP required to reach |
| --- | --- |
| 2 | 200 |
| 3 | 350 |
| 4 | 500 |
| 5 | 650 |
| ... | +150 each level |

The formula is `200 + (current_level - 1) * 150`. This is a linear-slope curve, not exponential, but it grows steadily and is easy to reason about.

---

## components/level.py

Create `game/components/level.py`:

```python
from __future__ import annotations

from typing import TYPE_CHECKING

from game.components.base_component import BaseComponent

if TYPE_CHECKING:
    from game.engine import Engine


class Level(BaseComponent):
    def __init__(
        self,
        current_level: int = 1,
        current_xp: int = 0,
        level_up_base: int = 200,
        level_up_factor: int = 150,
        xp_given: int = 0,
    ) -> None:
        self.current_level = current_level
        self.current_xp = current_xp
        self.level_up_base = level_up_base
        self.level_up_factor = level_up_factor
        self.xp_given = xp_given

    @property
    def experience_to_next_level(self) -> int:
        return self.level_up_base + (self.current_level - 1) * self.level_up_factor

    @property
    def requires_level_up(self) -> bool:
        return self.current_xp >= self.experience_to_next_level

    def add_xp(self, xp: int) -> bool:
        if xp == 0 or self.level_up_base == 0:
            return False
        self.current_xp += xp
        return self.requires_level_up

    def increase_level(self) -> None:
        self.current_xp -= self.experience_to_next_level
        self.current_level += 1

    def increase_max_hp(self, amount: int = 20) -> None:
        self.entity.fighter.max_hp += amount
        self.entity.fighter.hp += amount
        self.increase_level()

    def increase_attack(self, amount: int = 1) -> None:
        self.entity.fighter.base_attack += amount
        self.increase_level()

    def increase_defense(self, amount: int = 1) -> None:
        self.entity.fighter.base_defense += amount
        self.increase_level()
```

`xp_given` is how much XP an enemy awards on death. Enemies set this; the player leaves it at 0 (the player does not give XP to anything).

!!! info "base_attack and base_defense"
    We renamed the stored values to `base_attack` and `base_defense` to make room for equipment bonuses in Part 13. Keep `attack` and `defense` as properties for now, so existing combat code can keep using `fighter.attack` and `fighter.defense`.

Update `Fighter.__init__` and add the two properties:

```python
self.base_defense = defense
self.base_attack = attack

@property
def defense(self) -> int:
    return self.base_defense

@property
def attack(self) -> int:
    return self.base_attack
```

---

## Award XP on kill

Update `Fighter.die()` in `game/components/fighter.py`:

```python
    def die(self, engine: Engine) -> None:
        from game.constants import colors, sprites
        from game.message_log import MessageLog
        if self.entity.ai is None:
            death_message = "You died!"
            death_message_color = colors.PLAYER_DEATH
        else:
            death_message = f"The {self.entity.name} is dead!"
            death_message_color = colors.ENEMY_DEATH
            if engine.player.level.add_xp(self.entity.level.xp_given):
                MessageLog.add_message(
                    "You feel your experience grow!", colors.LEVEL_UP
                )

        self.entity.char = sprites.CORPSE
        self.entity.color = colors.CORPSE
        self.entity.blocks_movement = False
        self.entity.ai = None
        self.entity.name = f"remains of {self.entity.name}"
        self.entity.render_order = RenderOrder.CORPSE

        MessageLog.add_message(death_message, death_message_color)
```

Add to `game/constants/colors.py`:

```python
LEVEL_UP = (0xFF, 0xFF, 0x00)
```

Add to `game/constants/sprites.py`:

```python
DOWN_STAIRS = ">"
```

Add to `game/constants/colors.py`:

```python
DOWN_STAIRS = (255, 255, 100)
```

---

## Update entity_factories.py with Level component

```python
from __future__ import annotations

from game.components.level import Level
from game.constants import colors, sprites

player = Actor(
    char=sprites.PLAYER,
    color=colors.PLAYER,
    name="Player",
    ai=None,
    fighter=Fighter(hp=30, defense=2, attack=5),
    inventory=Inventory(capacity=26),
    level=Level(level_up_base=200),
)

orc = Actor(
    char=sprites.ORC,
    color=colors.ORC,
    name="Orc",
    ai=HostileEnemy(),
    fighter=Fighter(hp=10, defense=0, attack=3),
    inventory=Inventory(capacity=0),
    level=Level(xp_given=35),
)

troll = Actor(
    char=sprites.TROLL,
    color=colors.TROLL,
    name="Troll",
    ai=HostileEnemy(),
    fighter=Fighter(hp=16, defense=1, attack=4),
    inventory=Inventory(capacity=0),
    level=Level(xp_given=100),
)
```

Update `Actor.__init__` in `game/entity.py` to accept and wire up the `level` component:

```python
class Actor(Entity):
    def __init__(
        self,
        *,
        ...
        level: Level,
    ) -> None:
        ...
        self.level = level
        self.level.entity = self
```

---

## GameWorld: separating map params from the live map

`Engine` currently holds `game_map` directly. When the player descends stairs, we need to generate a new map with the same parameters but a higher floor number. We extract those parameters into a `GameWorld` class.

Create `game/game_world.py`:

```python
from __future__ import annotations

from typing import TYPE_CHECKING

if TYPE_CHECKING:
    from game.engine import Engine


class GameWorld:
    """Holds the settings for map generation and tracks the current floor."""

    def __init__(
        self,
        *,
        engine: Engine,
        map_width: int,
        map_height: int,
        max_rooms: int,
        room_min_size: int,
        room_max_size: int,
        max_monsters_per_room: int,
        max_items_per_room: int,
        current_floor: int = 0,
    ) -> None:
        self.engine = engine
        self.map_width = map_width
        self.map_height = map_height
        self.max_rooms = max_rooms
        self.room_min_size = room_min_size
        self.room_max_size = room_max_size
        self.max_monsters_per_room = max_monsters_per_room
        self.max_items_per_room = max_items_per_room
        self.current_floor = current_floor

    def generate_floor(self) -> None:
        from game.map.map_generator import generate_dungeon

        self.current_floor += 1
        self.engine.game_map = generate_dungeon(
            max_rooms=self.max_rooms,
            room_min_size=self.room_min_size,
            room_max_size=self.room_max_size,
            map_width=self.map_width,
            map_height=self.map_height,
            max_monsters_per_room=self.max_monsters_per_room,
            max_items_per_room=self.max_items_per_room,
            player=self.engine.player,
        )
```

Update `Engine` to hold `game_world` instead of being given a `game_map` directly:

```python
class Engine:
    game_map: GameMap
    game_world: GameWorld
```

Update `game/setup_game.py`, replace direct `generate_dungeon` call with `GameWorld`:

```python
import copy

from game.game_world import GameWorld
from game.message_log import MessageLog

def new_game() -> Engine:
    player = copy.deepcopy(entity_factories.player)
    engine = Engine(player=player)

    engine.game_world = GameWorld(
        engine=engine,
        max_rooms=MAX_ROOMS,
        room_min_size=ROOM_MIN_SIZE,
        room_max_size=ROOM_MAX_SIZE,
        map_width=MAP_WIDTH,
        map_height=MAP_HEIGHT,
        max_monsters_per_room=MAX_MONSTERS_PER_ROOM,
        max_items_per_room=MAX_ITEMS_PER_ROOM,
    )
    engine.game_world.generate_floor()
    engine.update_fov()

    MessageLog.add_message(
        "Hello and welcome, adventurer, to yet another dungeon!",
        colors.WELCOME_TEXT,
    )
    return engine
```

---

## Stairs in the map generator

Place a staircase at the center of the last room generated. Update `generate_dungeon()`:

```python
from game.entity import Actor, Item
from game import entity_factories

def generate_dungeon(...) -> GameMap:
    ...
    for r, room in enumerate(rooms):
        place_entities(room, dungeon, max_monsters_per_room, max_items_per_room)

    # Place the player in the first room, stairs in the last.
    dungeon.downstairs_location = rooms[-1].center
    entity_factories.down_stairs.spawn(dungeon, *rooms[-1].center)

    return dungeon
```

Add stairs to `game/entity_factories.py`:

```python
down_stairs = Entity(
    char=sprites.DOWN_STAIRS,
    color=colors.DOWN_STAIRS,
    name="Stairs",
    blocks_movement=False,
    render_order=RenderOrder.ITEM,
)
```

Add `downstairs_location` to `GameMap`:

```python
class GameMap:
    downstairs_location: tuple[int, int] = (0, 0)
```

---

## TakeStairsAction

Add to `game/actions.py`:

```python
class TakeStairsAction(Action):
    def perform(self, engine: Engine, entity: Entity) -> None:
        if (entity.x, entity.y) == engine.game_map.downstairs_location:
            engine.game_world.generate_floor()
            MessageLog.add_message(
                "You descend the staircase.", colors.DESCEND
            )
        else:
            raise Impossible("There are no stairs here.")
```

Add to `game/constants/colors.py`:

```python
DESCEND = (0x9F, 0x3F, 0xFF)
```

Add the `>` keybinding in `MainGameEventHandler`:

```python
        if key == tcod.event.KeySym.PERIOD and event.mod & tcod.event.Modifier.LSHIFT:
            return TakeStairsAction()
```

(`>` is `Shift+.` on standard keyboards.)

---

## LevelUpEventHandler

Add to `game/input_handlers.py`:

```python
class LevelUpEventHandler(EventHandler):
    TITLE = "Level Up"

    def on_render(self, console: tcod.Console) -> None:
        super().on_render(console)

        x, y = 40, 0
        width = 35

        console.draw_frame(
            x=x, y=y, width=width, height=8,
            title=self.TITLE, clear=True,
            fg=colors.WHITE, bg=colors.BLACK,
        )

        console.print(x=x + 1, y=y + 1, text="Congratulations! You level up!")
        console.print(x=x + 1, y=y + 2, text="Select an attribute to increase.")

        fighter = self.engine.player.fighter
        console.print(
            x=x + 1, y=y + 4,
            text=f"a) Constitution (+20 HP, from {fighter.max_hp})",
        )
        console.print(
            x=x + 1, y=y + 5,
            text=f"b) Strength (+1 attack, from {fighter.base_attack})",
        )
        console.print(
            x=x + 1, y=y + 6,
            text=f"c) Agility (+1 defense, from {fighter.base_defense})",
        )

    def event_keydown(self, event: tcod.event.KeyDown) -> BaseEventHandler | None:
        player = self.engine.player
        index = event.sym - tcod.event.KeySym.a

        if index == 0:
            player.level.increase_max_hp()
        elif index == 1:
            player.level.increase_attack()
        elif index == 2:
            player.level.increase_defense()
        else:
            MessageLog.add_message("Invalid entry.", colors.INVALID)
            return None

        return MainGameEventHandler(self.engine)

    def handle_events(self, event: tcod.event.Event) -> BaseEventHandler:
        result = super().handle_events(event)
        if result is not self:
            return result
        # Stay in level-up screen until a valid choice is made.
        return self
```

Trigger the modal from `EventHandler.handle_events()` after enemy turns and death handling:

```python
            if self.engine.player.is_alive:
                self.engine.handle_enemy_turns()

            if not self.engine.player.is_alive:
                new_handler = GameOverEventHandler(self.engine)
                new_handler.on_enter()
                return new_handler

            self.engine.update_fov()

            if self.engine.player.level.requires_level_up:
                return LevelUpEventHandler(self.engine)
```

---

## Show floor and level in the UI

Update `game/hud.py`, add a function to print dungeon floor and player level:

```python
def render_dungeon_level(
    console: Console,
    dungeon_level: int,
    location: tuple[int, int],
) -> None:
    x, y = location
    console.print(x=x, y=y, text=f"Dungeon level: {dungeon_level}")
```

Call it from `Engine.render()`:

```python
        hud.render_dungeon_level(
            console=console,
            dungeon_level=self.game_world.current_floor,
            location=(0, 47),
        )
```

---

## Testing your work

Run `python main.py`:

- [ ] Killing an orc prints XP gained; killing a troll gives more XP
- [ ] After enough kills, a level-up screen appears with three stat choices
- [ ] Selecting an option increases the stat and closes the modal
- [ ] Finding `>` stairs and pressing `Shift+.` generates a new floor
- [ ] The player's HP, inventory, and level persist across floor transitions
- [ ] Enemies and items are freshly generated on each new floor
- [ ] The floor counter in the UI increments when descending

---

## Summary

Character progression and dungeon depth are now linked. Key additions:

- **`Level` component**: XP tracking, level-up threshold, stat-increase methods
- **`GameWorld`**: owns generation parameters, creates new floors on demand
- **`TakeStairsAction`**: triggers floor generation when player is on stairs
- **`LevelUpEventHandler`**: modal stat-selection screen
- **XP on kill**: `Fighter.die()` awards XP to the player

**Current architecture**:

- `GameWorld`: owns dungeon generation parameters and current floor number
- `Engine`: owns the current `GameMap` plus a `GameWorld` for new floors
- `Level`: component that owns XP, level thresholds, and stat increases
- `TakeStairsAction`: asks `GameWorld` to generate the next floor
- `LevelUpEventHandler`: modal state entered when the player must choose a stat

**Files created**: `game/components/level.py`, `game/game_world.py`

**Files modified**: `game/entity.py`, `game/entity_factories.py`, `game/map/map_generator.py`, `game/actions.py`, `game/input_handlers.py`, `game/engine.py`, `game/setup_game.py`, `game/hud.py`, `game/components/fighter.py`, `game/constants/sprites.py`, `game/constants/colors.py`

---

## Exercises

1. **XP display**:

    Show current XP and XP-to-next-level in the UI panel: `"XP: 120 / 200"`. Add a second small bar next to the HP bar.

2. **Persistent floors**:

    Store each generated `GameMap` in a list inside `GameWorld`. When the player descends, generate a new floor. When they ascend (add `<` stairs), restore the previous map. This requires saving entity positions and the player being removed from the old map before being placed on the new one.

3. **Level cap**:

    Cap the player at level 10. Above level 10, `add_xp` still accumulates but `requires_level_up` always returns `False`. Print `"You are at maximum level."` instead of the modal.

**Next**: Part 12: Procedural Difficulty
