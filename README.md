# Wager Platformer

A web-based, Mario-style side-scrolling game where the player wagers money on every stretch of the course. A C++ backend calculates the probability of surviving each stretch, and an LLM character tries to convince the player to keep playing.

## Game Design

### Goal
Finish with as much money as possible. The player's money at exit is their score, and the player tries to beat the top score on the high score list. The game has no finish line.

### Rules
- The player starts with **$100**.
- The player makes a **wager at the start and at every checkpoint**. The wager limit is **$1**.
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
- **Death:** the player loses the money they bet and restarts from the last checkpoint.
- **Game over:** the player runs out of money.
- **Exit:** the player can only exit at a checkpoint. Their money becomes their score on the high score list.
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

Raised earlier and not yet answered:
- Whether blocks can stack (a block followed directly by another block).
- Whether the $1 wager limit is a minimum or a maximum, and whether the player can go all-in.
- Where the player restarts if they die before reaching the first checkpoint.
- Whether the player enters a name for the high score list.

## Technical Design

### Stack
- **Frontend:** HTML, CSS, JavaScript (canvas)
- **Backend:** C++ (HTTP server)
- **LLM:** called from the C++ backend so the API key is never exposed in the browser

### Where the course concepts are used
- **Recursion:** a square's probability is defined in terms of earlier squares (for example, a pit is less likely right after a pit).
- **Dynamic programming:** results of those recursive calculations are cached so each square is computed only once, and the survival probability of a stretch is built from them.
- **LLM call:** the backend sends the game state and computed probabilities to the LLM and returns the character's dialogue.

### Data Flow
1. `game.js` sends user input and game state to the C++ server through `api.js`.
2. `main.cpp` passes the state to `probability.cpp` and gets the survival odds back, which are shown before the wager.
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