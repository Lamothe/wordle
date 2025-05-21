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

interface LetterStateDictionary {
  [key: string]: CellState;
}

const currentRow = ref(0);
const currentCol = ref(0);
const word = ref('');
const status = ref('');
const complete = ref(false);
const rows = ref<({ letter: string, state: CellState })[][]>([]);
const letters = ref<LetterStateDictionary>({});
const keyboardKeys: string[][] = [];
const keyboardRows = ref<string[][]>([]);

// Initialisation
keyboardKeys.push(['q', 'w', 'e', 'r', 't', 'y', 'u', 'i', 'o', 'p']);
keyboardKeys.push(['a', 's', 'd', 'f', 'g', 'h', 'j', 'k', 'l']);
keyboardKeys.push(['z', 'x', 'c', 'v', 'b', 'n', 'm']);

for (let x = 0; x < keyboardKeys.length; x++) {
  keyboardRows.value[x] = [];
  for (let y = 0; y < keyboardKeys[x]!.length; y++) {
    keyboardRows.value[x]![y] = keyboardKeys[x]![y]!;
  }
}

const resetBoard = (): void => {
  for (let x = 0; x < numRows; x++) {
    rows.value[x] = [];
    for (let y = 0; y < numCols; y++) {
      rows.value[x]![y] = { letter: '', state: CellState.Unknown };
    }
  }
}

const reset = (): void => {

  currentRow.value = 0;
  currentCol.value = 0;
  status.value = '';
  complete.value = false;
  resetBoard();

  // Reset keyboard key states
  for (let x = 0; x < keyboardKeys.length; x++) {
    for (let y = 0; y < keyboardKeys[x]!.length; y++) {
      letters.value[keyboardKeys[x]![y]!] = CellState.Unknown;
    }
  }

  // Load a new word
  word.value = words[Math.floor(Math.random() * words.length)]!;
}
reset();

// Methods
const isAlphabet = (char: string): boolean => /^[a-zA-Z]$/.test(char);

const getCurrentRowWord = () => {
  let rowWord = '';
  for (let y = 0; y < numCols; y++) {
    rowWord += rows.value[currentRow.value]![y]!.letter;
  }
  return rowWord;
}

const updateRow = () => {
  const letterCounts: { [key: string]: number } = {};

  for (let colIndex = 0; colIndex < numCols; colIndex++) {
    const wordLetter = word.value[colIndex]!;
    const cellLetter = rows.value[currentRow.value]![colIndex]!.letter;

    // Exact match?
    if (cellLetter === wordLetter) {
      console.log(`Exact match on ${cellLetter} at column ${colIndex}`);
      updateCellState(colIndex, CellState.ExactMatch);
      updateKeyboardKeyState(cellLetter, CellState.ExactMatch);
    }
    else { // No? Then add it to a list of uncounted letters.
      if (!letterCounts[wordLetter]) {
        letterCounts[wordLetter] = 1;
      } else {
        letterCounts[wordLetter]++;
      }
      console.log(`Added letterCount[${wordLetter}] now ${letterCounts[wordLetter]}`);
    }
  }

  for (let colIndex = 0; colIndex < numCols; colIndex++) {
    const cellLetter = rows.value[currentRow.value]![colIndex]!.letter;

    console.log(`Value for letterCount[${cellLetter}] = ${letterCounts[cellLetter]}`);
    if (letterCounts[cellLetter] && letterCounts[cellLetter] > 0) {
      letterCounts[cellLetter]--;
      updateCellState(colIndex, CellState.Match);
      updateKeyboardKeyState(cellLetter, CellState.Match);
    }
    else {
      updateKeyboardKeyState(cellLetter, CellState.Unused);
    }
  }
}

const processGuess = async (): Promise<void> => {
  const rowWord = getCurrentRowWord();

  // Is it a valid word in our dictionary?
  if (words.includes(rowWord)) {

    updateRow();

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
    reset();
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

const updateCellState = (colIndex: number, state: CellState) => {
  const currentState = rows.value[currentRow.value]![colIndex]?.state;
  if (currentState !== CellState.ExactMatch && currentState !== CellState.Unused) {
    rows.value[currentRow.value]![colIndex]!.state = state;
  }
}

const updateKeyboardKeyState = (key: string, state: CellState) => {
  const currentState = letters.value[key];
  if (currentState !== CellState.ExactMatch && currentState !== CellState.Unused) {
    letters.value[key] = state;
  }
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

/*
const test = async (): Promise<void> => {
  const testWordGuess = async (testWord: string, guessWord: string): Promise<void> => {
    word.value = testWord;
    for (let i = 0; i < guessWord.length; i++) {
      await handleKeyStroke(guessWord[i]!);
    }
    await handleKeyStroke("Enter");
  }

  await testWordGuess("rawer", "waver");
  //await testWordGuess("moron", "droll");
  //await testWordGuess("droll", "moron");
  //await testWordGuess("moron", "drool");
  //await testWordGuess("flags", "slags");
  //await testWordGuess("jowly", "clamp");
}

test().then(() => { }).catch(() => { });
*/
</script>

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
