---
title: "Part 7 - Creating the Interface"
date: 2019-03-30T09:33:53-07:00
draft: false
aliases: /tutorials/tcod/part-7
---

Our game is looking more and more playable by the chapter, but before we
move forward with the gameplay, we ought to take a moment to focus on
how the project *looks*. Despite what roguelike traditionalists may tell
you, a good UI goes a long way.

Let's start by fixing up our HP section. With not all that much code, we
can add a neat little health bar, that will tell the player how much
health is remaining before death. We'll start by adding some needed
variables in `engine.py`:

{{< codetab >}} {{< diff-tab >}} {{< highlight diff >}}
    ...
    screen_height = 50

+   bar_width = 20
+   panel_height = 7
+   panel_y = screen_height - panel_height

    map_width = 80
-   map_height = 45
+   map_height = 43
    ...
    con = tcod.console.Console(screen_width, screen_height, order='F')
+   panel = tcod.console.Console(screen_width, panel_height, order='F')
{{</ highlight >}}
{{</ diff-tab >}}
{{< original-tab >}}
<pre>    ...
    screen_height = 50

    <span class="new-text">bar_width = 20
    panel_height = 7
    panel_y = screen_height - panel_height</span>

    map_width = 80
    <span class="crossed-out-text">map_height = 45</span>
    <span class="new-text">map_height = 43</span>
    ...
    con = tcod.console.Console(screen_width, screen_height, order='F')
    <span class="new-text">panel = tcod.console.Console(screen_width, panel_height, order='F')</span></pre>
{{</ original-tab >}}
{{</ codetab >}}

We're creating a new console, called `panel`, which will hold our HP bar
and message log. We also modified the map height while we were at it, to
give the HP bar and (soon to come) message log more room.

Now we'll want a function that draws the health bar, or any other bar we
desire. You might want to add a Mana or Stamina bar later on, so it's
best that we make this function as reusable as possible. Put the
following in `render_functions.py`, right under the `RenderOrder` enum,
but above
    `render_all`.

{{< highlight py3 >}}
def render_bar(panel, x, y, total_width, name, value, maximum, bar_color, back_color):
    bar_width = int(float(value) / maximum * total_width)

    panel.bg[x:x + total_width, y] = back_color
    if bar_width > 0:
        panel.bg[x:x + bar_width, y] = bar_color

    panel.print(int(x + total_width / 2), y, '{0}: {1}/{2}'.format(name, value, maximum),
                fg=(255, 255, 255), alignment=tcod.CENTER)
{{</ highlight >}}

Now let's use this function in `render_all`. Remove the HP indicator we
had put in before, and put the code for the stats panel at the end of
the
    function.

{{< codetab >}} {{< diff-tab >}} {{< highlight diff >}}
-def render_all(con, root_console, entities, player, game_map, fov_map, fov_recompute, screen_width, screen_height, colors):
+def render_all(con, root_console, panel, entities, player, game_map, fov_map, fov_recompute, screen_width, screen_height,
+              bar_width, panel_height, panel_y, colors):
            ...
-   con.print(1, screen_height - 2, 'HP: {0:02}/{1:02}'.format(player.fighter.hp, player.fighter.max_hp),
-             fg=(255, 255, 255))

    con.blit(dest=root_console)

+   panel.clear()
+
+   render_bar(panel, 1, 1, bar_width, 'HP', player.fighter.hp, player.fighter.max_hp,
+              (255, 114, 114), (127, 0, 0))
+
+   panel.blit(dest=root_console, dest_x=0, dest_y=panel_y)
{{</ highlight >}}
{{</ diff-tab >}}
{{< original-tab >}}
<pre><span class="crossed-out-text">def render_all(con, root_console, entities, player, game_map, fov_map, fov_recompute, screen_width, screen_height, colors):</span>
<span class="new-text">def render_all(con, root_console, panel, entities, player, game_map, fov_map, fov_recompute, screen_width, screen_height,
               bar_width, panel_height, panel_y, colors):</span>
            ...
    <span class="crossed-out-text">con.print(1, screen_height - 2, 'HP: {0:02}/{1:02}'.format(player.fighter.hp, player.fighter.max_hp),</span>
              <span class="crossed-out-text">fg=(255, 255, 255))</span>

    con.blit(dest=root_console)

    <span class="new-text">panel.clear()

    render_bar(panel, 1, 1, bar_width, 'HP', player.fighter.hp, player.fighter.max_hp,
               (255, 114, 114), (127, 0, 0))

    panel.blit(dest=root_console, dest_x=0, dest_y=panel_y)</span></pre>
{{</ original-tab >}}
{{</ codetab >}}

Be sure to update the call to `render_all` in
`engine.py`:

{{< codetab >}} {{< diff-tab >}} {{< highlight diff >}}
-       render_all(con, root_console, entities, player, game_map, fov_map, fov_recompute, screen_width, screen_height, colors)
+       render_all(con, root_console, panel, entities, player, game_map, fov_map, fov_recompute, screen_width, screen_height,
+                  bar_width, panel_height, panel_y, colors)
{{</ highlight >}}
{{</ diff-tab >}}
{{< original-tab >}}
<pre>        <span class="crossed-out-text">render_all(con, root_console, entities, player, game_map, fov_map, fov_recompute, screen_width, screen_height, colors)</span>
        <span class="new-text">render_all(con, root_console, panel, entities, player, game_map, fov_map, fov_recompute, screen_width, screen_height,
                   bar_width, panel_height, panel_y, colors)</span></pre>
{{</ original-tab >}}
{{</ codetab >}}

Now we've got a nice looking HP bar at the bottom of the screen. It'll
decrease when the player takes damage, and increase when we heal (that's
coming next chapter).

Let's keep things going, and create our message log. Add the following
variables to `engine.py` to start:

{{< codetab >}} {{< diff-tab >}} {{< highlight diff >}}
    ...
    panel_y = screen_height - panel_height

+   message_x = bar_width + 2
+   message_width = screen_width - bar_width - 2
+   message_height = panel_height - 1

    map_width = 80
    ...
{{</ highlight >}}
{{</ diff-tab >}}
{{< original-tab >}}
<pre>    ...
    panel_y = screen_height - panel_height

    <span class="new-text">message_x = bar_width + 2
    message_width = screen_width - bar_width - 2
    message_height = panel_height - 1</span>

    map_width = 80
    ...</pre>
{{</ original-tab >}}
{{</ codetab >}}

To implement the message log, we'll need two classes: one for the log,
and one for the messages inside it. Start by creating a new file, called
`game_messages.py`. Put the following code inside it:

{{< highlight py3 >}}
import textwrap


class Message:
    def __init__(self, text, color=(255, 255, 255)):
        self.text = text
        self.color = color


class MessageLog:
    def __init__(self, x, width, height):
        self.messages = []
        self.x = x
        self.width = width
        self.height = height

    def add_message(self, message):
        # Split the message if necessary, among multiple lines
        new_msg_lines = textwrap.wrap(message.text, self.width)

        for line in new_msg_lines:
            # If the buffer is full, remove the first line to make room for the new one
            if len(self.messages) == self.height:
                del self.messages[0]

            # Add the new line as a Message object, with the text and the color
            self.messages.append(Message(line, message.color))
{{< /highlight >}}

That's a lot to take in at once, so let's go through it.

`Message` is pretty simple. We store the message text and the color to
draw it with. You can opt to not pass a color, in which case, white is
used by default.

The `MessageLog` is the more interesting class. It holds a list of
messages (the `Message` class), holds its x coordinate (for
convenience), and its width and height. Width and height are useful so
that we'll know when we need to cut off the top messages (the message
log will "scroll" as new messages come up).

In the `add_message` method, we're splitting up the message text into
multiple lines if needed, using the `textwrap.wrap` function. We then
check if the message log is at its capacity, and if so, we delete the
top lines. Finally, we append the new message.

Let's start putting this new message log into place. Add a new message
log to `engine.py`:

{{< codetab >}} {{< diff-tab >}} {{< highlight diff >}}
    fov_map = initialize_fov(game_map)

+   message_log = MessageLog(message_x, message_width, message_height)
+   mouse_pos = (0, 0)
{{</ highlight >}}
{{</ diff-tab >}}
{{< original-tab >}}
<pre>    fov_map = initialize_fov(game_map)

    <span class="new-text">message_log = MessageLog(message_x, message_width, message_height)
    mouse_pos = (0, 0)</span></pre>
{{</ original-tab >}}
{{</ codetab >}}

Remember to import the `MessageLog` at the top as well:

{{< codetab >}} {{< diff-tab >}} {{< highlight diff >}}
from fov_functions import initialize_fov, recompute_fov
+from game_messages import MessageLog
from game_states import GameStates
{{</ highlight >}}
{{</ diff-tab >}}
{{< original-tab >}}
<pre>from fov_functions import initialize_fov, recompute_fov
<span class="new-text">from game_messages import MessageLog</span>
from game_states import GameStates</pre>
{{</ original-tab >}}
{{</ codetab >}}

With our message log in place, let's go through the project and remove
all the `print` statements, replacing them with the message log.

Let's start with the death functions. In `death_functions.py`:

{{< codetab >}} {{< diff-tab >}} {{< highlight diff >}}
+from game_messages import Message

from game_states import GameStates

from render_functions import RenderOrder


def kill_player(player):
    player.char = '%'
    player.color = (139, 0, 0)

-   return 'You died!', GameStates.PLAYER_DEAD
+   return Message('You died!', (255, 0, 0)), GameStates.PLAYER_DEAD


def kill_monster(monster):
-   death_message = '{0} is dead!'.format(monster.name.capitalize())
+   death_message = Message('{0} is dead!'.format(monster.name.capitalize()), (255, 127, 0))
    ...
{{</ highlight >}}
{{</ diff-tab >}}
{{< original-tab >}}
<pre><span class="new-text">from game_messages import Message</span>

from game_states import GameStates

from render_functions import RenderOrder


def kill_player(player):
    player.char = '%'
    player.color = (139, 0, 0)

    <span class="crossed-out-text">return 'You died!', GameStates.PLAYER_DEAD</span>
    <span class="new-text">return Message('You died!', (255, 0, 0)), GameStates.PLAYER_DEAD</span>


def kill_monster(monster):
    <span class="crossed-out-text">death_message = '{0} is dead!'.format(monster.name.capitalize())</span>
    <span class="new-text">death_message = Message('{0} is dead!'.format(monster.name.capitalize()), (255, 127, 0))</span>
    ...</pre>
{{</ original-tab >}}
{{</ codetab >}}

Then, back in `engine.py`, replace the corresponding `print` statements
like this:

{{< codetab >}} {{< diff-tab >}} {{< highlight diff >}}
             ...
            (In the player's results loop)
            ...
            if dead_entity:
                if dead_entity == player:
                    message, game_state = kill_player(dead_entity)
                else:
                    message = kill_monster(dead_entity)

-               print(message)
+               message_log.add_message(message)
            ...
            (In the enemy results loop)
            ...
                        if dead_entity:
                            if dead_entity == player:
                                message, game_state = kill_player(dead_entity)
                            else:
                                message = kill_monster(dead_entity)

-                           print(message)
+                           message_log.add_message(message)
                            ...
{{</ highlight >}}
{{</ diff-tab >}}
{{< original-tab >}}
<pre>             ...
            (In the player's results loop)
            ...
            if dead_entity:
                if dead_entity == player:
                    message, game_state = kill_player(dead_entity)
                else:
                    message = kill_monster(dead_entity)

                <span class="crossed-out-text">print(message)</span>
                <span class="new-text">message_log.add_message(message)</span>
            ...
            (In the enemy results loop)
            ...
                        if dead_entity:
                            if dead_entity == player:
                                message, game_state = kill_player(dead_entity)
                            else:
                                message = kill_monster(dead_entity)

                            <span class="crossed-out-text">print(message)</span>
                            <span class="new-text">message_log.add_message(message)</span>
                            ...</pre>
{{</ original-tab >}}
{{</ codetab >}}

Now for our action messages. In `fighter.py`:

{{< codetab >}} {{< diff-tab >}} {{< highlight diff >}}
        ...
        if damage > 0:
-           results.append({'message': '{0} attacks {1} for {2} hit points.'.format(self.owner.name.capitalize(),
-                                                                                   target.name, str(damage))})
+           results.append({'message': Message('{0} attacks {1} for {2} hit points.'.format(
+               self.owner.name.capitalize(), target.name, str(damage)), (255, 255, 255))})
            results.extend(target.fighter.take_damage(damage))
        else:
-           results.append({'message': '{0} attacks {1} but does no damage.'.format(self.owner.name.capitalize(),
-                                                                                   target.name)})
+           results.append({'message': Message('{0} attacks {1} but does no damage.'.format(
+               self.owner.name.capitalize(), target.name), (255, 255, 255))})

        return results
{{</ highlight >}}
{{</ diff-tab >}}
{{< original-tab >}}
<pre>        ...
        if damage > 0:
            <span class="crossed-out-text">results.append({'message': '{0} attacks {1} for {2} hit points.'.format(self.owner.name.capitalize(),</span>
                                                                                    <span class="crossed-out-text">target.name, str(damage))})</span>
            <span class="new-text">results.append({'message': Message('{0} attacks {1} for {2} hit points.'.format(
                self.owner.name.capitalize(), target.name, str(damage)), (255, 255, 255))})</span>
            results.extend(target.fighter.take_damage(damage))
        else:
            <span class="crossed-out-text">results.append({'message': '{0} attacks {1} but does no damage.'.format(self.owner.name.capitalize(),</span>
                                                                                    <span class="crossed-out-text">target.name)})</span>
            <span class="new-text">results.append({'message': Message('{0} attacks {1} but does no damage.'.format(
                self.owner.name.capitalize(), target.name), (255, 255, 255))})</span>

        return results</pre>
{{</ original-tab >}}
{{</ codetab >}}

You'll need to import `Message` for this to work:

{{< codetab >}} {{< diff-tab >}} {{< highlight diff >}}
+from game_messages import Message


class Fighter:
    ...
{{</ highlight >}}
{{</ diff-tab >}}
{{< original-tab >}}
<pre><span class="new-text">from game_messages import Message</span>


class Fighter:
    ...</pre>
{{</ original-tab >}}
{{</ codetab >}}

And in `engine.py`:

{{< codetab >}} {{< diff-tab >}} {{< highlight diff >}}
            ...
            (In the player's results loop)
            ...
            if message:
-               print(message)
+               message_log.add_message(message)
            ...
            (In the enemy results loop)
            ...
                        if message:
-                           print(message)
+                           message_log.add_message(message)
{{</ highlight >}}
{{</ diff-tab >}}
{{< original-tab >}}
<pre>            ...
            (In the player's results loop)
            ...
            if message:
                <span class="crossed-out-text">print(message)</span>
                <span class="new-text">message_log.add_message(message)</span>
            ...
            (In the enemy results loop)
            ...
                        if message:
                            <span class="crossed-out-text">print(message)</span>
                            <span class="new-text">message_log.add_message(message)</span></pre>
{{</ original-tab >}}
{{</ codetab >}}

Great, now we're adding all the messages to the log. However, nothing
shows up yet. Let's modify `render_all` to display the message log we've
created.

{{< codetab >}} {{< diff-tab >}} {{< highlight diff >}}
-def render_all(con, root_console, panel, entities, player, game_map, fov_map, fov_recompute, screen_width, screen_height,
-              bar_width, panel_height, panel_y, colors):
+def render_all(con, root_console, panel, entities, player, game_map, fov_map, fov_recompute, message_log, screen_width,
+              screen_height, bar_width, panel_height, panel_y, colors):
    ...
    panel.clear()

+   # Print the game messages, one line at a time
+   y = 1
+   for message in message_log.messages:
+       panel.print(message_log.x, y, message.text, fg=message.color)
+       y += 1
    ...
{{</ highlight >}}
{{</ diff-tab >}}
{{< original-tab >}}
<pre><span class="crossed-out-text">def render_all(con, root_console, panel, entities, player, game_map, fov_map, fov_recompute, screen_width, screen_height,</span>
               <span class="crossed-out-text">bar_width, panel_height, panel_y, colors):</span>
<span class="new-text">def render_all(con, root_console, panel, entities, player, game_map, fov_map, fov_recompute, message_log, screen_width,
               screen_height, bar_width, panel_height, panel_y, colors):</span>
    ...
    panel.clear()

    <span class="new-text"># Print the game messages, one line at a time
    y = 1
    for message in message_log.messages:
        panel.print(message_log.x, y, message.text, fg=message.color)
        y += 1</span>
    ...</pre>
{{</ original-tab >}}
{{</ codetab >}}

Then modify the call to `render_all` in `engine.py` to include the
message
log:

{{< codetab >}} {{< diff-tab >}} {{< highlight diff >}}
-       render_all(con, root_console, panel, entities, player, game_map, fov_map, fov_recompute, screen_width,
-                  screen_height, bar_width, panel_height, panel_y, colors)
+       render_all(con, root_console, panel, entities, player, game_map, fov_map, fov_recompute, message_log,
+                  screen_width, screen_height, bar_width, panel_height, panel_y, colors)
{{</ highlight >}}
{{</ diff-tab >}}
{{< original-tab >}}
<pre>        <span class="crossed-out-text">render_all(con, root_console, panel, entities, player, game_map, fov_map, fov_recompute, screen_width,</span>
                   <span class="crossed-out-text">screen_height, bar_width, panel_height, panel_y, colors)</span>
        <span class="new-text">render_all(con, root_console, panel, entities, player, game_map, fov_map, fov_recompute, message_log,
                   screen_width, screen_height, bar_width, panel_height, panel_y, colors)</span></pre>
{{</ original-tab >}}
{{</ codetab >}}

Run the project now. All our previous printed statements should now
appear in a scrolling message log. From here on out, we won't be doing
any more `print` statements, we'll just add everything to our message
log.

What's next? How about a little mouse-driven action? Our game is only
orcs and trolls right now, but perhaps someday it will have dozens
(hundreds?) of different monster and item types. It would be nice if we
could see what they are by moving our mouse over them.

The `tcod.event.wait()` loop already receives all events including mouse
movement. We just need to handle `tcod.event.MouseMotion` events to track
the current tile position. Update the event loop in `engine.py`:

{{< codetab >}} {{< diff-tab >}} {{< highlight diff >}}
            for event in tcod.event.wait():
+               if isinstance(event, tcod.event.MouseMotion):
+                   mouse_pos = (event.tile.x, event.tile.y)
                action = handle_keys(event)
{{</ highlight >}}
{{</ diff-tab >}}
{{< original-tab >}}
<pre>            for event in tcod.event.wait():
                <span class="new-text">if isinstance(event, tcod.event.MouseMotion):
                    mouse_pos = (event.tile.x, event.tile.y)</span>
                action = handle_keys(event)</pre>
{{</ original-tab >}}
{{</ codetab >}}

Put the following function in `render_functions.py`, above `render_bar`:

{{< codetab >}} {{< diff-tab >}} {{< highlight diff >}}
+def get_names_under_mouse(mouse_pos, entities, fov_map):
+   (x, y) = mouse_pos

+   names = [entity.name for entity in entities
+            if entity.x == x and entity.y == y and fov_map[entity.x, entity.y]]
+   names = ', '.join(names)

+   return names.capitalize()


def render_bar(panel, x, y, total_width, name, value, maximum, bar_color, back_color):
    ...
{{</ highlight >}}
{{</ diff-tab >}}
{{< original-tab >}}
<pre><span class="new-text">def get_names_under_mouse(mouse_pos, entities, fov_map):
    (x, y) = mouse_pos

    names = [entity.name for entity in entities
             if entity.x == x and entity.y == y and fov_map[entity.x, entity.y]]
    names = ', '.join(names)

    return names.capitalize()</span>


def render_bar(panel, x, y, total_width, name, value, maximum, bar_color, back_color):
    ...</pre>
{{</ original-tab >}}
{{</ codetab >}}

Now we'll once again modify our `render_all` function (that sure has had
a lot of changes this chapter, hasn't it?) to account for the mouse and
take advantage of our new
    function.

{{< codetab >}} {{< diff-tab >}} {{< highlight diff >}}
-def render_all(con, root_console, panel, entities, player, game_map, fov_map, fov_recompute, message_log, screen_width,
-              screen_height, bar_width, panel_height, panel_y, colors):
+def render_all(con, root_console, panel, entities, player, game_map, fov_map, fov_recompute, message_log, screen_width,
+              screen_height, bar_width, panel_height, panel_y, mouse_pos, colors):
    ...
    render_bar(panel, 1, 1, bar_width, 'HP', player.fighter.hp, player.fighter.max_hp,
               (255, 114, 114), (127, 0, 0))

+   panel.print(1, 0, get_names_under_mouse(mouse_pos, entities, fov_map), fg=(191, 191, 191))

    panel.blit(dest=root_console, dest_x=0, dest_y=panel_y)
{{</ highlight >}}
{{</ diff-tab >}}
{{< original-tab >}}
<pre><span class="crossed-out-text">def render_all(con, root_console, panel, entities, player, game_map, fov_map, fov_recompute, message_log, screen_width,</span>
               <span class="crossed-out-text">screen_height, bar_width, panel_height, panel_y, colors):</span>
<span class="new-text">def render_all(con, root_console, panel, entities, player, game_map, fov_map, fov_recompute, message_log, screen_width,
               screen_height, bar_width, panel_height, panel_y, mouse_pos, colors):</span>
    ...
    render_bar(panel, 1, 1, bar_width, 'HP', player.fighter.hp, player.fighter.max_hp,
               (255, 114, 114), (127, 0, 0))

    <span class="new-text">panel.print(1, 0, get_names_under_mouse(mouse_pos, entities, fov_map), fg=(191, 191, 191))</span>

    panel.blit(dest=root_console, dest_x=0, dest_y=panel_y)</pre>
{{</ original-tab >}}
{{</ codetab >}}

And, of course, we'll need to modify the call to `render_all` in
`engine.py` to match our new
definition.

{{< codetab >}} {{< diff-tab >}} {{< highlight diff >}}
-       render_all(con, root_console, panel, entities, player, game_map, fov_map, fov_recompute, message_log,
-                  screen_width, screen_height, bar_width, panel_height, panel_y, colors)
+       render_all(con, root_console, panel, entities, player, game_map, fov_map, fov_recompute, message_log,
+                  screen_width, screen_height, bar_width, panel_height, panel_y, mouse_pos, colors)
{{</ highlight >}}
{{</ diff-tab >}}
{{< original-tab >}}
<pre>        <span class="crossed-out-text">render_all(con, root_console, panel, entities, player, game_map, fov_map, fov_recompute, message_log,</span>
                   <span class="crossed-out-text">screen_width, screen_height, bar_width, panel_height, panel_y, colors)</span>
        <span class="new-text">render_all(con, root_console, panel, entities, player, game_map, fov_map, fov_recompute, message_log,
                   screen_width, screen_height, bar_width, panel_height, panel_y, mouse_pos, colors)</span></pre>
{{</ original-tab >}}
{{</ codetab >}}

Our game is now looking much, much better. If you ever intend for your
game to be played by more people than just yourself (it's okay if you
don't\!) then changes like these will be of paramount importance to your
project.

If you want to see the code so far in its entirety, [click
here](https://github.com/TStand90/roguelike_tutorial_revised/tree/part7).

[Click here to move on to the next part of this
tutorial.](/tutorials/tcod/2019/part-8)

<script src="/js/codetabs.js"></script>
