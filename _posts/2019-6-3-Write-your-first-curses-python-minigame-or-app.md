---
layout: post
title: Write your first Python curses app or minigame!
---

Curses is a very useful Python package that allows for the creation of text-based user interfaces. While it doesn't produce the most visually beautiful results, it's simplicity is also it's strength as very few lines of code are required to create an interface, making this package very handy for quick projects.

Getting started with Python curses can be a little tricky though, especially since it isn't a very well documented package  (<a href="https://docs.python.org/3/howto/curses.html">their website</a> does offer some very useful information, but the format isn't very beginner friendly).

Nevertheless I think that when it comes to learning something like curses, the best thing is a **practical approach**.

To serve this purpose, I have prepared a small script that implements the famous mathematical game ***Chomp*** (for those of you who don't know what *Chomp* is, you can read about the it <a href="https://en.wikipedia.org/wiki/Chomp">here</a>).

## The rules

To make everyone's life simpler, I have summarized *Chomp*'s rules as a few bullet points

* There is a **chocolate bar** of $$m\times n$$ pieces
  * All pieces are fine, except the **lower left piece**, which **is poisonous** 
* The **objective** of the game is to **force your opponent to eat the poisonous piece**
  * The **players take turns** chosing which **piece to eat** 
  * When a **player** eats a piece, he **must also eat all the pieces** that are **further up and to the right**
    * E.g.: if the player eats the piece located at (2, 3) he must also eat all the blue colored pieces (see image)

![chocolate](/Users/marcel/Documents/GitHub/MarcelFerrari.github.io/images/posts/2019-6-3-Write-your-first-curses-python-minigame-or-app/chocolate.png)

## The script

Now that we understand how the game works, we can take a look at the full script:

```python
# Curses modules
import curses
from sys import argv
from curses import KEY_MOUSE
from random import randint
from time import time, sleep

# Get chocolate bar size and player names
players = {}
try:
    x = int(argv[1])
    y = int(argv[2])
    players[1] = str(argv[3])
    players[2] = str(argv[4])
except:
    print("Error: please supply a valid chocolate bar size and two valid player names.") 
    # E.g. "python3 chomp.py 15 5 Player1 Player2"
    exit(1)

# Init curses
main = curses.initscr()

# Check if window is big enough
window_y,window_x = main.getmaxyx()
if window_y <= y + 2 or window_x <= x + 2:
    print("Terminal must be at least " + str(x + 2) + "x" + str(y + 2) + ".")
    curses.endwin()
    exit(1)

# Create new curses window of size (x+2)*(y+2) to accomodate border
win = curses.newwin(y + 2, x + 2, 0, 0)
win.keypad(1)
curses.noecho()
curses.curs_set(0)
win.border(0)
win.nodelay(1)
curses.mousemask(1)

# Define useful functions
# Return list with all coordinates of chocolate bar pieces
def get_chocolate(y,x):
    coords = []
    for i in range(1, y + 1): # Go over all horizontal lines (border uses y = 0)
        for p in range(1, x + 1): # Go over every piece in line (border uses x = 0)
            coords.append((i, p)) # Curses works with (y,x) coordinates system
    return coords

# Print chocolate pieces
def print_chocolate(coords,y,x): # It is easier to go over all coordinates
    for i in range(1, y + 1):    # checking if (y,x) is a piece of chocolate
        for p in range(1, x + 1):
            if (i,p) == (y, 1):  # Bottom left corner is the poisonous piece
                win.addch(i, p, "*")
            elif (i,p) in coords:
                win.addch(i, p, '#')
            else:
                win.addch(i, p, ' ')

# Eat chocolate pieces
def eat(coords, x, click_y, click_x):
    for i in range(1, click_y + 1): # Go over rows from top to chocolate piece included (+1)
        for p in range(click_x, x + 1): # Go over columns from clicked point to right edge
            if (i,p) in coords:
                coords.pop(coords.index((i, p)))
    return coords

coords = get_chocolate(y,x) # Define coordinates list

player = 1 # Define player turn

moves = 0 # Define moves

game_start = time() # Start timer

# Start curses print loop
while True:
    win.border(0) # Set window border
    key = win.getch() # Listen for keys pressed or mouse clicks
    if key == ord('q'): # When 'q' is hit quit the game
        while True: # Ask for confirmation
            key = win.getch()
            win.addstr(0, 1, "| Quit? (y/n) |")
            if key == ord('n'): # Resume game
                break
            if key == ord('y'): # Close window and exit
                curses.endwin()
                exit(0)

    if key == KEY_MOUSE: # If mouse is clicked
        _, click_x, click_y, _, _ = curses.getmouse() # Get clicked coordinates
        # Poisonous piece cannot be chosen
        if (click_y,click_x) in coords and (click_y,click_x) != (y, 1):
            moves += 1 # Increase number of moves by one
            player = abs(player - 3) # Switch between 1 and 2 --> |1-3|=2 and |2-3|=1
            coords = eat(coords, x, click_y, click_x)

    print_chocolate(coords, y, x) # Print chocolate pieces

    if len(coords) == 1: # If only 1 piece is left, then it must be the poisonous one
        break # End curses loop

game_end = time() # End timer

game_time = game_end - game_start # Calculate game time

curses.endwin() # Close window

# Print results
print("----------------")
print(str(players[abs(player - 3)]) + " WINS!")
print("Number of moves: " + str(moves))
print("Game time: " + str(round(game_time, 3)) + "s")
print("----------------")

exit(0)

```



## Break-down

Let's now take a look at the main components of the script.

The `x` and `y` variables are used to define the chocolate bar size. The values for these two variables are passed as command line arguments to make everything easier.

A `players = {}` dictionary is used to keep track of which player is winning. Player number 1 is passed as the first name and player number 2 as the second. Using a dictionary is handy because we only need to keep track of a `player` number, which is either 1 or 2 (corresponding to player 1 and player 2).

```python
# Get chocolate bar size
players = {}
try:
    x = int(argv[1])
    y = int(argv[2])
    players[1] = str(argv[3])
    players[2] = str(argv[4])
except:
    print("Error: please supply a valid chocolate bar size and two valid player names.") 
    # E.g. "python3 chomp.py 15 5 Player1 Player2"
    exit(1)
```



We now initialize curses by calling the `curses.initscr()` function.

It is important to check if the window is large enough to fit the chocolate bar. To do this we call the `getmxyx` function, which returns the `window_y` and `window_x` values. Keep in mind that curses uses an **unorthodox coordinate** system, which places the **origin in the top left corner** (see image).

The values of `window_y` and `window_x` are compared to `y + 2` and `x + 2` to make space for a border that we will set later.

```python
# Init curses
main = curses.initscr()

# Check if window is big enough
window_y,window_x = main.getmaxyx()
if window_y <= y + 2 or window_x <= x + 2:
    print("Terminal must be at least " + str(x + 2) + "x" + str(y + 2) + ".")
    curses.endwin()
    exit(1)
```

![curses-graph](/Users/marcel/Documents/GitHub/MarcelFerrari.github.io/images/posts/2019-6-3-Write-your-first-curses-python-minigame-or-app/curses-graph.png)

*Curses coordinate system, source: <a href="http://www.whiz.se/tag/curses.html">whizfoo</a>*



A window is now created by calling the `curse.newwin()` function. The arguments passed are the size (mind that `y + 2` is entered before `x + 2` ) and the initial coordinates, which normally are `(0, 0)`.

```python
# Create new curses window of size (x+2)*(y+2) to accomodate border
win = curses.newwin(y + 2, x + 2, 0, 0)
win.keypad(1)
curses.noecho()
curses.curs_set(0)
win.border(0)
win.nodelay(1)
curses.mousemask(1)
```

