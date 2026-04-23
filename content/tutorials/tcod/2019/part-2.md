---
title: "Part 2 - The generic Entity, the render functions, and the map"
date: 2019-03-30T08:39:20-07:00
draft: false
aliases: /tutorials/tcod/part-2
---

Now that we can move our little '@' symbol around, we need to give it
something to move around *in*. But before that, let's stop for a moment
and think about the player object itself.

Right now, we just represent the player with the '@' symbol, and its x
and y coordinates. Shouldn't we tie those things together in an object,
along with some other data and functions that pertain to it?

Let's create a generic class to represent not just the player, but just
about *everything* in our game world. Enemies, items, and whatever other
foreign entities we can dream of will be part of this class, which we'll
call `Entity`.

Create a new file, and call it `entity.py`. In that file, put the
following class:

{{< highlight py3 >}}
class Entity:
    """
    A generic object to represent players, enemies, items, etc.
    """
    def __init__(self, x, y, char, color):
        self.x = x
        self.y = y
        self.char = char
        self.color = color

    def move(self, dx, dy):
        # Move the entity by a given amount
        self.x += dx
        self.y += dy
{{</ highlight >}}

This is pretty self explanatory. The `Entity` class holds the x and y
coordinates, along with the character (the '@' symbol in the player's
case) and the color (white for the player by default). We also have a
method called `move`, which will allow the entity to be moved around by
a given x and y.

Let's put our fancy new class into action\! Modify the first part of
`engine.py` to look like this:

{{< codetab >}}
{{< diff-tab >}}
{{< highlight diff >}}
import tcod

+from entity import Entity
from input_handlers import handle_keys


def main():
    screen_width = 80
    screen_height = 50

-   player_x = int(screen_width / 2)
-   player_y = int(screen_height / 2)

+   player = Entity(int(screen_width / 2), int(screen_height / 2), '@', (255, 255, 255))
+   npc = Entity(int(screen_width / 2 - 5), int(screen_height / 2), '@', (255, 255, 0))
+   entities = [npc, player]
    ...
{{</ highlight >}}
{{</ diff-tab >}}
{{< original-tab >}}
<pre>import tcod

<span class="new-text">from entity import Entity</span>
from input_handlers import handle_keys


def main():
    screen_width = 80
    screen_height = 50

    <span class="crossed-out-text">player_x = int(screen_width / 2)</span>
    <span class="crossed-out-text">player_y = int(screen_height / 2)</span>

    <span class="new-text">player = Entity(int(screen_width / 2), int(screen_height / 2), '@', (255, 255, 255))
    npc = Entity(int(screen_width / 2 - 5), int(screen_height / 2), '@', (255, 255, 0))
    entities = [npc, player]</span>
    ...</pre>
{{</ original-tab >}}
{{</ codetab >}}

We're importing the `Entity` class into `engine.py`, and using it to
initialize the player and a new NPC. Colors are specified as RGB tuples —
`(255, 255, 255)` for white and `(255, 255, 0)` for yellow. We store
these two in a list that will eventually hold all our entities on the map.

Also modify the part where we handle movement so that the Entity class
handles the actual movement.

{{< codetab >}}
{{< diff-tab >}}
{{< highlight diff >}}
                if move:
                    dx, dy = move
-                   player_x += dx
-                   player_y += dy
+                   player.move(dx, dy)
{{</ highlight >}}
{{</ diff-tab >}}
{{< original-tab >}}
<pre>                if move:
                    dx, dy = move
                    <span class="crossed-out-text">player_x += dx</span>
                    <span class="crossed-out-text">player_y += dy</span>
                    <span class="new-text">player.move(dx, dy)</span></pre>
{{</ original-tab >}}
{{</ codetab >}}

Lastly, update the drawing function to use the new player object's
coordinates:

{{< codetab >}}
{{< diff-tab >}}
{{< highlight diff >}}
        while True:
-           con.print(player_x, player_y, '@', fg=(255, 255, 255))
+           con.print(player.x, player.y, '@', fg=(255, 255, 255))
            con.blit(dest=root_console)
            context.present(root_console)
            con.clear()
        ...
{{</ highlight >}}
{{</ diff-tab >}}
{{< original-tab >}}
<pre>        while True:
            <span class="crossed-out-text">con.print(player_x, player_y, '@', fg=(255, 255, 255))</span>
            <span class="new-text">con.print(player.x, player.y, '@', fg=(255, 255, 255))</span>
            con.blit(dest=root_console)
            context.present(root_console)
            con.clear()
        ...</pre>
{{</ original-tab >}}
{{</ codetab >}}

Now we need to alter the way that the entities are drawn to the screen.
If you run the code right now, only the player gets drawn. Let's write
some functions to draw not only the player, but any entity currently in
our entities list.

Create a new file called `render_functions.py`. This will hold our
functions for drawing to the screen. Put the following code in that file.

{{< highlight py3 >}}
import tcod


def render_all(con, root_console, entities, screen_width, screen_height):
    # Draw all entities in the list
    for entity in entities:
        draw_entity(con, entity)

    con.blit(dest=root_console)


def draw_entity(con, entity):
    con.print(entity.x, entity.y, entity.char, fg=entity.color)
{{</ highlight >}}

Here's a quick breakdown of what these functions do:

  - The `render_all` function is what we'll call from our game loop to
    draw entities and, shortly, the map. For now, it takes the console
    (`con`), the root console, a list of entities, and the screen
    width/height as parameters. It calls `draw_entity` on each, then
    blits the offscreen console to the root.
  - `draw_entity` is what does the actual drawing. It calls `con.print()`
    with the entity's position, character, and color. This makes it
    flexible enough to draw any entity we pass to it.

Note that we no longer need a separate `clear_entity` or `clear_all`
function — calling `con.clear()` in the game loop (which we already do)
erases the entire console each frame, which is simpler and more thorough.

Now that we've gotten a few functions to assist drawing the entities,
let's put them to use. Make the following modifications to the section
where we drew the player (in `engine.py`).

{{< codetab >}}
{{< diff-tab >}}
{{< highlight diff >}}
        while True:
-           con.print(player.x, player.y, '@', fg=(255, 255, 255))
-           con.blit(dest=root_console)
+           render_all(con, root_console, entities, screen_width, screen_height)
            context.present(root_console)
            con.clear()
        ...
{{</ highlight >}}
{{</ diff-tab >}}
{{< original-tab >}}
<pre>        while True:
            <span class="crossed-out-text">con.print(player.x, player.y, '@', fg=(255, 255, 255))</span>
            <span class="crossed-out-text">con.blit(dest=root_console)</span>
            <span class="new-text">render_all(con, root_console, entities, screen_width, screen_height)</span>
            context.present(root_console)
            con.clear()
        ...</pre>
{{</ original-tab >}}
{{</ codetab >}}

Don't forget to import `render_all` at the top of your file. Your
imports section should now look something like this:

{{< codetab >}}
{{< diff-tab >}}
{{< highlight diff >}}
import tcod

from entity import Entity
from input_handlers import handle_keys
+from render_functions import render_all
{{</ highlight >}}
{{</ diff-tab >}}
{{< original-tab >}}
<pre>import tcod

from entity import Entity
from input_handlers import handle_keys
<span class="new-text">from render_functions import render_all</span></pre>
{{</ original-tab >}}
{{</ codetab >}}

If you run the project now, you should see your '@' symbol, along with a
yellow one representing our NPC. It doesn't do anything, yet, but now we
have a method for drawing more than one character to the screen.

It's time to shift gears a bit and get our map in place. The map will
consist of a 2d array of Tile objects. Tiles will have a few properties
that define whether we can move through them, or see through them.

We'd better start by defining the size of our map. Add these variables
right below where you defined the screen width and height:

{{< codetab >}}
{{< diff-tab >}}
{{< highlight diff >}}
    ...
    screen_height = 50
+   map_width = 80
+   map_height = 45
{{</ highlight >}}
{{</ diff-tab >}}
{{< original-tab >}}
<pre>    ...
    screen_height = 50
    <span class="new-text">map_width = 80
    map_height = 45</span></pre>
{{</ original-tab >}}
{{</ codetab >}}

Simple enough. Now we need a place for our Tile class, along with a few
other map classes, to live. I prefer to keep similar classes in the same
folder, so create a new Python package (that's a directory with a file
called `__init__.py` in it, \_\_init\_\_.py is empty in this case)
called `map_objects`. In there, create a file called `tile.py` and put
the following code in it.

{{< highlight py3 >}}
class Tile:
    """
    A tile on a map. It may or may not be blocked, and may or may not block sight.
    """
    def __init__(self, blocked, block_sight=None):
        self.blocked = blocked

        # By default, if a tile is blocked, it also blocks sight
        if block_sight is None:
            block_sight = blocked

        self.block_sight = block_sight

{{</ highlight >}}

Nothing too complicated here. The `Tile` class holds whether or not the
tile is blocked (if it's blocked, you can't move through it), and
whether or not it blocks sight (for our FOV algorithm later). Notice
that you don't have to pass `block_sight` every time; it's assumed to be
the same as `blocked`. By keeping the two separate, we can have a tile
that can be seen-through, but not crossed (a lava pit maybe?), or vice
versa (a dark room perhaps).

Now that we have the tile class, we need some sort of container to hold
our tiles. Let's call this class `GameMap`, which will hold the 2d array
of tiles and some methods for setting up and interacting with it. Create
a file (in the map\_objects folder) and call it `game_map.py`, and put
the following in it:

{{< highlight py3 >}}
from map_objects.tile import Tile


class GameMap:
    def __init__(self, width, height):
        self.width = width
        self.height = height
        self.tiles = self.initialize_tiles()

    def initialize_tiles(self):
        tiles = [[Tile(False) for y in range(self.height)] for x in range(self.width)]

        tiles[30][22].blocked = True
        tiles[30][22].block_sight = True
        tiles[31][22].blocked = True
        tiles[31][22].block_sight = True
        tiles[32][22].blocked = True
        tiles[32][22].block_sight = True

        return tiles
{{</ highlight >}}

We're passing in the width and height of the map (which we've defined in
our engine), and initializing a 2d array of Tiles, set to non-blocking
by default. We're setting a few tiles as blocked, just for demonstration
purposes. I've kept the code setting the tiles up out of the `__init__`
function, for two reasons. One, it may get called outside the
initialization, and two, because I prefer keeping `__init__` functions
as simple as possible.

Go back to `engine.py`, where we'll make a few changes so that our map
gets initialized and then drawn to the screen.

Firstly, we need to define what colors to draw for blocked and
non-blocked tiles. Let's set up a dictionary that holds the colors we'll
be using for now (it will expand as this tutorial goes on).

{{< codetab >}}
{{< diff-tab >}}
{{< highlight diff >}}
    ...
    map_height = 45

+   colors = {
+       'dark_wall': (0, 0, 100),
+       'dark_ground': (50, 50, 150)
+   }

    player = Entity(int(screen_width / 2), int(screen_height / 2), '@', (255, 255, 255))
{{</ highlight >}}
{{</ diff-tab >}}
{{< original-tab >}}
<pre>    ...
    map_height = 45

    <span class="new-text">colors = {
        'dark_wall': (0, 0, 100),
        'dark_ground': (50, 50, 150)
    }</span>

    player = Entity(int(screen_width / 2), int(screen_height / 2), '@', (255, 255, 255))</pre>
{{</ original-tab >}}
{{</ codetab >}}

These colors will serve as our wall and ground outside the FOV, when we
get there (hence the 'dark' in the names). Colors are specified as
`(red, green, blue)` tuples, with each value between 0 and 255.

Now let's initialize the game map itself. This can go anywhere before
the main loop; I put mine inside the `with` block, right below the
console initialization.

{{< codetab >}}
{{< diff-tab >}}
{{< highlight diff >}}
        root_console = tcod.console.Console(screen_width, screen_height, order='F')
        con = tcod.console.Console(screen_width, screen_height, order='F')

+       game_map = GameMap(map_width, map_height)

        while True:
{{</ highlight >}}
{{</ diff-tab >}}
{{< original-tab >}}
<pre>        root_console = tcod.console.Console(screen_width, screen_height, order='F')
        con = tcod.console.Console(screen_width, screen_height, order='F')

        <span class="new-text">game_map = GameMap(map_width, map_height)</span>

        while True:</pre>
{{</ original-tab >}}
{{</ codetab >}}

Don't forget to import the actual GameMap object so that we can use it
in the engine.

{{< codetab >}}
{{< diff-tab >}}
{{< highlight diff >}}
from entity import Entity
from input_handlers import handle_keys
+from map_objects.game_map import GameMap
from render_functions import render_all
{{</ highlight >}}
{{</ diff-tab >}}
{{< original-tab >}}
<pre>from entity import Entity
from input_handlers import handle_keys
<span class="new-text">from map_objects.game_map import GameMap</span>
from render_functions import render_all</pre>
{{</ original-tab >}}
{{</ codetab >}}

Now that our map object is ready to go, let's pass it to `render_all` so
that we can draw it. We'll also pass the `colors` dictionary, because
`render_all` will need to know what colors to draw the various parts of
the map.

{{< codetab >}}
{{< diff-tab >}}
{{< highlight diff >}}
-       render_all(con, root_console, entities, screen_width, screen_height)
+       render_all(con, root_console, entities, game_map, screen_width, screen_height, colors)
{{</ highlight >}}
{{</ diff-tab >}}
{{< original-tab >}}
<pre>        <span class="crossed-out-text">render_all(con, root_console, entities, screen_width, screen_height)</span>
        <span class="new-text">render_all(con, root_console, entities, game_map, screen_width, screen_height, colors)</span></pre>
{{</ original-tab >}}
{{</ codetab >}}

Open up `render_functions.py` and modify `render_all` like this:

{{< codetab >}}
{{< diff-tab >}}
{{< highlight diff >}}
-def render_all(con, root_console, entities, screen_width, screen_height):
+def render_all(con, root_console, entities, game_map, screen_width, screen_height, colors):
+   # Draw all the tiles in the game map
+   for y in range(game_map.height):
+       for x in range(game_map.width):
+           wall = game_map.tiles[x][y].block_sight
+
+           if wall:
+               con.bg[x, y] = colors.get('dark_wall')
+           else:
+               con.bg[x, y] = colors.get('dark_ground')
+
    # Draw all entities in the list
    for entity in entities:
        draw_entity(con, entity)

    con.blit(dest=root_console)
{{</ highlight >}}
{{</ diff-tab >}}
{{< original-tab >}}
<pre><span class="crossed-out-text">def render_all(con, root_console, entities, screen_width, screen_height):</span>
<span class="new-text">def render_all(con, root_console, entities, game_map, screen_width, screen_height, colors):
    # Draw all the tiles in the game map
    for y in range(game_map.height):
        for x in range(game_map.width):
            wall = game_map.tiles[x][y].block_sight

            if wall:
                con.bg[x, y] = colors.get('dark_wall')
            else:
                con.bg[x, y] = colors.get('dark_ground')
</span>
    # Draw all entities in the list
    for entity in entities:
        draw_entity(con, entity)

    con.blit(dest=root_console)</pre>
{{</ original-tab >}}
{{</ codetab >}}

`render_all` now loops through each tile in the game map and checks if
it blocks sight or not. If it does, it draws a wall background; if not,
a floor background. `con.bg[x, y]` directly sets the background color of
tile (x, y) on the console.

Run the project now, and you should see the 'map' drawn with some color
to it. You'll see our three block wall as well, but there's one problem:
We can move through the wall\!

We need to add two more things before we can call it a day. Modify the
part where the player's move function gets called to look like this:

{{< codetab >}}
{{< diff-tab >}}
{{< highlight diff >}}
                if move:
                    dx, dy = move

+                   if not game_map.is_blocked(player.x + dx, player.y + dy):
+                       player.move(dx, dy)
-                   player.move(dx, dy)
{{</ highlight >}}
{{</ diff-tab >}}
{{< original-tab >}}
<pre>                if move:
                    dx, dy = move
<span class="new-text">
                    if not game_map.is_blocked(player.x + dx, player.y + dy):
                        player.move(dx, dy)</span>
                    <span class="crossed-out-text">player.move(dx, dy)</span></pre>
{{</ original-tab >}}
{{</ codetab >}}

*\* Note the indentation change for player.move(dx, dy). This is Python,
so indentation matters\!*

Now we just have to create the `is_blocked` method in the game map. Open
up the `game_map.py` file and add this method to the class:

{{< highlight py3 >}}
    def is_blocked(self, x, y):
        if self.tiles[x][y].blocked:
            return True

        return False
{{</ highlight >}}

*\* Note: You could shorten the is\_blocked function, by simply writing*
`return self.tiles[x][y].blocked`. *But we'll be modifying this function
to do more checks soon, hence we're taking the more explicit route.*

Run the project again, and you will get stuck on the walls.

That's going to do it for this tutorial. It may not look like much, but
we've set ourselves up to create some real looking dungeons in the next
tutorial.

If you want to see the code so far in its entirety, [click
here](https://github.com/TStand90/roguelike_tutorial_revised/tree/part2).

[Click here to move on to the next part of this
tutorial.](/tutorials/tcod/2019/part-3)

<script src="/js/codetabs.js"></script>
