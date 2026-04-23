---
title: "Part 1 - Drawing the '@' symbol and moving it around"
date: 2019-03-30T08:39:15-07:00
draft: false
aliases: /tutorials/tcod/part-1
---

Welcome to part 1 of the **Roguelike Tutorial Revised**\! This series
will help you create your very first roguelike game, written in Python\!

This tutorial is largely based off the [one found on
Roguebasin](http://www.roguebasin.com/index.php?title=Complete_Roguelike_Tutorial,_using_python%2Blibtcod).
Many of the design decisions were mainly to keep this tutorial in lockstep
with that one (at least in terms of chapter composition and general
direction). This tutorial would not have been possible without the
guidance of those who wrote that tutorial, along with all the wonderful
contributors to libtcod and python-tcod over the years.

This part assumes that you have either checked [Part
0](/tutorials/tcod/2019/part-0) and are already set up and ready to go. If
not, be sure to check that page, and make sure that you've got Python
and TCOD installed, and a file called `engine.py` created in the
directory that you want to work in.

Assuming that you've done all that, let's get started. Modify (or
create, if you haven't already) the file `engine.py` to look like this:

{{< highlight py3 >}}
import tcod


def main():
    print('Hello World!')


if __name__ == '__main__':
    main()
{{</ highlight >}}

You can run the program like any other Python program, but for those
who are brand new, you do that by typing `python engine.py` in the
terminal. If you have both Python 2 and 3 installed on your machine, you
might have to use `python3 engine.py` to run (it depends on your default
python, and whether you're using a virtualenv or not).

Okay, not the most exciting program in the world, I admit, but we've
already got our first major difference from the other tutorial. Namely,
this funky looking thing here:

{{< highlight py3 >}}
if __name__ == '__main__':
    main()
{{< /highlight >}}

So what does that do? Basically, we're saying that we're only going to
run the "main" function when we explicitly run the script, using `python
engine.py`. It's not super important that you understand this now, but
if you want a more detailed explanation, [this answer on Stack
Overflow](https://stackoverflow.com/a/419185) gives a pretty good
overview.

Confirm that the above program runs (if not, there's probably an issue
with your tcod setup). Once that's done, we can move on to bigger and
better things. The first major step to creating any roguelike is getting
an '@' character on the screen and moving, so let's get started with
that.

Modify `engine.py` to look like this:

{{< highlight py3 >}}
import tcod


def main():
    screen_width = 80
    screen_height = 50

    tileset = tcod.tileset.load_tilesheet(
        'arial10x10.png', 32, 8, tcod.tileset.CHARMAP_TCOD
    )

    with tcod.context.new(
        columns=screen_width,
        rows=screen_height,
        tileset=tileset,
        title='libtcod tutorial revised',
        vsync=True,
    ) as context:
        root_console = tcod.console.Console(screen_width, screen_height, order='F')

        while True:
            root_console.print(1, 1, '@', fg=(255, 255, 255))
            context.present(root_console)
            root_console.clear()

            for event in tcod.event.wait():
                if isinstance(event, tcod.event.Quit):
                    raise SystemExit()
                if isinstance(event, tcod.event.KeyDown):
                    if event.sym == tcod.event.KeySym.ESCAPE:
                        raise SystemExit()


if __name__ == '__main__':
    main()
{{</ highlight >}}

Run `engine.py` again, and you should see an '@' symbol on the screen.
Once you've fully soaked in the glory on the screen in front of you, you
can hit the `Esc` key to exit the program.

There's a lot going on here, so let's break it down line by line.

{{< highlight py3 >}}
    screen_width = 80
    screen_height = 50
{{</ highlight >}}

This is simple enough. We're defining some variables for the screen
size. Eventually, we'll load these values from a JSON file rather than
hard coding them in the source, but we won't worry about that until we
have some more variables like
this.

{{< highlight py3 >}}
    tileset = tcod.tileset.load_tilesheet(
        'arial10x10.png', 32, 8, tcod.tileset.CHARMAP_TCOD
    )
{{</ highlight >}}

Here, we're telling tcod which font to use. The `'arial10x10.png'`
bit is the actual file we're reading from (this should exist in your
project folder). The numbers `32, 8` specify the layout of the tile
sheet (32 columns, 8 rows of characters), and `tcod.tileset.CHARMAP_TCOD`
tells tcod which character mapping to use.

{{< highlight py3 >}}
    with tcod.context.new(
        columns=screen_width,
        rows=screen_height,
        tileset=tileset,
        title='libtcod tutorial revised',
        vsync=True,
    ) as context:
{{</ highlight >}}

This is what actually creates the window. We use a `with` statement so
that the window is automatically cleaned up when our program exits. We
pass it the `screen_width` and `screen_height` values from before (80 and
50, respectively), the tileset we just loaded, a title for the window,
and `vsync=True` to enable vertical sync (smooth rendering).

{{< highlight py3 >}}
        while True:
{{</ highlight >}}

This is what's called our 'game loop'. Basically, this is a loop that
won't ever end until we explicitly exit. Every game has some sort of
game loop or another.

{{< highlight py3 >}}
            root_console.print(1, 1, '@', fg=(255, 255, 255))
{{</ highlight >}}

This prints our '@' character to the console at position (1, 1). The
`fg=(255, 255, 255)` sets the foreground color to white (as an RGB
tuple). If you want your character to be a different color, try changing
those numbers (they represent red, green, and blue values from 0 to 255).

{{< highlight py3 >}}
            context.present(root_console)
{{</ highlight >}}

This is the part that presents everything on the screen. The
`root_console` is passed to `context.present()`, which draws it to the
window. Pretty straightforward.

{{< highlight py3 >}}
            root_console.clear()
{{</ highlight >}}

After presenting, we clear the console so it's ready for the next frame.
This erases everything we've drawn, so the next loop iteration starts
with a blank slate.

{{< highlight py3 >}}
            for event in tcod.event.wait():
                if isinstance(event, tcod.event.Quit):
                    raise SystemExit()
                if isinstance(event, tcod.event.KeyDown):
                    if event.sym == tcod.event.KeySym.ESCAPE:
                        raise SystemExit()
{{</ highlight >}}

`tcod.event.wait()` pauses the program and returns events one at a time
as the user interacts with the window. We check if the event is a
`Quit` event (the user closed the window) or a `KeyDown` event (the user
pressed a key). If the user presses `Esc`, we raise `SystemExit()` to
close the program gracefully.

So we've got our '@' symbol drawn, now let's get it moving around\!

We need to keep track of the player's position at all times, so let's
create two variables, `player_x` and `player_y` to keep track of this.

{{< codetab >}}
{{< diff-tab >}}
{{< highlight diff >}}
    ...
    screen_height = 50
+
+   player_x = int(screen_width / 2)
+   player_y = int(screen_height / 2)
+
    tileset = tcod.tileset.load_tilesheet(
    ...
{{</ highlight >}}
{{</ diff-tab >}}
{{< original-tab >}}
<pre>    ...
    screen_height = 50
    <span class="new-text">
    player_x = int(screen_width / 2)
    player_y = int(screen_height / 2)
    </span>
    tileset = tcod.tileset.load_tilesheet(
    ...</pre>
{{</ original-tab >}}
{{</ codetab >}}

*Note: Ellipses denote omitted parts of the code. I'll include lines
around the code to be inserted so that you'll know exactly where to put
new pieces of code, but I won't be showing the entire file every time.
The green lines denote code that you should be adding.*

We're placing the player right in the middle of the screen. What's with
the `int()` function though? Well, Python 3 doesn't automatically
truncate division like Python 2 does, so we have to cast the division
result (a float) to an integer. If we don't, tcod will give an error.

We also have to modify the command to print the '@' symbol to use these
new coordinates.

{{< codetab >}}
{{< diff-tab >}}
{{< highlight diff >}}
        ...
-       root_console.print(1, 1, '@', fg=(255, 255, 255))
+       root_console.print(player_x, player_y, '@', fg=(255, 255, 255))
        context.present(root_console)
        ...
{{</ highlight >}}
{{</ diff-tab >}}
{{< original-tab >}}
<pre>        ...
        <span class="crossed-out-text">root_console.print(1, 1, '@', fg=(255, 255, 255))</span>
        <span class="new-text">root_console.print(player_x, player_y, '@', fg=(255, 255, 255))</span>
        context.present(root_console)
        ...</pre>
{{</ original-tab >}}
{{</ codetab >}}

*Note: The red lines denote code that has been removed.*

Run the code now and you should see the '@' in the center of the screen.
Let's take care of moving it around now.

Okay, so we're rendering the '@' but it just sits there. We need to
define a function to handle the user's input. It will essentially
translate the user's key presses into game actions.

Up until now, this tutorial hasn't deviated all that much from the
original one, but here's a critical turning point. We're about to define
a function, called `handle_keys` to take care of keyboard input. We
*could* put this in our `engine.py` file... but should it be there? I
would argue no. The engine (game loop) captures input and should do
something with it; but, translating from one to the other is not
something it needs to know about.

So rather than putting the `handle_keys` function in `engine.py`, let's
create a new file, called `input_handlers.py`. Put the following code
inside that new file.

{{< highlight py3 >}}
import tcod


def handle_keys(event):
    if isinstance(event, tcod.event.KeyDown):
        key = event.sym

        # Movement keys
        if key == tcod.event.KeySym.UP:
            return {'move': (0, -1)}
        elif key == tcod.event.KeySym.DOWN:
            return {'move': (0, 1)}
        elif key == tcod.event.KeySym.LEFT:
            return {'move': (-1, 0)}
        elif key == tcod.event.KeySym.RIGHT:
            return {'move': (1, 0)}

        if key == tcod.event.KeySym.RETURN and event.mod & tcod.event.Modifier.LALT:
            # Alt+Enter: toggle full screen
            return {'fullscreen': True}

        elif key == tcod.event.KeySym.ESCAPE:
            # Exit the game
            return {'exit': True}

    # No key was pressed
    return {}
{{</ highlight >}}

That's a lot to take in all at once, so again, let's break it down a
bit.

{{< highlight py3 >}}
def handle_keys(event):
{{</ highlight >}}

We're defining a function called `handle_keys`, which takes one
argument, `event`. `event` in this case will be a tcod event object from
the event loop.

{{< highlight py3 >}}
    if isinstance(event, tcod.event.KeyDown):
        key = event.sym
{{</ highlight >}}

First we check if the event is actually a key press (`tcod.event.KeyDown`).
If it is, we grab the key symbol from `event.sym` and store it in `key`
for convenience.

{{< highlight py3 >}}
        if key == tcod.event.KeySym.UP:
{{</ highlight >}}

This if statement (along with the other elifs) just tell us which key
was pressed. Right now, it's one of the arrow keys for movement. What's
more interesting is the code inside these if statements.

{{< highlight py3 >}}
        return {'move': (0, -1)}
{{</ highlight >}}

So what's going on here? Well, when we return from this function, the
engine is going to have to do something. In this case, we want our
character to move. But what if we hit a different key? Then we might not
be moving; we may be using an item, casting a spell, or exiting the
game. One way to handle all these different possibilities is to return a
dictionary from this function, which the engine will read and decide
what to do.

In this instance, we're returning a dictionary with the key `'move'`,
and the value is a pair of numbers. The numbers will tell the engine in
what direction to move the player. So for example, the 'up' key will
move us '0' on the x axis, and '-1' on the y axis.

{{< highlight py3 >}}
        if key == tcod.event.KeySym.RETURN and event.mod & tcod.event.Modifier.LALT:
            # Alt+Enter: toggle full screen
            return {'fullscreen': True}
        elif key == tcod.event.KeySym.ESCAPE:
            # Exit the game
            return {'exit': True}
{{</ highlight >}}

These are our non-movement actions that we're allowing for now. If the
user pressed ALT+Enter, the game will go full screen. If the user
presses 'Esc', the game will exit. Note that `event.mod &
tcod.event.Modifier.LALT` checks whether the left Alt key is held down
at the same time as the Enter key.

{{< highlight py3 >}}
    return {}
{{</ highlight >}}

Because our engine will be expecting a dictionary, we have to return
*something*, even if nothing happened.

This may seem confusing, but it will likely make sense in a minute.
Let's return to our `engine.py` file and call our `handle_keys`
function.

{{< codetab >}}
{{< diff-tab >}}
{{< highlight diff >}}
            ...
            for event in tcod.event.wait():
                if isinstance(event, tcod.event.Quit):
                    raise SystemExit()
-               if isinstance(event, tcod.event.KeyDown):
-                   if event.sym == tcod.event.KeySym.ESCAPE:
-                       raise SystemExit()
+               action = handle_keys(event)
+
+               move = action.get('move')
+               exit = action.get('exit')
+               fullscreen = action.get('fullscreen')
+
+               if move:
+                   dx, dy = move
+                   player_x += dx
+                   player_y += dy
+
+               if exit:
+                   raise SystemExit()
+
+               if fullscreen:
+                   context.sdl_window.fullscreen = not context.sdl_window.fullscreen
            ...
{{</ highlight >}}
{{</ diff-tab >}}
{{< original-tab >}}
<pre>            ...
            for event in tcod.event.wait():
                if isinstance(event, tcod.event.Quit):
                    raise SystemExit()
                <span class="crossed-out-text">if isinstance(event, tcod.event.KeyDown):
                    if event.sym == tcod.event.KeySym.ESCAPE:
                        raise SystemExit()</span>
                <span class="new-text">action = handle_keys(event)

                move = action.get('move')
                exit = action.get('exit')
                fullscreen = action.get('fullscreen')

                if move:
                    dx, dy = move
                    player_x += dx
                    player_y += dy

                if exit:
                    raise SystemExit()

                if fullscreen:
                    context.sdl_window.fullscreen = not context.sdl_window.fullscreen</span>
            ...</pre>
{{</ original-tab >}}
{{</ codetab >}}

Note: I'll denote lines to delete in red. So in this case, remove the
inline key-check block and replace it with the calls to `handle_keys`.

Also be sure to import the `handle_keys` function at the top of
`engine.py`.

{{< codetab >}}
{{< diff-tab >}}
{{< highlight diff >}}
import tcod

+from input_handlers import handle_keys
{{</ highlight >}}
{{</ diff-tab >}}
{{< original-tab >}}
    <pre>import tcod

<span class="new-text">from input_handlers import handle_keys</span></pre>
{{</ original-tab >}}
{{</ codetab >}}

Hopefully now the dictionary madness in `handle_keys` makes a little
more sense. We're capturing the return value of `handle_keys` in the
variable `action` (which should be a dictionary, no matter what we
pressed), and checking what keys are inside it. If it contains a key
called 'move', then we know to look for the (x, y) coordinates. If it
contains 'exit', then we know we need to exit the game.

Try running the engine.py file now. You should be able to move around.
Exciting!

One last thing before we move on. Take a look at our drawing functions.
Notice how we're drawing directly to `root_console`? Rather than doing
that, we'll want to use an offscreen console, then 'blit' (copy) it to
the root console before presenting. This will make it easier to manage
multiple consoles when we get to the GUI portion of this series.

Modify the `engine.py` file like this:

{{< codetab >}}
{{< diff-tab >}}
{{< highlight diff >}}
        ...
        root_console = tcod.console.Console(screen_width, screen_height, order='F')
+       con = tcod.console.Console(screen_width, screen_height, order='F')

        while True:
-           root_console.print(player_x, player_y, '@', fg=(255, 255, 255))
+           con.print(player_x, player_y, '@', fg=(255, 255, 255))
+           con.blit(dest=root_console)
            context.present(root_console)
-           root_console.clear()
+           con.clear()
        ...
{{</ highlight >}}
{{</ diff-tab >}}
{{< original-tab >}}
<pre>        ...
        root_console = tcod.console.Console(screen_width, screen_height, order='F')
        <span class="new-text">con = tcod.console.Console(screen_width, screen_height, order='F')</span>

        while True:
            <span class="crossed-out-text">root_console.print(player_x, player_y, '@', fg=(255, 255, 255))</span>
            <span class="new-text">con.print(player_x, player_y, '@', fg=(255, 255, 255))
            con.blit(dest=root_console)</span>
            context.present(root_console)
            <span class="crossed-out-text">root_console.clear()</span>
            <span class="new-text">con.clear()</span>
        ...</pre>
{{</ original-tab >}}
{{</ codetab >}}

`con.blit(dest=root_console)` is used to copy `con` to the root console,
which is then presented to the screen by `context.present(root_console)`.

That wraps up part one of this tutorial\! If you're using git or some
other form of version control (and I recommend you do), commit your
changes now.

If you want to see the code so far in its entirety, [click
here](https://github.com/TStand90/roguelike_tutorial_revised/tree/part1).
The files you'll want to check are `engine.py` and `input_handlers.py`

[Click here to move on to the next part of this
tutorial.](/tutorials/tcod/2019/part-2)

<script src="/js/codetabs.js"></script>
