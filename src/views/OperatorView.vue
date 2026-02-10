<template>
  <div class="w-screen h-screen bg-gray-100 m-0 overflow-hidden flex flex-col">
    <!-- Toast notification -->
    <transition name="fade">
      <div v-if="toast.show" 
        class="fixed top-4 right-4 z-50 px-6 py-4 rounded-lg shadow-lg"
        :class="toast.type === 'error' ? 'bg-red-500 text-white' : 'bg-green-500 text-white'">
        {{ toast.message }}
      </div>
    </transition>

    <!-- Modal de Confirmación -->
    <div v-if="confirmModal.show" class="fixed inset-0 bg-black/50 flex items-center justify-center z-50">
      <div class="bg-white p-6 rounded-lg max-w-md w-full">
        <h3 class="text-lg font-bold mb-4">{{ confirmModal.title }}</h3>
        <p class="text-gray-600 mb-6">{{ confirmModal.message }}</p>
        <div class="flex gap-4">
          <button @click="confirmModal.onConfirm" class="flex-1 px-4 py-2 bg-red-500 text-white rounded hover:bg-red-600 transition">
            Confirmar
          </button>
          <button @click="confirmModal.show = false" class="flex-1 px-4 py-2 bg-gray-200 text-gray-700 rounded hover:bg-gray-300 transition">
            Cancelar
          </button>
        </div>
      </div>
    </div>

    <div class="bg-white px-8 py-6 shadow-md flex justify-between items-center">
      <div class="flex items-center gap-4">
        <h1 class="text-3xl text-gray-800">Panel de Operador</h1>
        <div class="flex gap-2">
          <span v-if="assignedWindow" class="text-sm bg-purple-100 text-purple-800 px-3 py-1 rounded-full font-semibold">
            <i class="fas fa-desktop mr-1"></i>{{ assignedWindow.name }} (#{{ assignedWindow.number }})
          </span>
          <span v-else-if="windowError" class="text-sm bg-red-100 text-red-800 px-3 py-1 rounded-full">
            <i class="fas fa-exclamation-triangle mr-1"></i>Sin ventanilla asignada
          </span>
          <span v-if="user?.services" class="text-sm bg-green-100 text-green-800 px-3 py-1 rounded-full">
            {{ user.services.length }} servicio(s) asignado(s)
          </span>
        </div>
      </div>
      <div class="flex gap-4 items-center">
        <div class="text-right">
          <p class="font-semibold">{{ user?.name || 'Cargando...' }}</p>
          <p class="text-sm text-gray-600">{{ user?.email }}</p>
        </div>
        <button @click="handleLogout" class="px-4 py-2 bg-red-500 text-white rounded hover:bg-red-600 transition">
          Salir
        </button>
      </div>
    </div>

    <!-- Mensaje de error si no tiene ventanilla asignada -->
    <div v-if="windowError && !assignedWindow" class="flex-1 flex items-center justify-center p-8">
      <div class="max-w-2xl p-8 bg-red-50 border-2 border-red-300 rounded-lg shadow-lg">
        <div class="flex items-start gap-6">
          <i class="fas fa-exclamation-circle text-red-600 text-6xl"></i>
          <div>
            <h3 class="text-2xl font-bold text-red-800 mb-3">Esta PC no está asignada a ninguna ventanilla</h3>
            <p class="text-red-700 mb-3 text-lg">{{ windowError }}</p>
            <p class="text-red-600 mb-4">
              No puede operar turnos sin una ventanilla asignada.
            </p>
            <div class="bg-red-100 border border-red-400 rounded p-4 text-sm text-red-800">
              <p class="font-semibold mb-2">Solución:</p>
              <ol class="list-decimal ml-5 space-y-1">
                <li>Contacte al administrador del sistema</li>
                <li>Solicite que asigne una ventanilla a esta IP en el panel de administración (tab "Ventanillas")</li>
                <li>Recargue esta página después de la asignación</li>
              </ol>
            </div>
          </div>
        </div>
      </div>
    </div>

    <!-- Paneles de operación (solo si tiene ventanilla asignada) -->
    <div v-if="assignedWindow" class="flex-1 grid grid-cols-2 gap-8 p-8 overflow-hidden">
      <!-- Panel izquierdo: Turno actual -->
      <div class="bg-white rounded-lg p-6 shadow-lg flex flex-col">
        <h2 class="text-2xl mb-6 text-gray-800">Turno Actual</h2>
        
        <div v-if="currentTurn" class="flex flex-col gap-6">
          <div class="flex justify-between items-center pb-4 border-b-2 border-gray-200">
            <div class="text-5xl font-bold" :style="{ color: currentTurn.service.color }">
              {{ currentTurn.ticket_number }}
            </div>
            <div>
              <span 
                class="px-4 py-2 rounded-full font-semibold text-sm"
                :class="{
                  'bg-yellow-100 text-yellow-800': currentTurn.status === 'called',
                  'bg-blue-100 text-blue-800': currentTurn.status === 'in_progress'
                }"
              >
                {{ statusText(currentTurn.status) }}
              </span>
            </div>
          </div>

          <div class="flex flex-col gap-2">
            <div class="flex justify-between py-2 border-b border-gray-100">
              <strong>Servicio:</strong>
              <span class="px-2 py-1 rounded text-sm" :style="{ backgroundColor: currentTurn.service.color + '20', color: currentTurn.service.color }">
                {{ currentTurn.service.name }}
              </span>
            </div>
            <div v-if="currentTurn.customer_name" class="flex justify-between py-2 border-b border-gray-100">
              <strong>Cliente:</strong>
              <span class="font-semibold text-lg">{{ currentTurn.customer_name }}</span>
            </div>
            <div class="flex justify-between py-2 border-b border-gray-100">
              <strong>Hora emisión:</strong>
              <span class="text-gray-600">{{ formatTime(currentTurn.issued_at) }}</span>
            </div>
            <div v-if="currentTurn.called_at" class="flex justify-between py-2 border-b border-gray-100">
              <strong>Hora llamado:</strong>
              <span class="text-blue-600">{{ formatTime(currentTurn.called_at) }}</span>
            </div>
            <div v-if="currentTurn.attended_at" class="flex justify-between py-2 border-b border-gray-100">
              <strong>Hora atención:</strong>
              <span class="text-green-600">{{ formatTime(currentTurn.attended_at) }}</span>
            </div>
            <!-- Tiempos calculados -->
            <div v-if="currentTurn.status === 'in_progress'" class="mt-2 p-3 bg-blue-50 rounded-lg text-sm">
              <div class="flex justify-between">
                <span class="text-gray-600">Tiempo esperando:</span>
                <span class="font-semibold">{{ calculateWaitTime(currentTurn) }}</span>
              </div>
            </div>
          </div>

          <!-- Botones cuando está LLAMADO -->
          <div v-if="currentTurn.status === 'called'" class="flex flex-col gap-3 mt-4">
            <button
              @click="startAttention"
              class="w-full p-4 border-0 rounded-lg font-semibold cursor-pointer bg-blue-500 text-white hover:bg-blue-600 transition"
            >
              <i class="fas fa-play mr-2"></i>Iniciar Atención
            </button>
            
            <div class="flex gap-3">
              <button
                @click="markNoShow"
                class="flex-1 p-3 border-0 rounded-lg font-semibold cursor-pointer bg-gray-500 text-white hover:bg-gray-600 transition"
              >
                <i class="fas fa-user-slash mr-2"></i>No se presentó
              </button>
              
              <button
                @click="returnToQueue"
                class="flex-1 p-3 border-0 rounded-lg font-semibold cursor-pointer bg-yellow-500 text-white hover:bg-yellow-600 transition"
              >
                <i class="fas fa-undo mr-2"></i>Devolver a cola
              </button>
            </div>
          </div>
          
          <!-- Botones cuando está EN ATENCIÓN -->
          <div v-if="currentTurn.status === 'in_progress'" class="flex gap-4 mt-4">
            <button
              @click="completeTurn"
              class="flex-1 p-4 border-0 rounded-lg font-semibold cursor-pointer bg-green-500 text-white hover:bg-green-600 transition"
            >
              <i class="fas fa-check mr-2"></i>Completar
            </button>
            
            <button
              @click="cancelTurn"
              class="flex-1 p-4 border-0 rounded-lg font-semibold cursor-pointer bg-red-500 text-white hover:bg-red-600 transition"
            >
              <i class="fas fa-times mr-2"></i>Cancelar
            </button>
          </div>
        </div>

        <div v-else class="text-center py-12 text-gray-500">
          <p class="mb-8">No hay turno en atención</p>
          <button 
            @click="callNext" 
            class="text-xl px-8 py-6 mt-8 border-0 rounded-lg font-semibold cursor-pointer bg-blue-500 text-white hover:bg-blue-600 transition"
          >
            Llamar Siguiente Turno
          </button>
        </div>
      </div>

      <!-- Panel derecho: Cola de espera -->
      <div class="bg-white rounded-lg p-6 shadow-lg flex flex-col overflow-hidden">
        <div class="flex justify-between items-center mb-6">
          <h2 class="text-2xl text-gray-800">En Espera ({{ waitingTurns.length }})</h2>
          <button 
            @click="loadQueue" 
            class="px-4 py-2 bg-gray-100 border border-gray-300 rounded cursor-pointer hover:bg-gray-200 transition"
          >
            🔄 Actualizar
          </button>
        </div>

        <div class="flex flex-col gap-3 overflow-y-auto">
          <!-- Mensaje si ya hay un turno activo -->
          <div v-if="currentTurn" class="bg-yellow-50 border border-yellow-200 rounded-lg p-3 text-sm text-yellow-800">
            <i class="fas fa-info-circle mr-2"></i>
            Tienes un turno activo. Complétalo antes de llamar otro.
          </div>

          <div
            v-for="turn in waitingTurns"
            :key="turn.id"
            class="flex items-center gap-4 p-4 bg-gray-50 rounded-lg border-l-4 transition"
            :class="currentTurn ? 'opacity-50 cursor-not-allowed' : 'cursor-pointer hover:bg-gray-100'"
            :style="{ borderLeftColor: turn.service.color }"
            @click="!currentTurn && callSpecific(turn)"
          >
            <div class="text-2xl font-bold text-gray-700 min-w-[100px]">
              {{ turn.ticket_number }}
            </div>
            <div class="flex-1">
              <div class="font-semibold text-gray-900">{{ turn.service.name }}</div>
              <div class="text-sm text-gray-600">{{ formatTime(turn.created_at) }}</div>
            </div>
            <div v-if="turn.priority === 'priority'" class="text-2xl">
              ⭐
            </div>
          </div>

          <div v-if="waitingTurns.length === 0" class="text-center py-12 text-gray-400">
            No hay turnos en espera
          </div>
        </div>
      </div>
    </div>
  </div>
</template>

<script setup>
import { ref, onMounted, onUnmounted, reactive } from 'vue';
import { useRouter } from 'vue-router';
import { getOperatorQueue, getUser, logout as apiLogout, getWindowByIp } from '../api';
import api from '../api';

const router = useRouter();
const user = ref(null);
const assignedWindow = ref(null);
const windowError = ref('');
const currentTurn = ref(null);
const waiting = ref([]);
const waitingTurns = ref([]); // Alias para compatibilidad
const USE_SSE = false; // Cambiar a true para usar SSE en vez de polling
let intervalId = null;
let eventSource = null;

// Toast notification
const toast = reactive({
  show: false,
  message: '',
  type: 'success'
});

const showToast = (message, type = 'success') => {
  toast.message = message;
  toast.type = type;
  toast.show = true;
  setTimeout(() => {
    toast.show = false;
  }, 3000);
};

// Modal de confirmación
const confirmModal = reactive({
  show: false,
  title: '',
  message: '',
  onConfirm: () => {}
});

const showConfirm = (title, message, onConfirm) => {
  confirmModal.title = title;
  confirmModal.message = message;
  confirmModal.onConfirm = () => {
    confirmModal.show = false;
    onConfirm();
  };
  confirmModal.show = true;
};

const loadQueue = async () => {
  // No cargar cola si no tiene ventanilla asignada
  if (!assignedWindow.value) {
    return;
  }
  
  try {
    const response = await getOperatorQueue(); // Usar endpoint protegido
    waiting.value = response.data.waiting || [];
    waitingTurns.value = response.data.waiting || []; // Mantener compatibilidad
    
    // El backend nos devuelve directamente el turno activo del operador (si tiene)
    if (response.data.my_turn) {
      currentTurn.value = response.data.my_turn;
    } else {
      currentTurn.value = null;
    }
  } catch (error) {
    console.error('Error cargando cola:', error);
  }
};

const callNext = async () => {
  // Verificar si tiene ventanilla asignada
  if (!assignedWindow.value) {
    showToast('No tienes ventanilla asignada. No puedes llamar turnos.', 'error');
    return;
  }

  // Verificar si ya tiene un turno activo
  if (currentTurn.value) {
    showToast('Ya tienes un turno activo. Complétalo o cancélalo primero.', 'error');
    return;
  }

  if (waitingTurns.value.length === 0) {
    showToast('No hay turnos en espera', 'error');
    return;
  }

  // Ordenar por prioridad y fecha
  const sortedTurns = [...waitingTurns.value].sort((a, b) => {
    if (a.priority === 'priority' && b.priority !== 'priority') return -1;
    if (a.priority !== 'priority' && b.priority === 'priority') return 1;
    return new Date(a.created_at) - new Date(b.created_at);
  });

  const nextTurn = sortedTurns[0];
  await callSpecific(nextTurn);
};

const callSpecific = async (turn) => {
  // Verificar si tiene ventanilla asignada
  if (!assignedWindow.value) {
    showToast('No tienes ventanilla asignada. No puedes llamar turnos.', 'error');
    return;
  }

  // Verificar si ya tiene un turno activo
  if (currentTurn.value) {
    showToast('Ya tienes un turno activo. Complétalo o cancélalo primero.', 'error');
    return;
  }

  try {
    const response = await api.post(`/turns/${turn.id}/call`, {
      window_id: assignedWindow.value.id,
    });
    
    currentTurn.value = response.data;
    await loadQueue();
  } catch (error) {
    console.error('Error llamando turno:', error);
    const message = error.response?.data?.message || 'Error al llamar turno';
    showToast(message, 'error');
  }
};

const startAttention = async () => {
  try {
    const response = await api.post(`/turns/${currentTurn.value.id}/attend`);
    // Actualizar con los datos del servidor (mantiene created_at original)
    currentTurn.value = response.data;
    showToast('Atención iniciada');
  } catch (error) {
    console.error('Error iniciando atención:', error);
    showToast('Error al iniciar atención', 'error');
  }
};

const completeTurn = async () => {
  try {
    await api.post(`/turns/${currentTurn.value.id}/complete`);
    currentTurn.value = null;
    await loadQueue();
    showToast('Turno completado');
  } catch (error) {
    console.error('Error completando turno:', error);
    showToast('Error al completar turno', 'error');
  }
};

const cancelTurn = () => {
  showConfirm(
    'Cancelar Turno',
    '¿Está seguro de cancelar este turno?',
    async () => {
      try {
        await api.post(`/turns/${currentTurn.value.id}/cancel`);
        currentTurn.value = null;
        await loadQueue();
        showToast('Turno cancelado');
      } catch (error) {
        console.error('Error cancelando turno:', error);
        showToast('Error al cancelar turno', 'error');
      }
    }
  );
};

const markNoShow = () => {
  showConfirm(
    'Cliente no se presentó',
    '¿Confirma que el cliente no se presentó?',
    async () => {
      try {
        await api.post(`/turns/${currentTurn.value.id}/no-show`);
        currentTurn.value = null;
        await loadQueue();
        showToast('Turno marcado como no presentado');
      } catch (error) {
        console.error('Error marcando no show:', error);
        showToast('Error al marcar turno', 'error');
      }
    }
  );
};

const returnToQueue = () => {
  showConfirm(
    'Devolver a cola',
    '¿Desea devolver este turno a la cola de espera?',
    async () => {
      try {
        await api.post(`/turns/${currentTurn.value.id}/return-to-queue`);
        currentTurn.value = null;
        await loadQueue();
        showToast('Turno devuelto a la cola');
      } catch (error) {
        console.error('Error devolviendo turno:', error);
        showToast('Error al devolver turno', 'error');
      }
    }
  );
};

const statusText = (status) => {
  const texts = {
    waiting: 'En espera',
    called: 'Llamado',
    in_progress: 'En atención',
    completed: 'Completado',
    cancelled: 'Cancelado',
    no_show: 'No se presentó'
  };
  return texts[status] || status;
};

const formatTime = (datetime) => {
  if (!datetime) return '-';
  return new Date(datetime).toLocaleTimeString('es-AR', {
    hour: '2-digit',
    minute: '2-digit'
  });
};

const calculateWaitTime = (turn) => {
  if (!turn.issued_at || !turn.called_at) return '-';
  const issued = new Date(turn.issued_at);
  const called = new Date(turn.called_at);
  const diffMinutes = Math.round((called - issued) / 60000);
  if (diffMinutes < 60) return `${diffMinutes} min`;
  const hours = Math.floor(diffMinutes / 60);
  const mins = diffMinutes % 60;
  return mins > 0 ? `${hours}h ${mins}m` : `${hours}h`;
};

const handleLogout = async () => {
  try {
    await apiLogout();
  } catch (error) {
    console.error('Error en logout:', error);
  } finally {
    localStorage.removeItem('token');
    localStorage.removeItem('user');
    router.push('/login');
  }
};

const loadUser = async () => {
  try {
    const response = await getUser();
    user.value = response.data;
  } catch (error) {
    console.error('Error cargando usuario:', error);
    // Si falla, redirigir a login
    router.push('/login');
  }
};

const loadWindow = async () => {
  try {
    const response = await getWindowByIp();
    assignedWindow.value = response.data;
    windowError.value = '';
    console.log(`✅ Ventanilla asignada: ${assignedWindow.value.name} (${assignedWindow.value.ip_address})`);
  } catch (error) {
    console.error('Error cargando ventanilla:', error);
    windowError.value = error.response?.data?.error || 'Error al cargar ventanilla';
    
    // Mostrar toast de error
    showToast(
      `No hay ventanilla asignada a esta PC. IP: ${error.response?.data?.ip || 'desconocida'}`,
      'error'
    );
  }
};

const connectSSE = () => {
  const token = localStorage.getItem('token');
  
  if (!token) {
    console.error('No hay token para SSE');
    return;
  }

  if (eventSource) {
    eventSource.close();
  }

  eventSource = new EventSource(`http://192.168.0.51:8000/api/sse/operator?token=${token}`);

  eventSource.onmessage = (event) => {
    try {
      const data = JSON.parse(event.data);
      waiting.value = data.waiting || [];
      inProgress.value = data.in_progress || [];
      waitingTurns.value = data.waiting || [];
      
      if (data.in_progress && data.in_progress.length > 0) {
        currentTurn.value = data.in_progress[0];
      } else if (currentTurn.value && currentTurn.value.status === 'completed') {
        currentTurn.value = null;
      }
      
      console.log('✅ SSE Operador: Datos actualizados');
    } catch (error) {
      console.error('Error parseando SSE:', error);
    }
  };

  eventSource.onerror = (error) => {
    console.error('❌ Error SSE, volviendo a polling...', error);
    eventSource.close();
    eventSource = null;
    
    // Volver a polling
    intervalId = setInterval(loadQueue, 5000);
  };

  eventSource.onopen = () => {
    console.log('🟢 SSE Operador conectado');
  };
};

onMounted(async () => {
  await loadUser();
  await loadWindow(); // Verificar ventanilla asignada por IP
  
  loadQueue(); // Carga inicial
  
  // Elegir método de actualización
  if (USE_SSE) {
    connectSSE();
  } else {
    intervalId = setInterval(loadQueue, 5000); // Polling cada 5 segundos
  }
});

onUnmounted(() => {
  if (eventSource) {
    eventSource.close();
  }
  if (intervalId) clearInterval(intervalId);
});
</script>

<style scoped>
.fade-enter-active,
.fade-leave-active {
  transition: opacity 0.3s ease;
}

.fade-enter-from,
.fade-leave-to {
  opacity: 0;
}
</style>
