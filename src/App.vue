<template>
  <div class="max-w-6xl mx-auto p-4 flex flex-col gap-4">
    <header class="flex items-center justify-between bg-neutral-800 border border-neutral-700 rounded-lg p-3 shadow">
      <div class="flex flex-wrap gap-4 text-xs md:text-sm">
        <div class="flex items-center gap-2">
          <span class="px-2 py-1 bg-neutral-700 rounded">💰 Gold</span>
          <span class="font-semibold">{{ Math.floor(state.gold) }}</span>
          <span class="text-neutral-400">({{ perTurn.gold >= 0 ? '+' : '' }}{{ perTurn.gold }})</span>
        </div>
        <div class="flex items-center gap-2">
          <span class="px-2 py-1 bg-neutral-700 rounded">👥 Pop</span>
          <span class="font-semibold">{{ state.population }}</span>
        </div>
        <div class="flex items-center gap-2">
          <span class="px-2 py-1 bg-neutral-700 rounded">⚡ Energy</span>
          <span class="font-semibold">{{ Math.floor(state.energy) }}/{{ state.maxEnergy }}</span>
          <span class="text-neutral-400">({{ perTurn.energy >= 0 ? '+' : '' }}{{ perTurn.energy }})</span>
        </div>
        <div class="flex items-center gap-2">
          <span class="px-2 py-1 bg-neutral-700 rounded">📅 Turn</span>
          <span class="font-semibold">{{ state.turn }}</span>
        </div>
      </div>
      <div class="flex gap-2 text-xs">
        <button @click="endTurn" class="px-3 py-2 bg-emerald-600 hover:bg-emerald-500 rounded shadow">
          End Turn
        </button>
        <button @click="resetState" class="px-3 py-2 bg-neutral-700 hover:bg-neutral-600 rounded shadow">
          Reset
        </button>
      </div>
    </header>

    <div class="grid grid-cols-1 md:grid-cols-4 gap-4">
      <div class="md:col-span-3 bg-neutral-800 border border-neutral-700 rounded-lg p-3 flex flex-col items-center gap-3">
        <canvas
          ref="canvasRef"
          width="640"
          height="640"
          class="bg-neutral-900 border border-neutral-700 rounded"
          @mousemove="handleMouseMove"
          @mouseleave="handleMouseLeave"
          @click="handleClick"
        ></canvas>
        <p class="text-xs text-neutral-400 text-center">
          Click a card, hover to preview, click a tile to build. Terrain cannot be built on.
        </p>
      </div>
      <aside class="bg-neutral-800 border border-neutral-700 rounded-lg p-3 text-xs space-y-2">
        <h2 class="text-sm font-semibold">Legend</h2>
        <ul class="space-y-1">
          <li class="flex items-center gap-2"><span class="w-4 h-4 bg-emerald-600 border border-neutral-900"></span> Residential (+2 Pop, likes Parks/Water, dislikes Industry)</li>
          <li class="flex items-center gap-2"><span class="w-4 h-4 bg-amber-500 border border-neutral-900"></span> Commercial (Gold/turn, bonus near Residential)</li>
          <li class="flex items-center gap-2"><span class="w-4 h-4 bg-orange-600 border border-neutral-900"></span> Industrial (High Gold, -9 Energy)</li>
          <li class="flex items-center gap-2"><span class="w-4 h-4 bg-lime-500 border border-neutral-900"></span> Park (Boosts Residential)</li>
          <li class="flex items-center gap-2"><span class="w-4 h-4 bg-sky-400 border border-neutral-900"></span> Water (Terrain)</li>
          <li class="flex items-center gap-2"><span class="w-4 h-4 bg-slate-500 border border-neutral-900"></span> Mountain (Terrain)</li>
          <li class="flex items-center gap-2"><span class="w-4 h-4 bg-yellow-400 border border-neutral-900"></span> Solar Plant (+5 Energy/turn, +10 Max)</li>
        </ul>
        <p class="text-neutral-400 pt-2">Every 3 turns, draft a new card to add to your deck. Run out of Energy and it's Game Over.</p>
      </aside>
    </div>

    <section class="bg-neutral-800 border border-neutral-700 rounded-lg p-3">
      <h2 class="text-sm font-semibold mb-2">Hand</h2>
      <div class="grid grid-cols-2 sm:grid-cols-4 md:grid-cols-6 gap-2">
        <button
          v-for="(card, index) in state.hand"
          :key="index + card.key"
          class="text-left rounded-lg border border-neutral-700 bg-neutral-900 p-3 hover:border-emerald-500 transition shadow"
          :class="{ 'ring-2 ring-emerald-400': state.selectedCard === card }"
          @click="selectCard(card)"
        >
          <div class="flex items-center justify-between mb-2">
            <span class="text-lg">{{ card.icon }}</span>
            <span class="text-xs px-2 py-1 bg-neutral-800 rounded">{{ card.cost }}💰</span>
          </div>
          <div class="font-semibold text-sm">{{ card.key }}</div>
          <p class="text-[10px] text-neutral-300 leading-tight mt-1">{{ card.description }}</p>
        </button>
      </div>
    </section>
  </div>

  <div v-if="showDraft" class="fixed inset-0 bg-black/70 flex items-center justify-center">
    <div class="bg-neutral-900 border border-neutral-700 rounded-lg p-4 w-11/12 max-w-2xl shadow-xl">
      <h3 class="text-lg font-semibold mb-3">Draft a card</h3>
      <div class="grid grid-cols-1 md:grid-cols-3 gap-3">
        <button
          v-for="card in draftChoices"
          :key="card.key"
          class="rounded-lg border border-neutral-700 bg-neutral-800 p-3 text-left hover:border-emerald-500 transition"
          @click="pickDraft(card)"
        >
          <div class="flex items-center justify-between mb-1">
            <span class="text-lg">{{ card.icon }}</span>
            <span class="text-xs px-2 py-1 bg-neutral-900 rounded">{{ card.cost }}💰</span>
          </div>
          <div class="font-semibold">{{ card.key }}</div>
          <p class="text-[10px] text-neutral-300 leading-tight mt-1">{{ card.description }}</p>
        </button>
      </div>
      <p class="text-xs text-neutral-400 mt-3">Pick one card to add permanently to your deck.</p>
    </div>
  </div>

  <div v-if="state.gameOver" class="fixed inset-0 bg-black/80 flex items-center justify-center">
    <div class="bg-neutral-900 border border-red-600 rounded-lg p-6 text-center space-y-3 shadow-xl">
      <h2 class="text-2xl font-semibold text-red-400">Game Over</h2>
      <p class="text-neutral-200 text-sm">You ran out of Energy. Your citizens await a new leader.</p>
      <button @click="resetState" class="px-4 py-2 bg-emerald-600 hover:bg-emerald-500 rounded">Restart</button>
    </div>
  </div>
</template>

<script setup>
import { computed, onMounted, reactive, ref } from 'vue';

const tileSize = 80;
const gridSize = 8;

const cardLibrary = [
  { key: 'Residential', icon: '🏠', cost: 10, description: '+2 Pop. Bonus near Parks/Water, penalty near Industry.', type: 'build', building: 'residential' },
  { key: 'Commercial', icon: '🛍️', cost: 15, description: 'Generates Gold each turn. Bonus near Residential.', type: 'build', building: 'commercial' },
  { key: 'Industrial', icon: '🏭', cost: 20, description: 'High Gold/turn. -9 Energy upkeep.', type: 'build', building: 'industrial' },
  { key: 'Park', icon: '🌳', cost: 5, description: 'Boosts adjacent Residential.', type: 'build', building: 'park' },
  { key: 'Power', icon: '🔆', cost: 30, description: 'Solar Plant: +5 Energy/turn, +10 Max Energy.', type: 'build', building: 'solar' },
  { key: 'Demolish', icon: '🛠️', cost: 5, description: 'Remove a building. Terrain cannot be removed.', type: 'action', building: 'demolish' }
];

let ctx = null;
const canvasRef = ref(null);
const draftChoices = ref([]);
const showDraft = ref(false);
let audioCtx;

const state = reactive({
  grid: [],
  gold: 60,
  population: 0,
  energy: 15,
  maxEnergy: 20,
  turn: 1,
  deck: [],
  discard: [],
  hand: [],
  selectedCard: null,
  hover: { x: null, y: null },
  gameOver: false
});

const perTurn = computed(() => computePerTurn());

function beep({ frequency = 440, duration = 0.1, type = 'sine' } = {}) {
  try {
    if (!audioCtx) audioCtx = new AudioContext();
    const osc = audioCtx.createOscillator();
    const gain = audioCtx.createGain();
    osc.type = type;
    osc.frequency.value = frequency;
    gain.gain.value = 0.05;
    osc.connect(gain).connect(audioCtx.destination);
    osc.start();
    osc.stop(audioCtx.currentTime + duration);
  } catch (err) {
    console.warn('Audio unavailable', err);
  }
}

function initGrid() {
  state.grid = new Array(gridSize).fill(null).map(() => new Array(gridSize).fill(null).map(() => ({ terrain: null, building: null })));
  const pickTerrain = (kind, count) => {
    let placed = 0;
    while (placed < count) {
      const x = Math.floor(Math.random() * gridSize);
      const y = Math.floor(Math.random() * gridSize);
      if (!state.grid[y][x].terrain) {
        state.grid[y][x].terrain = kind;
        placed += 1;
      }
    }
  };
  pickTerrain('mountain', 6);
  pickTerrain('water', 6);
}

function shuffle(arr) {
  const copy = [...arr];
  for (let i = copy.length - 1; i > 0; i -= 1) {
    const j = Math.floor(Math.random() * (i + 1));
    [copy[i], copy[j]] = [copy[j], copy[i]];
  }
  return copy;
}

function addStartingDeck() {
  const basics = ['Residential', 'Commercial', 'Industrial', 'Park', 'Power', 'Residential', 'Residential', 'Commercial', 'Demolish'];
  basics.forEach((name) => {
    const card = cardLibrary.find((c) => c.key === name);
    state.deck.push({ ...card });
  });
}

function drawHand() {
  state.hand = [];
  for (let i = 0; i < 4; i += 1) {
    if (state.deck.length === 0) {
      state.deck = shuffle(state.discard.splice(0));
    }
    if (state.deck.length === 0) break;
    state.hand.push(state.deck.pop());
  }
}

function discardHand() {
  state.discard.push(...state.hand);
  state.hand = [];
}

function computePerTurn() {
  let goldDelta = 0;
  let energyDelta = 0;
  for (let y = 0; y < gridSize; y += 1) {
    for (let x = 0; x < gridSize; x += 1) {
      const cell = state.grid[y][x];
      if (!cell.building) continue;
      switch (cell.building.kind) {
        case 'commercial': {
          const resAdj = countAdjacent(x, y, (b) => b.building?.kind === 'residential');
          goldDelta += 6 + resAdj * 3;
          break;
        }
        case 'industrial': {
          goldDelta += 12;
          energyDelta -= 9;
          break;
        }
        case 'solar': {
          energyDelta += 5;
          break;
        }
        default:
          break;
      }
    }
  }
  return { gold: goldDelta, energy: energyDelta };
}

function countAdjacent(x, y, predicate) {
  const dirs = [
    [1, 0],
    [-1, 0],
    [0, 1],
    [0, -1]
  ];
  return dirs.reduce((acc, [dx, dy]) => {
    const nx = x + dx;
    const ny = y + dy;
    if (nx >= 0 && nx < gridSize && ny >= 0 && ny < gridSize) {
      acc += predicate(state.grid[ny][nx]) ? 1 : 0;
    }
    return acc;
  }, 0);
}

function placeBuilding(x, y, card) {
  if (state.gameOver) return false;
  const cell = state.grid[y][x];
  if (card.building === 'demolish') {
    if (!cell.building) return false;
    cell.building = null;
    beep({ frequency: 200, duration: 0.12, type: 'sawtooth' });
    return true;
  }
  if (cell.building || cell.terrain) return false;
  const building = { kind: card.building };
  cell.building = building;
  if (building.kind === 'residential') {
    const parks = countAdjacent(x, y, (c) => c.building?.kind === 'park');
    const water = countAdjacent(x, y, (c) => c.terrain === 'water');
    const industry = countAdjacent(x, y, (c) => c.building?.kind === 'industrial');
    const popGain = Math.max(0, 2 + parks + water - industry);
    state.population += popGain;
  }
  if (building.kind === 'solar') {
    state.maxEnergy += 10;
  }
  beep({ frequency: 480, duration: 0.08, type: 'triangle' });
  return true;
}

function canPlayCard(card, x, y) {
  if (!card) return false;
  const cell = state.grid[y][x];
  if (card.building === 'demolish') {
    return !!cell.building;
  }
  if (cell.terrain || cell.building) return false;
  return true;
}

function attemptPlay(x, y) {
  const card = state.selectedCard;
  if (!card) return;
  if (state.gold < card.cost) {
    beep({ frequency: 140, duration: 0.1 });
    return;
  }
  if (!canPlayCard(card, x, y)) {
    beep({ frequency: 160, duration: 0.08 });
    return;
  }
  const success = placeBuilding(x, y, card);
  if (success) {
    state.gold -= card.cost;
    const handIndex = state.hand.indexOf(card);
    if (handIndex >= 0) {
      state.discard.push(...state.hand.splice(handIndex, 1));
    }
    state.selectedCard = null;
    render();
  }
}

function applyTurnIncome() {
  const { gold, energy } = computePerTurn();
  state.gold += gold;
  const projectedEnergy = state.energy + energy;
  if (projectedEnergy < 0) {
    state.energy = 0;
    triggerGameOver();
    return;
  }
  state.energy = Math.min(state.maxEnergy, projectedEnergy);
}

function startTurn() {
  if (state.turn % 3 === 0) {
    openDraft();
  }
  drawHand();
  state.selectedCard = null;
  render();
}

function endTurn() {
  if (state.gameOver) return;
  discardHand();
  applyTurnIncome();
  if (state.gameOver) return;
  state.turn += 1;
  startTurn();
  beep({ frequency: 620, duration: 0.05, type: 'square' });
}

function openDraft() {
  draftChoices.value = shuffle(cardLibrary.filter((c) => c.key !== 'Demolish')).slice(0, 3);
  showDraft.value = true;
}

function pickDraft(card) {
  state.deck.push({ ...card });
  beep({ frequency: 720, duration: 0.08, type: 'square' });
  showDraft.value = false;
}

function triggerGameOver() {
  state.gameOver = true;
  beep({ frequency: 110, duration: 0.6, type: 'sawtooth' });
}

function resetState() {
  state.gold = 60;
  state.population = 0;
  state.energy = 15;
  state.maxEnergy = 20;
  state.turn = 1;
  state.deck = [];
  state.discard = [];
  state.hand = [];
  state.selectedCard = null;
  state.gameOver = false;
  state.hover = { x: null, y: null };
  initGrid();
  addStartingDeck();
  state.deck = shuffle(state.deck);
  startTurn();
  render();
}

function selectCard(card) {
  state.selectedCard = card;
  render();
}

function render() {
  if (!ctx) return;
  ctx.clearRect(0, 0, 640, 640);
  for (let y = 0; y < gridSize; y += 1) {
    for (let x = 0; x < gridSize; x += 1) {
      const cell = state.grid[y][x];
      const px = x * tileSize;
      const py = y * tileSize;
      ctx.fillStyle = '#111827';
      ctx.fillRect(px, py, tileSize, tileSize);
      ctx.strokeStyle = '#27272a';
      ctx.strokeRect(px, py, tileSize, tileSize);

      if (cell.terrain === 'water') {
        ctx.fillStyle = '#38bdf8';
        ctx.fillRect(px + 4, py + 4, tileSize - 8, tileSize - 8);
      }
      if (cell.terrain === 'mountain') {
        ctx.fillStyle = '#6b7280';
        ctx.fillRect(px + 4, py + 4, tileSize - 8, tileSize - 8);
      }

      if (cell.building) {
        const { kind } = cell.building;
        const colors = {
          residential: '#10b981',
          commercial: '#f59e0b',
          industrial: '#ea580c',
          park: '#84cc16',
          solar: '#facc15'
        };
        ctx.fillStyle = colors[kind] || '#e5e7eb';
        ctx.fillRect(px + 6, py + 6, tileSize - 12, tileSize - 12);
        ctx.fillStyle = '#0f172a';
        ctx.font = '12px "Press Start 2P", sans-serif';
        ctx.fillText(kind.slice(0, 3).toUpperCase(), px + 10, py + 24);
      }
    }
  }

  if (state.selectedCard && state.hover.x !== null) {
    const { x, y } = state.hover;
    const px = x * tileSize;
    const py = y * tileSize;
    const valid = canPlayCard(state.selectedCard, x, y) && state.gold >= state.selectedCard.cost;
    ctx.fillStyle = valid ? 'rgba(16, 185, 129, 0.25)' : 'rgba(239, 68, 68, 0.25)';
    ctx.fillRect(px, py, tileSize, tileSize);
  }
}

function handleMouseMove(e) {
  if (!canvasRef.value) return;
  const rect = canvasRef.value.getBoundingClientRect();
  const scaleX = rect.width / canvasRef.value.width;
  const scaleY = rect.height / canvasRef.value.height;
  const canvasX = (e.clientX - rect.left) / scaleX;
  const canvasY = (e.clientY - rect.top) / scaleY;
  const x = Math.floor(canvasX / tileSize);
  const y = Math.floor(canvasY / tileSize);
  if (x >= 0 && x < gridSize && y >= 0 && y < gridSize) {
    state.hover = { x, y };
  } else {
    state.hover = { x: null, y: null };
  }
  render();
}

function handleMouseLeave() {
  state.hover = { x: null, y: null };
  render();
}

function handleClick(e) {
  if (!canvasRef.value) return;
  const rect = canvasRef.value.getBoundingClientRect();
  const scaleX = rect.width / canvasRef.value.width;
  const scaleY = rect.height / canvasRef.value.height;
  const canvasX = (e.clientX - rect.left) / scaleX;
  const canvasY = (e.clientY - rect.top) / scaleY;
  const x = Math.floor(canvasX / tileSize);
  const y = Math.floor(canvasY / tileSize);
  if (x >= 0 && x < gridSize && y >= 0 && y < gridSize) {
    attemptPlay(x, y);
  }
}

onMounted(() => {
  ctx = canvasRef.value?.getContext('2d') || null;
  resetState();
});
</script>
