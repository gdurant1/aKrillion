# Wager Platformer

A web-based, Mario-style side-scrolling game where the player wagers money on every stretch of the course. A C++ backend calculates the probability of surviving each stretch, and an LLM character tries to convince the player to keep playing.

## Game Design

### Goal
Finish with as much money as possible. The player's money at exit is their score, and the player tries to beat the top score on the high score list. The game has no finish line.

### Rules
- The player starts with **$100**.
- The player makes a **wager at the start and at every checkpoint**. The **minimum wager is $1**, and the player may go **all-in** (wager their entire balance).
- Before each wager the player is shown the **odds**, which are the probability of surviving the next stretch (odds *for*, not odds *against*; wins against a total).
- After the wager, the server **rolls all 10 squares** of the stretch. The player then moves forward with the **arrow keys**, like a Mario game, and each square is revealed as the player steps onto it.
- Each square is a **floor**, a **block**, or a **pit**.

| Square | What happens |
|--------|--------------|
| Floor | Player moves forward onto the floor |
| Block | Player presses **Up** to jump onto the block; the block becomes the new floor |
| Pit | Player falls and dies |

- **Checkpoints:** every **10 spaces**. At each checkpoint the player receives their reward and makes a new wager with new odds.
- **Reward:** calculated like a casino, from the wager and the odds (see Payout below).
- **Death:** the player loses the money they bet and restarts from the last checkpoint. If they die before reaching the first checkpoint, they restart from the start.
- **Game over:** the player runs out of money.
- **Exit:** the player can only exit at a checkpoint. Their money becomes their score on the high score list.
- **Initials:** the player can enter their initials at the start of the game, for the high score list.
- **LLM character:** talks to the player and tries to convince them to continue playing.

### Controls and Visibility
- The player **cannot see what is in front of them** until they press **Up** or **Forward**.
- **Forward:** moves ahead one square onto a floor.
- **Up:** at a block, the player presses Up to jump onto it.
- If the player does not press Up at a block, they are stuck in front of it.

### Payout
Fair payout based on the survival probability `p`, with one optional house-edge constant:

```
payout = wager * (1 - edge) / p
profit = payout - wager
```

`edge` is a single constant in `config.h`. Setting it to 0 gives the fair payout.

### Undecided
- **LLM choice** (hosted API or local model).
- **LLM character's personality** (name, tone, and whether it may state the real odds).
- **Jumping over pits.** One proposed rule: Forward jumps onto a block, and Up jumps over the square directly in front of the player and lands on the following square, which would let the player jump over a pit. This conflicts with the current block rule (Up jumps onto a block), so it is not part of the game yet. Also undecided under this rule: what happens if the landing square is a pit or a block.

## Technical Design

### Stack
- **Frontend:** HTML, CSS, JavaScript (canvas)
- **Backend:** C++ (HTTP server)
- **LLM:** called from the C++ backend so the API key is never exposed in the browser
- **Visualization:** BRIDGES (C++), required for the assignment; used to visualize the data structures and algorithm results

### Where the course concepts are used
- **Recursion:** a square's probability is defined in terms of earlier squares (for example, a pit is less likely right after a pit).
- **Dynamic programming:** results of those recursive calculations are cached so each square is computed only once, and the survival probability of a stretch is built from them.
- **LLM call:** the backend sends the game state and computed probabilities to the LLM and returns the character's dialogue.
- **BRIDGES:** visualizes the data structures and algorithm results behind the recursion and DP.

### Data Flow
1. `game.js` sends user input and game state to the C++ server through `api.js`.
2. `main.cpp` passes the state to `probability.cpp` and gets the survival odds back, which are shown before the wager. `probability.cpp` also sends its data structures and results to `bridges_view.cpp` for visualization with BRIDGES.
3. After the wager, the server rolls all 10 squares of the stretch and stores them. Only one square at a time is sent to the browser, so players cannot read the pits ahead of time.
4. `main.cpp` updates money, position, and checkpoint through `game_state.cpp`.
5. On key moments, `main.cpp` passes the state and probabilities to `llm_client.cpp`.
6. The server returns the result and dialogue to the browser, which displays them.

The browser talks to one server only. Probability and LLM are modules inside that server, not separate programs. Money, wager, and square outcomes live on the server so players cannot edit them in the browser.

### File Layout

```
wager-platformer/
├── README.md
├── frontend/
│   ├── index.html        # page structure, canvas, speech bubble
│   ├── style.css         # layout and styling
│   ├── game.js           # game loop, movement, drawing, keyboard input
│   └── api.js            # fetch calls to the C++ server
└── backend/
    ├── CMakeLists.txt    # build file (or Makefile)
    ├── main.cpp          # HTTP server and routes
    ├── probability.h     # probability calculator interface
    ├── probability.cpp   # recursion + dynamic programming
    ├── bridges_view.h    # BRIDGES visualization interface
    ├── bridges_view.cpp  # visualizes data structures and algorithm results with BRIDGES
    ├── game_state.h      # money, wager, position, checkpoint
    ├── game_state.cpp    # state updates, payout, game-over check
    ├── llm_client.h      # LLM client interface
    ├── llm_client.cpp    # prompt building and LLM API call
    └── config.h          # API key loading, model name, port, house edge
```

### Planned Routes

| Route | Purpose |
|-------|---------|
| `/start` | Begin a game with $100 and return the first odds |
| `/wager` | Submit a wager; the server rolls the 10-square stretch |
| `/step` | Move forward or jump; returns the square outcome and new state |
| `/dialogue` | Get LLM character dialogue |
| `/exit` | Exit at a checkpoint and finalize the score |
| `/scores` | Get the high score list |

### LLM Triggers
The LLM is called on key moments rather than every step:
- Before a wager
- After a death
- At a checkpoint (including when the exit option appears)
- When the player is ahead
- When the player tries to exit

Design notes: call the LLM asynchronously so the game never freezes, keep canned fallback lines in case the call fails, keep a short conversation history for consistency, and limit calls per player.

### Implementation Notes
- Store money as whole cents (integers) and round payouts to a cent so totals never drift.
- Set a minimum survival probability (for example, 0.01) so a very unlikely stretch cannot produce a huge payout.
- Store the high score list in a small file such as `scores.json`.

## BRIDGES Visualization

[BRIDGES](https://bridgesuncc.github.io/index.html) is a library for visualizing data structures and algorithms, with C++, Java, and Python versions. Using BRIDGES is required for the assignment. It is used to visualize the data structures and algorithm results from the C++ backend, while the HTML/CSS/JavaScript game stays as the player-facing game.

### What to visualize
The BRIDGES page lists arrays (including 2D), trees, and line charts, which map onto the game as follows:

| Game data | BRIDGES visual |
|-----------|----------------|
| DP table of cached probabilities | 2D array |
| Recursion behind a square's probability | Binary tree |
| Survival probability and the player's money over time | Line chart |

### Where it fits
- The visualization is called from the C++ backend, in or alongside `probability.cpp`, so it sees the same recursion and DP results that produce the odds.
- Planned module: `bridges_view.h` and `bridges_view.cpp` in `backend/`, so the visualization code stays out of the probability logic.
- The game's design, routes, and data flow do not change.

### Requirements to confirm
BRIDGES needs an account, an API key, and an internet connection from the machine that runs the C++ code. Check the BRIDGES documentation for the exact setup steps before building this.

## Team Split

The work is split among three team members: two parts for the backend and a third for the frontend.

### Part 1: Probability and Game State
**Files:** `probability.h`, `probability.cpp`, `game_state.h`, `game_state.cpp`

- Calculate the probability of a floor, block, or pit, and the survival probability of a 10-square stretch, using recursion and dynamic programming.
- Track the game state: money, wager, position, checkpoint, payout, and the game-over check.
- Provides the game state and the computed probabilities that Part 2 uses for dialogue.

### Part 2: LLM Client
**Files:** `llm_client.h`, `llm_client.cpp`

- Build the prompt from the game state and probabilities, call the LLM API, and return the character's dialogue text.
- Call the LLM asynchronously, keep canned fallback lines, keep a short conversation history, and limit calls per player (see LLM Triggers above).
- Depends on the undecided LLM choice and character personality.

### Part 3: Frontend
**Files:** `index.html`, `style.css`, `game.js`, `api.js`

- Draw the game and handle the game loop, character movement, and keyboard input.
- Show the speech bubble for the LLM character, and take the player's wager and initials.
- Talk to the C++ server through `api.js`.
- Completed by the third team member.

### Not Yet Assigned

**`main.cpp`**
- Starts the HTTP server and defines the routes: `/start`, `/wager`, `/step`, `/dialogue`, `/exit`, and `/scores`.
- Connects the parts: passes the state to `probability.cpp`, updates money and position through `game_state.cpp`, calls `llm_client.cpp` on key moments, and calls `bridges_view.cpp` to send the visualization data.
- It touches all three parts of the team split, so whoever owns it needs to coordinate closely with the other two backend members and the frontend member.

**`config.h`**
- Holds the settings: how the API key is loaded, the model name, the port, and the house-edge constant (0 gives the fair payout).

**`CMakeLists.txt`**
- The build file (or a Makefile) that compiles the backend.

**`bridges_view.h` and `bridges_view.cpp`**
- The BRIDGES visualization module. It receives the data structures and results from `probability.cpp`, such as the DP table, the recursion tree, and the survival probability and money over time, and sends them to BRIDGES.

### Where It Runs
The game will run on a GitHub Codespace or on a local server.