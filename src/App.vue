<script setup lang="ts">
import { ref, onMounted, computed, watch } from 'vue';
import { TYPE_CHART, TYPE_COLORS, ALL_TYPES, type PokemonType } from './data/typeChart';
import { TRANSLATIONS, REGIONS } from './data/translations';

interface Pokemon {
  id: number;
  names: { en: string; ja: string };
  types: PokemonType[];
  sprite: string;
  stats: Record<string, number>;
}

interface GameState {
  score: number;
  history: Array<{ 
    id: number; 
    names: { en: string; ja: string }; 
    types: PokemonType[]; 
    selected: PokemonType; 
    multiplier: number; 
    points: number;
    bestTypes: PokemonType[];
    maxMultiplier: number;
  }>;
  wrongIds: number[];
  language: 'en' | 'ja';
  selectedRegions: string[];
}

const currentPokemon = ref<Pokemon | null>(null);
const loading = ref(false);
const showResult = ref(false);
const showSettings = ref(false);
const lastMultiplier = ref(1);
const lastPoints = ref(0);
const selectedType = ref<PokemonType | null>(null);

const gameState = ref<GameState>({
  score: 0,
  history: [],
  wrongIds: [],
  language: 'en',
  selectedRegions: REGIONS.map(r => r.id)
});

const t = computed(() => TRANSLATIONS[gameState.value.language]);

// 計算當前選擇地區的所有寶可夢 ID 集合
const currentPool = computed(() => {
  const pool: number[] = [];
  const activeRanges = REGIONS.filter(r => gameState.value.selectedRegions.includes(r.id)).map(r => r.range);
  activeRanges.forEach(([start, end]) => {
    for (let i = start; i <= end; i++) pool.push(i);
  });
  return pool;
});

// 計算當前 Pool 中已回答過的 ID
const answeredInPool = computed(() => {
  const historyIds = new Set(gameState.value.history.map(h => h.id));
  return currentPool.value.filter(id => historyIds.has(id));
});

const bestTypesForCurrent = computed(() => {
  if (!currentPokemon.value) return { types: [], multiplier: 0 };
  let maxM = 0;
  let types: PokemonType[] = [];
  ALL_TYPES.forEach(type => {
    const m = calculateMultiplier(type, currentPokemon.value!.types);
    if (m > maxM) {
      maxM = m;
      types = [type];
    } else if (m === maxM) {
      types.push(type);
    }
  });
  return { types, multiplier: maxM };
});

onMounted(() => {
  const saved = localStorage.getItem('poke_type_master_v3');
  if (saved) {
    gameState.value = { ...gameState.value, ...JSON.parse(saved) };
  }
  fetchNewPokemon();
});

const saveGame = () => {
  localStorage.setItem('poke_type_master_v3', JSON.stringify(gameState.value));
};

// 監聽地區變化，自動重新整理
watch(() => gameState.value.selectedRegions, () => {
  fetchNewPokemon();
}, { deep: true });

const fetchNewPokemon = async () => {
  if (currentPool.value.length === 0) return;
  
  loading.value = true;
  showResult.value = false;
  selectedType.value = null;
  
  try {
    // 優先挑選未回答過的
    const unanswered = currentPool.value.filter(id => !answeredInPool.value.includes(id));
    const targetPool = unanswered.length > 0 ? unanswered : currentPool.value;
    const id = targetPool[Math.floor(Math.random() * targetPool.length)];

    const [res, specRes] = await Promise.all([
      fetch(`https://pokeapi.co/api/v2/pokemon/${id}`),
      fetch(`https://pokeapi.co/api/v2/pokemon-species/${id}`)
    ]);
    const data = await res.json();
    const speciesData = await specRes.json();
    
    const jaName = speciesData.names.find((n: any) => n.language.name === 'ja-Hrkt')?.name || 
                   speciesData.names.find((n: any) => n.language.name === 'ja')?.name || data.name;
    const enName = speciesData.names.find((n: any) => n.language.name === 'en')?.name || data.name;

    currentPokemon.value = {
      id: data.id,
      names: { en: enName, ja: jaName },
      types: data.types.map((t: any) => t.type.name as PokemonType),
      sprite: data.sprites.other['official-artwork'].front_default || data.sprites.front_default,
      stats: {
        hp: data.stats[0].base_stat,
        atk: data.stats[1].base_stat,
        def: data.stats[2].base_stat,
        spa: data.stats[3].base_stat,
        spd: data.stats[4].base_stat,
        spe: data.stats[5].base_stat
      }
    };
  } catch (error) {
    console.error("Failed to fetch pokemon", error);
  } finally {
    loading.value = false;
  }
};

const calculateMultiplier = (attackType: PokemonType, defenderTypes: PokemonType[]): number => {
  let multiplier = 1;
  defenderTypes.forEach(defType => {
    const effect = TYPE_CHART[attackType][defType];
    if (effect !== undefined) multiplier *= effect;
  });
  return multiplier;
};

const handleTypeSelect = (type: PokemonType) => {
  if (showResult.value || !currentPokemon.value) return;
  
  selectedType.value = type;
  const multiplier = calculateMultiplier(type, currentPokemon.value.types);
  let points = 0;
  
  if (multiplier === 4) points = 10;
  else if (multiplier === 2) points = 1;
  else if (multiplier === 1) points = 0;
  else points = -5;
  
  lastMultiplier.value = multiplier;
  lastPoints.value = points;
  gameState.value.score += points;
  
  const { types: bTypes, multiplier: bMult } = bestTypesForCurrent.value;

  gameState.value.history.push({
    id: currentPokemon.value.id,
    names: { ...currentPokemon.value.names },
    types: [...currentPokemon.value.types],
    selected: type,
    multiplier,
    points,
    bestTypes: bTypes,
    maxMultiplier: bMult
  });
  
  if (points < 0 && !gameState.value.wrongIds.includes(currentPokemon.value.id)) {
    gameState.value.wrongIds.push(currentPokemon.value.id);
  }
  
  showResult.value = true;
  saveGame();
};

const resetProgress = () => {
  if (confirm(gameState.value.language === 'ja' ? 'すべての進度をリセットしますか？' : 'Reset all progress?')) {
    gameState.value.score = 0;
    gameState.value.history = [];
    gameState.value.wrongIds = [];
    saveGame();
    fetchNewPokemon();
  }
};

const getEffectText = (m: number) => {
  if (m === 4) return t.value.effect_4x;
  if (m === 2) return t.value.effect_2x;
  if (m === 1) return t.value.effect_1x;
  if (m === 0) return t.value.effect_0x;
  return t.value.effect_low;
};

const toggleRegion = (rid: string) => {
  const index = gameState.value.selectedRegions.indexOf(rid);
  if (index > -1) {
    if (gameState.value.selectedRegions.length > 1) {
      gameState.value.selectedRegions.splice(index, 1);
    }
  } else {
    gameState.value.selectedRegions.push(rid);
  }
  saveGame();
};

const getStatColor = (key: string) => {
  const colors: any = { hp: '#FF0000', atk: '#F08030', def: '#F8D030', spa: '#6890F0', spd: '#78C850', spe: '#F85888' };
  return colors[key] || '#ccc';
};
</script>

<template>
  <div class="app-container" :class="gameState.language">
    <header>
      <div class="title-row">
        <h1>{{ t.title }}</h1>
        <button class="settings-toggle" @click="showSettings = !showSettings">⚙️</button>
      </div>
      
      <div class="stats">
        <div class="stat-card">
          <span class="label">{{ t.score }}</span>
          <span class="value">{{ gameState.score }}</span>
        </div>
        <div class="stat-card">
          <span class="label">{{ t.wrong }}</span>
          <span class="value">{{ gameState.wrongIds.length }}</span>
        </div>
        <div class="stat-card progress">
          <span class="label">{{ gameState.language === 'ja' ? '進度' : 'Progress' }}</span>
          <span class="value">{{ answeredInPool.length }} / {{ currentPool.length }}</span>
        </div>
      </div>
    </header>

    <div v-if="showSettings" class="settings-panel">
      <div class="settings-group">
        <label>{{ t.language }}</label>
        <div class="lang-btns">
          <button :class="{ active: gameState.language === 'en' }" @click="gameState.language = 'en'; saveGame()">EN</button>
          <button :class="{ active: gameState.language === 'ja' }" @click="gameState.language = 'ja'; saveGame()">JA</button>
        </div>
      </div>
      
      <div class="settings-group">
        <label>{{ t.regions }}</label>
        <div class="region-grid">
          <button 
            v-for="reg in REGIONS" 
            :key="reg.id"
            :class="{ active: gameState.selectedRegions.includes(reg.id) }"
            @click="toggleRegion(reg.id)"
          >
            {{ reg.name[gameState.language] }}
          </button>
        </div>
      </div>

      <div class="settings-actions">
        <button class="reset-btn" @click="resetProgress">
          {{ gameState.language === 'ja' ? '進度リセット' : 'Reset Progress' }}
        </button>
        <button class="close-settings" @click="showSettings = false">OK</button>
      </div>
    </div>

    <main v-if="currentPokemon">
      <div class="pokemon-display" :class="{ 'is-loading': loading }">
        <div v-if="loading" class="loader">{{ t.loading }}</div>
        <template v-else>
          <img :src="currentPokemon.sprite" :alt="currentPokemon.names[gameState.language]" class="pokemon-img" />
          <h2 class="pokemon-name">#{{ currentPokemon.id }} {{ currentPokemon.names[gameState.language] }}</h2>
          <div class="reveal-types" v-if="showResult">
            <span v-for="type in currentPokemon.types" :key="type" class="type-badge" :style="{ backgroundColor: TYPE_COLORS[type] }">
              {{ (t as any).typeNames[type] }}
            </span>
          </div>

          <div class="stats-bar-container">
            <div v-for="(val, key) in currentPokemon.stats" :key="key" class="stat-row">
              <span class="stat-label">{{ (t as any)[key] }}</span>
              <div class="stat-bar-bg">
                <div class="stat-bar-fill" :style="{ width: `${Math.min((val / 160) * 100, 100)}%`, backgroundColor: getStatColor(key) }"></div>
              </div>
              <span class="stat-val">{{ val }}</span>
            </div>
          </div>
        </template>
      </div>

      <div class="feedback" v-if="showResult" :class="lastPoints > 0 ? 'pos' : (lastPoints < 0 ? 'neg' : 'neu')">
        <div class="result-title">{{ getEffectText(lastMultiplier) }}</div>
        
        <div v-if="bestTypesForCurrent.multiplier === 4" class="hint-4x">
          {{ t.hint_4x }}
        </div>

        <div class="best-hint" v-if="lastMultiplier < bestTypesForCurrent.multiplier">
          {{ t.best }}:
          <span v-for="type in bestTypesForCurrent.types" :key="type" class="mini-type" :style="{ backgroundColor: TYPE_COLORS[type] }">
            {{ (t as any).typeNames[type] }}
          </span>
          ({{ bestTypesForCurrent.multiplier }}x)
        </div>

        <div class="result-score">{{ lastPoints > 0 ? '+' : '' }}{{ lastPoints }}</div>
        <button @click="fetchNewPokemon" class="next-btn">{{ t.next }}</button>
      </div>

      <div class="type-grid" :class="{ disabled: showResult }">
        <button 
          v-for="type in ALL_TYPES" 
          :key="type"
          class="type-btn"
          :style="{ 
            backgroundColor: TYPE_COLORS[type],
            outline: selectedType === type ? '4px solid white' : 'none',
            opacity: showResult && selectedType !== type ? 0.5 : 1
          }"
          @click="handleTypeSelect(type)"
        >
          {{ (t as any).typeNames[type] }}
        </button>
      </div>
    </main>

    <footer v-if="gameState.history.length > 0">
      <h3>{{ t.recent }}</h3>
      <div class="history-list">
        <div v-for="(h, i) in gameState.history.slice().reverse().slice(0, 5)" :key="i" class="history-item">
          <div class="h-info">
            <span class="h-name">#{{ h.id }} {{ h.names[gameState.language] }}</span>
            <div class="h-best">
              {{ t.best }}: 
              <span v-for="bt in h.bestTypes" :key="bt" class="mini-type" :style="{ backgroundColor: TYPE_COLORS[bt] }">
                {{ (t as any).typeNames[bt] }}
              </span>
              ({{ h.maxMultiplier }}x)
            </div>
          </div>
          <span :class="h.points > 0 ? 'pos-text' : (h.points < 0 ? 'neg-text' : '')" class="h-points">
            {{ h.points > 0 ? '+' : '' }}{{ h.points }}
          </span>
        </div>
      </div>
    </footer>
  </div>
</template>

<style scoped>
.app-container {
  max-width: 600px;
  margin: 0 auto;
  padding: 15px;
  font-family: 'Segoe UI', 'Hiragino Kaku Gothic ProN', 'Meiryo', sans-serif;
  color: #fff;
  background-color: #1a1a1a;
  min-height: 100vh;
}

header { margin-bottom: 15px; }
.title-row { display: flex; justify-content: space-between; align-items: center; margin-bottom: 10px; }
h1 { font-size: 1.1rem; margin: 0; color: #ffcb05; text-shadow: 2px 2px #3b4cca; }

.settings-toggle { background: none; border: none; font-size: 1.4rem; cursor: pointer; padding: 5px; }

.stats { display: flex; gap: 8px; }
.stat-card {
  background: #333;
  padding: 4px 8px;
  border-radius: 8px;
  display: flex;
  flex-direction: column;
  align-items: center;
  flex: 1;
}
.stat-card.progress { flex: 1.5; background: #222; border: 1px solid #444; }
.label { font-size: 0.6rem; color: #aaa; text-transform: uppercase; }
.value { font-weight: 800; font-size: 0.9rem; }

.settings-panel {
  background: #2a2a2a;
  padding: 15px;
  border-radius: 12px;
  margin-bottom: 15px;
  border: 1px solid #444;
  box-shadow: 0 10px 30px rgba(0,0,0,0.5);
}

.settings-group { margin-bottom: 12px; }
.settings-group label { display: block; font-size: 0.8rem; color: #888; margin-bottom: 5px; }

.lang-btns { display: flex; gap: 8px; }
.lang-btns button, .region-grid button {
  background: #333;
  color: #888;
  border: 1px solid #444;
  padding: 6px 12px;
  border-radius: 6px;
  cursor: pointer;
  font-size: 0.75rem;
  transition: all 0.2s;
}
.lang-btns button.active, .region-grid button.active {
  background: #3b4cca;
  color: white;
  border-color: #ffcb05;
}

.region-grid { display: grid; grid-template-columns: repeat(3, 1fr); gap: 6px; }

.settings-actions { display: flex; gap: 10px; margin-top: 15px; }
.reset-btn {
  flex: 1;
  padding: 8px;
  background: rgba(239, 68, 68, 0.2);
  border: 1px solid #ef4444;
  color: #ef4444;
  border-radius: 6px;
  font-size: 0.75rem;
  font-weight: 800;
  cursor: pointer;
}
.reset-btn:hover { background: #ef4444; color: white; }

.close-settings {
  flex: 1;
  padding: 8px;
  background: #555;
  border: none;
  color: white;
  border-radius: 6px;
  font-size: 0.75rem;
  font-weight: 800;
  cursor: pointer;
}

.pokemon-display {
  background: #2a2a2a;
  border-radius: 20px;
  padding: 12px;
  text-align: center;
  margin-bottom: 12px;
  box-shadow: 0 8px 25px rgba(0,0,0,0.5);
  min-height: 280px;
  display: flex;
  flex-direction: column;
  justify-content: center;
  align-items: center;
}

.pokemon-img {
  width: 120px;
  height: 120px;
  object-fit: contain;
  filter: drop-shadow(0 0 8px rgba(255,255,255,0.15));
}

.pokemon-name { margin: 6px 0; font-size: 1.1rem; letter-spacing: 1px; }

.stats-bar-container {
  width: 100%;
  max-width: 300px;
  margin-top: 10px;
  display: flex;
  flex-direction: column;
  gap: 3px;
}
.stat-row { display: flex; align-items: center; gap: 8px; font-size: 0.6rem; }
.stat-label { width: 45px; text-align: right; color: #888; font-weight: 800; }
.stat-bar-bg { flex: 1; height: 4px; background: #444; border-radius: 2px; overflow: hidden; }
.stat-bar-fill { height: 100%; border-radius: 2px; transition: width 0.5s ease-out; }
.stat-val { width: 25px; text-align: left; font-weight: 800; }

.type-badge {
  padding: 4px 14px;
  border-radius: 15px;
  font-size: 0.75rem;
  font-weight: 800;
  text-shadow: 1px 1px 3px rgba(0,0,0,0.8);
  box-shadow: 0 2px 5px rgba(0,0,0,0.3);
}

.feedback {
  text-align: center;
  padding: 10px;
  border-radius: 12px;
  margin-bottom: 12px;
  animation: slideDown 0.3s ease-out;
}
.feedback.pos { background: rgba(74, 222, 128, 0.15); border: 1px solid #4ade80; }
.feedback.neg { background: rgba(248, 113, 113, 0.15); border: 1px solid #f87171; }
.feedback.neu { background: rgba(156, 163, 175, 0.15); border: 1px solid #9ca3af; }

.result-title { font-weight: 800; font-size: 1rem; margin-bottom: 3px; }
.result-score { font-size: 1.3rem; font-weight: 900; margin: 5px 0; }

.hint-4x { color: #fbbf24; font-weight: 800; font-size: 0.8rem; margin: 3px 0; }
.best-hint { font-size: 0.8rem; color: #ddd; margin: 3px 0; }

.mini-type {
  display: inline-block;
  padding: 2px 8px;
  border-radius: 4px;
  font-size: 0.7rem;
  margin: 0 3px;
  font-weight: 800;
  text-shadow: 1px 1px 2px rgba(0,0,0,0.8);
}

.next-btn {
  background: #3b4cca;
  color: white;
  border: none;
  padding: 6px 20px;
  border-radius: 20px;
  font-weight: 800;
  cursor: pointer;
  margin-top: 5px;
}

.type-grid { display: grid; grid-template-columns: repeat(3, 1fr); gap: 8px; }
.type-grid.disabled { pointer-events: none; }

.type-btn {
  border: 1px solid rgba(255,255,255,0.2);
  padding: 12px;
  border-radius: 8px;
  color: white;
  font-weight: 800;
  cursor: pointer;
  font-size: 0.75rem;
  text-shadow: 1px 1px 4px rgba(0,0,0,0.9);
  transition: transform 0.1s, box-shadow 0.1s, filter 0.1s;
}
.type-btn:hover { transform: translateY(-2px); filter: brightness(1.1); box-shadow: 0 5px 15px rgba(0,0,0,0.4); }
.type-btn:active { transform: scale(0.95); }

footer { margin-top: 20px; border-top: 1px solid #333; padding-top: 10px; }
footer h3 { font-size: 0.85rem; color: #888; margin-bottom: 8px; }
.history-list { display: flex; flex-direction: column; gap: 5px; }
.history-item {
  display: flex;
  justify-content: space-between;
  align-items: center;
  background: #222;
  padding: 5px 10px;
  border-radius: 6px;
}
.h-info { display: flex; flex-direction: column; }
.h-name { font-size: 0.75rem; font-weight: 800; }
.h-best { font-size: 0.65rem; color: #aaa; display: flex; align-items: center; }
.h-points { font-weight: 800; font-size: 0.9rem; }

.pos-text { color: #4ade80; }
.neg-text { color: #f87171; }

@keyframes slideDown {
  from { opacity: 0; transform: translateY(-10px); }
  to { opacity: 1; transform: translateY(0); }
}

@media (min-width: 400px) {
  .type-grid { grid-template-columns: repeat(6, 1fr); }
}
</style>
