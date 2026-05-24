# Part 13: Equipment

## What You Will Build

By the end of this part, the player can find, equip, and swap weapons and armor, and their combat stats will reflect what they are carrying. With this in place, the main tutorial game is complete.

## Learning goals

- Add `Equippable` and `Equipment` components
- Define equipment slots (weapon, armor) as an enum
- Layer equipment bonuses on top of Fighter base stats
- Show equipped items in the inventory UI
- Spawn starting gear and find better weapons on deeper floors

---

## How equipment works

Equipment is a two-sided relationship:

- An **`Equippable`** component sits on an `Item` and says "I give +2 attack when equipped in the weapon slot"
- An **`Equipment`** component sits on an `Actor` and holds references to whatever is currently equipped

When combat damage is calculated, `Fighter.attack` and `Fighter.defense` include bonuses from the `Equipment` component.

```txt
Actor (player)
  ├── Fighter  (base_attack=5, base_defense=2)
  └── Equipment
        ├── weapon  → Item("Sword")
        │               └── Equippable(attack_bonus=4)
        └── armor   → Item("Chain Mail")
                        └── Equippable(defense_bonus=3)

Effective attack = 5 + 4 = 9
Effective defense = 2 + 3 = 5
```

---

## EquipmentType enum

Create `game/entities/equipment_type.py`:

```python
from __future__ import annotations

from enum import auto, Enum


class EquipmentType(Enum):
    WEAPON = auto()
    ARMOR  = auto()
```

---

## game/entities/components/equippable.py

Create `game/entities/components/equippable.py`:

```python
from __future__ import annotations

from typing import TYPE_CHECKING

from game.entities.components.base_component import ItemComponent
from game.entities.equipment_type import EquipmentType

if TYPE_CHECKING:
    from game.entities.entity import Item


class Equippable(ItemComponent):
    def __init__(
        self,
        equipment_type: EquipmentType,
        attack_bonus: int = 0,
        defense_bonus: int = 0,
    ) -> None:
        self.equipment_type = equipment_type
        self.attack_bonus = attack_bonus
        self.defense_bonus = defense_bonus


class Dagger(Equippable):
    def __init__(self) -> None:
        super().__init__(equipment_type=EquipmentType.WEAPON, attack_bonus=2)


class Sword(Equippable):
    def __init__(self) -> None:
        super().__init__(equipment_type=EquipmentType.WEAPON, attack_bonus=4)


class LeatherArmor(Equippable):
    def __init__(self) -> None:
        super().__init__(equipment_type=EquipmentType.ARMOR, defense_bonus=1)


class ChainMail(Equippable):
    def __init__(self) -> None:
        super().__init__(equipment_type=EquipmentType.ARMOR, defense_bonus=3)
```

Subclassing for each item type is optional, we could pass parameters directly. Subclasses make `game/entities/factories.py` easier to read (`Dagger()` vs `Equippable(EquipmentType.WEAPON, attack_bonus=2)`).

---

## game/entities/components/equipment.py

Create `game/entities/components/equipment.py`:

```python
from __future__ import annotations

from typing import TYPE_CHECKING

from game.entities.components.base_component import ActorComponent
from game.entities.equipment_type import EquipmentType
from game.message_log import MessageLog

if TYPE_CHECKING:
    from game.entities.entity import Actor, Item


class Equipment(ActorComponent):
    def __init__(
        self,
        weapon: Item | None = None,
        armor: Item | None = None,
    ) -> None:
        self.weapon = weapon
        self.armor = armor

    @property
    def defense_bonus(self) -> int:
        bonus = 0
        if self.weapon and self.weapon.equippable:
            bonus += self.weapon.equippable.defense_bonus
        if self.armor and self.armor.equippable:
            bonus += self.armor.equippable.defense_bonus
        return bonus

    @property
    def attack_bonus(self) -> int:
        bonus = 0
        if self.weapon and self.weapon.equippable:
            bonus += self.weapon.equippable.attack_bonus
        if self.armor and self.armor.equippable:
            bonus += self.armor.equippable.attack_bonus
        return bonus

    def item_is_equipped(self, item: Item) -> bool:
        return self.weapon is item or self.armor is item

    def toggle_equip(self, equippable_item: Item) -> None:
        if equippable_item.equippable.equipment_type == EquipmentType.WEAPON:
            slot = "weapon"
        else:
            slot = "armor"

        if getattr(self, slot) is equippable_item:
            self.unequip_from_slot(slot)
        else:
            self.equip_to_slot(slot, equippable_item)

    def equip_to_slot(self, slot: str, item: Item) -> None:
        current_item = getattr(self, slot)
        if current_item is not None:
            self.unequip_from_slot(slot)
        setattr(self, slot, item)
        MessageLog.add_message(f"You equip the {item.name}.")

    def unequip_from_slot(self, slot: str) -> None:
        current_item = getattr(self, slot)
        MessageLog.add_message(f"You remove the {current_item.name}.")
        setattr(self, slot, None)
```

Because `MessageLog` is static, the component can log directly without needing a callback injected from outside.

---

## Update Fighter to include equipment bonuses

`Fighter.attack` and `Fighter.defense` should now add equipment bonuses. Replace the raw attributes with properties:

Update `game/entities/components/fighter.py`:

```python
class Fighter(ActorComponent):
    def __init__(self, hp: int, defense: float, attack: float) -> None:
        self.max_hp             = hp
        self._hp: float         = float(hp)
        self.base_defense: float = float(defense)
        self.base_attack: float  = float(attack)

    @property
    def defense(self) -> float:
        bonus = 0
        if hasattr(self.entity, "equipment") and self.entity.equipment:
            bonus = self.entity.equipment.defense_bonus
        return self.base_defense + bonus

    @property
    def attack(self) -> float:
        bonus = 0
        if hasattr(self.entity, "equipment") and self.entity.equipment:
            bonus = self.entity.equipment.attack_bonus
        return self.base_attack + bonus
```

All existing code that reads `fighter.attack` and `fighter.defense` automatically picks up equipment bonuses, no other changes needed at the call site.

---

## Update game/entities/entity.py: Item gets equippable, Actor gets equipment

Update the `Item` class:

```python
class Item(Entity):
    def __init__(
        self,
        *,
        ...
        consumable: Consumable | None = None,
        equippable: Equippable | None = None,
    ) -> None:
        ...
        self.consumable = consumable
        if self.consumable:
            self.consumable.entity = self

        self.equippable = equippable
        if self.equippable:
            self.equippable.entity = self
```

Update `Actor`:

```python
from game.entities.components.equipment import Equipment

class Actor(Entity):
    def __init__(
        self,
        *,
        ...
        equipment: Equipment,
    ) -> None:
        ...
        self.equipment = equipment
        self.equipment.entity = self
```

---

## EquipAction in actions.py

```python
class EquipAction(Action):
    def __init__(self, item: Item) -> None:
        super().__init__()
        self.item = item

    def perform(self, engine: Engine, entity: Entity) -> None:
        entity.equipment.toggle_equip(self.item)
```

Also update `DropItem` so dropping equipped gear unequips it first:

```python
class DropItem(ItemAction):
    def perform(self, engine: Engine, entity: Entity) -> None:
        if entity.equipment.item_is_equipped(self.item):
            entity.equipment.toggle_equip(self.item)

        entity.inventory.drop_item(self.item, engine.game_map)
        MessageLog.add_message(f"You dropped the {self.item.name}.")
```

---

## Update InventoryActivateHandler

When the selected item is equippable, return an `EquipAction` instead of an `ItemAction`:

```python
class InventoryActivateHandler(InventoryEventHandler):
    TITLE = "Select an item to use"

    def on_item_selected(self, item) -> Action | None:
        if item.equippable:
            from game.actions import EquipAction
            return EquipAction(item=item)
        return item.consumable.get_action(self.engine.player, self.engine)
```

---

## Show equipped status in the inventory overlay

Update `InventoryEventHandler.on_render` to mark equipped items with `(E)`:

```python
        if number_of_items_in_inventory > 0:
            for i, item in enumerate(self.engine.player.inventory.items):
                item_key = chr(ord("a") + i)
                is_equipped = self.engine.player.equipment.item_is_equipped(item)
                item_string = f"({item_key}) {item.name}"
                if is_equipped:
                    item_string = f"{item_string} (E)"
                console.print(x + 1, y + i + 1, item_string)
```

---

## Adding weapon and armor constants

Each piece of equipment gets its own named sprite and color, even though daggers and swords share `/` and the two armors share `[`. Per the convention from Part 5, that means `colors.SWORD` is more readable at a call site than reusing `colors.DAGGER` or hardcoding `(0, 191, 255)`.

Extend `game/constants/sprites.py` in a new equipment sprites section:

```diff
 # Equipment sprites
+
+DAGGER = "/"
+SWORD = "/"
+LEATHER_ARMOR = "["
+CHAIN_MAIL = "["
```

Extend `game/constants/colors.py` in a new equipment colors section:

```diff
 # Equipment colors
+
+DAGGER = (0, 191, 255)
+SWORD = (0, 191, 255)
+LEATHER_ARMOR = (139, 69, 19)
+CHAIN_MAIL = (139, 69, 19)
```

---

## game/entities/factories.py: add weapons and armor

```python
from __future__ import annotations

from game.entities.components.equippable import ChainMail, Dagger, LeatherArmor, Sword
from game.entities.components.equipment import Equipment
from game.constants import colors, sprites

player = Actor(
    char=sprites.PLAYER,
    color=colors.PLAYER,
    name="Player",
    ai=None,
    fighter=Fighter(hp=30, defense=2, attack=5),
    inventory=Inventory(capacity=26),
    level=Level(level_up_base=200),
    equipment=Equipment(),
)

orc = Actor(
    ...
    equipment=Equipment(),
)

troll = Actor(
    ...
    equipment=Equipment(),
)

dagger = Item(
    char=sprites.DAGGER,
    color=colors.DAGGER,
    name="Dagger",
    equippable=Dagger(),
)

sword = Item(
    char=sprites.SWORD,
    color=colors.SWORD,
    name="Sword",
    equippable=Sword(),
)

leather_armor = Item(
    char=sprites.LEATHER_ARMOR,
    color=colors.LEATHER_ARMOR,
    name="Leather Armor",
    equippable=LeatherArmor(),
)

chain_mail = Item(
    char=sprites.CHAIN_MAIL,
    color=colors.CHAIN_MAIL,
    name="Chain Mail",
    equippable=ChainMail(),
)
```

---

## Give the player a starting dagger

In `setup_game.new_game()`, place a dagger in the player's inventory and equip it immediately:

```python
import copy


def new_game() -> Engine:
    player = copy.deepcopy(factories.player)
    engine = Engine(player=player)

    # Starting equipment
    dagger = copy.deepcopy(factories.dagger)
    dagger.owner = player.inventory
    player.inventory.items.append(dagger)
    player.equipment.toggle_equip(dagger)

    leather_armor = copy.deepcopy(factories.leather_armor)
    leather_armor.owner = player.inventory
    player.inventory.items.append(leather_armor)
    player.equipment.toggle_equip(leather_armor)
    ...
```

Starting equipment is equipped at game start; the messages go to the log as normal.

---

## Add equipment to the item spawn tables

Update `item_chances` in `game/map/map_generator.py`:

```python
item_chances: dict[str, list[tuple[int, int]]] = {
    "health_potion":    [(1, 35)],
    "confusion_scroll": [(2, 10)],
    "lightning_scroll": [(4, 25)],
    "fireball_scroll":  [(6, 25)],
    "dagger":           [(1, 5)],
    "sword":            [(4, 5)],
    "leather_armor":    [(1, 5)],
    "chain_mail":       [(6, 15)],
}
```

Add entries to `ITEM_FACTORIES`:

```python
ITEM_FACTORIES = {
    ...
    "dagger":        factories.dagger,
    "sword":         factories.sword,
    "leather_armor": factories.leather_armor,
    "chain_mail":    factories.chain_mail,
}
```

---

## Testing your work

Run `python main.py`:

- [ ] The player starts with a Dagger `(E)` and Leather Armor `(E)` already equipped
- [ ] The inventory screen shows `(E)` next to equipped items
- [ ] Pressing `i → a` on the dagger shows `"You remove the Dagger."` (toggle off)
- [ ] Pressing `i → a` again equips it: `"You equip the Dagger."`
- [ ] Finding a Sword and equipping it automatically removes the Dagger first
- [ ] Checking stats: player with Sword has effective attack = 5 (base) + 4 (sword) = 9
- [ ] Chain Mail appears on floor 6+; equipping it noticeably reduces incoming damage

---

## Summary

Equipment is complete. The game is now feature-complete. Key additions:

- **`Equippable` component**: carries slot type and stat bonuses
- **`Equipment` component**: tracks what is in each slot, applies bonuses to `Fighter`
- **`Fighter.attack`/`.defense`**: properties that include equipment bonuses transparently
- **`EquipAction`**: toggles equip/unequip via `equipment.toggle_equip()`
- **Inventory UI**: shows `(E)` next to equipped items
- **Starting gear**: player begins with dagger and leather armor

**Current architecture**:

- `EquipmentType`: enum for equipment slots
- `Equippable`: item component that defines slot and bonuses
- `Equipment`: actor component that tracks equipped items
- `Fighter`: exposes effective `attack` and `defense`, including equipment bonuses
- `EquipAction`: routes inventory activation into equip/unequip behavior
- `setup_game.py`: creates starting gear and equips it silently

**File structure**:

```txt
main.py
game/
├── __init__.py
├── actions.py                  ← modified
├── engine.py
├── exceptions.py
├── game_world.py
├── hud.py
├── input_handlers.py           ← modified
├── message_log.py
├── setup_game.py               ← modified
├── constants/
│   ├── __init__.py
│   ├── colors.py               ← modified
│   └── sprites.py              ← modified
├── entities/
│   ├── __init__.py
│   ├── entity.py               ← modified
│   ├── factories.py            ← modified
│   ├── render_order.py
│   ├── equipment_type.py       ← new
│   └── components/
│       ├── __init__.py
│       ├── ai.py
│       ├── base_component.py
│       ├── consumable.py
│       ├── equipment.py        ← new
│       ├── equippable.py       ← new
│       ├── fighter.py          ← modified
│       ├── inventory.py
│       └── level.py
└── map/
    ├── __init__.py
    ├── game_map.py
    ├── tile_types.py
    └── map_generator.py        ← modified
```

---

## Exercises

1. **Equipment stats in the UI.** Show effective attack and defense in the panel: `"ATK: 9  DEF: 5"`. Read from `player.fighter.attack` and `player.fighter.defense` (which already include bonuses).

2. **Ring slot.** Add `EquipmentType.RING` and a third slot in `Equipment`. Create a `RingOfProtection(Equippable)` with `defense_bonus=1`. Add it to the item tables from floor 5.

3. **Cursed items.** Add a `cursed: bool` flag to `Equippable`. Cursed items cannot be unequipped, `unequip_from_slot` checks and raises `Impossible("You cannot remove the cursed item!")`. Add a `Remove Curse Scroll` consumable that clears the flag.

---

## What's next

You have built a complete roguelike. Here are directions to take it further:

**Content**
- More monster types per Part 12's weighted tables (troll variants, vampires, dragons)
- More spell scrolls: confusion variants, area heals, walls of fire
- Ranged weapons (bow + arrow inventory item)

**Depth**
- Persistent dungeon floors (store each `GameMap` in a list; ascend/descend freely)
- Named artifacts: unique items with special effects that only spawn once per run
- Status effects: poison, blindness, haste, each as a component with a countdown

**Polish**
- Graphical tiles: replace the tileset with a 16×16 pixel art set; tcod supports it with no code changes beyond the tileset loader
- Sound: `pygame.mixer` can play `.wav` files alongside tcod's rendering
- Fullscreen: `tcod.context.new(..., renderer=tcod.RENDERER_SDL2, flags=tcod.libtcodpy.FULLSCREEN)`

**Publishing**
- The tutorial files are plain Markdown. `mkdocs build` generates a self-contained `site/` folder you can zip and share
- For the official python-tcod docs: convert admonitions (`!!! type "Title"\n    body` → `:::{type}\nTitle\nbody\n:::`) with a single regex pass, then open a PR to the `python-tcod` repository's `docs/` folder

The architecture you built (Entity, components, actions, event handlers as a state machine, pickle serialization) scales to a much larger game. The reference implementation at [python-tcod-tutorial-revised](https://github.com/TStand90/tcod_tutorial_v2) follows the same structure and is a good comparison point.

Good luck with the dungeon.
