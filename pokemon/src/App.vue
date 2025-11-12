<template>
  <div class="pokedex-root" :style="{ background: gradientBackground }">
    
    <div class="header-white">
      <div class="header-content">
        <h1 class="header-title">{{ pokemonData ? capitalize(pokemonData.name) : 'Pokédex' }}</h1>
        <p class="header-subtitle">{{ pokemonData ? `#${pokemonData.id.toString().padStart(3,'0')}` : 'Busca tu Pokémon favorito' }}</p>
      </div>
    </div>

    <div class="search-bar-top">
      <input
        v-model="searchQuery"
        @keyup.enter="searchPokemon"
        placeholder="Nombre o número del Pokémon..."
        class="search-input"
      />
      <button @click="searchPokemon" class="search-btn">{{ loading ? 'Buscando...' : '🔍 Buscar' }}</button>
      <button v-if="pokemonData" @click="reset" class="clear-btn">Clear</button>
    </div>

    <div class="main-canvas">
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

      <div class="right-panel">
        <h3 class="stats-title">Estadísticas</h3>
        <div class="stat-row" v-for="stat in pokemonData?.stats || []" :key="stat.stat.name">
          <div class="stat-header">
            <span class="stat-name">{{ formatStatName(stat.stat.name) }}</span>
            <span class="stat-value">{{ stat.base_stat }} / 255</span>
          </div>
          <div class="stat-bar-outer">
            <div
              class="stat-bar-inner"
              :style="{ width: (stat.base_stat/255*100) + '%', background: leftCardColor }"
            ></div>
          </div>
        </div>
      </div>
    </div>

    <div v-if="error" class="error-toast">{{ error }}</div>
  </div>
</template>

<script setup>
import { ref, computed } from 'vue'
import axios from 'axios'

const iconEevee = 'https://raw.githubusercontent.com/PokeAPI/sprites/master/sprites/pokemon/133.png'
const iconPikachu = 'https://raw.githubusercontent.com/PokeAPI/sprites/master/sprites/pokemon/25.png'
const iconPsyduck = 'https://raw.githubusercontent.com/PokeAPI/sprites/master/sprites/pokemon/54.png'
const logoUrl = 'https://upload.wikimedia.org/wikipedia/commons/9/98/International_Pokémon_logo.svg'

const searchQuery = ref('')
const pokemonData = ref(null)
const speciesInfo = ref(null)
const loading = ref(false)
const error = ref('')

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

const dominantColor = ref('#E5C07B')
const leftCardColor = computed(() => dominantColor.value)
const imageSrc = computed(() => {
  if (!pokemonData.value) return ''
  return (pokemonData.value.sprites.other?.['official-artwork']?.front_default
    || pokemonData.value.sprites.front_default || '')
})
const weaknesses = ref([])

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

    const sp = await axios.get(data.species.url)
    speciesInfo.value = sp.data

    const sc = speciesInfo.value.color?.name
    if (sc && speciesColorMap[sc]) dominantColor.value = speciesColorMap[sc]
    else {
      const mainType = data.types[0].type.name
      dominantColor.value = typeColors[mainType] || '#E5C07B'
    }

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

const gradientBackground = computed(() => {
  if (!pokemonData.value) return 'linear-gradient(135deg, #667eea 0%, #764ba2 100%)'
  const types = pokemonData.value.types.map(t => typeColors[t.type.name])
  if (types.length === 1) {
    return `linear-gradient(135deg, ${types[0]} 0%, ${types[0]}80 100%)`
  }
  return `linear-gradient(135deg, ${types[0]} 0%, ${types[1]} 100%)`
})
</script>

<style scoped>
@import url('https://fonts.googleapis.com/css2?family=Montserrat:wght@300;600;800;900&display=swap');

* {
  margin: 0;
  padding: 0;
  box-sizing: border-box;
}

html, body {
  margin: 0;
  padding: 0;
  height: 100%;
  overflow: hidden;
}

.pokedex-root {
  width: 100vw;
  display: flex;
  flex-direction: column;
  font-family: 'Montserrat', sans-serif;
 
  transition: background 0.5s ease;
}

.top-bar {
  height: 70px;
  display: flex;
  align-items: center;
  justify-content: space-between;
  padding: 0 30px;
  flex-shrink: 0;
  background: rgba(255, 255, 255, 0.95);
  backdrop-filter: blur(10px);
  border-bottom: 1px solid rgba(0,0,0,0.1);
}

.top-left, .top-right {
  display: flex;
  gap: 10px;
  align-items: center;
}
.icon-small {
  width: 42px;
  height: 42px;
  image-rendering: pixelated;
  transition: transform 0.3s ease;
}
.icon-small:hover {
  transform: scale(1.2) rotate(-5deg);
}
.logo-wrap {
  width: 360px;
  display: flex;
  justify-content: center;
}
.poke-logo {
  height: 50px;
  opacity: 0.95;
  transition: transform 0.4s ease;
}
.poke-logo:hover {
  transform: scale(1.05);
}

.header-white {
  width: 100%;
  padding: 30px 0;
  text-align: center;
  background: white;
  flex-shrink: 0;
  box-shadow: 0 2px 10px rgba(0,0,0,0.1);
}

.header-title {
  font-size: 3rem;
  font-weight: 900;
  letter-spacing: 2px;
  margin-bottom: 8px;
  color: #1f2937;
  text-transform: uppercase;
}

.header-subtitle {
  font-size: 1.1rem;
  font-weight: 600;
  color: #6b7280;
}

.search-bar-top {
  display: flex;
  justify-content: center;
  align-items: center;
  gap: 12px;
  margin: 25px 0;
  flex-shrink: 0;
}
.search-input {
  width: 420px;
  padding: 14px 18px;
  border-radius: 12px;
  border: 2px solid #e2e8f0;
  background: #fff;
  color: #111;
  font-weight: 600;
  font-size: 16px;
  box-shadow: 0 4px 12px rgba(0,0,0,0.1);
  transition: all 0.3s ease;
}
.search-input:focus {
  border-color: #3B82F6;
  box-shadow: 0 0 0 3px rgba(59,130,246,0.1);
  outline: none;
}
.search-btn {
  padding: 14px 24px;
  border-radius: 12px;
  background: #1f2937;
  color: #fff;
  font-weight: 700;
  font-size: 16px;
  border: none;
  cursor: pointer;
  box-shadow: 0 4px 12px rgba(0,0,0,0.15);
  transition: all 0.3s ease;
}
.search-btn:hover {
  background: #374151;
  transform: translateY(-2px);
}
.clear-btn {
  padding: 12px 20px;
  border-radius: 10px;
  background: #f1f5f9;
  color: #475569;
  font-weight: 700;
  border: 2px solid #e2e8f0;
  cursor: pointer;
  transition: all 0.3s ease;
}
.clear-btn:hover {
  background: #e2e8f0;
  transform: translateY(-1px);
}

.main-canvas {
  flex: 1;
  display: flex;
  gap: 24px;
  align-items: center;
  justify-content: center;
  padding: 0 30px 20px;
}

.left-card, .center-panel, .right-panel {
  height: 100%;
  max-height: 500px;
  overflow: hidden;
  flex-shrink: 0;
}

.left-card {
  width: 33%;
  border-radius: 20px;
  display: flex;
  flex-direction: column;
  align-items: center;
  padding: 30px;
  transition: all 0.5s ease;
  box-shadow: 0 8px 32px rgba(0,0,0,0.2);
  border: 1px solid rgba(255,255,255,0.3);
  backdrop-filter: blur(10px);
}
.pokemon-name {
  font-size: 32px;
  font-weight: 800;
  color: #fff;
  text-shadow: 2px 2px 8px rgba(0,0,0,0.3);
  margin-bottom: 20px;
}
.image-wrap {
  flex: 1;
  display: flex;
  align-items: center;
  justify-content: center;
  width: 100%;
}
.image-wrap img {
  max-width: 90%;
  max-height: 250px;
  object-fit: contain;
  filter: drop-shadow(0 15px 30px rgba(0,0,0,0.3));
  transition: transform 0.3s ease;
}
.image-wrap img:hover {
  transform: scale(1.05);
}
.basic-stats {
  width: 100%;
  display: flex;
  justify-content: space-between;
  margin-top: 20px;
  color: rgba(255,255,255,0.9);
  font-size: 18px;
  font-weight: 600;
}

.center-panel {
  width: 34%;
  text-align: center;
  display: flex;
  flex-direction: column;
  gap: 20px;
}
.number-big {
  font-size: 4rem;
  font-weight: 900;
  color: white;
  text-shadow: 2px 2px 8px rgba(0,0,0,0.3);
}
.section {
  background: rgba(255, 255, 255, 0.95);
  padding: 20px;
  border-radius: 16px;
  box-shadow: 0 8px 32px rgba(0,0,0,0.1);
  backdrop-filter: blur(10px);
  border: 1px solid rgba(255,255,255,0.2);
}
.section-title {
  font-size: 1.3rem;
  font-weight: 800;
  margin-bottom: 15px;
  color: #1f2937;
}
.types-row {
  display: flex;
  gap: 10px;
  justify-content: center;
  flex-wrap: wrap;
}
.type-pill, .weak-pill {
  padding: 10px 20px;
  border-radius: 12px;
  color: white;
  font-weight: 700;
  border: none;
  cursor: pointer;
  box-shadow: 0 4px 12px rgba(0,0,0,0.2);
  transition: all 0.3s ease;
}
.type-pill:hover, .weak-pill:hover {
  transform: translateY(-2px);
  box-shadow: 0 6px 20px rgba(0,0,0,0.3);
}

.right-panel {
  width: 33%;
  background: rgba(255, 255, 255, 0.95);
  padding: 25px;
  border-radius: 20px;
  box-shadow: 0 8px 32px rgba(0,0,0,0.1);
  backdrop-filter: blur(10px);
  border: 1px solid rgba(255,255,255,0.2);
}
.stats-title {
  font-size: 1.5rem;
  font-weight: 800;
  margin-bottom: 20px;
  color: #1f2937;
  text-align: center;
}
.stat-row {
  margin-bottom: 16px;
}
.stat-header {
  display: flex;
  justify-content: space-between;
  font-size: 14px;
  font-weight: 600;
  color: #374151;
  margin-bottom: 6px;
}
.stat-bar-outer {
  width: 100%;
  height: 12px;
  background: #f1f5f9;
  border-radius: 8px;
  overflow: hidden;
}
.stat-bar-inner {
  height: 100%;
  border-radius: 8px;
  transition: all 0.8s ease;
}

.error-toast {
  position: fixed;
  left: 50%;
  transform: translateX(-50%);
  top: 120px;
  background: #fee2e2;
  color: #dc2626;
  padding: 12px 24px;
  border-radius: 12px;
  font-weight: 700;
  box-shadow: 0 8px 25px rgba(0,0,0,0.15);
  border: 1px solid #fecaca;
  z-index: 1000;
}

.muted {
  color: #9ca3af;
  font-style: italic;
}
</style>
