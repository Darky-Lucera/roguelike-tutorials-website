# Roguelike Tutorial: Python & tcod

A step-by-step guide to building a classic roguelike game in Python, using the [tcod](https://python-tcod.readthedocs.io/) library.

This tutorial is designed around a different goal than most: instead of just showing you *what* to type, it explains *why* each decision was made and *what* each concept means. Every chapter opens with the theory before the code.

!!! info "Coming from another tutorial?"
    - **vs. RogueBasin's classic Python+libtcod tutorial**: we use modern `tcod` (the Pythonic successor to `libtcod`), Python 3.12+, and a more layered architecture (events → actions → engine).
    - **vs. Yet Another Roguelike Tutorial**: the structure is similar, but each chapter leads with the design rationale before the code, with more attention to *why* a pattern exists (components, separation of concerns, event flow) and not just *what* to type.

---

## What you will build

A complete, playable roguelike with:

- Procedurally generated dungeons
- Turn-based combat
- Items and inventory
- Spells and targeting
- Multiple dungeon levels with XP
- Equipment
- Save and load

By the end you will have a working game and, more importantly, an understanding of the architecture behind it.

---

## Prerequisites

- **Python**: You know functions, classes, and basic data structures. You don't need to be an expert.
- **No game development experience required.** This tutorial explains every new concept.
- **Tools**: We use [uv](https://docs.astral.sh/uv/) for project setup, which requires no prior knowledge.

---

## Tools and versions

| Tool | Version |
| --- | --- |
| Python | 3.12+ |
| tcod | 21.2.0+ |
| numpy | 2.4+ |
| Package manager | uv |

---

## How code is presented

**New files** are shown in full so you can read them top to bottom.

**Changes to existing files** use diff blocks to show exactly what changed:

```diff
-    console.print(x=1, y=1, text="@")
+    console.print(x=player.x, y=player.y, text=player.char)
```

Lines starting with `-` are removed. Lines starting with `+` are added.

---

## Table of contents

- [Part 0: Introduction and Setup](part-0.md):

  Install the tools, create the project, and verify everything runs.

- [Part 1: Drawing the @ and Moving It Around](part-1.md):

  Open a window, draw the player character, and move it with the keyboard.

- [Part 2: Entities, the Map, and the Engine](part-2.md):

  Introduce the Entity class, a numpy tile system, and the Engine as a central coordinator.

- [Part 3: Generating a Dungeon](part-3.md):

  Procedurally generate rooms and corridors; place the player in the first room.

- [Part 4: Field of View](part-4.md):

  Show only what the player can see; remember explored tiles.

- [Part 5: Enemies and the Turn System](part-5.md):

  Spawn enemies, give them AI, and build the alternating turn loop.

- [Part 6: Combat](part-6.md):

  Add HP, attack stats, A* pathfinding, and player death.

- [Part 7: The User Interface](part-7.md):

  Build the HUD: health bar, message log, and mouse-hover entity names.

- **Part 8: Items and Inventory**:

  Add items to the dungeon and give the player an inventory.

  Available in two formats:

  - [Part 8](part-8-intro.md): Full chapter.
  - [Part 8a: The Item System](part-8a.md) and [Part 8b: Inventory](part-8b.md): Split into two shorter chapters.

  Which format works better for you? Let us know.

- [Part 9: Spells and Targeting](part-9.md):

  Implement a targeting cursor and three scroll types with spell effects.

- [Part 10: Save and Load](part-10.md):

  Serialize game state with pickle; add a main menu with background art.

- [Part 11: Dungeon Levels and Experience](part-11.md):

  Add dungeon floors, stairs, XP, and character level-ups.

- [Part 12: Procedural Difficulty](part-12.md):

  Scale encounter difficulty with floor-keyed weighted spawn tables.

- [Part 13: Equipment](part-13.md):

  Add weapons and armor with attack and defense bonuses.

---

## Appendices

- [Appendix 1: Damage Formulas](append-1.md)
- [Appendix 2: Combat Effects](append-2.md)
- [Appendix 3: Consumable Scaling](append-3.md)
- [Appendix 4: Advanced Dungeon Generation Ideas](append-4.md)

---

## Analytics and privacy

This site uses Google Analytics to collect anonymous usage statistics. No personal data is collected or stored.

The only thing tracked is which pages are visited and in what order. This helps answer questions like: do readers finish the tutorial? Which parts get the most traffic? Where do people stop?

There are no ads, no user accounts, no cookies used for tracking individuals, and the data is never shared or sold. If you prefer not to be tracked, a browser extension like [uBlock Origin](https://ublockteam.github.io/uBlockOrigin/) will block Google Analytics automatically.
