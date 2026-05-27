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
          <div class="respuesta-body">
            <template v-if="taskList">
              <p class="response-title">Plan de tareas</p>
              <ul class="price-list">
                <li v-for="item in taskPlan.items" :key="item.name">
                  <span>{{ item.name }}</span>
                </li>
              </ul>
              <p class="response-total">Total de tareas: <strong>{{ taskPlan.total }}</strong></p>
            </template>
            <template v-else>
              <p>Respuesta</p>
            </template>
          </div>
          <button
            class="accion-btn"
            @click="handleAction"
            @mousedown="pressed = true"
            @mouseup="pressed = false"
            @mouseleave="pressed = false"
            :class="{ pressed }"
          >
            {{ taskList ? 'Ver agenda' : 'Acción' }}
          </button>
        </div>
      </transition>
    </aside>
  </main>
</template>

<script setup>
import { ref, computed, onBeforeUnmount } from 'vue'

const text = ref('')
const showResponse = ref(false)
const pressed = ref(false)
let timerId = null

const taskKeywords = [
  'reunión', 'reunion', 'meeting', 'cita', 'call', 'llamada',
  'tarea', 'pendiente', 'agenda', 'recordar', 'hoy', 'mañana', 'manana', 'horario'
]

const timeRegex = /\b(?:hoy|mañana|manana|esta semana|este viernes|el lunes|a las \d{1,2}(?::\d{2})?|\b\d{1,2}:\d{2}\b|\b\d{1,2}h\b)\b/i
const listMarkerRegex = /^[\-\*\d\.]+\s+/
const sentenceSeparatorRegex = /[,;•]+|\s+y\s+|\s+and\s+/i

const isTaskLike = (content) => {
  return taskKeywords.some((word) => content.includes(word)) || timeRegex.test(content)
}

const taskList = computed(() => {
  const content = text.value.toLowerCase().trim()
  if (!content) return false

  const lines = content.split(/\r?\n/).map((line) => line.trim()).filter(Boolean)
  const listStyle = lines.some((line) => listMarkerRegex.test(line) || line.includes(','))
  const hasTaskWord = isTaskLike(content)

  return hasTaskWord && (lines.length > 1 || listStyle || timeRegex.test(content))
})

const taskPlan = computed(() => {
  if (!taskList.value) return { items: [], total: 0 }

  const rawSegments = text.value
    .split(/\r?\n/)
    .flatMap((line) => line.split(sentenceSeparatorRegex))
    .map((segment) => segment.trim().replace(listMarkerRegex, ''))
    .filter((segment) => segment.length > 0)

  const items = rawSegments
    .map((segment) => {
      const cleaned = segment
        .replace(/\b(a las|por la tarde|por la mañana|en la tarde|en la mañana|esta semana|el lunes|el martes|el miércoles|el jueves|el viernes|el sábado|el domingo)\b/gi, '')
        .replace(/\b(hoy|mañana|manana)\b/gi, '')
        .trim()
      return { name: cleaned || segment }
    })
    .filter((item) => item.name.length > 0)

  const uniqueItems = Array.from(new Set(items.map((item) => item.name))).map((name) => ({ name }))

  return { items: uniqueItems.slice(0, 8), total: uniqueItems.length }
})

const onInput = () => {
  showResponse.value = false
  resetResponseTimer()
}

const resetResponseTimer = () => {
  clearTimeout(timerId)
  timerId = window.setTimeout(() => {
    if (text.value.trim().length > 0) {
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
  background: #f7f5ef;
}

body {
  margin: 0;
  font-family: 'Patrick Hand', 'Segoe Print', 'Comic Sans MS', 'Lucida Grande', sans-serif;
  font-size: 1rem;
  line-height: 1.75;
  background: #fefcf8;
}

.app-layout {
  display: grid;
  grid-template-columns: 70% auto;
  gap: 0;
  min-height: 100vh;
  padding: 2rem;
  background: #fefcf8;
}

.lienzo,
.respuesta {
  background: transparent;
  border-radius: 0;
  box-shadow: none;
  padding: 2rem;
}

.lienzo {
  min-height: 75vh;
  border-right: 1px solid rgba(60, 60, 60, 0.12);
}

.editor {
  width: 100%;
  min-height: 75vh;
  resize: none;
  border: none;
  outline: none;
  background: transparent;
  color: #3a3a3a;
  font: inherit;
  line-height: 1.8;
  padding: 0;
}

.editor::placeholder {
  color: rgba(58, 58, 58, 0.5);
}

.respuesta {
  min-height: 75vh;
  display: flex;
  align-items: flex-start;
  justify-content: center;
}

.respuesta-card {
  width: 100%;
  display: flex;
  flex-direction: column;
  justify-content: space-between;
  min-height: 70vh;
  padding: 1.5rem 0;
}

.respuesta-body {
  opacity: 0.98;
  color: #3a3a3a;
  font-size: 1.03rem;
}

.response-title {
  margin: 0 0 1rem 0;
  font-weight: 700;
}

.price-list {
  list-style: none;
  padding: 0;
  margin: 0 0 1rem 0;
}

.price-list li {
  display: flex;
  justify-content: space-between;
  padding: 0.4rem 0;
  border-bottom: 1px dashed rgba(58, 58, 58, 0.14);
}

.price-list li:last-child {
  border-bottom: none;
}

.response-total {
  margin: 1rem 0 0;
  font-weight: 700;
}

.accion-btn {
  align-self: flex-start;
  margin-top: auto;
  padding: 0.9rem 1.6rem;
  background: rgba(60, 60, 60, 0.06);
  color: #3a3a3a;
  border: 1px solid rgba(60, 60, 60, 0.12);
  border-radius: 999px;
  cursor: pointer;
  transition: background 220ms ease, transform 120ms ease, color 220ms ease, border-color 220ms ease, box-shadow 220ms ease;
  font: inherit;
}

.accion-btn:hover {
  background: rgba(60, 60, 60, 0.12);
  box-shadow: 0 4px 10px rgba(60, 60, 60, 0.08);
}

.accion-btn.pressed,
.accion-btn:active {
  background: rgba(60, 60, 60, 0.18);
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
