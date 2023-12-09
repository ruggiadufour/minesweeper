<script setup lang="ts">
import { ref, onBeforeMount, reactive, watch } from 'vue';

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
const showed = ref(0);
const currentGame: 'win' | 'lose' | 'playing' = ref('playing');
const elements: Element[] = ref([]);

const getRandom = (n: number) => Math.floor(Math.random() * n);

const getRowColNum = (str: RowCol) => {
  const [row, col] = str.split('-');
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
    } while (bombs.has(position));
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

const reset = () => {
  counter.value = 0;
  showed.value = 0;
  currentGame.value = 'playing';
};

const play = () => {
  reset();
  generateBoard();
  generateBombs();
  generateArounds();
};

const around = (data: {
  row: number;
  col: number;
  callback: (el: Element) => void;
}) => {
  const subArr00 = [data.col - 1, data.row - 1];
  for (let c = subArr00[0]; c < subArr00[0] + 3; c++) {
    if (c < 0 || c >= cols.value) continue;

    for (let r = subArr00[1]; r < subArr00[1] + 3; r++) {
      if (r < 0 || r >= rows.value) continue;

      const el = getElement(r, c);
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

const handleRightClick = (element) => {
  element.flag = !element.flag;
};

const handleShowElement = (element) => {
  element.show = true;
  showed.value++;

  const total = rows.value * cols.value;
  if (total - bombsCount.value === showed.value) {
    handleWin();
  }
};

const handleWin = () => {
  currentGame.value = 'win';
};

const handleLose = () => {
  currentGame.value = 'lose';
  showAllBombs();
};

const handleClick = (element) => {
  if (element.isBomb) {
    handleLose();
  } else {
    handleShowElement(element);
    if (element.count === 0) {
      spreadAround(element);
    }
  }
};

onBeforeMount(() => {
  play();
});

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
    <br />
    <button @click="play">Reset</button>
  </div>
  <h1 v-if="currentGame !== 'playing'">
    {{ currentGame === 'win' ? 'You Won!' : 'You Lose!' }}
  </h1>
  <div class="board">
    <button
      v-for="(element, i) of elements"
      @click="handleClick(element)"
      @contextmenu.prevent="handleRightClick(element)"
      class="element"
      :class="{ show: element.show }"
      :disabled="currentGame !== 'playing'"
    >
      <div v-if="element.show">
        <span v-if="element.isBomb">💣</span>
        <span v-else-if="element.count > 0">{{ element.count }}</span>
      </div>
      <span v-if="!element.show && element.flag">🚩</span>
    </button>
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
