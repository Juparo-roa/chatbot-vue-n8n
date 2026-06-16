<template>
  <main class="page">
    <section class="chat">
      <header class="chat-header">
        <div>
          <h1>Asistente Virtual Inteligente</h1>
          <p>Soluciones guiadas con IA</p>
        </div>

        <span :class="['status', online ? 'online' : 'offline']">
          {{ online ? 'Activo' : 'Sin conexión' }}
        </span>
      </header>

      <div ref="messagesContainer" class="messages">
        <div
          v-for="(msg, index) in messages"
          :key="index"
          :class="['message', msg.role]"
        >
          <strong>{{ msg.role === 'user' ? 'Tú' : 'Asistente' }}</strong>
          <div v-html="msg.text"></div>
        </div>

        <div v-if="loading" class="message bot">
          <strong>Asistente</strong>
          <p>Analizando tu consulta...</p>
        </div>
      </div>

      <form @submit.prevent="sendMessage" class="form">
        <input
          v-model="userMessage"
          placeholder="Describe tu problema o pregunta..."
          :disabled="loading"
        />

        <button type="submit" :disabled="loading">
          {{ loading ? 'Enviando...' : 'Enviar' }}
        </button>
      </form>

      <div class="actions">
        <button class="secondary" @click="clearChat">
          Limpiar conversación
        </button>
      </div>
    </section>
  </main>
</template>

<script setup>
import { ref, nextTick } from 'vue'
import { marked } from 'marked'

const userMessage = ref('')
const loading = ref(false)
const online = ref(true)
const messagesContainer = ref(null)

const sessionId = crypto.randomUUID()

const messages = ref([
  {
    role: 'bot',
    text: 'Hola, soy tu asistente virtual inteligente. Describe tu problema y te ayudaré paso a paso.'
  }
])

const scrollToBottom = async () => {
  await nextTick()

  if (messagesContainer.value) {
    messagesContainer.value.scrollTop = messagesContainer.value.scrollHeight
  }
}

const sendMessage = async () => {
  const text = userMessage.value.trim()

  if (!text || loading.value) return

  messages.value.push({
    role: 'user',
    text
  })

  userMessage.value = ''
  loading.value = true

  await scrollToBottom()

  const controller = new AbortController()
  const timeout = setTimeout(() => controller.abort(), 30000)

  try {
  const API_URL = import.meta.env.VITE_API_URL || 'http://localhost:5678';

  console.log("API_URL:", API_URL);
  
  const response = await fetch(`${API_URL}/webhook/chatbot`, {
  method: 'POST',
  headers: {
    'Content-Type': 'application/json'
    },
  body: JSON.stringify({
    message: text,
    sessionId,
    timestamp: new Date().toISOString()
    }),
  signal: controller.signal
  });

    clearTimeout(timeout)

    if (!response.ok) {
      throw new Error('El servidor no respondió correctamente')
    }

    const data = await response.json()

    messages.value.push({
      role: 'bot',
      text: marked.parse(data.reply || 'No recibí una respuesta válida.')
    })

    if (!data.success) {
      throw new Error('La respuesta del asistente no fue exitosa')
    }

    online.value = true
  } catch (error) {
    messages.value.push({
      role: 'bot',
      text: 'No pude conectarme con el asistente en este momento. Revisa n8n, Groq o intenta nuevamente en unos segundos.'
    })

    online.value = false
  } finally {
    loading.value = false
    await scrollToBottom()
  }
}

const clearChat = () => {
  messages.value = [
    {
      role: 'bot',
      text: 'Conversación reiniciada. Describe tu problema y te ayudaré.'
    }
  ]

  online.value = true
}
</script>

<style>
body {
  margin: 0;
  font-family: Arial, sans-serif;
  background: #f3f4f6;
}

.page {
  min-height: 100vh;
  display: grid;
  place-items: center;
}

.chat {
  width: 560px;
  background: white;
  padding: 24px;
  border-radius: 18px;
  box-shadow: 0 15px 40px rgba(0, 0, 0, 0.12);
}

.chat-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
}

.chat-header h1 {
  margin: 0;
  font-size: 28px;
}

.chat-header p {
  margin: 6px 0 0;
  color: #666;
}

.status {
  padding: 6px 10px;
  border-radius: 999px;
  font-size: 12px;
  color: white;
}

.status.online {
  background: #16a34a;
}

.status.offline {
  background: #dc2626;
}

.messages {
  height: 420px;
  overflow-y: auto;
  border: 1px solid #ddd;
  padding: 16px;
  border-radius: 14px;
  margin: 18px 0;
  background: #fafafa;
}

.message {
  margin-bottom: 14px;
  padding: 12px 14px;
  border-radius: 12px;
  line-height: 1.4;
}

.message strong {
  display: block;
  margin-bottom: 4px;
  font-size: 13px;
}

.message p {
  margin: 0;
  white-space: pre-line;
}

.message div {
  line-height: 1.6;
}

.message ul,
.message ol {
  padding-left: 20px;
}

.message pre {
  background: #f4f4f4;
  padding: 10px;
  border-radius: 8px;
  overflow-x: auto;
}

.message code {
  background: #f4f4f4;
  padding: 2px 4px;
  border-radius: 4px;
}

.user {
  background: #dbeafe;
  text-align: right;
}

.bot {
  background: #e5e7eb;
}

.form {
  display: flex;
  gap: 10px;
}

input {
  flex: 1;
  padding: 12px;
  border: 1px solid #aaa;
  border-radius: 8px;
}

button {
  padding: 12px 18px;
  border: none;
  border-radius: 8px;
  cursor: pointer;
  background: #111827;
  color: white;
}

button:disabled {
  background: #9ca3af;
  cursor: not-allowed;
}

.actions {
  margin-top: 12px;
}

.secondary {
  width: 100%;
  background: #ef4444;
}
</style>