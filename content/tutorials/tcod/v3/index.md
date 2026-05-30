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

- [Part 0: Introduction and Setup](part-0.md)
- [Part 1: Drawing the @ and Moving It Around](part-1.md)
- [Part 2: Entities, the Map, and the Engine](part-2.md)
- [Part 3: Generating a Dungeon](part-3.md)
- [Part 4: Field of View](part-4.md)
- [Part 5: Enemies and the Turn System](part-5.md)
- [Part 6: Combat](part-6.md)
- [Part 7: The User Interface](part-7.md)
- [Part 8A: Items and Inventory](part-8.md)
- [Part 8B: Items and Inventory (split)](part-8_split.md)
- [Part 9: Spells and Targeting](part-9.md)
- [Part 10: Save and Load](part-10.md)
- [Part 11: Dungeon Levels and Experience](part-11.md)
- [Part 12: Procedural Difficulty](part-12.md)
- [Part 13: Equipment](part-13.md)

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
