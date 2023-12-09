<script setup lang="ts">
import { ref, watch, reactive } from "vue";

type Element = {
  row: number;
  col: number;
  count: number;
  show: boolean;
  flag: boolean;
  isBomb: boolean;
};

type RowCol = `${number}-${number}`;

const rows = ref(10);
const cols = ref(7);
const bombsCount = ref(5);
const counter = ref(0);
const showBombs = ref(false);
const showed = ref(0);
const currentGame = ref<"win" | "lose" | "playing" | "waiting">("waiting");
const elements = ref<Element[]>([]);
const intervalId = ref();
const logs = reactive(new Map());
const showingReplay = ref(false);

const getRandom = (n: number) => Math.floor(Math.random() * n);

const getRowColNum = (str: RowCol) => {
  const [row, col] = str.split("-");
  return { row: Number(row), col: Number(col) };
};

const generateBombs = () => {
  const bombs = new Map();

  for (let i = 0; i < bombsCount.value; i++) {
    let position: RowCol;
    do {
      const row = getRandom(rows.value),
        col = getRandom(cols.value);

      position = `${row}-${col}`;
    } while (bombs.has(position) && bombs.size < elements.value.length);
    bombs.set(position, true);

    const { row, col } = getRowColNum(position);
    const el = getElement(row, col);
    el.isBomb = true;
  }
};

const showAllBombs = () => {
  elements.value.forEach((el) => {
    if (el.isBomb) el.show = true;
  });
};

const generateBoard = () => {
  elements.value = Array.from({ length: cols.value * rows.value }, (_, i) => ({
    col: i % cols.value,
    row: Math.floor(i / cols.value),
    count: 0,
    show: false,
    flag: false,
    isBomb: false,
  }));
};

const generateLog = (element: Element, type: "flag" | "select") => {
  if (showingReplay.value) return;

  logs.set(counter.value, { element, type });
};

const generateTimer = () => {
  const INTERVAL = 250;
  intervalId.value = setInterval(() => {
    counter.value += INTERVAL;
    showingReplay && checkReplayStep();
  }, INTERVAL);
};

const closeTimer = () => clearInterval(intervalId.value);

const getElement = (row: number, col: number): Element => {
  const index = row * cols.value + col;
  return elements.value[index];
};

const generateArounds = () => {
  elements.value
    .filter((el) => el.isBomb)
    .forEach((el) => {
      const { row, col } = el;

      around({ col, row, callback: (el) => el.count++ });
    });
};

const reset = (clearLogs = true) => {
  counter.value = 0;
  showed.value = 0;
  currentGame.value = "playing";
  clearLogs && logs.clear();
  closeTimer();
};

const play = () => {
  reset();
  showingReplay.value = false;
  generateBoard();
  generateBombs();
  generateArounds();
  generateTimer();
};

const resetAllStatus = () => {
  elements.value.forEach((el) => {
    el.show = false;
    el.flag = false;
  });
};

const checkReplayStep = () => {
  const { element, type } = logs.get(counter.value);
  if (element) {
    if (type === "select") handleClick(element);
    else handleRightClick(element);
  }
};

const replay = () => {
  showingReplay.value = true;
  reset(false);
  resetAllStatus();
  generateTimer();
};

const around = (data: {
  row: number;
  col: number;
  callback: (el: Element) => void;
}) => {
  const subArrColStart = data.col - 1;
  const subArrRowStart = data.row - 1;
  const ARR_LENGTH = 3;

  for (let col = subArrColStart; col < subArrColStart + ARR_LENGTH; col++) {
    if (col < 0 || col >= cols.value) continue;

    for (let row = subArrRowStart; row < subArrRowStart + ARR_LENGTH; row++) {
      if (row < 0 || row >= rows.value) continue;

      const el = getElement(row, col);
      data.callback(el);
    }
  }
};

const spreadAround = (el: Element) => {
  around({
    col: el.col,
    row: el.row,
    callback: (thisEl) => {
      if (!thisEl.show && !thisEl.isBomb) {
        handleShowElement(thisEl);
        if (thisEl.count === 0) {
          spreadAround(thisEl);
        }
      }
    },
  });
};

const handleRightClick = (element: Element) => {
  generateLog(element, "flag");
  element.flag = !element.flag;
};

const handleShowElement = (element: Element) => {
  element.show = true;
  showed.value++;

  const total = rows.value * cols.value;
  if (total - bombsCount.value === showed.value) {
    handleWin();
  }
};

const handleWin = () => {
  currentGame.value = "win";
  closeTimer();
};

const handleLose = () => {
  currentGame.value = "lose";
  showAllBombs();
  closeTimer();
};

const handleClick = (element: Element) => {
  if (element.show) return;

  generateLog(element, "select");

  if (element.isBomb) {
    handleLose();
  } else {
    handleShowElement(element);
    if (element.count === 0) {
      spreadAround(element);
    }
  }
};

watch([rows, cols, bombsCount], play);
</script>

<template>
  <div>
    <div>
      <label>Rows: </label>
      <input v-model="rows" type="number" />
    </div>
    <div>
      <label>Columns: </label>
      <input v-model="cols" type="number" />
    </div>
    <div>
      <label>Bombs: </label>
      <input v-model="bombsCount" type="number" />
    </div>
    <div>
      <label>Show bombs: </label>
      <input v-model="showBombs" type="checkbox" />
    </div>
    <br />
    <span>Time: {{ Math.round(counter / 1000) }}</span>
    <br />
    <button @click="play">
      {{ currentGame === "waiting" ? "Play" : "Reset" }}
    </button>
    <button :disabled="logs.size === 0" @click="replay">Replay</button>
  </div>

  <div v-if="currentGame !== 'waiting'">
    <h1 v-if="currentGame !== 'playing'">
      {{ currentGame === "win" ? "You Won!" : "You Lose!" }}
    </h1>
    <div class="board">
      <button
        v-for="element of elements"
        @click="handleClick(element)"
        @contextmenu.prevent="handleRightClick(element)"
        class="element"
        :class="{ show: element.show }"
        :disabled="currentGame !== 'playing'"
      >
        <div v-if="showBombs && element.isBomb">
          <span v-if="element.isBomb">💣</span>
        </div>
        <div v-else-if="element.show">
          <span v-if="element.isBomb">💣</span>
          <span v-else-if="element.count > 0">{{ element.count }}</span>
        </div>
        <span v-if="!element.show && element.flag">🚩</span>
      </button>
    </div>
  </div>
</template>

<style scoped>
.board {
  display: grid;
  grid-template-columns: repeat(v-bind(cols), 1fr);
  gap: 5px;
  margin-top: 2rem;
}

.element {
  width: 50px;
  height: 50px;
  text-align: center;
}

.show {
  background-color: gray;
}
</style>
