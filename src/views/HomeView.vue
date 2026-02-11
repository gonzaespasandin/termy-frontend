<template>
  <div class="w-screen h-screen px-8 py-6 overflow-hidden m-0" style="background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);">
    
    <!-- Mensaje de error global -->
    <transition name="fade">
      <div v-if="errorMessage" class="fixed top-4 left-1/2 transform -translate-x-1/2 z-50 px-6 py-4 bg-red-500 text-white rounded-lg shadow-lg">
        {{ errorMessage }}
      </div>
    </transition>
    
    <div class="max-w-full h-full mx-auto flex flex-col">
      <!-- Header con logos -->
      <div class="flex justify-between items-center mb-4" style="height: 12vh;">
        <!-- Logo empresa cliente (izquierda) -->
        <div class="bg-white/20 backdrop-blur-sm rounded-2xl flex items-center justify-center" style="width: 20%; height: 100%; padding: 1rem;">
          <div class="text-white text-center">
            <div class="text-sm font-semibold mb-1">Logo</div>
            <div class="text-xs">Cliente</div>
          </div>
        </div>

        <!-- Título central -->
        <div class="flex-1 text-center px-8">
          <p class="text-white/90" :style="{ fontSize: getHeaderFontSize() }">Seleccione el servicio que desea solicitar</p>
        </div>

        <!-- Logo nuestra empresa (derecha) -->
        <div class="bg-white/20 backdrop-blur-sm rounded-2xl flex items-center justify-center" style="width: 20%; height: 100%; padding: 1rem;">
          <div class="text-white text-center">
            <div class="text-sm font-semibold mb-1">Logo</div>
            <div class="text-xs">Turnero</div>
          </div>
        </div>
      </div>

      <div v-if="loading" class="text-center text-white flex-1 flex items-center justify-center" :style="{ fontSize: getHeaderFontSize() }">
        Cargando servicios...
      </div>

      <!-- Grid de servicios responsive -->
      <div v-else class="flex-1 flex items-center justify-center" style="height: calc(88vh - 2rem);">
        <div 
          class="services-grid w-full h-full"
          :style="getGridStyle()"
        >
          <div
            v-for="service in services"
            :key="service.id"
            class="service-card bg-white rounded-2xl text-center cursor-pointer transition-all duration-200 border-4 border-transparent hover:-translate-y-2 hover:scale-105 hover:shadow-2xl flex flex-col justify-center"
            :style="{ borderColor: service.color, ...getCardStyle() }"
            @click="selectService(service)"
          >
            <div 
              class="icon-circle mx-auto mb-2 rounded-full flex items-center justify-center text-white font-bold"
              :style="{ backgroundColor: service.color, ...getIconStyle() }"
            >
              {{ service.code }}
            </div>
            <h3 class="my-2 text-gray-800 font-semibold" :style="{ fontSize: getServiceNameFontSize() }">{{ service.name }}</h3>
            <p class="text-gray-600" :style="{ fontSize: getServiceDescFontSize() }">{{ service.description }}</p>
          </div>
        </div>
      </div>

      <!-- Modal para confirmar (solo si el perfil pide nombre) -->
      <div v-if="selectedService && askCustomerName" class="fixed inset-0 bg-black/80 flex items-center justify-center z-50">
        <div class="bg-white p-12 rounded-3xl max-w-2xl w-11/12 text-center">
          <h2 class="mb-4 text-gray-800 text-3xl">¿Desea solicitar un turno para:</h2>
          <h3 class="text-5xl mb-6" style="color: #667eea;">{{ selectedService.name }}</h3>

          <input
            v-model="customerName"
            type="text"
            placeholder="Nombre (opcional)"
            class="w-full p-4 border-2 border-gray-300 rounded-lg my-6 text-2xl focus:outline-none focus:border-blue-500"
          />

          <div class="flex gap-4 mt-6">
            <button 
              @click="confirmTurn" 
              class="flex-1 p-6 border-0 rounded-lg text-2xl cursor-pointer font-semibold text-white hover:opacity-90 transition-opacity"
              style="background-color: #667eea;"
            >
              Confirmar
            </button>
            <button 
              @click="selectedService = null" 
              class="flex-1 p-6 border-0 rounded-lg text-2xl cursor-pointer font-semibold bg-gray-200 text-gray-700 hover:opacity-90 transition-opacity"
            >
              Cancelar
            </button>
          </div>
        </div>
      </div>

      <!-- Ticket generado -->
      <div v-if="generatedTurn" class="fixed inset-0 bg-black/80 flex items-center justify-center z-50">
        <div class="bg-white p-12 rounded-3xl max-w-2xl w-11/12 text-center">
          <h2 class="text-gray-800 mb-4 text-4xl">Su turno es:</h2>
          <div class="text-8xl font-bold my-6 tracking-wide" style="color: #667eea;">{{ generatedTurn.ticket_number }}</div>
          <p class="text-2xl text-gray-600 my-3">Servicio: {{ generatedTurn.service.name }}</p>
          <p class="text-2xl text-gray-600 my-3">Hora: {{ formatTime(generatedTurn.created_at) }}</p>
          
          <!-- Estado de impresión -->
          <p v-if="printing" class="text-lg text-blue-600 my-4">🖨️ Imprimiendo ticket...</p>
          <p v-if="printError" class="text-lg text-red-600 my-4">⚠️ {{ printError }}</p>
          <p v-if="printSuccess" class="text-lg text-green-600 my-4">✅ Ticket impreso correctamente</p>
          
          <div class="flex gap-4 mt-6">
            <button 
              @click="resetAndPrint" 
              class="flex-1 px-12 py-6 border-0 rounded-lg text-2xl cursor-pointer font-semibold text-white hover:opacity-90 transition-opacity"
              style="background-color: #667eea;"
              :disabled="printing"
            >
              {{ printing ? 'Imprimiendo...' : 'Imprimir Ticket' }}
            </button>
            <button 
              @click="closeTicketModal" 
              class="flex-1 px-12 py-6 border-0 rounded-lg text-2xl cursor-pointer font-semibold bg-gray-200 text-gray-700 hover:opacity-90 transition-opacity"
            >
              Cerrar
            </button>
          </div>
        </div>
      </div>
    </div>
  </div>
</template>
  
<script setup>
import { ref, onMounted, computed } from 'vue';
import { getTotemServices, createTurn, getServices } from '../api';
import * as printService from '../services/printService';

// Estado del totem
const totemConfigured = ref(false);
const totemCode = ref('');
const totemId = ref(null);
const totemName = ref('');
const askCustomerName = ref(true); // Por defecto pide nombre

// Estado de servicios y turnos
const services = ref([]);
const loading = ref(true);
const selectedService = ref(null);
const customerName = ref('');
const generatedTurn = ref(null);

// Estado de impresión
const printing = ref(false);
const printError = ref('');
const printSuccess = ref(false);

// Detectar si estamos en Electron (app de escritorio)
const isElectronApp = ref(false);

// Mensaje de error global
const errorMessage = ref('');

const showError = (message) => {
  errorMessage.value = message;
  setTimeout(() => {
    errorMessage.value = '';
  }, 4000);
};

// Calcular columnas y filas según cantidad de servicios
const getGridLayout = () => {
  const count = services.value.length;
  
  if (count === 0) return { cols: 1, rows: 1 };
  if (count <= 3) return { cols: 3, rows: 1 };
  if (count <= 6) return { cols: 3, rows: 2 };
  if (count <= 9) return { cols: 3, rows: 3 };
  if (count <= 12) return { cols: 4, rows: 3 };
  if (count <= 15) return { cols: 5, rows: 3 };
  return { cols: 6, rows: Math.ceil(count / 6) };
};

// Calcular estilo del grid
const getGridStyle = () => {
  const layout = getGridLayout();
  const count = services.value.length;
  
  // Gap más pequeño para muchos servicios
  const gap = count <= 6 ? '2.5rem' : count <= 9 ? '1.5rem' : '1rem';
  
  return {
    display: 'grid',
    gridTemplateColumns: `repeat(${layout.cols}, 1fr)`,
    gridTemplateRows: `repeat(${layout.rows}, 1fr)`,
    gap: gap,
    padding: '1rem',
  };
};

// Calcular estilo de las cards
const getCardStyle = () => {
  const count = services.value.length;
  
  // Padding más pequeño para muchos servicios
  const padding = count <= 6 ? '2rem' : count <= 9 ? '1.5rem' : '1rem';
  
  return {
    padding: padding,
  };
};

// Calcular estilo del icono circular
const getIconStyle = () => {
  const count = services.value.length;
  
  // Tamaño del icono según cantidad de servicios
  let size, fontSize;
  if (count <= 3) {
    size = '6rem';
    fontSize = '2.5rem';
  } else if (count <= 6) {
    size = '5rem';
    fontSize = '2rem';
  } else if (count <= 9) {
    size = '4rem';
    fontSize = '1.5rem';
  } else {
    size = '3rem';
    fontSize = '1.25rem';
  }
  
  return {
    width: size,
    height: size,
    fontSize: fontSize,
  };
};

// Tamaño de fuente para el título del header
const getHeaderFontSize = () => {
  const count = services.value.length;
  if (count <= 6) return '2rem';
  if (count <= 9) return '1.75rem';
  return '1.5rem';
};

// Tamaño de fuente para nombres de servicios
const getServiceNameFontSize = () => {
  const count = services.value.length;
  if (count <= 3) return '2rem';
  if (count <= 6) return '1.75rem';
  if (count <= 9) return '1.5rem';
  return '1.25rem';
};

// Tamaño de fuente para descripciones
const getServiceDescFontSize = () => {
  const count = services.value.length;
  if (count <= 3) return '1.125rem';
  if (count <= 6) return '1rem';
  if (count <= 9) return '0.875rem';
  return '0.75rem';
};

onMounted(async () => {
  // Detectar si estamos en Electron
  isElectronApp.value = printService.isElectron();
  
  if (isElectronApp.value) {
    console.log('🖥️ Ejecutando en modo Electron (App de Escritorio)');
    
    // Si estamos en Electron, obtener configuración del totem desde el .env de Electron
    const electronConfig = await printService.getTotemConfig();
    if (electronConfig && electronConfig.totemCode !== 'TOTEM-DEFAULT') {
      totemCode.value = electronConfig.totemCode;
      await loadTotemServices();
      return;
    }
  }
  
  // Si no estamos en Electron o no hay configuración, cargar todos los servicios
  console.log('🌐 Ejecutando en modo Web - Mostrando todos los servicios');
  await loadAllServices();
});

const loadTotemServices = async () => {
  loading.value = true;
  try {
    const response = await getTotemServices(totemCode.value);
    totemId.value = response.data.totem.id;
    totemName.value = response.data.totem.name;
    services.value = response.data.services;
    totemConfigured.value = true;
    // Cargar configuración de si pide nombre (por defecto true si no está definido)
    askCustomerName.value = response.data.totem.ask_customer_name !== false;
    console.log('📋 Configuración del perfil - Pedir nombre:', askCustomerName.value);
  } catch (error) {
    console.error('Error cargando servicios del totem:', error);
    showError('Error al cargar servicios del totem. Mostrando todos los servicios.');
    await loadAllServices();
  } finally {
    loading.value = false;
  }
};

const loadAllServices = async () => {
  loading.value = true;
  try {
    const response = await getServices();
    services.value = response.data;
    totemName.value = 'Todos los servicios';
  } catch (error) {
    console.error('Error cargando servicios:', error);
    showError('Error al cargar servicios');
  } finally {
    loading.value = false;
  }
};

const selectService = async (service) => {
  selectedService.value = service;
  customerName.value = '';
  
  // Si el perfil no pide nombre, crear turno directamente sin mostrar modal
  if (!askCustomerName.value) {
    console.log('⚡ Modo rápido: creando turno sin pedir nombre');
    await confirmTurn();
  }
};

const confirmTurn = async () => {
  try {
    // Si no se pide nombre, enviar vacío (no "Anónimo")
    // Si se pide nombre pero está vacío, enviar vacío también (antes era "Anónimo")
    const finalCustomerName = askCustomerName.value ? (customerName.value || '') : '';
    
    const response = await createTurn({
      service_id: selectedService.value.id,
      customer_name: finalCustomerName,
      totem_id: totemId.value,
    });
    
    generatedTurn.value = response.data;
    selectedService.value = null;
    
    // Imprimir automáticamente después de crear el turno
    await printTicket();
  } catch (error) {
    console.error('Error creando turno:', error);
    showError('Error al crear turno');
  }
};

const printTicket = async () => {
  if (!generatedTurn.value) return;
  
  printing.value = true;
  printError.value = '';
  printSuccess.value = false;
  
  try {
    // Usar el servicio unificado de impresión
    // Detecta automáticamente si estamos en Electron o navegador
    // Si no se pide nombre, enviar vacío para que no se imprima en el ticket
    const finalCustomerName = askCustomerName.value ? (customerName.value || '') : '';
    
    const result = await printService.printTicket({
      ticket_number: generatedTurn.value.ticket_number,
      service_name: generatedTurn.value.service.name,
      customer_name: finalCustomerName,
    });
    
    if (result.success) {
      printSuccess.value = true;
      console.log('✅ Ticket impreso:', result.message);
    } else {
      throw new Error(result.error || 'Error al imprimir');
    }
    
  } catch (error) {
    console.error('❌ Error de impresión:', error);
    printError.value = error.message || 'Error desconocido al imprimir';
  } finally {
    printing.value = false;
  }
};

const resetAndPrint = async () => {
  // Intentar imprimir de nuevo
  await printTicket();
};

const closeTicketModal = () => {
  generatedTurn.value = null;
  customerName.value = '';
  printError.value = '';
  printSuccess.value = false;
};

const formatTime = (datetime) => {
  return new Date(datetime).toLocaleTimeString('es-AR', {
    hour: '2-digit',
    minute: '2-digit'
  });
};
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

/* Asegurar que el grid se adapte correctamente */
.services-grid {
  max-width: 100%;
  max-height: 100%;
}

/* Cards responsivas */
.service-card {
  min-height: 0; /* Permitir que se encoja */
  overflow: hidden;
}

/* Icono circular responsive */
.icon-circle {
  flex-shrink: 0;
}

/* NOTA PARA AGREGAR LOGOS:
   Reemplazar los divs con clase "text-white text-center" en el header
   por tags <img> con las rutas a los logos reales:
   
   Logo Cliente: <img src="/ruta/al/logo-cliente.png" alt="Logo Cliente" class="w-full h-full object-contain" />
   Logo Turnero: <img src="/ruta/al/logo-turnero.png" alt="Logo Turnero" class="w-full h-full object-contain" />
*/
</style>