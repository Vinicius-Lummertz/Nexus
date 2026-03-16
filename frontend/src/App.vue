<script setup lang="ts">
import { computed, onMounted, ref } from 'vue'

type Tone = 'neutro' | 'acido' | 'tecnico' | 'amigavel'

interface NewsCard {
  id: number | string
  title: string
  summary: string
  tags: string[]
  validationQuestion?: string | null
}

interface FeedSection {
  id: number | string
  name: string
  affinity: number
  cards: NewsCard[]
}

interface FeedPayload {
  sections?: FeedSection[]
}

const apiBaseUrl = (import.meta.env.VITE_API_BASE_URL ?? '').replace(/\/$/, '')
const feedRoute = import.meta.env.VITE_FEED_ROUTE ?? '/feed'
const searchRoute = import.meta.env.VITE_SEARCH_ROUTE ?? '/search'
const validationRoute = import.meta.env.VITE_VALIDATION_ROUTE ?? '/validation'

const tone = ref<Tone>('tecnico')
const query = ref('')
const loading = ref(false)
const error = ref('')
const sections = ref<FeedSection[]>([])

const hasApiBase = computed(() => apiBaseUrl.length > 0)
const hasData = computed(() => sections.value.length > 0)

const normalizeCards = (rawCards: unknown): NewsCard[] => {
  if (!Array.isArray(rawCards)) return []

  return rawCards.map((raw, index) => {
    const card = raw as Partial<NewsCard>
    return {
      id: card.id ?? index + 1,
      title: typeof card.title === 'string' ? card.title : '',
      summary: typeof card.summary === 'string' ? card.summary : '',
      tags: Array.isArray(card.tags) ? card.tags.filter((tag): tag is string => typeof tag === 'string') : [],
      validationQuestion: typeof card.validationQuestion === 'string' ? card.validationQuestion : null,
    }
  })
}

const normalizeSections = (rawSections: unknown): FeedSection[] => {
  if (!Array.isArray(rawSections)) return []

  return rawSections.map((raw, index) => {
    const section = raw as Partial<FeedSection>
    return {
      id: section.id ?? index + 1,
      name: typeof section.name === 'string' ? section.name : '',
      affinity: typeof section.affinity === 'number' ? section.affinity : 0,
      cards: normalizeCards(section.cards),
    }
  })
}

const request = async (url: string, init?: RequestInit) => {
  const response = await fetch(url, init)
  if (!response.ok) {
    throw new Error(`HTTP ${response.status}`)
  }

  return (await response.json()) as FeedPayload | FeedSection[]
}

const applyPayload = (payload: FeedPayload | FeedSection[]) => {
  const rawSections = Array.isArray(payload) ? payload : payload.sections
  sections.value = normalizeSections(rawSections)
}

const loadFeed = async () => {
  if (!hasApiBase.value) return

  loading.value = true
  error.value = ''

  try {
    const payload = await request(`${apiBaseUrl}${feedRoute}`)
    applyPayload(payload)
  } catch (err) {
    error.value = err instanceof Error ? err.message : 'Falha ao carregar feed'
  } finally {
    loading.value = false
  }
}

const submitSearch = async () => {
  if (!hasApiBase.value) {
    error.value = 'Configure VITE_API_BASE_URL'
    return
  }

  if (!query.value.trim()) return

  loading.value = true
  error.value = ''

  try {
    const payload = await request(`${apiBaseUrl}${searchRoute}`, {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({ query: query.value, tone: tone.value }),
    })
    applyPayload(payload)
  } catch (err) {
    error.value = err instanceof Error ? err.message : 'Falha ao pesquisar'
  } finally {
    loading.value = false
  }
}

const submitValidation = async (cardId: NewsCard['id'], accepted: boolean) => {
  if (!hasApiBase.value) return

  try {
    await request(`${apiBaseUrl}${validationRoute}`, {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({ cardId, accepted }),
    })
  } catch {
    error.value = 'Falha ao validar preferencia'
  }
}

onMounted(() => {
  void loadFeed()
})
</script>

<template>
  <main class="app-shell">
    <header class="topbar">
      <h1>Nexus</h1>
      <form class="search-form" @submit.prevent="submitSearch">
        <input v-model="query" type="search" />
        <select v-model="tone" aria-label="Tom da IA">
          <option value="tecnico">Tecnico</option>
          <option value="acido">Acido</option>
          <option value="amigavel">Amigavel</option>
          <option value="neutro">Neutro</option>
        </select>
        <button type="submit" :disabled="loading">Buscar</button>
      </form>
    </header>

    <section v-if="!hasApiBase" class="notice">
      <p>Defina VITE_API_BASE_URL no arquivo .env para ativar o consumo da API.</p>
    </section>

    <section v-if="error" class="notice error">
      <p>{{ error }}</p>
    </section>

    <section v-if="loading" class="notice">
      <p>Carregando...</p>
    </section>

    <section v-else-if="hasData" class="feed-grid">
      <article v-for="section in sections" :key="section.id" class="feed-section">
        <header class="section-header">
          <h2>{{ section.name }}</h2>
          <span>{{ section.affinity }}</span>
        </header>

        <article v-for="card in section.cards" :key="card.id" class="news-card">
          <h3>{{ card.title }}</h3>
          <p>{{ card.summary }}</p>

          <ul v-if="card.tags.length > 0" class="tags">
            <li v-for="tag in card.tags" :key="tag">{{ tag }}</li>
          </ul>

          <div v-if="card.validationQuestion" class="validation">
            <p>{{ card.validationQuestion }}</p>
            <div>
              <button type="button" @click="submitValidation(card.id, true)">Sim</button>
              <button type="button" @click="submitValidation(card.id, false)">Nao</button>
            </div>
          </div>
        </article>
      </article>
    </section>

    <section v-else class="notice">
      <p>Sem dados no feed.</p>
    </section>
  </main>
</template>
