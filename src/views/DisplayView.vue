<template>
  <div class="w-screen h-screen text-white flex flex-col p-6 m-0 overflow-hidden" style="background: linear-gradient(135deg, #1e3c72 0%, #2a5298 100%);">
    
    <!-- Header: Clima y Hora -->
    <div class="flex justify-between items-center mb-6 pb-4 border-b-4 border-white/30">
      <!-- Espacio reservado para clima -->
      <div class="bg-white/10 backdrop-blur-lg rounded-2xl px-8 py-4 flex items-center gap-4" style="min-width: 250px;">
        <div class="text-5xl">🌤️</div>
        <div>
          <div class="text-xl font-semibold">Clima</div>
          <div class="text-sm opacity-75">Próximamente</div>
        </div>
      </div>
      
      <!-- Hora y fecha -->
      <div class="flex flex-col items-end">
        <div class="text-5xl font-bold tracking-widest">{{ currentTime }}</div>
        <div class="text-lg opacity-75">{{ currentDate }}</div>
      </div>
    </div>

    <div class="flex-1 flex gap-6 overflow-hidden">
      <!-- Columna izquierda: Lista de turnos llamados (60%) -->
      <div class="flex flex-col" style="width: 60%;">
        <h2 class="text-2xl text-center mb-2 bg-white/20 p-2 rounded-lg font-bold">
          TURNOS LLAMADOS
        </h2>
        
        <div v-if="recentCalled.length > 0" class="flex flex-col gap-2 overflow-y-auto custom-scrollbar pr-2 pl-1">
          <div
            v-for="(turn, index) in recentCalled"
            :key="turn.id"
            class="bg-white rounded-xl flex items-center justify-between transition-all"
            :class="{ 
              'blink-card': turn.status === 'called'
            }"
            :style="{ 
              padding: index === 0 ? '1.1rem 1.25rem' : '0.875rem 1rem'
            }"
          >
            <!-- Número del turno -->
            <div class="text-gray-800 font-black" :style="{ fontSize: index === 0 ? '2.6rem' : '2.1rem' }">
              {{ turn.ticket_number }}
            </div>
            
            <!-- Ventanilla -->
            <div v-if="turn.window" class="flex flex-col items-end">
              <div class="text-gray-500 font-semibold" :style="{ fontSize: index === 0 ? '1rem' : '0.85rem' }">
                Ventanilla
              </div>
              <div 
                class="text-gray-800 font-black"
                :style="{ fontSize: index === 0 ? '1.9rem' : '1.6rem' }"
              >
                {{ turn.window.name }}
              </div>
            </div>
          </div>
        </div>
        
        <div v-else class="flex-1 flex items-center justify-center opacity-60">
          <div class="text-center">
            <div class="text-6xl mb-4">⏳</div>
            <p class="text-2xl">Esperando próximo llamado...</p>
          </div>
        </div>
      </div>

      <!-- Columna derecha: Publicidad (40%) -->
      <div class="flex flex-col" style="width: 40%;">
        <h2 class="text-3xl text-center mb-4 bg-white/20 p-4 rounded-lg font-bold">
          INFORMACIÓN
        </h2>
        
        <!-- Espacio para publicidad -->
        <div class="flex-1 bg-white/10 backdrop-blur-lg rounded-2xl p-8 flex flex-col items-center justify-center gap-6">
          <div class="text-center">
            <div class="text-7xl mb-6">📢</div>
            <p class="text-2xl font-semibold mb-3">Espacio publicitario</p>
            <p class="text-lg opacity-75">Próximamente contenido personalizado</p>
          </div>
          
          <!-- Mensaje informativo -->
          <div class="mt-8 text-center border-t-2 border-white/30 pt-6 w-full">
            <p class="text-xl opacity-90">Por favor, permanezca atento al llamado de su turno</p>
          </div>
        </div>
      </div>
    </div>
  </div>
</template>

<script setup>
import { ref, onMounted, onUnmounted, computed, watch } from 'vue';
import { getQueue } from '../api';

const waiting = ref([]);
const inProgress = ref([]);
const recentCalled = ref([]); // Nueva: lista de turnos llamados
const currentTime = ref('');
const currentDate = ref('');
const lastCalledId = ref(null); // Para detectar nuevo turno llamado

const USE_SSE = false; // Cambiar a true para usar SSE en vez de polling

let eventSource = null;
let intervalId = null;
let clockId = null;

// Turno más reciente llamado (el primero de la lista)
const lastCalledTurn = computed(() => {
  return recentCalled.value.length > 0 ? recentCalled.value[0] : null;
});

// Función para reproducir sonido de notificación
const playNotificationSound = () => {
  // Crear un contexto de audio
  const audioContext = new (window.AudioContext || window.webkitAudioContext)();
  
  // Crear un oscilador (generador de tono)
  const oscillator = audioContext.createOscillator();
  const gainNode = audioContext.createGain();
  
  // Conectar el oscilador al gain y al destino
  oscillator.connect(gainNode);
  gainNode.connect(audioContext.destination);
  
  // Configurar el sonido (tono de notificación agradable)
  oscillator.type = 'sine';
  oscillator.frequency.setValueAtTime(800, audioContext.currentTime); // Frecuencia inicial
  oscillator.frequency.exponentialRampToValueAtTime(600, audioContext.currentTime + 0.1);
  
  // Configurar el volumen (fade in/out)
  gainNode.gain.setValueAtTime(0, audioContext.currentTime);
  gainNode.gain.linearRampToValueAtTime(0.3, audioContext.currentTime + 0.05);
  gainNode.gain.exponentialRampToValueAtTime(0.01, audioContext.currentTime + 0.5);
  
  // Reproducir el sonido
  oscillator.start(audioContext.currentTime);
  oscillator.stop(audioContext.currentTime + 0.5);
  
  // Segundo tono (eco)
  setTimeout(() => {
    const oscillator2 = audioContext.createOscillator();
    const gainNode2 = audioContext.createGain();
    
    oscillator2.connect(gainNode2);
    gainNode2.connect(audioContext.destination);
    
    oscillator2.type = 'sine';
    oscillator2.frequency.setValueAtTime(900, audioContext.currentTime);
    oscillator2.frequency.exponentialRampToValueAtTime(700, audioContext.currentTime + 0.1);
    
    gainNode2.gain.setValueAtTime(0, audioContext.currentTime);
    gainNode2.gain.linearRampToValueAtTime(0.25, audioContext.currentTime + 0.05);
    gainNode2.gain.exponentialRampToValueAtTime(0.01, audioContext.currentTime + 0.4);
    
    oscillator2.start(audioContext.currentTime);
    oscillator2.stop(audioContext.currentTime + 0.4);
  }, 150);
  
  console.log('🔔 Sonido de notificación reproducido');
};

const loadQueue = async () => {
  try {
    const response = await getQueue();
    waiting.value = response.data.waiting || [];
    inProgress.value = response.data.in_progress || [];
    
    const newRecentCalled = response.data.recent_called || [];
    
    // Detectar si hay un nuevo turno llamado (comparar el ID del primero)
    if (newRecentCalled.length > 0) {
      const newFirstId = newRecentCalled[0].id;
      
      // Si hay un turno nuevo y está en estado "called", reproducir sonido
      if (lastCalledId.value !== null && 
          newFirstId !== lastCalledId.value && 
          newRecentCalled[0].status === 'called') {
        playNotificationSound();
        console.log('🆕 Nuevo turno llamado:', newRecentCalled[0].ticket_number);
      }
      
      lastCalledId.value = newFirstId;
    }
    
    recentCalled.value = newRecentCalled;
    
  } catch (error) {
    console.error('Error cargando cola:', error);
  }
};

const connectSSE = () => {
  if (eventSource) {
    eventSource.close();
  }

  eventSource = new EventSource('http://192.168.0.51:8000/api/sse/display');

  eventSource.onmessage = (event) => {
    try {
      const data = JSON.parse(event.data);
      waiting.value = data.waiting || [];
      inProgress.value = data.in_progress || [];
      
      const newRecentCalled = data.recent_called || [];
      
      // Detectar nuevo turno llamado
      if (newRecentCalled.length > 0) {
        const newFirstId = newRecentCalled[0].id;
        
        if (lastCalledId.value !== null && 
            newFirstId !== lastCalledId.value && 
            newRecentCalled[0].status === 'called') {
          playNotificationSound();
          console.log('🆕 Nuevo turno llamado (SSE):', newRecentCalled[0].ticket_number);
        }
        
        lastCalledId.value = newFirstId;
      }
      
      recentCalled.value = newRecentCalled;
      
      console.log('✅ SSE: Datos actualizados');
    } catch (error) {
      console.error('Error parseando SSE:', error);
    }
  };

  eventSource.onerror = (error) => {
    console.error('❌ Error SSE, volviendo a polling...', error);
    eventSource.close();
    eventSource = null;
    
    // Volver a polling
    intervalId = setInterval(loadQueue, 3000);
  };

  eventSource.onopen = () => {
    console.log('🟢 SSE conectado');
  };
};

const updateClock = () => {
  const now = new Date();
  currentTime.value = now.toLocaleTimeString('es-AR', {
    hour: '2-digit',
    minute: '2-digit',
    second: '2-digit'
  });
  currentDate.value = now.toLocaleDateString('es-AR', {
    weekday: 'long',
    year: 'numeric',
    month: 'long',
    day: 'numeric'
  });
};

const formatTime = (datetime) => {
  if (!datetime) return '';
  return new Date(datetime).toLocaleTimeString('es-AR', {
    hour: '2-digit',
    minute: '2-digit'
  });
};

onMounted(() => {
  loadQueue(); // Carga inicial
  updateClock();
  
  // Elegir método de actualización
  if (USE_SSE) {
    connectSSE();
  } else {
    intervalId = setInterval(loadQueue, 2000); // Polling cada 2 segundos (más frecuente para mejor respuesta)
  }
  
  clockId = setInterval(updateClock, 1000);
});

onUnmounted(() => {
  if (eventSource) {
    eventSource.close();
  }
  if (intervalId) clearInterval(intervalId);
  if (clockId) clearInterval(clockId);
});
</script>

<style scoped>
/* Animación de titileo para la card del turno llamado (blanco-gris oscuro) */
@keyframes blinkCard {
  0%, 100% {
    background-color: #ffffff;
    box-shadow: 0 8px 30px rgba(0, 0, 0, 0.15);
  }
  50% {
    background-color: #9ca3af;
    box-shadow: 0 12px 40px rgba(0, 0, 0, 0.35);
  }
}

.blink-card {
  animation: blinkCard 0.8s ease-in-out infinite;
}

/* Scrollbar personalizado para la lista de turnos */
.custom-scrollbar::-webkit-scrollbar {
  width: 8px;
}

.custom-scrollbar::-webkit-scrollbar-track {
  background: rgba(255, 255, 255, 0.1);
  border-radius: 10px;
}

.custom-scrollbar::-webkit-scrollbar-thumb {
  background: rgba(255, 255, 255, 0.3);
  border-radius: 10px;
}

.custom-scrollbar::-webkit-scrollbar-thumb:hover {
  background: rgba(255, 255, 255, 0.5);
}
</style>
