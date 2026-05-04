<script setup>
import { ref, onMounted, nextTick } from 'vue';
import axios from 'axios';
import { 
  MessageSquare, 
  MoreVertical, 
  Search, 
  Paperclip, 
  Smile, 
  Mic, 
  Send,
  RefreshCw
} from 'lucide-vue-next';

const messages = ref([]);
const newMessage = ref('');
const messageContainer = ref(null);
const fileInput = ref(null);
const isTyping = ref(false);

const N8N_WEBHOOK_URL = 'https://dmncrt.app.n8n.cloud/webhook/agente-novacasa';
const API_URL = import.meta.env.VITE_API_URL || 'https://notaria-server.vercel.app/api';

// --- Session Management ---
const getOrGenerateSessionID = () => {
  let id = localStorage.getItem('whatsapp_session_id');
  if (!id) {
    const chars = 'ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz0123456789';
    id = '';
    for (let i = 0; i < 9; i++) {
      id += chars.charAt(Math.floor(Math.random() * chars.length));
    }
    localStorage.setItem('whatsapp_session_id', id);
  }
  return id;
};

const currentSessionID = ref(getOrGenerateSessionID());
// --------------------------

const fetchMessages = async () => {
  try {
    const response = await axios.get(`${API_URL}/messages`);
    const serverMessages = response.data;
    
    // Logic to prevent optimistic messages from disappearing:
    // Only update if:
    // 1. The server has MORE messages than we have locally.
    // 2. OR the server has the SAME amount but the content of the last one is different (it became an incoming message).
    // 3. AND never clear the list if the server returns 0 but we have local messages pending.
    
    const lastLocal = messages.value[messages.value.length - 1];
    const lastServer = serverMessages[serverMessages.length - 1];

    const hasNewContent = lastLocal && lastServer && lastLocal.text !== lastServer.text;
    const serverHasMore = serverMessages.length > messages.value.length;
    const serverHasSameButDiff = (serverMessages.length === messages.value.length) && hasNewContent;

    if (serverHasMore || serverHasSameButDiff || (messages.value.length === 0 && serverMessages.length > 0)) {
      messages.value = serverMessages;
      scrollToBottom();
    }
  } catch (error) {
    console.error('Error fetching messages:', error);
  }
};

const processBotMessages = (respuesta, idOffset = 0) => {
  const mensajesSeparados = respuesta.split('[SEP]');
  
  mensajesSeparados.forEach((texto, index) => {
    let currentText = texto.trim();
    if (currentText) {
      let imageUrl = null;
      
      // 1. Extraer imágenes en formato Markdown: ![alt](url)
      const markdownImageRegex = /!\[.*?\]\((https?:\/\/[^\s)]+)\)/;
      const markdownMatch = currentText.match(markdownImageRegex);
      
      if (markdownMatch) {
        imageUrl = markdownMatch[1];
        // Remover el markdown completo del texto
        currentText = currentText.replace(markdownMatch[0], '').trim();
      } else {
        // 2. Fallback para URLs planas
        const urlRegex = /(https?:\/\/[^\s)]+)/g;
        const urls = currentText.match(urlRegex);
        
        if (urls) {
          // Detecta si la URL es de una imagen (DALL-E, Cloudinary, o tiene extensión de imagen)
          const imageRegex = /\.(jpeg|jpg|gif|png|webp)|oaidalleapiprodscus\.blob\.core\.windows\.net|res\.cloudinary\.com/i;
          for (let i = 0; i < urls.length; i++) {
            if (imageRegex.test(urls[i])) {
              imageUrl = urls[i];
              // Quita la URL del texto para que no se vea duplicada
              currentText = currentText.replace(urls[i], '').trim();
              break; // Extrae la primera imagen
            }
          }
        }
      }

      if (currentText || imageUrl) {
        messages.value.push({
          id: Date.now() + index + idOffset,
          sender: 'Bot',
          text: currentText,
          image: imageUrl,
          timestamp: new Date(),
          incoming: true
        });
      }
    }
  });
};

const sendMessage = async () => {
  if (!newMessage.value.trim()) return;

  const text = newMessage.value;
  const tempId = Date.now(); // Temporary ID for immediate UI feedback
  
  // Add to UI immediately for better UX
  const localMsg = {
    id: tempId,
    sender: 'Me',
    text,
    timestamp: new Date(),
    incoming: false
  };
  
  messages.value.push(localMsg);
  newMessage.value = '';
  scrollToBottom();
  isTyping.value = true;

  try {
    const response = await axios.post(N8N_WEBHOOK_URL, { 
      message: text, 
      sessionId: currentSessionID.value 
    });
    
    const datosRecibidos = response.data;
    const respuesta = datosRecibidos.respuesta_bot || datosRecibidos.reply || datosRecibidos.output || (typeof datosRecibidos === 'string' ? datosRecibidos : '');
    processBotMessages(respuesta, 0);
    scrollToBottom();
  } catch (error) {
    console.error('Error sending message to n8n:', error);
    // Remove the message if it failed to send (optional, or show error)
    // messages.value = messages.value.filter(m => m.id !== tempId);
    
    const errorMsg = {
      id: Date.now() + 2,
      sender: 'System',
      text: 'Error al conectar con el asistente. Por favor intenta de nuevo.',
      timestamp: new Date(),
      incoming: true,
      isError: true
    };
    messages.value.push(errorMsg);
    scrollToBottom();
  } finally {
    isTyping.value = false;
    scrollToBottom();
  }
};

const triggerFileUpload = () => {
  fileInput.value.click();
};

const onFileSelected = async (event) => {
  const file = event.target.files[0];
  if (!file) return;

  const validTypes = ['image/jpeg', 'image/png', 'image/webp'];
  if (!validTypes.includes(file.type)) {
    alert('Por favor selecciona una imagen en formato .jpg, .png o .webp');
    if (fileInput.value) {
      fileInput.value.value = '';
    }
    return;
  }

  // Mensaje optimista
  const tempId = Date.now();
  const localMsg = {
    id: tempId,
    sender: 'Me',
    text: 'Subiendo imagen...',
    image: null,
    timestamp: new Date(),
    incoming: false
  };
  messages.value.push(localMsg);
  scrollToBottom();
  isTyping.value = true;

  try {
    const formData = new FormData();
    formData.append('file', file);
    // IMPORTANTE: Debes crear un "Upload Preset" de tipo "Unsigned" en Cloudinary
    formData.append('upload_preset', 'novacasa_unsigned'); 
    formData.append('cloud_name', 'djlphhqgb');

    // Subir a Cloudinary
    const cloudinaryRes = await axios.post(
      'https://api.cloudinary.com/v1_1/djlphhqgb/image/upload',
      formData
    );

    const imageUrl = cloudinaryRes.data.secure_url;

    // Actualizar mensaje local con la imagen
    const msgIndex = messages.value.findIndex(m => m.id === tempId);
    if (msgIndex !== -1) {
      messages.value[msgIndex].text = '';
      messages.value[msgIndex].image = imageUrl;
    }
    scrollToBottom();

    // Enviar la URL de la imagen al agente n8n
    const n8nRes = await axios.post(N8N_WEBHOOK_URL, {
      message: `[El usuario ha enviado una imagen: ${imageUrl}]`,
      imageUrl: imageUrl,
      sessionId: currentSessionID.value
    });

    const datosRecibidos = n8nRes.data;
    const respuesta = datosRecibidos.respuesta_bot || datosRecibidos.reply || datosRecibidos.output || (typeof datosRecibidos === 'string' ? datosRecibidos : '');
    processBotMessages(respuesta, 100);

  } catch (error) {
    console.error('Error al subir la imagen o contactar a n8n:', error);
    const msgIndex = messages.value.findIndex(m => m.id === tempId);
    if (msgIndex !== -1) {
      messages.value[msgIndex].text = 'Error al subir la imagen.';
    }
  } finally {
    isTyping.value = false;
    scrollToBottom();
    // Limpiar el input para permitir subir la misma imagen de nuevo
    if (fileInput.value) {
      fileInput.value.value = '';
    }
  }
};

const scrollToBottom = async () => {
  await nextTick();
  if (messageContainer.value) {
    messageContainer.value.scrollTop = messageContainer.value.scrollHeight;
  }
};

const resetChat = async () => {
  if (confirm('¿Estás seguro de que quieres reiniciar el chat y generar un nuevo SessionID?')) {
    try {
      try {
        await axios.post(`${API_URL}/reset`);
      } catch (e) {
        // Ignore if the reset endpoint doesn't exist
      }
      localStorage.removeItem('whatsapp_session_id');
      currentSessionID.value = getOrGenerateSessionID();
      messages.value = [];
    } catch (error) {
      console.error('Error al reiniciar el chat:', error);
    }
  }
};

const formatTime = (timestamp) => {
  const date = new Date(timestamp);
  return date.toLocaleTimeString([], { hour: '2-digit', minute: '2-digit' });
};

const formatMessage = (text) => {
  if (!text) return '';
  // Reemplaza " - " con un salto de línea seguido de "- " para que se muestre como lista
  return text.replace(/ - /g, '\n- ');
};

onMounted(() => {
  // fetchMessages(); // Desactivamos el polling ya que ahora usamos n8n directo
  // setInterval(fetchMessages, 3000);
});
</script>

<template>
  <div class="whatsapp-app">
    <!-- Sidebar -->
    <div class="sidebar">
      <header class="sidebar-header">
        <div class="avatar">R</div>
        <div class="actions">
          <MessageSquare class="icon" />
          <MoreVertical class="icon" />
        </div>
      </header>
      
      <div class="search-container">
        <div class="search-bar">
          <Search class="icon-small" />
          <input type="text" placeholder="Buscar o empezar un chat nuevo" />
        </div>
      </div>
      
      <div class="chat-list">
        <div class="chat-item active">
          <div class="avatar profile-pic"></div>
          <div class="chat-info">
            <div class="chat-name">novacasa</div>
            <div class="chat-last-msg">
              <span v-if="messages.length > 0">
                {{ messages[messages.length - 1].text || 'Imagen' }}
              </span>
              <span v-else>No hay mensajes</span>
            </div>
          </div>
        </div>
        
        <!-- Placeholder chats -->
        <div class="chat-item" v-for="i in 5" :key="i">
          <div class="avatar" :style="{ background: `hsl(${i * 60}, 70%, 40%)` }">{{ String.fromCharCode(65 + i) }}</div>
          <div class="chat-info">
            <div class="chat-name">Contacto {{ i }}</div>
            <div class="chat-last-msg">Último mensaje del contacto...</div>
          </div>
        </div>
      </div>
    </div>

    <!-- Main Chat Window -->
    <div class="main-chat">
      <header class="chat-header">
        <div class="avatar profile-pic"></div>
        <div class="contact-info">
          <div class="name">novacasa</div>
          <div class="status">en línea</div>
        </div>
        <div class="actions">
          <RefreshCw class="icon" @click="resetChat" title="Reiniciar chat y sesión" />
          <MoreVertical class="icon" />
        </div>
      </header>

      <div class="message-container" ref="messageContainer">
        <div 
          v-for="msg in messages" 
          :key="msg.id" 
          :class="['message-bubble', msg.incoming ? 'incoming' : 'outgoing']"
        >
          <div v-if="msg.image" class="message-image">
            <img :src="msg.image" alt="Uploaded image" />
          </div>
          <div v-if="msg.text" class="message-text">{{ formatMessage(msg.text) }}</div>
          <div class="message-time">{{ formatTime(msg.timestamp) }}</div>
        </div>
        
        <!-- Typing Indicator -->
        <div v-if="isTyping" class="message-bubble incoming typing-indicator">
          <div class="dots">
            <span></span>
            <span></span>
            <span></span>
          </div>
        </div>
      </div>

      <footer class="chat-footer">
        <input type="file" ref="fileInput" hidden accept=".jpg,.jpeg,.png,.webp,image/jpeg,image/png,image/webp" @change="onFileSelected" />
        <Smile class="icon" />
        <Paperclip class="icon" @click="triggerFileUpload" />
        <input 
          v-model="newMessage" 
          @keyup.enter="sendMessage" 
          type="text" 
          placeholder="Escribe un mensaje aquí" 
        />
        <Send 
          v-if="newMessage.trim()" 
          @click="sendMessage" 
          class="icon" 
          style="color: #00a884" 
        />
        <Mic v-else class="icon" />
      </footer>
    </div>
  </div>
</template>

<style>
/* Style is in style.css, but App.vue might need some local overrides or we just import it in main.js */
</style>
