Como copiar elemento:

Abrir dev tools haciendo click derecho en el juego y "inspect".  Buscar el elemento con el nombre que digan las instrucciones y click derecho -> copiar elemento

## Monte (trilero)
Pegar en dev tools -> consola tras cada mano (en la última no sirve, hay truco ;) )

```(function(factor) {
  let target = shellMonteGridState.revealPos;
shellMonteGridState.swaps.forEach(([a, b]) => {
  if (target === a) target = b;
  else if (target === b) target = a;
});
console.log("Posición ganadora final:", target);


```
Izquierda 0, centro 1, derecha 2

## Knight round

<img width="702" height="677" alt="image" src="https://github.com/user-attachments/assets/32861dbf-34c7-40b0-9b3b-09c0eba64afa" />



## Pares 
pegar en dev tools. Si falla, recargar

```
async function runPairsAutoSolver() {
  // 1) Instala los hooks de memoria si aún no están activos
  if (!window.__pairsMemoryInstalled) {
    window.__pairsMemory = {};
    const key = (r, c) => r + "," + c;

    const origReveal = revealPairsCell;
    revealPairsCell = function (row, col, symbol) {
      window.__pairsMemory[key(row, col)] = symbol;
      return origReveal(row, col, symbol);
    };

    const origMatched = markPairsCellMatched;
    markPairsCellMatched = function (row, col, symbol) {
      window.__pairsMemory[key(row, col)] = symbol;
      return origMatched(row, col, symbol);
    };

    window.__pairsMemoryInstalled = true;
    console.log("Memoria instalada.");
  }

  const wait = (ms) => new Promise((r) => setTimeout(r, ms));
  const waitUnlocked = async () => {
    while (pairsGridState.locked) await wait(80);
  };
  const click = (r, c) => pairsCellEl(r, c)?.click();

  function allCells() {
    return [...document.querySelectorAll(".pairs-cell:not(.central)")].map((el) => [
      +el.dataset.r,
      +el.dataset.c,
    ]);
  }
  function isMatched(r, c) {
    return pairsCellEl(r, c)?.classList.contains("matched");
  }
  function knownPair() {
    const bySymbol = {};
    for (const k in window.__pairsMemory) {
      const [r, c] = k.split(",").map(Number);
      if (isMatched(r, c)) continue;
      const sym = window.__pairsMemory[k];
      (bySymbol[sym] ||= []).push([r, c]);
      if (bySymbol[sym].length >= 2) return bySymbol[sym].slice(0, 2);
    }
    return null;
  }
  function unexploredCells() {
    return allCells().filter(
      ([r, c]) => !isMatched(r, c) && window.__pairsMemory[r + "," + c] === undefined
    );
  }

  let totalPairs = 0;
  const total = allCells().length;

  console.log(`Iniciando auto-resolución (${total} casillas, ${total / 2} pares)...`);

  while (true) {
    await waitUnlocked();

    const remaining = allCells().filter(([r, c]) => !isMatched(r, c));
    if (remaining.length === 0) {
      console.log(`Listo. ${totalPairs} pares resueltos.`);
      break;
    }

    let pair = knownPair();
    if (!pair) {
      const rem = unexploredCells();
      if (rem.length < 2) {
        // ya no quedan casillas nuevas por explorar pero el tablero no cerró: esperar y reintentar
        await wait(200);
        continue;
      }
      pair = [rem[0], rem[1]];
    }

    click(...pair[0]);
    await wait(220);
    await waitUnlocked();
    click(...pair[1]);
    await wait(220);
    await waitUnlocked();
    totalPairs++;
  }
}

runPairsAutoSolver();
```

## Piano

<img width="246" height="242" alt="image" src="https://github.com/user-attachments/assets/ff10db5c-fdf7-4347-ad81-5a06b6d3fc5b" />



## Concepto

The mechanics, from the code:

- **Goal:** accumulate 165,000 "G" (resources) before time runs out — that's the `CONCEPTO_VICTORY_GOAL`.
- **Core** (the diamond, center cell): click it for a flat resource gain that scales up the more you've earned overall (`conceptoCoreClickGainLocal`).
- **Generator** (the bird/pelican icon): passive income, base rate ~1.11/s.
- **Multiplier** (the cross-burst icon): boosts adjacent generators/reactors by +60% each.
- **Amplifier** (the line icon): boosts every producer in its full row _and_ column by ×2, stacking multiplicatively per amplifier.
- **Bank** (the square-with-dot icon): adds a flat +32% to _global_ income, no matter where it's placed.
- **Reactor** (the concentric-circle icon): stronger generator (1.55/s base) that also buffs neighboring reactors via an aura effect.

Each structure type unlocks at a lifetime-earnings threshold, and costs scale up the more of that type you've already built (~17% per copy for most types).

<img width="632" height="605" alt="image" src="https://github.com/user-attachments/assets/32f2d15c-4bf1-4a3e-b8bb-ae2608b4e1b8" />

Bottom line: with optimal play you can hit 165,000 in **about 5 minutes 45 seconds** of game time, using only ~14 manual clicks at the very start. I confirmed this by coding up the exact formulas from your file and running a search that, at every moment, picks whichever action (click, or buy X at position Y) minimizes total time to the goal.

**The one insight that matters most:** amplifiers don't care about adjacency — they boost their _entire row and entire column_, and multiple amps stack multiplicatively. So the winning move is to dedicate one row and one column entirely to amps rather than scattering them. Every producer in that row or column gets ×2 per amp; a producer sitting at the intersection gets _both_ bonuses at once. Multipliers (which only help direct neighbors) and reactors are comparatively weak — multipliers are just useful cheap filler before amps unlock.

**The build order the simulation converged on:**

1. **Bootstrap (0–60s):** Click the core about 14 times to scrape together 14 gold — just enough for your first generator. Then fill the rest of row 0 with generators (7 total) as gold trickles in, plus one more generator at (row 1, col 0).
2. **One multiplier (~65s):** Place it where it touches two existing generators at once.
3. **The amp phase (~120–325s, the slow part):** Build amps along row 1 and down column 0 — five each. This is where most of your time actually goes, since amp costs climb fast (~17% per amp). Stay patient here; nothing else beats building "just one more amp" even as they get pricey, because the payoff compounds. The generator sitting at the intersection (row 1, col 0) ends up getting boosted by _all ten_ amps at once.
4. **Bank avalanche (~325s on):** Once your rate has exploded past a few hundred per second, banks (flat +32% global multiplier each, gentle cost growth) become the best buy. Fill the block around the core with them.
5. **Mop-up (last couple seconds):** Rate is now in the thousands per second, so remaining cells fill almost instantly — whatever's cheapest, mostly generators.

One thing to skip: **reactors**. They unlock late (8,500 lifetime earned) and cost a lot, and for this specific 165k target the payback window is too short — amps and banks already capture the value by the time reactors are available.

Caveat: this is a near-optimal result from simulating your exact formulas with a greedy time-minimizing search, not a formally proven global optimum — a hand-crafted layout might shave off a few more seconds, but the core technique (stack amps on one shared row+column, exploit the intersection cell, switch to banks late, skip reactors) is the dominant lever by a wide margin.

## Slitherlink

https://www.noq.solutions/slitherlink
https://www.kakuro-online.com/slitherlink/

## Sudoku Battleship

https://logic-puzzles-online.com/tools/battleship-solver/

Para IA ()
```
Solve this 7x7 battleship sudoku game.

Ship lengths

1x4

1x3

3x2

3x1

Grid:


<div class="bs-layout"> COPIAR ELEMENTO
```


## gray scale


```
function waitForGreyUnlock(timeoutMs = 5000) {
  return new Promise((resolve, reject) => {
    const start = Date.now();
    const iv = setInterval(() => {
      if (!greyGridState || !greyGridState.locked) {
        clearInterval(iv);
        resolve();
      } else if (Date.now() - start > timeoutMs) {
        clearInterval(iv);
        reject(new Error("Timed out waiting for grey grid click result"));
      }
    }, 30);
  });
}

async function playGreyscale({ selector = ".grey-cell" } = {}) {
  let step = 0;

  while (true) {
    const cells = Array.from(
      document.querySelectorAll(`${selector}:not(.central):not(.done)`)
    );

    const withBrightness = cells
      .map((cell) => {
        const rgb = parseRgb(cell.style.backgroundColor);
        return rgb ? { cell, brightness: (rgb.r + rgb.g + rgb.b) / 3 } : null;
      })
      .filter(Boolean);

    if (withBrightness.length === 0) {
      console.log("No more cells to click. Done.");
      break;
    }

    withBrightness.sort((a, b) => b.brightness - a.brightness);
    const { cell, brightness } = withBrightness[0];
    const row = Number(cell.dataset.r);
    const col = Number(cell.dataset.c);

    step++;
    console.log(`Click ${step}: r=${row}, c=${col}, brightness=${brightness.toFixed(1)}`);

    // Wait for previous click to resolve, just in case
    if (greyGridState.locked) await waitForGreyUnlock();

    onGreyCellClick(cell, row, col);
    await waitForGreyUnlock();

    // Check whether that click was actually correct before continuing
    if (cell.classList.contains("fail-flash")) {
      console.error(`Click ${step} was WRONG (r=${row}, c=${col}). Stopping — order assumption is off.`);
      break;
    }
  }
}

// Run it:
playGreyscale();
```


## fruit ninja

pegar en dev tools -> consola con juego empezado
gana solo

```
(() => {
    const canvas = document.getElementById("fruitNinjaCanvas");
    if (!canvas || !fruitNinjaGridState) {
        console.log("Game not running.");
        return;
    }

    const rect = () => canvas.getBoundingClientRect();

    function fire(type, xNorm, yNorm, id = 1) {
        const r = rect();

        canvas.dispatchEvent(new PointerEvent(type, {
            bubbles: true,
            cancelable: true,
            pointerId: id,
            pointerType: "mouse",
            clientX: r.left + xNorm * r.width,
            clientY: r.top + yNorm * r.height,
            buttons: 1
        }));
    }

    function slash(x1, y1, x2, y2) {
        fire("pointerdown", x1, y1);

        const steps = 8;

        for (let i = 1; i <= steps; i++) {
            const t = i / steps;
            fire(
                "pointermove",
                x1 + (x2 - x1) * t,
                y1 + (y2 - y1) * t
            );
        }

        fire("pointerup", x2, y2);
    }

    const timer = setInterval(() => {

        if (!fruitNinjaGridState || fruitNinjaGridState.finished) {
            clearInterval(timer);
            return;
        }

        const elapsed = fruitNinjaElapsedMs();

        fruitNinjaGridState.schedule.forEach(entry => {

            if (entry.kind !== "fruit")
                return;

            const pos = fruitNinjaEntityPos(entry, elapsed);

            if (!pos)
                return;

            if (!fruitNinjaEntityActive(entry, elapsed))
                return;

            // Avoid slicing the same fruit twice
            const idx = fruitNinjaGridState.schedule.indexOf(entry);
            if (fruitNinjaGridState.sliceSet.has(idx))
                return;

            // Is a bomb close to our intended slash?
            let dangerous = false;

            for (const other of fruitNinjaGridState.schedule) {

                if (other.kind !== "bomb")
                    continue;

                const bombPos = fruitNinjaEntityPos(other, elapsed);

                if (!bombPos)
                    continue;

                if (!fruitNinjaEntityActive(other, elapsed))
                    continue;

                const dist = fruitNinjaDistToSegment(
                    bombPos.x,
                    bombPos.y,
                    pos.x - 0.03,
                    pos.y,
                    pos.x + 0.03,
                    pos.y
                );

                if (dist < fruitNinjaGridState.hitRadius * 1.5) {
                    dangerous = true;
                    break;
                }
            }

            if (dangerous)
                return;

            // Tiny horizontal slash
            slash(
                pos.x - 0.03,
                pos.y,
                pos.x + 0.03,
                pos.y
            );

        });

    }, 20);

    console.log("Auto slicer running.");
})();
```


## Binary


```
**Binary Grid Solver**

I'm playing a binary puzzle on a 7x7 grid. Here are the rules:

- Each cell is either Black (B) or White (W)
- Every row and every column must have exactly 4 of one and 3 of the other (the balance is fixed per puzzle — you may need to infer it from the clues)
- No three consecutive same colour in any row or column
- All rows must be unique
- All columns must be unique

I'll paste the raw HTML of the grid element from my browser dev tools. Parse it as follows:

- `class="binary-cell dot"` = fixed clue (cannot be changed)
- `class="binary-piece black"` inside a cell = Black
- `class="binary-piece white"` inside a cell = White
- A cell with no `binary-piece` span = empty

Solve the puzzle and give me:

1. The full solution grid (7 rows, using B and W)
2. Only the cells I need to fill in (empty cells in the original), listed as R#C# = B/W
3. If there are multiple solutions, tell me which cells are forced and which are ambiguous

Here is the grid HTML:  
`[PASTE HTML HERE]`
```

## Bridges


Claude

```
This is a Hashiwokakero ("Bridges") puzzle, not the card game. Each numbered 
island must be connected to other islands by a total number of bridges equal 
to its number. Rules:

1. Bridges run only horizontally or vertically — never diagonally.
2. A bridge connects exactly two islands, with no other island in between.
3. There can be at most 2 bridges between any given pair of islands.
4. Bridges cannot cross each other or pass through other islands.
5. All islands must end up connected into a single network (no isolated 
   groups).

Given the island positions and numbers below, solve the puzzle:
1. List each island with its (row, col) position and number.
2. Determine which pairs of islands are "adjacent" (same row or column, 
   nothing in between).
3. Work out how many bridges (0, 1, or 2) go on each possible edge so every 
   island's total matches its number.
4. Verify: no crossing bridges, and the whole graph is connected.
5. Present the final answer as a diagram (islands as circles with their 
   numbers, single lines for single bridges, double lines for double 
   bridges) rather than just a table.


Here is the puzzle:
[paste the bridge-board HTML or island list here]
```

## Sliding

1 - Solve in this site: https://jweilhammer.github.io/sliding-puzzle-solver/
2- Enter the solution in this code and paste in console

```
// === Sliding puzzle autoplayer ===
// Paste into the browser console while the puzzle is on screen.

// 1) Paste your move list here as the raw text you were given:
const MOVE_TEXT = `
1: DOWN
2: DOWN
3: LEFT
4: UP
5: RIGHT
... (paste the full list)
`;

// 2) Parse "N: DIRECTION" lines into an ordered array of directions
function parseMoves(text) {
  return text
    .split("\n")
    .map((line) => line.trim())
    .filter(Boolean)
    .map((line) => {
      const m = line.match(/^\d+:\s*(UP|DOWN|LEFT|RIGHT)$/i);
      if (!m) throw new Error("Bad move line: " + line);
      return m[1].toUpperCase();
    });
}

// Offset of the tile (relative to empty) that must be clicked for a given
// "empty moves in this direction" instruction.
const DIR_TO_TILE_OFFSET = {
  DOWN:  { dr:  1, dc:  0 }, // tile below empty slides up
  UP:    { dr: -1, dc:  0 }, // tile above empty slides down
  RIGHT: { dr:  0, dc:  1 }, // tile right of empty slides left
  LEFT:  { dr:  0, dc: -1 }, // tile left of empty slides right
};

function findEmpty() {
  const cellMap = slidingCellMap(slidingGridState.cells);
  const { empty } = slidingEmptyAndAdjacent(cellMap);
  return empty;
}

function waitForUnlock(timeoutMs = 5000) {
  return new Promise((resolve, reject) => {
    const start = Date.now();
    const iv = setInterval(() => {
      if (!slidingGridState || !slidingGridState.locked) {
        clearInterval(iv);
        resolve();
      } else if (Date.now() - start > timeoutMs) {
        clearInterval(iv);
        reject(new Error("Timed out waiting for click result"));
      }
    }, 30);
  });
}

async function playMoves(moves, { delayMs = 120 } = {}) {
  for (let i = 0; i < moves.length; i++) {
    const dir = moves[i];
    const empty = findEmpty();
    if (!empty) throw new Error("No empty cell found at move " + (i + 1));

    const off = DIR_TO_TILE_OFFSET[dir];
    const r = empty.r + off.dr;
    const c = empty.c + off.dc;

    if (r < 0 || r > 6 || c < 0 || c > 6) {
      throw new Error(`Move ${i + 1} (${dir}) goes out of bounds from empty (${empty.r},${empty.c})`);
    }

    // Wait until previous click resolved (locked === false)
    if (slidingGridState.locked) {
      await waitForUnlock();
    }

    console.log(`Move ${i + 1}: ${dir} -> clicking tile at (${r}, ${c})`);
    onSlidingCellClick(r, c);

    // small pacing delay before checking lock state again, plus wait for unlock
    await new Promise((res) => setTimeout(res, delayMs));
    await waitForUnlock();
  }
  console.log("All moves sent.");
}

// 3) Run it:
playMoves(parseMoves(MOVE_TEXT));
```



## Mastermind

✗,○,✓,□,△,★
https://nebupookins.github.io/JS-Mastermind-Solver/

## calcudoku

https://billabob.github.io/kenkensolver/kenkensolver.html

# fillomino

https://www.kakuro-online.com/fillomino/
