---
title: "Part 4 - Field of View"
date: 2019-03-30T09:33:44-07:00
draft: false
aliases: /tutorials/tcod/part-4
---

We have a dungeon now, and we can move about it freely. But are we
really *exploring* the dungeon if we can just see it all from the
beginning?

Most roguelikes (not all\!) only let you see within a certain range of
your character, and ours will be no different. We need to implement a way
to calculate the "Field of View" for our adventurer, and fortunately,
tcod makes that easy\!

We'll need to define a few variables before we get started. Add these in
the same section as our screen and map variables:

{{< codetab >}} {{< diff-tab >}} {{< highlight diff >}}
    ...
    max_rooms = 30

+   fov_algorithm = 0
+   fov_light_walls = True
+   fov_radius = 10

    colors = {
    ...
{{</ highlight >}}
{{</ diff-tab >}}
{{< original-tab >}}
<pre>    ...
    max_rooms = 30

    <span class="new-text">fov_algorithm = 0
    fov_light_walls = True
    fov_radius = 10</span>

    colors = {
    ...</pre>
{{</ original-tab >}}
{{</ codetab >}}

'0' is just the default algorithm that tcod uses; it has more, and I
encourage you to experiment with them later. `fov_light_walls` just
tells us whether or not to 'light up' the walls we see; you can change
it if you don't like the way it looks. `fov_radius` is somewhat obvious,
it tells us how far we can actually see.

We also need to update the `colors` dictionary, because now we need two
more colors for the 'light' versions of both walls and floors. Walls and
floors in our fov will be 'lit', distinguishing them from the ones
outside what we can see.

{{< highlight py3 >}}
    colors = {
        'dark_wall': (0, 0, 100),
        'dark_ground': (50, 50, 150),
        'light_wall': (130, 110, 50),
        'light_ground': (200, 180, 50)
    }
{{</ highlight >}}

*\* Don't forget to add the comma after the 'dark\_ground' entry; Python
will throw an error without it\!*

If you don't like these colors, feel free to change them to your liking.

The thing about field of view is that it doesn't need to be computed
every turn. In fact, it would be quite a waste to do so\! We really only
need to change it when the player moves. Attacking, using an item, or
just standing still for a turn doesn't alter FOV. We can handle this by
having a boolean variable, which we'll call `fov_recompute`, which tells
us if we need to recompute. We can define it somewhere above our game
loop (I put mine right after the map initialization).

{{< codetab >}} {{< diff-tab >}} {{< highlight diff >}}
    ...
    game_map.make_map(max_rooms, room_min_size, room_max_size, map_width, map_height, player)

+   fov_recompute = True

        while True:
    ...
{{</ highlight >}}
{{</ diff-tab >}}
{{< original-tab >}}
<pre>    ...
    game_map.make_map(max_rooms, room_min_size, room_max_size, map_width, map_height, player)

    <span class="new-text">fov_recompute = True</span>

        while True:
    ...</pre>
{{</ original-tab >}}
{{</ codetab >}}

It's `True` by default, because we have to compute it right when the
game starts.

Now let's initialize our field of view, which we'll store in a variable
called `fov_map`. Instead of a special object, `fov_map` is simply a
numpy boolean array: `True` means that tile is currently visible, `False`
means it is not. Let's keep these functions out of `engine.py`, and
instead, put them in a new file, called `fov_functions.py`. In that
file, put the following:

{{< highlight py3 >}}
import numpy as np
import tcod


def initialize_fov(game_map):
    return np.zeros((game_map.width, game_map.height), dtype=bool)


def recompute_fov(fov_map, game_map, x, y, radius, light_walls=True, algorithm=0):
    transparent = np.array(
        [[not game_map.tiles[tx][ty].block_sight for ty in range(game_map.height)]
         for tx in range(game_map.width)],
        dtype=bool
    )
    fov_map[:] = tcod.map.compute_fov(
        transparent.T,
        (y, x),
        radius=radius,
        light_walls=light_walls,
        algorithm=algorithm,
    ).T
{{</ highlight >}}

`initialize_fov` returns an all-`False` array the same size as the map —
nothing is visible when the game starts. `recompute_fov` builds a
`transparent` array from the tile data (True where a tile doesn't block
sight), then calls `tcod.map.compute_fov`. The `.T` transposes between
the `[x, y]` layout we use and the `[y, x]` (row, column) layout tcod
expects. After recomputing, `fov_map[x, y]` is `True` wherever the
player can see.

Note that `recompute_fov` now takes `game_map` as a parameter — it needs
the tile data to build the transparency array.

Call `initialize_fov` in `engine.py` and store the result in `fov_map`.

{{< codetab >}} {{< diff-tab >}} {{< highlight diff >}}
    ...
    fov_recompute = True

+   fov_map = initialize_fov(game_map)

        while True:
    ...
{{</ highlight >}}
{{</ diff-tab >}}
{{< original-tab >}}
<pre>    ...
    fov_recompute = True

    <span class="new-text">fov_map = initialize_fov(game_map)</span>

        while True:
    ...</pre>
{{</ original-tab >}}
{{</ codetab >}}

Don't forget the import for the function.

{{< codetab >}} {{< diff-tab >}} {{< highlight diff >}}
...
from entity import Entity
+from fov_functions import initialize_fov
from input_handlers import handle_keys
...
{{</ highlight >}}
{{</ diff-tab >}}
{{< original-tab >}}
<pre>...
from entity import Entity
<span class="new-text">from fov_functions import initialize_fov</span>
from input_handlers import handle_keys
...</pre>
{{</ original-tab >}}
{{</ codetab >}}

While we're at it, let's modify the section where we move the player to
set `fov_recompute` to True.

{{< codetab >}} {{< diff-tab >}} {{< highlight diff >}}
                ...
                player.move(dx, dy)

+               fov_recompute = True
{{</ highlight >}}
{{</ diff-tab >}}
{{< original-tab >}}
<pre>                ...
                player.move(dx, dy)

                <span class="new-text">fov_recompute = True</span></pre>
{{</ original-tab >}}
{{</ codetab >}}

But where does the recompute actually *happen*? Put your fov
recomputation in `engine.py` at the start of the game loop, before the
render call.

{{< codetab >}} {{< diff-tab >}}
{{< highlight diff >}}
        while True:
+           if fov_recompute:
+               recompute_fov(fov_map, game_map, player.x, player.y, fov_radius, fov_light_walls, fov_algorithm)
+
            render_all(con, root_console, entities, game_map, screen_width, screen_height, colors)
        ...
{{</ highlight >}}
{{</ diff-tab >}}
{{< original-tab >}}
<pre>        while True:
            <span class="new-text">if fov_recompute:
                recompute_fov(fov_map, game_map, player.x, player.y, fov_radius, fov_light_walls, fov_algorithm)
</span>
            render_all(con, root_console, entities, game_map, screen_width, screen_height, colors)
        ...</pre>
{{</ original-tab >}}
{{</ codetab >}}

... And, of course, we have to import that function:

{{< codetab >}} {{< diff-tab >}} {{< highlight diff >}}
...
from entity import Entity
-from fov_functions import initialize_fov
+from fov_functions import initialize_fov, recompute_fov
from input_handlers import handle_keys
...
{{</ highlight >}}
{{</ diff-tab >}}
{{< original-tab >}}
<pre>...
from entity import Entity
from fov_functions import initialize_fov<span class="new-text">, recompute_fov</span>
from input_handlers import handle_keys
...</pre>
{{</ original-tab >}}
{{</ codetab >}}

Now, after the player successfully moves, the field of view will be set
to recalculate, but it won't if we do something else.

With our field of view calculated, we need to actually *display* it (if
you run the code now, you won't notice any visible change). Open up
`render_functions.py` and modify the `render_all` function like
this:

{{< codetab >}} {{< diff-tab >}} {{< highlight diff >}}
-def render_all(con, root_console, entities, game_map, screen_width, screen_height, colors):
+def render_all(con, root_console, entities, game_map, fov_map, fov_recompute, screen_width, screen_height, colors):
-   for y in range(game_map.height):
+   if fov_recompute:
-       for x in range(game_map.width):
+       for y in range(game_map.height):
-           wall = game_map.tiles[x][y].block_sight
-
-           if wall:
-               con.bg[x, y] = colors.get('dark_wall')
-           else:
-               con.bg[x, y] = colors.get('dark_ground')
+           for x in range(game_map.width):
+               visible = fov_map[x, y]
+               wall = game_map.tiles[x][y].block_sight

+               if visible:
+                   if wall:
+                       con.bg[x, y] = colors.get('light_wall')
+                   else:
+                       con.bg[x, y] = colors.get('light_ground')
+               else:
+                   if wall:
+                       con.bg[x, y] = colors.get('dark_wall')
+                   else:
+                       con.bg[x, y] = colors.get('dark_ground')

    # Draw all entities in the list
    for entity in entities:
        draw_entity(con, entity)

    con.blit(dest=root_console)
{{</ highlight >}}
{{</ diff-tab >}}
{{< original-tab >}}
<pre><span class="crossed-out-text">def render_all(con, root_console, entities, game_map, screen_width, screen_height, colors):</span>
<span class="new-text">def render_all(con, root_console, entities, game_map, fov_map, fov_recompute, screen_width, screen_height, colors):</span>
    <span class="new-text">if fov_recompute:</span>
        <span style="color: blue">for y in range(game_map.height):
            for x in range(game_map.width):</span>
                <span class="new-text">visible = fov_map[x, y]</span>
                <span style="color: blue">wall = game_map.tiles[x][y].block_sight</span>

                <span class="new-text">if visible:
                    if wall:
                        con.bg[x, y] = colors.get('light_wall')
                    else:
                        con.bg[x, y] = colors.get('light_ground')
                else:</span>
                    <span style="color: blue">if wall:
                        con.bg[x, y] = colors.get('dark_wall')
                    else:
                        con.bg[x, y] = colors.get('dark_ground')</span>

    # Draw all entities in the list
    for entity in entities:
        draw_entity(con, entity)

    con.blit(dest=root_console)</pre>
{{</ original-tab >}}
{{</ codetab >}}

*\* Note: Blue denotes lines that are exactly the same as before, except
for their indentation. The if statements for `fov_recompute` and
`visible` force certain lines to be indented farther than they were
before. Remember, this is Python, indentation matters\!*

Now our `render_all` function will display tiles differently, depending
on if they're in our field of view or not. If a tile's position is
`True` in `fov_map`, we draw it with the 'light' colors, and if not, we
draw the 'dark' version.

The definition of `render_all` has changed, so be sure to update it in
`engine.py`. While we're at it, let's set `fov_recompute` to `False`
after we call `render_all`.

{{< codetab >}} {{< diff-tab >}} {{< highlight diff >}}
        ...
-       render_all(con, root_console, entities, game_map, screen_width, screen_height, colors)
+       render_all(con, root_console, entities, game_map, fov_map, fov_recompute, screen_width, screen_height, colors)
+
+       fov_recompute = False
{{</ highlight >}}
{{</ diff-tab >}}
{{< original-tab >}}
<pre>        ...
        <span class="crossed-out-text">render_all(con, root_console, entities, game_map, screen_width, screen_height, colors)</span>
        <span class="new-text">render_all(con, root_console, entities, game_map, fov_map, fov_recompute, screen_width, screen_height, colors)

        fov_recompute = False</span></pre>
{{</ original-tab >}}
{{</ codetab >}}

Run the project now. The player's field of view is now visible\! But,
despite being able to "see" the FOV, it still doesn't really *do*
anything. We can still see the entire map, along with our NPC. Luckily,
the changes we have to make to fix this are fairly minimal.

Let's start with our NPC. We should just be able to modify our
`draw_entity` function to account for the field of view, which would
solve our problem.

{{< codetab >}} {{< diff-tab >}} {{< highlight diff >}}
-def draw_entity(con, entity):
+def draw_entity(con, entity, fov_map):
-   con.print(entity.x, entity.y, entity.char, fg=entity.color)
+   if fov_map[entity.x, entity.y]:
+       con.print(entity.x, entity.y, entity.char, fg=entity.color)
{{</ highlight >}}
{{</ diff-tab >}}
{{< original-tab >}}
<pre><span class="crossed-out-text">def draw_entity(con, entity):</span>
<span class="new-text">def draw_entity(con, entity, fov_map):
    if fov_map[entity.x, entity.y]:</span>
        <span style="color: blue">con.print(entity.x, entity.y, entity.char, fg=entity.color)</span></pre>
{{</ original-tab >}}
{{</ codetab >}}

*\* Again, the blue means the line is the same as before, except the
indentation has changed.*

Also be sure to update the part where we call the function:

{{< codetab >}} {{< diff-tab >}} {{< highlight diff >}}
    for entity in entities:
-       draw_entity(con, entity)
+       draw_entity(con, entity, fov_map)
{{</ highlight >}}
{{</ diff-tab >}}
{{< original-tab >}}
<pre>    for entity in entities:
        <span class="crossed-out-text">draw_entity(con, entity)</span>
        <span class="new-text">draw_entity(con, entity, fov_map)</span></pre>
{{</ original-tab >}}
{{</ codetab >}}

Run the project again, and you won't see the NPC unless it's in your
field of view.

Now for the map. In traditional roguelikes, your character can only see
what's inside its field of view, but it will "remember" areas that were
explored previously. We can accomplish this effect by adding a variable
called `explored` to our `Tile` class. Modify the `__init__` function in
`Tile` to include this new variable:

{{< codetab >}} {{< diff-tab >}} {{< highlight diff >}}
        ...
        self.block_sight = block_sight

+       self.explored = False
{{</ highlight >}}
{{</ diff-tab >}}
{{< original-tab >}}
<pre>        ...
        self.block_sight = block_sight

        <span class="new-text">self.explored = False</span></pre>
{{</ original-tab >}}
{{</ codetab >}}

This new variable needs to be taken into account in our `render_all`
function. Let's do that now. We'll only draw the tiles outside of our
field of view if we've explored them previously. Also, any tiles that
*are* in our field of view, we'll mark as 'explored'.

{{< codetab >}} {{< diff-tab >}} {{< highlight diff >}}
                ...
                visible = fov_map[x, y]
                wall = game_map.tiles[x][y].block_sight

                if visible:
                    if wall:
                        con.bg[x, y] = colors.get('light_wall')
                    else:
                        con.bg[x, y] = colors.get('light_ground')

+                   game_map.tiles[x][y].explored = True
-               else:
+               elif game_map.tiles[x][y].explored:
                    if wall:
                        con.bg[x, y] = colors.get('dark_wall')
                    else:
                        con.bg[x, y] = colors.get('dark_ground')
                    ...
{{</ highlight >}}
{{</ diff-tab >}}
{{< original-tab >}}
<pre>                ...
                visible = fov_map[x, y]
                wall = game_map.tiles[x][y].block_sight

                if visible:
                    if wall:
                        con.bg[x, y] = colors.get('light_wall')
                    else:
                        con.bg[x, y] = colors.get('light_ground')

                    <span class="new-text">game_map.tiles[x][y].explored = True</span>
                <span class="crossed-out-text">else:</span>
                <span class="new-text">elif game_map.tiles[x][y].explored:</span>
                    if wall:
                        con.bg[x, y] = colors.get('dark_wall')
                    else:
                        con.bg[x, y] = colors.get('dark_ground')
                    ...</pre>
{{</ original-tab >}}
{{</ codetab >}}

We now have a real, explorable dungeon\! True, there may not be much
*in* there right now, but this was a major step to a working game. In
the next few parts, we'll fill the dungeon with some evil(?) monsters to
punch.

If you want to see the code so far in its entirety, [click
here](https://github.com/TStand90/roguelike_tutorial_revised/tree/part4).

[Click here to move on to the next part of this
tutorial.](/tutorials/tcod/2019/part-5)

<script src="/js/codetabs.js"></script>
