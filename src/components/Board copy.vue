<template>
  <div class="game-container">
    <h2>🎲 Vue Backgammon 🎲</h2>

    <div v-if="winner" class="winner-modal">
      <div class="winner-content">
        <h1>🎉 {{ winner === 1 ? 'White (You)' : 'Black (CPU)' }} Wins! 🎉</h1>
        <button @click="initGame" class="reset-btn">Play Again</button>
      </div>
    </div>

    <div class="settings-bar">
      <label class="mode-switch">
        <input type="checkbox" v-model="isCpuMode"> 
        <span>VS Computer Mode</span>
      </label>
    </div>

    <div class="dice-controls">
      
      <div v-if="dice.values.length > 0" class="dice-display">
        <span class="die">{{ dice.values[0] }}</span>
        <span class="die">{{ dice.values[1] }}</span>
        
        <span v-if="dice.moves.length > 0" class="moves-left">
          (残り: {{ dice.moves.join(', ') }})
        </span>
        
        <span v-if="isCpuProcessing" class="cpu-status-text">
          🤖 CPU Move...
        </span>

        <button v-if="!isCpuProcessing && dice.moves.length > 0" @click="changeTurn" class="skip-btn">
          Skip
        </button>
      </div>
      
      <button 
        v-if="dice.moves.length === 0 && !winner && !isCpuProcessing" 
        @click="rollDice"
        class="roll-btn"
      >
        🎲 Roll Dice ({{ turn === 1 ? 'White' : 'Black' }})
      </button>

      <div v-if="isCpuProcessing && dice.values.length === 0" class="cpu-status-text">
        🤖 CPU Rolling...
      </div>
    </div>

    <div class="main-board">
      <div class="bar-area">
        <div class="bar-half top-bar">
          <div 
            v-for="n in bar.player1" :key="'w'+n" 
            class="checker white-checker"
            @click="handleBarClick(1)"
            :class="{ 'selected': selectedBar === 1 }"
          ></div>
        </div>
        <div class="bar-center">BAR</div>
        <div class="bar-half bottom-bar">
          <div 
            v-for="n in bar.player2" :key="'b'+n" 
            class="checker black-checker"
            @click="handleBarClick(2)"
            :class="{ 'selected': selectedBar === 2 }"
          ></div>
        </div>
      </div>

      <div class="points-area" v-if="points.length === 24">
        <div class="row top-row">
          <Cell
            v-for="i in topIndices" 
            :key="i"
            :index="i"
            :count="points[i].count"
            :owner="points[i].owner"
            :isTop="true"
            :isSelected="selectedPoint === i"
            :isMovable="movableIndices.has(i)"
            @clickPoint="handlePointClick"
          />
        </div>

        <div class="row bottom-row">
          <Cell
            v-for="i in bottomIndices" 
            :key="i"
            :index="i"
            :count="points[i].count"
            :owner="points[i].owner"
            :isTop="false"
            :isSelected="selectedPoint === i"
            :isMovable="movableIndices.has(i)"
            @clickPoint="handlePointClick"
          />
        </div>
      </div>

      <div class="bear-off-area">
        <div class="bear-half top-bear">
          <div v-for="n in bearOff.player1" :key="'off-w'+n" class="checker white-checker off-checker"></div>
        </div>
        <div class="bear-center">OFF</div>
        <div class="bear-half bottom-bear">
          <div v-for="n in bearOff.player2" :key="'off-b'+n" class="checker black-checker off-checker"></div>
        </div>
        
        <div 
           v-if="movableIndices.has(999)" 
           class="bear-off-highlight"
           @click="executeBearOff"
        >
          GOAL!
        </div>
      </div>
    </div>

    <div class="controls">
      <button @click="initGame" class="reset-btn">Reset Game</button>
      <div class="status">
        手番: <strong :style="{ color: turn === 1 ? '#fff' : '#aaa' }">
          {{ turn === 1 ? '白 (あなた)' : '黒 (CPU)' }}
        </strong>
      </div>
    </div>
  </div>
</template>

<script setup lang="ts">
import { reactive, ref, onMounted } from 'vue';
import Cell from './Cell.vue';

interface PointState { count: number; owner: 0 | 1 | 2; }

const points = reactive<PointState[]>([]);
for (let i = 0; i < 24; i++) points.push({ count: 0, owner: 0 });

const bar = reactive({ player1: 0, player2: 0 });
const bearOff = reactive({ player1: 0, player2: 0 });
const turn = ref<1 | 2>(1); 
const winner = ref<1 | 2 | null>(null);

const isCpuMode = ref(true);        
const isCpuProcessing = ref(false); 

const dice = reactive<{ values: number[], moves: number[] }>({ values: [], moves: [] });

const selectedPoint = ref<number | null>(null);
const selectedBar = ref<number | null>(null); 
const movableIndices = ref<Set<number>>(new Set()); 

const topIndices = [12, 13, 14, 15, 16, 17, 18, 19, 20, 21, 22, 23];
const bottomIndices = [11, 10, 9, 8, 7, 6, 5, 4, 3, 2, 1, 0];

function initGame() {
  points.length = 0;
  for (let i = 0; i < 24; i++) points.push({ count: 0, owner: 0 });
  bar.player1 = 0; bar.player2 = 0;
  bearOff.player1 = 0; bearOff.player2 = 0;
  turn.value = 1;
  winner.value = null;
  dice.values = []; dice.moves = [];
  isCpuProcessing.value = false;
  clearSelection();

  setupPoint(0, 2, 2);   
  setupPoint(11, 5, 2);  
  setupPoint(16, 3, 2);  
  setupPoint(18, 5, 2);  

  setupPoint(23, 2, 1);
  setupPoint(12, 5, 1);
  setupPoint(7, 3, 1);
  setupPoint(5, 5, 1);
}

function setupPoint(index: number, count: number, owner: 1 | 2) {
  if (points[index]) { points[index].count = count; points[index].owner = owner; }
}

function rollDice() {
  if (isCpuProcessing.value) return;

  const d1 = Math.floor(Math.random() * 6) + 1;
  const d2 = Math.floor(Math.random() * 6) + 1;
  dice.values = [d1, d2];
  if (d1 === d2) dice.moves = [d1, d1, d1, d1];
  else dice.moves = [d1, d2];
  clearSelection();
}

function clearSelection() {
  selectedPoint.value = null;
  selectedBar.value = null;
  movableIndices.value.clear();
}

function changeTurn() {
  turn.value = turn.value === 1 ? 2 : 1;
  dice.values = []; dice.moves = [];
  clearSelection();

  if (isCpuMode.value && turn.value === 2 && !winner.value) {
    playCpuTurn();
  }
}

// --- 🤖 CPUロジック (改良版) ---
async function playCpuTurn() {
  isCpuProcessing.value = true;
  
  // 1. 少し待つ
  await sleep(600);
  
  // 2. ダイスを振る
  const d1 = Math.floor(Math.random() * 6) + 1;
  const d2 = Math.floor(Math.random() * 6) + 1;
  dice.values = [d1, d2]; // ★ここで画面にサイコロが表示される
  if (d1 === d2) dice.moves = [d1, d1, d1, d1];
  else dice.moves = [d1, d2];

  // ★重要: ダイスを見せるために少し長めに待つ
  await sleep(1000);

  // 3. 移動開始
  while (dice.moves.length > 0 && !winner.value) {
    const allMoves = getAllLegalMoves(2);
    
    if (allMoves.length === 0) {
      console.log("CPU: 動かせる手がありません");
      break; 
    }

    // 戦略: ゴール > ヒット > ランダム
    let bestMove = allMoves[0];
    
    const goalMove = allMoves.find(m => m.target === 999);
    const hitMove = allMoves.find(m => {
       if (m.target === 999) return false;
       const p = points[m.target];
       return p.owner === 1 && p.count === 1; 
    });

    if (goalMove) bestMove = goalMove;
    else if (hitMove) bestMove = hitMove;
    else {
      bestMove = allMoves[Math.floor(Math.random() * allMoves.length)];
    }

    // 選択アニメーション
    if (bestMove.source === 'bar') {
      selectedBar.value = 2;
      handleBarClick(2); 
    } else {
      selectedPoint.value = bestMove.source as number;
      handlePointClick(bestMove.source as number);
    }
    
    await sleep(600); 

    // 移動実行
    if (bestMove.target === 999) {
      executeBearOff();
    } else {
      executeMove(bestMove.target);
    }

    await sleep(600); 
  }

  isCpuProcessing.value = false;
  changeTurn();
}

function getAllLegalMoves(player: 2): { source: number | 'bar', target: number }[] {
  const moves: { source: number | 'bar', target: number }[] = [];
  
  if (bar.player2 > 0) {
    selectedBar.value = 2;
    handleBarClick(2); 
    movableIndices.value.forEach(target => {
      moves.push({ source: 'bar', target });
    });
    clearSelection(); 
    return moves;
  }

  for (let i = 0; i < 24; i++) {
    if (points[i].owner === player && points[i].count > 0) {
      selectedPoint.value = i;
      calculateMovablePoints(i);
      movableIndices.value.forEach(target => {
        moves.push({ source: i, target });
      });
    }
  }
  clearSelection();
  return moves;
}

function sleep(ms: number) {
  return new Promise(resolve => setTimeout(resolve, ms));
}

// --- 共通ロジック ---

function canBearOff(player: 1 | 2): boolean {
  if ((player === 1 && bar.player1 > 0) || (player === 2 && bar.player2 > 0)) return false;
  for (let i = 0; i < 24; i++) {
    if (points[i].owner === player && points[i].count > 0) {
      if (player === 1 && i > 5) return false;
      if (player === 2 && i < 18) return false;
    }
  }
  return true;
}

function handleBarClick(owner: 1 | 2) {
  if (isCpuProcessing.value && owner === 1) return; 
  if (!isCpuProcessing.value && owner === 2) return; 

  if (winner.value || turn.value !== owner || dice.moves.length === 0) return;
  const count = owner === 1 ? bar.player1 : bar.player2;
  if (count <= 0) return;

  selectedPoint.value = null;
  selectedBar.value = owner;
  
  const possibleMoves = new Set<number>();
  const uniqueDice = [...new Set(dice.moves)];

  uniqueDice.forEach(die => {
    let target = -1;
    if (owner === 1) target = 24 - die; 
    if (owner === 2) target = die - 1; 
    if (isValidMove(target, owner)) possibleMoves.add(target);
  });
  movableIndices.value = possibleMoves;
}

function handlePointClick(index: number) {
  if (isCpuProcessing.value) return; 

  if (winner.value || dice.moves.length === 0) return;

  if (movableIndices.value.has(index)) {
    executeMove(index);
    return;
  }

  const myBarCount = turn.value === 1 ? bar.player1 : bar.player2;
  if (myBarCount > 0) {
    if (!isCpuMode.value) alert("バーに駒があります！先に復活させてください。");
    return;
  }

  const p = points[index];
  if (p.count > 0 && p.owner === turn.value) {
    selectedPoint.value = index;
    selectedBar.value = null;
    calculateMovablePoints(index);
  } else {
    clearSelection();
  }
}

function calculateMovablePoints(fromIndex: number) {
  const possibleMoves = new Set<number>();
  const uniqueDice = [...new Set(dice.moves)];
  const canOff = canBearOff(turn.value);

  uniqueDice.forEach(die => {
    let target = -1;
    let isOff = false;

    if (turn.value === 1) {
      target = fromIndex - die;
      if (target < 0) isOff = true;
    }
    if (turn.value === 2) {
      target = fromIndex + die;
      if (target > 23) isOff = true;
    }

    if (isOff) {
      if (canOff) {
        if (target === -1 || target === 24) {
             possibleMoves.add(999); 
        } else {
          if (checkNoPieceBehind(fromIndex, turn.value)) {
             possibleMoves.add(999);
          }
        }
      }
    } else {
      if (isValidMove(target, turn.value)) {
        possibleMoves.add(target);
      }
    }
  });
  movableIndices.value = possibleMoves;
}

function checkNoPieceBehind(currentIndex: number, player: 1 | 2): boolean {
  if (player === 1) {
    for (let i = currentIndex + 1; i <= 5; i++) {
      if (points[i].owner === 1 && points[i].count > 0) return false;
    }
  }
  if (player === 2) {
    for (let i = currentIndex - 1; i >= 18; i--) {
      if (points[i].owner === 2 && points[i].count > 0) return false;
    }
  }
  return true;
}

function isValidMove(targetIndex: number, player: 1 | 2): boolean {
  if (targetIndex < 0 || targetIndex > 23) return false; 
  const p = points[targetIndex];
  if (p.count === 0 || p.owner === player) return true;
  if (p.owner !== player && p.count === 1) return true;
  return false;
}

function executeBearOff() {
  if (selectedPoint.value === null) return;
  const player = turn.value;
  
  points[selectedPoint.value].count--;
  if (points[selectedPoint.value].count === 0) points[selectedPoint.value].owner = 0;

  if (player === 1) bearOff.player1++;
  else bearOff.player2++;

  let needed = 0;
  if (player === 1) needed = selectedPoint.value + 1;
  else needed = 24 - selectedPoint.value;

  let dieIdx = dice.moves.indexOf(needed);
  if (dieIdx === -1) {
    dice.moves.sort((a,b) => a-b);
    dieIdx = dice.moves.findIndex(m => m >= needed);
    if (dieIdx === -1) dieIdx = dice.moves.length - 1; 
  }
  
  dice.moves.splice(dieIdx, 1);
  clearSelection();
  checkWin();
  
  if (!winner.value && dice.moves.length === 0 && !isCpuProcessing.value) {
    setTimeout(() => changeTurn(), 500);
  }
}

function executeMove(toIndex: number) {
  let usedDie = 0;
  if (selectedBar.value !== null) {
    if (turn.value === 1) usedDie = 24 - toIndex;
    if (turn.value === 2) usedDie = toIndex + 1;
    if (turn.value === 1) bar.player1--; else bar.player2--;
  } else if (selectedPoint.value !== null) {
    usedDie = Math.abs(toIndex - selectedPoint.value);
    points[selectedPoint.value].count--;
    if (points[selectedPoint.value].count === 0) points[selectedPoint.value].owner = 0;
  }

  const target = points[toIndex];
  if (target.owner !== 0 && target.owner !== turn.value) {
    if (target.owner === 1) bar.player1++; else bar.player2++;
    target.count = 1; target.owner = turn.value;
  } else {
    target.count++; target.owner = turn.value;
  }

  const dieIdx = dice.moves.indexOf(usedDie);
  if (dieIdx > -1) dice.moves.splice(dieIdx, 1);
  else dice.moves.shift(); 

  clearSelection();
  
  if (dice.moves.length === 0 && !isCpuProcessing.value) {
     setTimeout(() => changeTurn(), 500);
  }
}

function checkWin() {
  if (bearOff.player1 === 15) winner.value = 1;
  if (bearOff.player2 === 15) winner.value = 2;
}

onMounted(() => { initGame(); });
</script>

<style scoped>
/* スタイル（以前と同じ） */
.game-container {
  display: flex; flex-direction: column; align-items: center;
  font-family: sans-serif; background-color: #222; color: #fff;
  min-height: 100vh; padding: 20px;
}
.settings-bar { margin-bottom: 10px; }
.mode-switch {
  background: #444; padding: 5px 10px; border-radius: 5px; cursor: pointer; display: flex; align-items: center; gap: 5px;
}
.dice-controls {
  height: 60px; display: flex; align-items: center; justify-content: center;
  margin-bottom: 10px; gap: 10px;
}
.cpu-status-text { font-weight: bold; color: yellow; margin-left: 10px; animation: pulse 1s infinite; }
.dice-display { display: flex; gap: 10px; align-items: center; }
.die {
  background: white; color: black; font-weight: bold; font-size: 1.5rem;
  width: 40px; height: 40px; display: flex; align-items: center; justify-content: center;
  border-radius: 8px;
}
.moves-left { margin-left: 10px; color: #ccc; font-size: 0.9rem; }
.roll-btn {
  padding: 10px 20px; font-size: 1.2rem; cursor: pointer;
  background-color: #4CAF50; color: white; border: none; border-radius: 5px;
}
.skip-btn {
  padding: 5px 10px; cursor: pointer; background-color: #666; color: white; border: none; border-radius: 3px;
}

.main-board {
  display: flex; background-color: #5c3a21; border: 10px solid #3e2714;
  border-radius: 8px; padding: 5px; height: 500px; width: 750px; 
}

.bar-area {
  width: 50px; background-color: #331f0f; display: flex; flex-direction: column;
  justify-content: space-between; align-items: center; border-right: 2px solid #222; padding: 5px 0;
}
.bar-center { font-weight: bold; writing-mode: vertical-rl; color: #aaa; }

.points-area { flex: 1; display: flex; flex-direction: column; }
.row { flex: 1; display: flex; }
.bottom-row { border-top: 2px solid rgba(0,0,0,0.2); }

.bear-off-area {
  width: 50px; background-color: #3e2714; display: flex; flex-direction: column;
  justify-content: space-between; align-items: center; border-left: 2px solid #222; padding: 5px 0;
  position: relative;
}
.bear-center { font-weight: bold; writing-mode: vertical-rl; color: #aaa; }
.off-checker { width: 28px; height: 10px; border-radius: 2px; margin-bottom: 1px; }

.checker {
  width: 28px; height: 28px; border-radius: 50%; box-shadow: 0 2px 2px rgba(0,0,0,0.3); margin-bottom: 2px;
  cursor: pointer;
}
.white-checker { background-color: #fff; border: 1px solid #ccc; }
.black-checker { background-color: #333; border: 1px solid #000; }
.selected { border: 2px solid orange; box-shadow: 0 0 10px orange; }

.bear-off-highlight {
  position: absolute; top: 0; left: 0; width: 100%; height: 100%;
  background-color: rgba(255, 215, 0, 0.4);
  color: yellow; font-weight: bold; writing-mode: vertical-rl;
  display: flex; align-items: center; justify-content: center;
  cursor: pointer; animation: pulse 1s infinite;
}
@keyframes pulse { 0% { opacity: 0.5; } 50% { opacity: 1; } 100% { opacity: 0.5; } }

.winner-modal {
  position: fixed; top: 0; left: 0; width: 100%; height: 100%;
  background: rgba(0,0,0,0.8); display: flex; align-items: center; justify-content: center; z-index: 999;
}
.winner-content { text-align: center; }
.reset-btn { padding: 5px 10px; cursor: pointer; }
.controls { margin-top: 20px; text-align: center; }
</style>