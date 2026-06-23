```vue
<template>
  <main class="page">
    <section class="chat">
      <!-- Encabezado -->
      <header class="chat-header">
        <div>
          <h1>AVI Rewdev</h1>
          <p>Soluciones guiadas con IA</p>
        </div>

        <span :class="['status', online ? 'online' : 'offline']">
          {{ online ? 'Activo' : 'Sin conexión' }}
        </span>
      </header>

      <!-- Conversación -->
      <div ref="messagesContainer" class="messages">
        <div v-for="(msg, index) in messages" :key="index" :class="['message', msg.role]">
          <strong>
            {{ msg.role === 'user' ? 'Tú' : 'Asistente' }}
          </strong>

          <div v-html="msg.text"></div>
        </div>

        <div v-if="loading" class="message bot">
          <strong>Asistente</strong>
          <p>Analizando tu consulta...</p>
        </div>
      </div>

      <!-- Formulario -->
      <form class="form" @submit.prevent="sendMessage">
        <input v-model="userMessage" type="text" placeholder="Escribe tu consulta..." :disabled="loading" />

        <button type="submit" :disabled="loading">
          {{ loading ? "Enviando..." : "Enviar" }}
        </button>
      </form>

      <!-- Botón limpiar -->
      <div class="actions" v-if="messages.some(msg => msg.role === 'user')">
        <button class="secondary" type="button" @click="clearChat">
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
  background: #0f0f0f;
}

.page {
  width: 100%;
  min-height: 100vh;

  display: flex;
  justify-content: flex-end;
  align-items: center;

  padding-right: 40px;
  box-sizing: border-box;

  background: transparent;
}

.chat {
  
  width: min(350px, calc(100vw - 24px));
  height: min(500px, calc(100vh - 24px));

  display: flex;
  flex-direction: column;

  background: #1a1a1a;
  color: #ffffff;

  padding: 12px;
  border-radius: 16px;

  border: 1px solid #2e2e2e;
  box-shadow: 0 10px 30px rgba(0, 0, 0, 0.35);

}

.chat-header {
  display: flex;
  justify-content: space-between;
  align-items: center;

  background: #111111;
  color: white;

  padding: 12px;
  border-radius: 12px;
}

.chat-header h1 {
  margin: 0;
  font-size: 20px;
}

.chat-header p {
  margin: 6px 0 0;
  font-size: 12px;
  color: #b3b3b3;
}

.status {
  padding: 6px 10px;
  border-radius: 999px;

  background: #16a34a;
  color: white;

  font-weight: bold;
}

.status.online {
  background: #16a34a;
}

.status.offline {
  background: #dc2626;
}

.messages {
  flex: 1;
  min-height: 0;
  overflow-y: auto;

  border: 1px solid #2e2e2e;
  border-radius: 12px;

  padding: 10px;
  margin: 12px 0;

  background: #181818;
}

.message {
  margin-bottom: 14px;
  padding: 8px 10px;
  font-size: 13px;
  border-radius: 16px;
  line-height: 1.5;
  background: #2a2a2a;
  color: #ffffff;
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
  background: #111111;
  color: #ffffff;
  padding: 10px;
  border-radius: 8px;
  overflow-x: auto;
}

.message code {
  background: #2d2d2d;
  color: #60a5fa;
  padding: 2px 4px;
  border-radius: 4px;
}

.user {
  background: #2563eb;
  color: white;
}

.bot {
  background: #2b2b2b;
  color: white;
}

.form {
  display: flex;
  gap: 8px;
}

.form input {
  flex: 1;
}

.form button {
  width: 70px;
  flex-shrink: 0;
}

input {
  flex: 1;
  padding: 8px 10px;
  border: 1px solid #333;
  border-radius: 10px;

  background: #111111;
  color: #ffffff;

}

input::placeholder {
  color: #888;
}

button {
  padding: 8px 12px;
  border: none;
  border-radius: 8px;
  cursor: pointer;
  background: #2563eb;
  color: white;
  transition: background 0.2s ease;
}

button:hover {
  background: #1d4ed8;
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
  background: #dc2626;
  color: white;
  transition: background 0.2s ease;
}

.secondary:hover {
  background: #b91c1c;
}
</style>