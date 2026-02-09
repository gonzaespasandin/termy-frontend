<template>
  <div class="w-screen h-screen text-white flex flex-col p-8 m-0 overflow-hidden" style="background: linear-gradient(135deg, #1e3c72 0%, #2a5298 100%);">
    <div class="flex justify-between items-center mb-8 pb-4 border-b-4 border-white/30">
      <h1 class="text-5xl font-bold">Sistema de Turnos</h1>
      <div class="text-3xl font-light tracking-widest">{{ currentTime }}</div>
    </div>

    <div class="flex-1 grid grid-cols-2 gap-8">
      <!-- Turno actual en atención -->
      <div class="flex flex-col">
        <h2 class="text-3xl text-center mb-6 bg-white/20 p-4 rounded-lg">ATENDIENDO AHORA</h2>
        
        <div v-if="inProgress.length > 0" class="flex flex-col gap-6">
          <div
            v-for="turn in inProgress"
            :key="turn.id"
            class="bg-white text-gray-800 rounded-2xl p-8 flex items-center gap-8 border-l-8 animate-pulse"
            :style="{ borderLeftColor: turn.service.color }"
          >
            <div class="text-7xl font-bold min-w-[200px] text-center" :style="{ color: turn.service.color }">
              {{ turn.ticket_number }}
            </div>
            <div class="flex-1">
              <div class="text-3xl font-semibold mb-2">{{ turn.service.name }}</div>
              <div v-if="turn.window" class="text-2xl text-gray-600">{{ turn.window.name }}</div>
            </div>
          </div>
        </div>
        
        <div v-else class="flex items-center justify-center h-full opacity-70 text-2xl">
          <p>No hay turnos en atención en este momento</p>
        </div>
      </div>

      <!-- Cola de espera -->
      <div class="flex flex-col">
        <h2 class="text-3xl text-center mb-6 bg-white/20 p-4 rounded-lg">EN ESPERA</h2>
        
        <div v-if="waiting.length > 0" class="grid grid-cols-3 gap-4">
          <div
            v-for="turn in waiting.slice(0, 9)"
            :key="turn.id"
            class="bg-white/10 backdrop-blur-lg p-6 rounded-lg border-l-4 flex flex-col items-center text-center"
            :style="{ borderLeftColor: turn.service.color }"
          >
            <span class="text-3xl font-bold mb-2">{{ turn.ticket_number }}</span>
            <span class="text-base opacity-90">{{ turn.service.name }}</span>
          </div>
        </div>
        
        <div v-else class="flex items-center justify-center h-full opacity-70 text-2xl">
          <p>No hay turnos en espera</p>
        </div>
      </div>
    </div>

    <div class="text-center mt-8 pt-4 border-t-2 border-white/30 text-xl opacity-90">
      <p>Por favor, permanezca atento al llamado de su turno</p>
    </div>
  </div>
</template>

<script setup>
import { ref, onMounted, onUnmounted } from 'vue';
import { getQueue } from '../api';

const waiting = ref([]);
const inProgress = ref([]);
const currentTime = ref('');

const USE_SSE = false; // Cambiar a true para usar SSE en vez de polling

let eventSource = null;
let intervalId = null;
let clockId = null;

const loadQueue = async () => {
  try {
    const response = await getQueue();
    waiting.value = response.data.waiting || [];
    inProgress.value = response.data.in_progress || [];
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
};

onMounted(() => {
  loadQueue(); // Carga inicial
  updateClock();
  
  // Elegir método de actualización
  if (USE_SSE) {
    connectSSE();
  } else {
    intervalId = setInterval(loadQueue, 3000); // Polling cada 3 segundos
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
