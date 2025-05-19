<script setup lang="ts">
import { ref } from 'vue';
import { onKeyStroke } from '@vueuse/core'
import confetti from 'canvas-confetti';
import { words } from 'assets/words.json';

// Declarations
const numRows = 6;
const numCols = 5;

enum CellState {
  Unknown,
  Unused,
  Match,
  ExactMatch
}

interface LetterDictionary {
  [key: string]: CellState;
}

const currentRow = ref(0);
const currentCol = ref(0);
const word = ref('');
const status = ref('');
const complete = ref(false);
const rows = ref<({ letter: string, state: CellState })[][]>([]);
const letters = ref<LetterDictionary>({});
const keyboardKeys: string[][] = [];
const keyboardRows = ref<string[][]>([]);

// Initialisation
for (let x = 0; x < numRows; x++) {
  rows.value[x] = [];
  for (let y = 0; y < numCols; y++) {
    rows.value[x]![y] = { letter: '', state: CellState.Unknown };
  }
}

keyboardKeys.push(['q', 'w', 'e', 'r', 't', 'y', 'u', 'i', 'o', 'p']);
keyboardKeys.push(['a', 's', 'd', 'f', 'g', 'h', 'j', 'k', 'l']);
keyboardKeys.push(['z', 'x', 'c', 'v', 'b', 'n', 'm']);

for (let x = 0; x < keyboardKeys.length; x++) {
  keyboardRows.value[x] = [];
  for (let y = 0; y < keyboardKeys[x]!.length; y++) {
    const key = keyboardKeys[x]![y] as string;
    letters.value[key] = CellState.Unknown;
    keyboardRows.value[x]![y] = key;
  }
}

word.value = words[Math.floor(Math.random() * words.length)]!;

// Methods
const isAlphabet = (char: string): boolean => /^[a-zA-Z]$/.test(char);

const getRowWord = () => {
  let rowWord = '';
  for (let y = 0; y < numCols; y++) {
    rowWord += rows.value[currentRow.value]![y]!.letter;
  }
  return rowWord;
}

const updateCell = (colIndex: number) => {
  const rowWord = getRowWord();

  const letter = rowWord[colIndex]!;
  const wordIndexes = getIndexes(word.value, letter);
  const rowWordIndexes = getIndexes(rowWord, letter);

  if (wordIndexes.includes(colIndex)) {
    rows.value[currentRow.value]![colIndex]!.state = CellState.ExactMatch;
    updateKeyboardKeyState(letter, CellState.ExactMatch);
  }
  else if (wordIndexes.length > 0) {
    // Magic: I bet you'll never be able to work out what this does ever again!
    const matchIndex = rowWordIndexes.indexOf(colIndex);

    if (matchIndex < wordIndexes.length) {
      rows.value[currentRow.value]![colIndex]!.state = CellState.Match;
      updateKeyboardKeyState(letter, CellState.Match);
    }
  }
  else {
    updateKeyboardKeyState(letter, CellState.Unused);
  }
}

const processGuess = async (): Promise<void> => {
  const rowWord = getRowWord();

  console.log(`Processing guess ${rowWord}`);

  // Is it a valid word in our dictionary?
  if (words.includes(rowWord)) {

    for (let i = 0; i < numCols; i++) {
      updateCell(i);
    }

    // Does it match our word?
    if (rowWord === word.value) {
      status.value = 'You win!'
      complete.value = true;
      await confetti();
    }
    else {
      // Is it the last guess?
      if (currentRow.value >= numRows - 1) {
        status.value = `You lose! The word was ${word.value.toLocaleUpperCase()}`
        complete.value = true;
      }
      else {
        currentRow.value++;
        currentCol.value = 0;
      }
    }
  }
  else {
    status.value = 'Invalid word'
  }
}

const handleKeyStroke = async (key: string): Promise<boolean> => {
  if (!complete.value) {
    status.value = '';

    if (isAlphabet(key)) {
      if (rows.value[currentRow.value]![currentCol.value]!.letter === '') {
        rows.value[currentRow.value]![currentCol.value]!.letter = key.toLocaleLowerCase();
        if (currentCol.value < numCols - 1) {
          currentCol.value++;
        }
      }
    }
    else if (key == 'Enter') {
      // Only do something if you're on the last column.
      if (currentCol.value === numCols - 1) {
        await processGuess();
      }
    }
    else if (key == 'Backspace') {
      // If you're on the last column and it's populated, don't step back, just clear the cell.
      if (rows.value[currentRow.value]![currentCol.value]!.letter && currentCol.value === numCols - 1) {
        rows.value[currentRow.value]![currentCol.value]!.letter = '';
      }
      else if (currentCol.value > 0) {
        currentCol.value--;
        rows.value[currentRow.value]![currentCol.value]!.letter = '';
      }
    }
  }

  if (key == 'F5') {
    return false;
  }
  else if (key == 'F12') {
    return false;
  }

  return true;
}

onKeyStroke((e: KeyboardEvent): void => {
  handleKeyStroke(e.key)
    .then((prevent) => { if (prevent) e.preventDefault(); })
    .catch(() => { });
});

const updateKeyboardKeyState = (key: string, state: CellState) => {
  const currentState = letters.value[key];
  if (currentState !== CellState.ExactMatch && currentState !== CellState.Unused) {
    letters.value[key] = state;
  }
}

const getIndexes = (str: string, char: string): number[] => {
  const indexes = [];
  for (let i = 0; i < str.length; i++) {
    if (str[i] === char) {
      indexes.push(i);
    }
  }
  return indexes;
}

const getClass = (state: CellState): string => {
  if (state === CellState.ExactMatch) {
    return 'exactMatch'
  }
  else if (state === CellState.Match) {
    return 'match'
  }
  else if (state === CellState.Unused) {
    return 'unused'
  }

  return '';
}

const getCellClass = (rowIndex: number, colIndex: number) => {
  const cellClass = getClass(rows.value[rowIndex]![colIndex]!.state);
  if (cellClass) {
    return cellClass;
  }
  return (rowIndex == currentRow.value && colIndex == currentCol.value) ? 'currentCell' : '';
};

const getKeyboardKeyClass = (state: CellState) => {
  const cellClass = getClass(state);
  return cellClass ? 'keyboardKey ' + cellClass : 'keyboardKey';
}
</script>

<template>
  <header>
    WORDLE
  </header>

  <main>
    <div class="status">
      {{ status }} &nbsp;
    </div>

    <div v-for="(row, rowIndex) in rows" :key="rowIndex">
      <div v-for="(_, colIndex) in row" class="cell" :key="colIndex">
        <input type="text" maxlength="1" v-model="rows[rowIndex]![colIndex]!.letter"
          :class="getCellClass(rowIndex, colIndex)">
      </div>
    </div>

    <div class="keyboard">
      <div class="keyboardRow" v-for="(keyboardRow, keyboardRowIndex) in keyboardRows" :key="keyboardRowIndex">
        <div :class="getKeyboardKeyClass(letters[keyboardKey]!)" v-for="keyboardKey in keyboardRow" :key="keyboardKey">
          {{ keyboardKey }}
        </div>
      </div>
    </div>
  </main>
</template>

<style scoped>
header {
  text-align: center;
  font-size: 40px;
  padding: 20px;
}

main {
  text-align: center;
}

.cell {
  display: inline;
}

.cell input {
  width: 50px;
  height: 50px;
  font-size: 40px;
  text-transform: capitalize;
  text-align: center;
  background-color: #222;
  border: 1px solid #000;
  color: #eee;
  outline: none;
  caret-color: transparent;
  border-radius: 5px;
}

.cell input.match {
  background-color: yellow;
  color: #111;
}

.cell input.exactMatch {
  background-color: green;
}

.cell input.currentCell {
  background-color: #333;
}

.status {
  font-size: 30px;
  padding: 20px;
}

.keyboard {
  padding: 30px;
}

.keyboardKey {
  width: 40px;
  height: 40px;
  font-size: 30px;
  text-transform: capitalize;
  text-align: center;
  background-color: #222;
  border: 1px solid #000;
  color: #eee;
  outline: none;
  caret-color: transparent;
  border-radius: 5px;
  display: inline-block;
  padding-top: 5px;
}

.keyboardKey.exactMatch {
  background-color: green;
}

.keyboardKey.match {
  background-color: yellow;
  color: #111;
}

.keyboardKey.unused {
  background-color: #111;
  color: #666;
}
</style>
