<template>
  <main class="app-layout">
    <section class="lienzo">
      <textarea
        v-model="text"
        @input="onInput"
        placeholder="Escribe lo que quieras aquí..."
        class="editor"
      ></textarea>
    </section>

    <aside class="respuesta">
      <transition name="fade">
        <div v-if="showResponse" class="respuesta-card">
          <transition name="fade" mode="out-in">
            <div :key="displayText" class="respuesta-body">
              <template v-if="noteMode">
                <p class="response-title">Notas estructuradas</p>

              <template v-if="noteStructure.ideas.length">
                <p class="section-title">Ideas</p>
                <ul class="price-list">
                  <li v-for="item in noteStructure.ideas" :key="item">
                    <span>{{ item }}</span>
                  </li>
                </ul>
              </template>

              <template v-if="noteStructure.actions.length">
                <p class="section-title">Acciones</p>
                <ul class="price-list">
                  <li v-for="item in noteStructure.actions" :key="item">
                    <span>{{ item }}</span>
                  </li>
                </ul>
              </template>

              <template v-if="noteStructure.questions.length">
                <p class="section-title">Preguntas</p>
                <ul class="price-list">
                  <li v-for="item in noteStructure.questions" :key="item">
                    <span>{{ item }}</span>
                  </li>
                </ul>
              </template>

              <template v-if="noteStructure.decisions.length">
                <p class="section-title">Decisiones</p>
                <ul class="price-list">
                  <li v-for="item in noteStructure.decisions" :key="item">
                    <span>{{ item }}</span>
                  </li>
                </ul>
              </template>

              <template v-if="noteStructure.others.length">
                <p class="section-title">Otros</p>
                <ul class="price-list">
                  <li v-for="item in noteStructure.others" :key="item">
                    <span>{{ item }}</span>
                  </li>
                </ul>
              </template>
            </template>
            <template v-else>
              <p>Respuesta</p>
            </template>
          </div>
          </transition>
          <button
            class="accion-btn"
            @click="handleAction"
            @mousedown="pressed = true"
            @mouseup="pressed = false"
            @mouseleave="pressed = false"
            :class="{ pressed }"
          >
            {{ noteMode ? 'Organizar notas' : 'Acción' }}
          </button>
        </div>
      </transition>
    </aside>
  </main>
</template>

<script setup>
import { ref, computed, onBeforeUnmount } from 'vue'

const text = ref('')
const displayText = ref('')
const showResponse = ref(false)
const pressed = ref(false)
let timerId = null

const noteKeywords = [
  'idea', 'ideas', 'pendiente', 'pendientes', 'pregunta', 'preguntas',
  'decisión', 'decisiones', 'decidir', 'recordar', 'nota', 'notas',
  'brainstorm', 'reunión', 'reunion', 'kickoff', 'cliente', 'campaña', 'proyecto', 'equipo'
]

const actionKeywords = [
  'hacer', 'revisar', 'llamar', 'enviar', 'preguntar', 'definir', 'terminar', 'completar', 'organizar', 'crear', 'coordinar', 'agendar'
]

const decisionKeywords = [
  'decidir', 'decisión', 'decisiones', 'elegir', 'optar', 'confirmar', 'aprobar'
]

const questionKeywords = [
  'qué', 'que', 'por qué', 'porque', 'cómo', 'como', 'cuándo', 'cuando', 'dónde', 'donde', 'quién', 'quien', 'para qué', 'para que'
]

const ideaKeywords = [
  'podría', 'podrias', 'podríamos', 'podemos', 'posible', 'quizá', 'quizas', 'tal vez', 'idea', 'ideas', 'sugerencia', 'sugerir'
]

const sentenceSeparatorRegex = /[,;•]+|\s+y\s+|\s+and\s+|\s+pero\s+/i
const listMarkerRegex = /^[\-\*\d\.]+\s+/
const questionRegex = /[¿?]|\b(?:qué|qué|que|cómo|como|cuándo|cuando|dónde|donde|quién|quien|por qué|porque)\b/i

const isNoteLike = (content) => {
  return noteKeywords.some((word) => content.includes(word)) || questionRegex.test(content)
}

const noteMode = computed(() => {
  const content = displayText.value.toLowerCase().trim()
  if (!content) return false

  const lines = content.split(/\r?\n/).map((line) => line.trim()).filter(Boolean)
  const hasNoteWord = isNoteLike(content)
  const listStyle = lines.length > 1 || lines.some((line) => listMarkerRegex.test(line) || line.includes(','))

  return hasNoteWord && listStyle
})

const noteStructure = computed(() => {
  if (!noteMode.value) return { ideas: [], actions: [], questions: [], decisions: [], others: [], total: 0 }

  const rawSegments = displayText.value
    .split(/\r?\n/)
    .flatMap((line) => line.split(sentenceSeparatorRegex))
    .map((segment) => segment.trim().replace(listMarkerRegex, ''))
    .filter((segment) => segment.length > 0)

  const classify = (segment) => {
    const normalized = segment.toLowerCase()
    if (questionRegex.test(normalized) || segment.includes('?')) {
      return 'questions'
    }
    if (decisionKeywords.some((word) => normalized.includes(word))) {
      return 'decisions'
    }
    if (actionKeywords.some((word) => normalized.includes(word))) {
      return 'actions'
    }
    if (ideaKeywords.some((word) => normalized.includes(word))) {
      return 'ideas'
    }
    if (/\b(pendiente|pendientes|tarea|tareas)\b/i.test(normalized)) {
      return 'actions'
    }
    return 'others'
  }

  const categories = {
    ideas: [],
    actions: [],
    questions: [],
    decisions: [],
    others: []
  }

  rawSegments.forEach((segment) => {
    const category = classify(segment)
    const cleaned = segment.replace(/^[\-\*\d\.\s]+/, '').trim()
    const finalText = cleaned.length > 0 ? cleaned : segment
    if (!categories[category].includes(finalText)) {
      categories[category].push(finalText)
    }
  })

  const total = Object.values(categories).reduce((sum, arr) => sum + arr.length, 0)
  return { ...categories, total }
})

const onInput = () => {
  if (!text.value.trim()) {
    showResponse.value = false
    displayText.value = ''
    clearTimeout(timerId)
    return
  }

  resetResponseTimer()
}

const resetResponseTimer = () => {
  clearTimeout(timerId)
  timerId = window.setTimeout(() => {
    if (text.value.trim().length > 0) {
      displayText.value = text.value
      showResponse.value = true
    }
  }, 2000)
}

const handleAction = () => {
  pressed.value = true
  setTimeout(() => {
    pressed.value = false
  }, 120)
}

onBeforeUnmount(() => {
  clearTimeout(timerId)
})
</script>

<style>
@import url('https://fonts.googleapis.com/css2?family=Patrick+Hand&display=swap');

:root {
  color-scheme: light;
  color: #3a3a3a;
  background: #f1ebe0;
}

body {
  margin: 0;
  font-family: 'Patrick Hand', 'Segoe Print', 'Comic Sans MS', 'Lucida Grande', sans-serif;
  font-size: 1rem;
  line-height: 1.8;
  background: radial-gradient(circle at top left, rgba(255,255,255,0.8), transparent 25%),
    linear-gradient(180deg, #f8f2e6 0%, #f5ede1 100%);
  color: #3a3a3a;
}

.app-layout {
  display: grid;
  grid-template-columns: 68% 32%;
  gap: 1.5rem;
  min-height: 100vh;
  padding: 2rem;
}

.lienzo,
.respuesta {
  position: relative;
  background: #fbf5e8;
  border-radius: 28px;
  box-shadow: 0 25px 60px rgba(58, 58, 58, 0.08);
  padding: 2rem;
  overflow: hidden;
}

.lienzo {
  min-height: 75vh;
  background-image:
    linear-gradient(0deg, rgba(249, 243, 229, 0.85) 0.5px, transparent 0.5px),
    radial-gradient(circle at 20% 10%, rgba(255,255,255,0.35), transparent 12%),
    radial-gradient(circle at 80% 90%, rgba(255,255,255,0.3), transparent 10%);
  background-size: 100% 2rem, cover;
  border: 1px solid rgba(60, 60, 60, 0.08);
}

.lienzo::before {
  content: '';
  position: absolute;
  top: 1.5rem;
  bottom: 1.5rem;
  left: 2.5rem;
  width: 2px;
  background: rgba(162, 60, 60, 0.18);
  opacity: 0.7;
}

.lienzo::after {
  content: '';
  position: absolute;
  top: 2rem;
  left: 1rem;
  width: 5px;
  height: calc(100% - 4rem);
  background-image: radial-gradient(circle, rgba(58,58,58,0.18) 12%, transparent 14%);
  background-size: 5px 36px;
  opacity: 0.5;
}

.editor {
  position: relative;
  width: 100%;
  min-height: 75vh;
  resize: none;
  border: none;
  outline: none;
  background: transparent;
  color: #3a3a3a;
  font: inherit;
  line-height: 1.9;
  padding: 2.5rem 2rem;
  z-index: 1;
}

.editor::placeholder {
  color: rgba(58, 58, 58, 0.45);
}

.respuesta {
  min-height: 75vh;
  display: flex;
  align-items: flex-start;
  justify-content: center;
  background: rgba(251, 245, 232, 0.7);
  border-left: 2px solid rgba(180, 150, 110, 0.22);
}

.respuesta-card {
  width: 100%;
  display: flex;
  flex-direction: column;
  justify-content: space-between;
  min-height: 70vh;
  padding: 1.5rem 1.5rem 1.2rem;
  background: transparent;
  border: none;
  border-radius: 20px;
  box-shadow: none;
  position: relative;
  z-index: 1;
}

.respuesta-body {
  opacity: 0.98;
  color: #3a3a3a;
  font-size: 1.05rem;
}

.response-title {
  margin: 0 0 0.8rem 0;
  font-size: 1.2rem;
  font-weight: 700;
}

.section-title {
  margin: 1rem 0 0.35rem;
  font-size: 1rem;
  font-weight: 700;
  letter-spacing: 0.02em;
  color: #3a3a3a;
}

.price-list {
  list-style: none;
  padding: 0;
  margin: 0;
}

.price-list li {
  display: block;
  padding: 0.4rem 0;
  border-bottom: 1px solid rgba(58, 58, 58, 0.08);
}

.price-list li:last-child {
  border-bottom: none;
}

.price-list li span {
  display: block;
  padding-left: 0.75rem;
  text-indent: -0.75rem;
}

.price-list li span::before {
  content: '•';
  margin-right: 0.55rem;
  color: rgba(58, 58, 58, 0.8);
}

.response-total {
  margin: 1.3rem 0 0;
  font-weight: 700;
}

.accion-btn {
  align-self: flex-start;
  margin-top: 1.5rem;
  padding: 0.75rem 1.4rem;
  background: rgba(255, 255, 255, 0.75);
  color: #3a3a3a;
  border: 1px solid rgba(60, 60, 60, 0.12);
  border-radius: 18px;
  cursor: pointer;
  transition: transform 120ms ease, background 220ms ease, border-color 220ms ease;
  font: inherit;
}

.accion-btn:hover {
  background: rgba(255, 255, 255, 0.95);
}

.accion-btn.pressed,
.accion-btn:active {
  transform: translateY(1px);
}

.fade-enter-active,
.fade-leave-active {
  transition: opacity 280ms ease, transform 280ms ease;
}

.fade-enter-from,
.fade-leave-to {
  opacity: 0;
  transform: translateY(8px);
}
</style>
