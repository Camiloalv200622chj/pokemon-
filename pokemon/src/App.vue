<template>
  <div class="pokedex-root">
    <!-- Top icons + logo -->
    <div class="top-bar">
      <div class="top-left">
        <img class="icon-small" :src="iconEevee" alt="eevee" />
        <img class="icon-small" :src="iconPikachu" alt="pikachu" />
      </div>
      <div class="logo-wrap">
        <img class="poke-logo" :src="logoUrl" alt="pokemon logo" />
      </div>
      <div class="top-right">
        <img class="icon-small" :src="iconPsyduck" alt="psyduck" />
      </div>
    </div>

    <!-- Main white canvas -->
    <div class="main-canvas">
      <!-- Left card: image + basic data -->
      <div class="left-card" :style="{ background: leftCardColor }">
        <h2 class="pokemon-name">{{ pokemonData?.name ? capitalize(pokemonData.name) : '' }}</h2>
        <div class="image-wrap">
          <img v-if="imageSrc" :src="imageSrc" :alt="pokemonData.name" />
        </div>

        <div class="basic-stats">
          <span>Altura: <strong>{{ pokemonData ? (pokemonData.height/10).toFixed(1) + 'm' : '-' }}</strong></span>
          <span>Peso: <strong>{{ pokemonData ? (pokemonData.weight/10).toFixed(1) + 'kg' : '-' }}</strong></span>
        </div>
      </div>

      <!-- Center panel: number, types, weaknesses -->
      <div class="center-panel">
        <div class="number-big"># {{ pokemonData ? pokemonData.id.toString().padStart(3,'0') : '---' }}</div>

        <div class="section">
          <h3 class="section-title">Tipos del pokémon</h3>
          <div class="types-row">
            <template v-if="pokemonData">
              <button
                v-for="t in pokemonData.types"
                :key="t.type.name"
                class="type-pill"
                :style="{ background: typeColors[t.type.name] }"
              >
                {{ t.type.name.toUpperCase() }}
              </button>
            </template>
          </div>
        </div>

        <div class="section">
          <h3 class="section-title">Debilidades del pokémon</h3>
          <div class="types-row">
            <template v-if="weaknesses.length">
              <button
                v-for="w in weaknesses"
                :key="w"
                class="weak-pill"
                :style="{ background: typeColors[w] || '#9CA3AF' }"
              >
                {{ w.toUpperCase() }}
              </button>
            </template>
            <div v-else class="muted">— N/A —</div>
          </div>
        </div>
      </div>

      <!-- Right panel: statistics -->
      <div class="right-panel">
        <h3 class="stats-title">Estadísticas</h3>

        <div class="stat-row" v-for="stat in pokemonData?.stats || []" :key="stat.stat.name">
          <div class="stat-header">
            <span class="stat-name">{{ formatStatName(stat.stat.name) }}</span>
            <span class="stat-value">{{ stat.base_stat }} / 255</span>
          </div>
          <div class="stat-bar-outer">
            <div class="stat-bar-inner" :style="{ width: (stat.base_stat/255*100) + '%', background: leftCardColor }"></div>
          </div>
        </div>
      </div>
    </div>

    <!-- Search overlay bottom center (always visible) -->
    <div class="search-bar">
      <input
        v-model="searchQuery"
        @keyup.enter="searchPokemon"
        placeholder="Nombre o número del Pokémon..."
        class="search-input"
      />
      <button @click="searchPokemon" class="search-btn">{{ loading ? 'Buscando...' : '🔍 Buscar' }}</button>
      <button v-if="pokemonData" @click="reset" class="clear-btn">Clear</button>
    </div>

    <!-- error toast -->
    <div v-if="error" class="error-toast">{{ error }}</div>
  </div>
</template>

<script setup>
import { ref, computed } from 'vue'
import axios from 'axios'

/* --- icons / logo (public sprite urls). You can replace these with local assets if you prefer --- */
const iconEevee = 'https://raw.githubusercontent.com/PokeAPI/sprites/master/sprites/pokemon/133.png'
const iconPikachu = 'https://raw.githubusercontent.com/PokeAPI/sprites/master/sprites/pokemon/25.png'
const iconPsyduck = 'https://raw.githubusercontent.com/PokeAPI/sprites/master/sprites/pokemon/54.png'
const logoUrl = 'https://upload.wikimedia.org/wikipedia/commons/9/98/International_Pokémon_logo.svg'

/* state */
const searchQuery = ref('')
const pokemonData = ref(null)
const speciesInfo = ref(null)
const loading = ref(false)
const error = ref('')

/* color maps */
const typeColors = {
  normal: '#A8A77A',
  fire: '#EE8130',
  water: '#6390F0',
  electric: '#F7D02C',
  grass: '#7AC74C',
  ice: '#96D9D6',
  fighting: '#C22E28',
  poison: '#A33EA1',
  ground: '#E2BF65',
  flying: '#A98FF3',
  psychic: '#F95587',
  bug: '#A6B91A',
  rock: '#B6A136',
  ghost: '#735797',
  dragon: '#6F35FC',
  dark: '#705746',
  steel: '#B7B7CE',
  fairy: '#D685AD'
}

/* species color name -> hex */
const speciesColorMap = {
  red: '#E4B858',
  blue: '#8BB9F1',
  yellow: '#E9D07A',
  green: '#9AE8A9',
  purple: '#CFAAFA',
  pink: '#F3A7C6',
  brown: '#CBAA88',
  gray: '#C4C4C4',
  white: '#FFFFFF',
  black: '#222222'
}

/* dominant color used for left card + stat bars */
const dominantColor = ref('#E5C07B') // default
const leftCardColor = computed(() => dominantColor.value)
const imageSrc = computed(() => {
  if (!pokemonData.value) return ''
  return (pokemonData.value.sprites.other?.['official-artwork']?.front_default
    || pokemonData.value.sprites.front_default || '')
})

/* weaknesses */
const weaknesses = ref([])

/* helpers */
function capitalize(s) {
  if (!s) return ''
  return s.charAt(0).toUpperCase() + s.slice(1)
}
function formatStatName(name) {
  const map = {
    hp: 'Hp',
    attack: 'Attack',
    defense: 'Defense',
    'special-attack': 'Special-attack',
    'special-defense': 'Special-defense',
    speed: 'Speed'
  }
  return map[name] || name
}

/* compute type weaknesses by calling /type/{name} */
async function computeWeaknesses(typesArr) {
  weaknesses.value = []
  try {
    const set = new Set()
    await Promise.all(typesArr.map(async (t) => {
      const res = await axios.get(`https://pokeapi.co/api/v2/type/${t.type.name}`)
      const dd = res.data.damage_relations.double_damage_from || []
      dd.forEach(x => set.add(x.name))
    }))
    weaknesses.value = Array.from(set)
  } catch (e) {
    weaknesses.value = []
  }
}

/* search function */
async function searchPokemon() {
  const q = (searchQuery.value || '').toString().trim().toLowerCase()
  if (!q) {
    error.value = 'Ingresa nombre o número del Pokémon'
    setTimeout(()=> error.value = '', 2500)
    return
  }

  loading.value = true
  error.value = ''
  pokemonData.value = null
  speciesInfo.value = null
  weaknesses.value = []

  try {
    const { data } = await axios.get(`https://pokeapi.co/api/v2/pokemon/${q}`)
    pokemonData.value = data

    // species (color, habitat, genera...)
    const sp = await axios.get(data.species.url)
    speciesInfo.value = sp.data

    // choose dominant color preferring species color
    const sc = speciesInfo.value.color?.name
    if (sc && speciesColorMap[sc]) dominantColor.value = speciesColorMap[sc]
    else {
      // fallback to main type color
      const mainType = data.types[0].type.name
      dominantColor.value = typeColors[mainType] || '#E5C07B'
    }

    // compute weaknesses
    await computeWeaknesses(data.types)
  } catch (e) {
    error.value = '❌ Pokémon no encontrado. Revisa el nombre o número.'
    pokemonData.value = null
    speciesInfo.value = null
    dominantColor.value = '#E5C07B'
    weaknesses.value = []
  } finally {
    loading.value = false
  }
}

function reset() {
  searchQuery.value = ''
  pokemonData.value = null
  speciesInfo.value = null
  weaknesses.value = []
  dominantColor.value = '#E5C07B'
  error.value = ''
}
</script>

<style scoped>
/* Import a clean font similar to the sample */
@import url('https://fonts.googleapis.com/css2?family=Montserrat:wght@300;600;800;900&display=swap');

:root, html, body {
  height: 100%;
  margin: 0;
}

/* root layout */
.pokedex-root {
  width: 100vw;
  height: 100vh;
  background: #ffffff; /* white background like the reference */
  font-family: 'Montserrat', sans-serif;
  display: flex;
  flex-direction: column;
  position: relative;
  overflow: hidden;
}

/* top icons & logo */
.top-bar {
  height: 90px;
  display: flex;
  align-items: center;
  justify-content: space-between;
  padding: 0 30px;
}
.top-left, .top-right {
  display:flex;
  gap:10px;
  align-items:center;
}
.icon-small {
  width:48px;
  height:48px;
  image-rendering: pixelated;
}
.logo-wrap {
  display:flex;
  align-items:center;
  justify-content:center;
  width: 360px;
}
.poke-logo {
  height:54px;
  opacity:0.95;
}

/* main canvas: 3 column layout */
.main-canvas {
  flex: 1;
  display:flex;
  gap: 30px;
  align-items: center;
  justify-content: center;
  padding: 10px 40px;
}

/* left card */
.left-card {
  width: 33%;
  border-radius: 14px;
  display:flex;
  flex-direction:column;
  align-items:center;
  justify-content:flex-start;
  padding: 26px;
  box-sizing: border-box;
  min-height: 70%;
}
.pokemon-name {
  font-size: 28px;
  font-weight: 900;
  margin-bottom: 10px;
  color: #111;
  text-shadow: 0 1px 0 rgba(255,255,255,0.6);
}
.image-wrap {
  width: 100%;
  display:flex;
  align-items:center;
  justify-content:center;
  padding: 8px;
}
.image-wrap img {
  max-width: 85%;
  max-height: 380px;
  object-fit: contain;
  filter: drop-shadow(0 20px 40px rgba(0,0,0,0.25));
}

/* basic stats bottom left */
.basic-stats {
  width: 100%;
  display:flex;
  justify-content:space-between;
  margin-top: 18px;
  color: rgba(0,0,0,0.85);
  font-size: 18px;
}

/* center panel */
.center-panel {
  width: 34%;
  display:flex;
  flex-direction:column;
  align-items:center;
  justify-content:flex-start;
  gap: 18px;
  padding-top: 40px;
}
.number-big {
  font-size: 84px;
  font-weight: 800;
  color: #d6b86a; /* gold-like default; dynamic color uses inline style in original example */
  margin-bottom: 6px;
}

.section-title {
  font-size: 26px;
  font-weight: 800;
  margin-bottom: 10px;
  color: #111;
}
.types-row {
  display:flex;
  gap:12px;
  align-items:center;
  justify-content:center;
  flex-wrap:wrap;
}
.type-pill {
  padding: 10px 18px;
  border-radius: 8px;
  color: #fff;
  font-weight:700;
  box-shadow: 0 4px 0 rgba(0,0,0,0.08);
}

/* weaknesses */
.weak-pill {
  padding: 8px 14px;
  border-radius: 8px;
  color: #fff;
  font-weight:700;
  box-shadow: 0 4px 0 rgba(0,0,0,0.06);
}
.muted { color: #666; }

/* right panel */
.right-panel {
  width: 33%;
  padding-top: 26px;
  height: 85%;
  display:flex;
  flex-direction:column;
  justify-content:flex-start;
  gap: 8px;
}
.stats-title {
  font-size: 28px;
  font-weight: 800;
  text-align:center;
  margin-bottom: 6px;
  color: #111;
}
.stat-row {
  margin-bottom: 8px;
}
.stat-header {
  display:flex;
  justify-content:space-between;
  font-size:14px;
  color:#111;
  margin-bottom:6px;
}
.stat-bar-outer {
  background: #ffffff;
  height: 18px;
  border-radius: 18px;
  padding: 3px;
  box-shadow: inset 0 0 0 2px rgba(0,0,0,0.06);
  background-color: #fff; /* bars background white with border look */
  border: 2px solid rgba(0,0,0,0.06);
}
.stat-bar-inner {
  height: 100%;
  border-radius: 14px;
  transition: width 0.7s ease;
}

/* search bar at bottom */
.search-bar {
  position: absolute;
  left: 50%;
  transform: translateX(-50%);
  bottom: 22px;
  display:flex;
  gap:10px;
  align-items:center;
  background: transparent;
}
.search-input {
  width: 420px;
  padding: 12px 14px;
  border-radius: 10px;
  border: 1px solid rgba(0,0,0,0.06);
  background: #fff;
  color: #111;
  font-weight:600;
  box-shadow: 0 6px 18px rgba(0,0,0,0.06);
}
.search-btn {
  padding: 12px 18px;
  border-radius: 10px;
  background: #111827;
  color: #fff;
  font-weight:700;
  box-shadow: 0 6px 18px rgba(0,0,0,0.08);
}
.clear-btn {
  padding: 10px 14px;
  border-radius: 8px;
  background: #efefef;
  color: #111;
  font-weight:700;
  border: 1px solid rgba(0,0,0,0.04);
}

/* error toast */
.error-toast {
  position: absolute;
  left: 50%;
  transform: translateX(-50%);
  top: 110px;
  background: #fee2e2;
  color: #7f1d1d;
  padding: 8px 12px;
  border-radius: 8px;
  font-weight:700;
  box-shadow: 0 4px 16px rgba(0,0,0,0.06);
}

/* responsive: stack vertically on narrow screens */
@media (max-width: 1000px) {
  .main-canvas { flex-direction: column; padding: 10px; align-items:flex-start; overflow:auto; }
  .left-card, .center-panel, .right-panel { width: 100%; }
  .search-input { width: 240px; }
  .number-big { font-size: 56px; }
  .poke-logo { height: 38px; }
}
</style>
