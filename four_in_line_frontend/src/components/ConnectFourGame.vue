<script setup lang="ts">
import { ref, computed } from "vue";

// Constants for game
const ROWS = 6;
const COLS = 7;
const EMPTY = 0;
const PLAYER1 = 1;
const PLAYER2 = 2;

// Color palette (from requirements)
const COLOR_ACCENT = "#FFD600";
const COLOR_PRIMARY = "#1976D2";
const COLOR_SECONDARY = "#388E3C";

// Symbols and names
const PLAYER_DETAILS = [
  {},
  { color: COLOR_PRIMARY, label: "Player 1" },
  { color: COLOR_SECONDARY, label: "Player 2" }
];

// Board state and control refs
const board = ref<number[][]>(Array.from({ length: ROWS }, () => Array(COLS).fill(EMPTY)));
const droppingRow = ref<number | null>(null);
const droppingCol = ref<number | null>(null);
const droppingPlayer = ref<number | null>(null);
const animating = ref(false);

const currentPlayer = ref<1 | 2>(PLAYER1);
const gameStatus = ref<"playing" | "win" | "draw">("playing");
const winner = ref<number | null>(null);

// Reset the game to empty
// PUBLIC_INTERFACE
function newGame() {
  board.value = Array.from({ length: ROWS }, () => Array(COLS).fill(EMPTY));
  currentPlayer.value = PLAYER1;
  gameStatus.value = "playing";
  winner.value = null;
  droppingRow.value = null;
  droppingCol.value = null;
  droppingPlayer.value = null;
  animating.value = false;
}

// Utility: Find the next empty spot in a column, from bottom up
function getAvailableRow(col: number) {
  for (let row = ROWS - 1; row >= 0; row--) {
    if (board.value[row][col] === EMPTY) return row;
  }
  return null;
}

// Handle user click to play disc
// PUBLIC_INTERFACE
async function playColumn(col: number) {
  if (animating.value || gameStatus.value !== "playing") return;

  const row = getAvailableRow(col);
  if (row === null) return; // column full

  // Animate disc drop: set dropping, wait, then commit
  droppingRow.value = 0;
  droppingCol.value = col;
  droppingPlayer.value = currentPlayer.value;
  animating.value = true;

  // Use timeouts for animation: simulate falling disc
  for (let r = 0; r <= row; r++) {
    droppingRow.value = r;
    await new Promise(res => setTimeout(res, 45));
  }

  // Place disc
  board.value[row][col] = currentPlayer.value;
  droppingRow.value = null;
  droppingCol.value = null;
  droppingPlayer.value = null;
  animating.value = false;

  // Check for win/draw
  if (checkWin(row, col, currentPlayer.value)) {
    winner.value = currentPlayer.value;
    gameStatus.value = "win";
  } else if (isBoardFull()) {
    gameStatus.value = "draw";
  } else {
    // Switch player
    currentPlayer.value = currentPlayer.value === PLAYER1 ? PLAYER2 : PLAYER1;
  }
}

// Win/draw detection
function isBoardFull() {
  return board.value.every(row => row.every(cell => cell !== EMPTY));
}
function checkWin(row: number, col: number, player: number) {
  // Directions: [deltaRow, deltaCol]
  const dirs = [
    [0, 1], [1, 0], [1, 1], [1, -1]
  ];
  for (const [dr, dc] of dirs) {
    let count = 1;
    // Check both directions
    for (let d = -1; d <= 1; d += 2) {
      let r = row + dr * d;
      let c = col + dc * d;
      while (r >= 0 && r < ROWS && c >= 0 && c < COLS && board.value[r][c] === player) {
        count++;
        r += dr * d;
        c += dc * d;
      }
    }
    if (count >= 4) return true;
  }
  return false;
}

 // For highlighting winning discs (UI): scan board if game won
const winningCoords = computed<[number, number][]>(() => {
  if (gameStatus.value !== "win" || winner.value === null) return [] as [number, number][];
  // try every cell; if part of winning four, return all such coords
  const winPlayer = winner.value;
  const coords: Array<[number, number]> = [];

  function getLine(r: number, c: number, dr: number, dc: number): Array<[number, number]> {
    const line = [];
    for (let i = 0; i < 4; i++) {
      line.push([r + dr * i, c + dc * i]);
    }
    return line;
  }
  const dirs = [
    [0, 1],
    [1, 0],
    [1, 1],
    [1, -1]
  ];
  for (let row = 0; row < ROWS; row++) {
    for (let col = 0; col < COLS; col++) {
      if (board.value[row][col] !== winPlayer) continue;
      for (const [dr, dc] of dirs) {
        const line = getLine(row, col, dr, dc);
        if (
          line.every(([r, c]) =>
            r >= 0 && r < ROWS && c >= 0 && c < COLS && board.value[r][c] === winPlayer
          )
        ) {
          coords.push(...line);
        }
      }
    }
  }
  return coords;
});
function cellIsWinning(row: number, col: number) {
  return winningCoords.value.some(([r, c]) => r === row && c === col);
}
</script>

<template>
  <div class="cf-root">
    <header class="cf-header">
      <h1>Connect Four</h1>
    </header>
    <section class="cf-board-wrapper">
      <div
        class="cf-board"
        :style="`grid-template-columns: repeat(${COLS}, 1fr); grid-template-rows: repeat(${ROWS}, 1fr);`"
      >
        <!-- Board clickable columns (transparent overlay) -->
        <button
          v-for="col in COLS"
          :key="'top-'+col"
          class="cf-col-input"
          aria-label="Play column"
          :disabled="animating || gameStatus !== 'playing' || getAvailableRow(col-1) === null"
          @click="playColumn(col - 1)"
          :style="`left: ${(col-1)*100/COLS}%; width: ${100/COLS}%;`"
        ></button>
        <!-- Board cells -->
        <template v-for="row in ROWS" :key="'r'+row">
          <div
            v-for="col in COLS"
            :key="'cell-'+row+'-'+col"
            class="cf-cell"
          >
            <div class="cf-hole"></div>
            <!-- Disc: placed -->
            <div
              v-if="board[row-1][col-1] !== EMPTY"
              class="cf-disc"
              :class="{
                win: cellIsWinning(row-1, col-1)
              }"
              :style="{
                background: board[row-1][col-1] === PLAYER1 ? COLOR_PRIMARY : COLOR_SECONDARY,
                boxShadow: cellIsWinning(row-1, col-1) ? `0 0 0 4px ${COLOR_ACCENT}` : 'none'
              }"
            ></div>
          </div>
        </template>
        <!-- Animated dropping disc -->
        <transition name="cf-fall">
          <div
            v-if="droppingRow !== null && droppingCol !== null && droppingPlayer !== null"
            class="cf-disc cf-disc-anim"
            :style="{
              background: droppingPlayer === PLAYER1 ? COLOR_PRIMARY : COLOR_SECONDARY,
              gridColumn: (droppingCol + 1).toString(),
              gridRow: (droppingRow + 1).toString()
            }"
          ></div>
        </transition>
      </div>
    </section>
    <section class="cf-status">
      <template v-if="gameStatus === 'playing'">
        <span
          :style="{
            color: currentPlayer === PLAYER1 ? COLOR_PRIMARY : COLOR_SECONDARY
          }"
        >
          ●
        </span>
        <span>
          {{ PLAYER_DETAILS[currentPlayer].label }}'s turn
        </span>
      </template>
      <template v-else-if="gameStatus === 'win' && winner">
        <span
          :style="{ color: winner === PLAYER1 ? COLOR_PRIMARY : COLOR_SECONDARY }"
        >
          ●
        </span>
        <span>
          {{ PLAYER_DETAILS[winner].label }} wins!
        </span>
      </template>
      <template v-else-if="gameStatus === 'draw'">
        <span style="color: #888;">Draw! No more moves.</span>
      </template>
    </section>
    <section class="cf-controls">
      <button class="cf-restart" @click="newGame">
        Restart Game
      </button>
    </section>
  </div>
</template>

<style scoped>
.cf-root {
  min-height: 100vh;
  display: flex;
  flex-direction: column;
  align-items: center;
  background: #fafcff;
  justify-content: center;
  padding: 0;
}
.cf-header {
  margin-top: 24px;
  margin-bottom: 28px;
  text-align: center;
}
.cf-header h1 {
  font-size: 2.2rem;
  font-weight: 700;
  letter-spacing: 0.01em;
  color: #222;
}
.cf-board-wrapper {
  display: flex;
  justify-content: center;
  align-items: center;
}
.cf-board {
  position: relative;
  display: grid;
  background: var(--cf-board-bg, #e3eae6);
  border-radius: 20px;
  box-shadow: 0 6px 26px 0 #17355c11;
  padding: 12px 10px 10px 12px;
  box-sizing: content-box;
  width: min(95vw,320px,90vh);
  aspect-ratio: 7 / 6;
  margin-bottom: 22px;
}
.cf-cell {
  width: 100%;
  height: 100%;
  position: relative;
  display: flex;
  justify-content: center;
  align-items: center;
}
.cf-hole {
  width: 86%;
  height: 86%;
  background: #f8fbfc;
  border-radius: 50%;
  box-shadow: inset 0 0 9px 2px #d1dbde;
  position: absolute;
  z-index: 0;
}
.cf-disc {
  width: 74%;
  height: 74%;
  border-radius: 50%;
  z-index: 1;
  box-shadow: 0 2px 10px 1px #2222;
  margin: 0 auto;
  border: 2.5px solid #fff;
  transition: box-shadow 300ms;
}
.cf-disc.win {
  box-shadow: 0 0 0 4px var(--cf-accent, #FFD600), 0 2px 10px 1px #2222;
  border-color: var(--cf-accent, #FFD600);
}
.cf-col-input {
  background: transparent;
  border: none;
  cursor: pointer;
  position: absolute;
  top: 0;
  height: 100%;
  transition: background 160ms;
  z-index: 5;
}
.cf-col-input:hover,
.cf-col-input:focus-visible {
  background: #ffd60022;
}
.cf-disc-anim {
  position: absolute;
  pointer-events: none;
  transition: none;
  /* handled by transition below */
}
.cf-status {
  margin-bottom: 18px;
  text-align: center;
  font-size: 1.18rem;
  font-weight: 500;
  letter-spacing: 0.01em;
  min-height: 2.2em;
  display: flex;
  justify-content: center;
  align-items: center;
  gap: 0.6em;
}
.cf-controls {
  text-align: center;
  margin-bottom: 16px;
}
.cf-restart {
  background: #fff8db;
  color: #A68C00;
  font-weight: 600;
  font-size: 1.07rem;
  border-radius: 16px;
  border: 1.5px solid #FFD600;
  box-shadow: 0 1.5px 8px #e8e8b322;
  padding: 0.7em 1.7em;
  cursor: pointer;
  outline: none;
  transition: filter 0.17s, background 0.17s;
}
.cf-restart:active {
  filter: brightness(0.93);
}
@media (max-width: 700px) {
  .cf-board {
    width: min(99vw,380px,96vh);
    padding: 6px 3px 3px 6px;
  }
  .cf-header h1 {
    font-size: 1.3rem;
  }
}
@media (max-width: 440px) {
  .cf-board {
    width: 99vw;
    border-radius: 0;
    box-shadow: none;
    padding: 1vw;
  }
  .cf-root {
    padding: 0;
  }
}
.cf-fall-enter-active,
.cf-fall-leave-active {
  transition: all 0.19s cubic-bezier(0.19, 1, 0.22, 1);
}
.cf-fall-enter-from,
.cf-fall-leave-to {
  opacity: 0.3;
  transform: translateY(-45px);
}
</style>
