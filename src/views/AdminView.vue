<template>
  <div class="w-screen h-screen bg-gray-100 m-0 overflow-hidden flex flex-col">
    <!-- Notificación Toast -->
    <transition name="fade">
      <div v-if="toast.show" 
        class="fixed top-4 right-4 z-50 px-6 py-4 rounded-lg shadow-lg max-w-md"
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

    <!-- Header -->
    <div class="bg-white px-8 py-6 shadow-md flex justify-between items-center">
      <h1 class="text-3xl text-gray-800 font-bold">Panel de Administración</h1>
      <div class="flex gap-4 items-center">
        <span class="text-sm bg-purple-100 text-purple-800 px-3 py-1 rounded-full">Administrador</span>
        <div class="text-right">
          <p class="font-semibold">{{ user?.name || 'Cargando...' }}</p>
          <p class="text-sm text-gray-600">{{ user?.email }}</p>
        </div>
        <button @click="handleLogout" class="px-4 py-2 bg-red-500 text-white rounded hover:bg-red-600 transition">
          <i class="fas fa-sign-out-alt mr-2"></i>Salir
        </button>
      </div>
    </div>

    <!-- Content -->
    <div class="flex-1 flex overflow-hidden">
      <!-- Sidebar -->
      <div class="w-64 bg-white shadow-lg p-6">
        <nav class="space-y-2">
          <button
            v-for="tab in tabs"
            :key="tab.id"
            @click="activeTab = tab.id"
            class="w-full text-left px-4 py-3 rounded-lg transition"
            :class="activeTab === tab.id 
              ? 'bg-blue-500 text-white font-semibold' 
              : 'text-gray-700 hover:bg-gray-100'"
          >
            <i :class="tab.icon" class="mr-2"></i>{{ tab.name }}
          </button>
        </nav>
      </div>

      <!-- Main Content -->
      <div class="flex-1 overflow-auto p-8">
        <!-- Dashboard -->
        <div v-if="activeTab === 'dashboard'" class="space-y-6">
          <div class="flex justify-between items-center">
            <h2 class="text-2xl font-bold text-gray-800">Dashboard</h2>
            <button @click="loadDashboardStats" class="px-4 py-2 bg-gray-100 border border-gray-300 rounded-lg hover:bg-gray-200 transition text-sm">
              <i class="fas fa-sync-alt mr-2" :class="{ 'fa-spin': dashboardLoading }"></i>Actualizar
            </button>
          </div>
          
          <!-- Stats principales -->
          <div class="grid grid-cols-5 gap-4">
            <div class="bg-white p-6 rounded-lg shadow">
              <div class="flex items-center justify-between">
                <div>
                  <div class="text-sm text-gray-600">Turnos Hoy</div>
                  <div class="text-3xl font-bold text-blue-600">{{ dashboardStats.today?.total ?? '-' }}</div>
                </div>
                <i class="fas fa-ticket-alt text-3xl text-blue-200"></i>
              </div>
            </div>
            <div class="bg-white p-6 rounded-lg shadow">
              <div class="flex items-center justify-between">
                <div>
                  <div class="text-sm text-gray-600">En Espera</div>
                  <div class="text-3xl font-bold text-yellow-600">{{ dashboardStats.today?.waiting ?? '-' }}</div>
                </div>
                <i class="fas fa-clock text-3xl text-yellow-200"></i>
              </div>
            </div>
            <div class="bg-white p-6 rounded-lg shadow">
              <div class="flex items-center justify-between">
                <div>
                  <div class="text-sm text-gray-600">Atendiendo</div>
                  <div class="text-3xl font-bold text-green-600">{{ dashboardStats.today?.attending ?? '-' }}</div>
                </div>
                <i class="fas fa-user-check text-3xl text-green-200"></i>
              </div>
            </div>
            <div class="bg-white p-6 rounded-lg shadow">
              <div class="flex items-center justify-between">
                <div>
                  <div class="text-sm text-gray-600">Completados</div>
                  <div class="text-3xl font-bold text-purple-600">{{ dashboardStats.today?.completed ?? '-' }}</div>
                </div>
                <i class="fas fa-check-circle text-3xl text-purple-200"></i>
              </div>
            </div>
            <div class="bg-white p-6 rounded-lg shadow">
              <div class="flex items-center justify-between">
                <div>
                  <div class="text-sm text-gray-600">Cancelados</div>
                  <div class="text-3xl font-bold text-red-600">{{ dashboardStats.today?.cancelled ?? '-' }}</div>
                </div>
                <i class="fas fa-times-circle text-3xl text-red-200"></i>
              </div>
            </div>
          </div>

          <div class="grid grid-cols-3 gap-6">
            <!-- Turnos por servicio -->
            <div class="bg-white p-6 rounded-lg shadow col-span-2 h-fit">
              <h3 class="text-lg font-semibold mb-4"><i class="fas fa-chart-bar mr-2 text-blue-500"></i>Turnos por Servicio (Hoy)</h3>
              <div v-if="!dashboardStats.by_service || dashboardStats.by_service.length === 0" class="text-gray-400 text-center py-8">
                Sin datos para hoy
              </div>
              <div v-else class="space-y-3">
                <div v-for="item in dashboardStats.by_service" :key="item.service_id" class="flex items-center gap-4">
                  <div class="w-3 h-3 rounded-full" :style="{ backgroundColor: item.service_color }"></div>
                  <div class="flex-1">
                    <div class="flex justify-between items-center mb-1">
                      <span class="font-medium">{{ item.service_name }}</span>
                      <span class="text-sm text-gray-600">{{ item.total }} turnos</span>
                    </div>
                    <div class="h-2 bg-gray-200 rounded-full overflow-hidden">
                      <div 
                        class="h-full rounded-full transition-all duration-500"
                        :style="{ 
                          width: `${(item.completed / item.total) * 100}%`,
                          backgroundColor: item.service_color 
                        }"
                      ></div>
                    </div>
                    <div class="flex justify-between text-xs text-gray-500 mt-1">
                      <span>{{ item.waiting }} en espera</span>
                      <span>{{ item.completed }} completados</span>
                    </div>
                  </div>
                </div>
              </div>
            </div>

            <!-- Info adicional -->
            <div class="space-y-6">
              <!-- Tiempo promedio -->
              <div class="bg-white p-6 rounded-lg shadow">
                <h3 class="text-lg font-semibold mb-4"><i class="fas fa-stopwatch mr-2 text-orange-500"></i>Tiempo Promedio</h3>
                <div class="text-center">
                  <div class="text-4xl font-bold text-orange-600">{{ dashboardStats.avg_wait_time ?? '-' }}</div>
                  <div class="text-sm text-gray-600">minutos de espera</div>
                </div>
              </div>

              <!-- Últimos completados -->
              <div class="bg-white p-6 rounded-lg shadow">
                <h3 class="text-lg font-semibold mb-4"><i class="fas fa-history mr-2 text-green-500"></i>Últimos Atendidos</h3>
                <div v-if="!dashboardStats.recent_completed || dashboardStats.recent_completed.length === 0" class="text-gray-400 text-center py-4">
                  Sin turnos completados
                </div>
                <div v-else class="space-y-2">
                  <div v-for="(turn, idx) in dashboardStats.recent_completed" :key="idx" class="flex justify-between items-center text-sm py-1 border-b border-gray-100 last:border-0">
                    <span class="font-mono font-bold">{{ turn.ticket_number }}</span>
                    <span class="text-gray-600 truncate mx-2">{{ turn.service_name }}</span>
                    <span class="text-gray-400">{{ turn.completed_at }}</span>
                  </div>
                </div>
              </div>
              <div class="bg-white p-6 rounded-lg shadow">
              <h3 class="text-lg font-semibold mb-4"><i class="fas fa-cogs mr-2 text-gray-500"></i>Sistema</h3>
              <div class="grid grid-cols-3 gap-4">
                <div class="text-center p-4 bg-blue-50 rounded-lg">
                  <div class="text-2xl font-bold text-blue-600">{{ dashboardStats.totals?.services ?? '-' }}</div>
                  <div class="text-xs text-gray-600">Servicios</div>
                </div>
                <div class="text-center p-4 bg-green-50 rounded-lg">
                  <div class="text-2xl font-bold text-green-600">{{ dashboardStats.totals?.operators ?? '-' }}</div>
                  <div class="text-xs text-gray-600">Operadores</div>
                </div>
                <div class="text-center p-4 bg-purple-50 rounded-lg">
                  <div class="text-2xl font-bold text-purple-600">{{ dashboardStats.totals?.profiles ?? '-' }}</div>
                  <div class="text-xs text-gray-600">Perfiles</div>
                </div>
              </div>
            </div>
            </div>
          </div>

          <!-- Stats del sistema + Accesos rápidos -->
          <div class="grid grid-cols-2 gap-6">
            <!-- Stats del sistema -->
            
          </div>
        </div>

        <!-- Historial -->
        <div v-if="activeTab === 'historial'" class="space-y-6">
          <div class="flex justify-between items-center">
            <h2 class="text-2xl font-bold text-gray-800">Historial de Turnos</h2>
            <div class="text-sm text-gray-500">
              Total: {{ historyPagination.total }} registros
            </div>
          </div>

          <!-- Filtros -->
          <div class="bg-white p-6 rounded-lg shadow">
            <div class="grid grid-cols-6 gap-4">
              <!-- Búsqueda por número -->
              <div class="col-span-1">
                <label class="block text-xs font-medium text-gray-600 mb-1">Nº Ticket</label>
                <input 
                  v-model="historyFilters.search" 
                  type="text" 
                  placeholder="Ej: AC-001"
                  class="w-full px-3 py-2 border rounded-lg focus:ring-2 focus:ring-blue-500 focus:border-blue-500 text-sm"
                  @keyup.enter="applyHistoryFilters"
                >
              </div>
              
              <!-- Estado -->
              <div class="col-span-1">
                <label class="block text-xs font-medium text-gray-600 mb-1">Estado</label>
                <select 
                  v-model="historyFilters.status" 
                  class="w-full px-3 py-2 border rounded-lg focus:ring-2 focus:ring-blue-500 focus:border-blue-500 text-sm"
                >
                  <option value="">Todos</option>
                  <option value="waiting">En espera</option>
                  <option value="called">Llamado</option>
                  <option value="in_progress">En atención</option>
                  <option value="completed">Completado</option>
                  <option value="cancelled">Cancelado</option>
                  <option value="no_show">No se presentó</option>
                </select>
              </div>
              
              <!-- Operador -->
              <div class="col-span-1">
                <label class="block text-xs font-medium text-gray-600 mb-1">Operador</label>
                <select 
                  v-model="historyFilters.operator_id" 
                  class="w-full px-3 py-2 border rounded-lg focus:ring-2 focus:ring-blue-500 focus:border-blue-500 text-sm"
                >
                  <option value="">Todos</option>
                  <option v-for="op in operators" :key="op.id" :value="op.id">{{ op.name }}</option>
                </select>
              </div>
              
              <!-- Servicio -->
              <div class="col-span-1">
                <label class="block text-xs font-medium text-gray-600 mb-1">Servicio</label>
                <select 
                  v-model="historyFilters.service_id" 
                  class="w-full px-3 py-2 border rounded-lg focus:ring-2 focus:ring-blue-500 focus:border-blue-500 text-sm"
                >
                  <option value="">Todos</option>
                  <option v-for="srv in services" :key="srv.id" :value="srv.id">{{ srv.name }}</option>
                </select>
              </div>
              
              <!-- Fecha desde -->
              <div class="col-span-1">
                <label class="block text-xs font-medium text-gray-600 mb-1">Desde</label>
                <input 
                  v-model="historyFilters.date_from" 
                  type="date" 
                  class="w-full px-3 py-2 border rounded-lg focus:ring-2 focus:ring-blue-500 focus:border-blue-500 text-sm"
                >
              </div>
              
              <!-- Fecha hasta -->
              <div class="col-span-1">
                <label class="block text-xs font-medium text-gray-600 mb-1">Hasta</label>
                <input 
                  v-model="historyFilters.date_to" 
                  type="date" 
                  class="w-full px-3 py-2 border rounded-lg focus:ring-2 focus:ring-blue-500 focus:border-blue-500 text-sm"
                >
              </div>
            </div>
            
            <!-- Botones de filtro -->
            <div class="flex gap-3 mt-4">
              <button 
                @click="applyHistoryFilters" 
                class="px-4 py-2 bg-blue-500 text-white rounded-lg hover:bg-blue-600 transition text-sm"
              >
                <i class="fas fa-search mr-2"></i>Buscar
              </button>
              <button 
                @click="clearHistoryFilters" 
                class="px-4 py-2 bg-gray-100 text-gray-700 rounded-lg hover:bg-gray-200 transition text-sm"
              >
                <i class="fas fa-times mr-2"></i>Limpiar
              </button>
              <button 
                @click="loadHistory(historyPagination.current_page)" 
                class="px-4 py-2 bg-gray-100 text-gray-700 rounded-lg hover:bg-gray-200 transition text-sm"
                :disabled="historyLoading"
              >
                <i class="fas fa-sync-alt mr-2" :class="{ 'fa-spin': historyLoading }"></i>Actualizar
              </button>
            </div>
          </div>

          <!-- Tabla de historial -->
          <div class="bg-white rounded-lg shadow overflow-hidden">
            <table class="w-full">
              <thead class="bg-gray-50">
                <tr>
                  <th class="px-4 py-3 text-left text-xs font-medium text-gray-500 uppercase w-8"></th>
                  <th class="px-4 py-3 text-left text-xs font-medium text-gray-500 uppercase">Ticket</th>
                  <th class="px-4 py-3 text-left text-xs font-medium text-gray-500 uppercase">Servicio</th>
                  <th class="px-4 py-3 text-left text-xs font-medium text-gray-500 uppercase">Cliente</th>
                  <th class="px-4 py-3 text-left text-xs font-medium text-gray-500 uppercase">Estado</th>
                  <th class="px-4 py-3 text-left text-xs font-medium text-gray-500 uppercase">Operador</th>
                  <th class="px-4 py-3 text-left text-xs font-medium text-gray-500 uppercase">Espera</th>
                  <th class="px-4 py-3 text-left text-xs font-medium text-gray-500 uppercase">Atención</th>
                  <th class="px-4 py-3 text-left text-xs font-medium text-gray-500 uppercase">Fecha</th>
                </tr>
              </thead>
              <tbody class="divide-y divide-gray-200">
                <tr v-if="historyLoading" class="text-center text-gray-500">
                  <td colspan="9" class="py-12">
                    <i class="fas fa-spinner fa-spin text-2xl mb-2"></i>
                    <p>Cargando historial...</p>
                  </td>
                </tr>
                <tr v-else-if="historyData.length === 0" class="text-center text-gray-500">
                  <td colspan="9" class="py-12">
                    <i class="fas fa-inbox text-4xl mb-2 text-gray-300"></i>
                    <p>No hay turnos que mostrar</p>
                  </td>
                </tr>
                <template v-else v-for="turn in historyData" :key="turn.id">
                  <!-- Fila principal -->
                  <tr 
                    class="hover:bg-gray-50 cursor-pointer transition"
                    :class="{ 'bg-blue-50': expandedTurn === turn.id }"
                    @click="toggleExpandTurn(turn.id)"
                  >
                    <td class="px-4 py-3 text-center">
                      <i 
                        class="fas transition-transform duration-200" 
                        :class="expandedTurn === turn.id ? 'fa-chevron-down' : 'fa-chevron-right'"
                      ></i>
                    </td>
                    <td class="px-4 py-3">
                      <span class="font-mono font-bold text-lg">{{ turn.ticket_number }}</span>
                      <span v-if="turn.priority === 'priority'" class="ml-2 text-xs bg-red-100 text-red-800 px-2 py-0.5 rounded">
                        Prioritario
                      </span>
                    </td>
                    <td class="px-4 py-3">
                      <div class="flex items-center gap-2">
                        <div 
                          class="w-3 h-3 rounded-full" 
                          :style="{ backgroundColor: turn.service?.color || '#gray' }"
                        ></div>
                        <span class="text-sm">{{ turn.service?.name || 'N/A' }}</span>
                      </div>
                    </td>
                    <td class="px-4 py-3 text-sm text-gray-600">
                      {{ turn.customer_name || '-' }}
                    </td>
                    <td class="px-4 py-3">
                      <span 
                        class="px-2 py-1 text-xs rounded-full font-medium"
                        :class="getStatusColor(turn.status)"
                      >
                        {{ getStatusLabel(turn.status) }}
                      </span>
                    </td>
                    <td class="px-4 py-3 text-sm text-gray-600">
                      {{ turn.operator?.name || '-' }}
                    </td>
                    <td class="px-4 py-3 text-sm text-gray-600">
                      {{ formatDuration(turn.wait_time) }}
                    </td>
                    <td class="px-4 py-3 text-sm text-gray-600">
                      {{ formatDuration(turn.attention_time) }}
                    </td>
                    <td class="px-4 py-3 text-sm text-gray-500">
                      {{ turn.created_at?.split(' ')[0] }}
                    </td>
                  </tr>
                  
                  <!-- Fila expandida con línea de tiempo -->
                  <tr v-if="expandedTurn === turn.id">
                    <td colspan="9" class="bg-gray-50 px-6 py-4">
                      <div class="grid grid-cols-3 gap-6">
                        <!-- Línea de tiempo -->
                        <div class="col-span-2">
                          <h4 class="text-sm font-semibold text-gray-700 mb-4">
                            <i class="fas fa-stream mr-2"></i>Línea de Tiempo
                          </h4>
                          <div class="relative">
                            <!-- Línea vertical -->
                            <div class="absolute left-4 top-2 bottom-2 w-0.5 bg-gray-200"></div>
                            
                            <!-- Eventos -->
                            <div class="space-y-4">
                              <div 
                                v-for="(event, idx) in turn.timeline" 
                                :key="idx" 
                                class="relative flex items-start gap-4 pl-10"
                              >
                                <!-- Punto en la línea -->
                                <div 
                                  class="absolute left-2 w-5 h-5 rounded-full flex items-center justify-center text-white text-xs"
                                  :class="getTimelineIconColor(event.color)"
                                >
                                  <i :class="`fas fa-${event.icon}`"></i>
                                </div>
                                
                                <!-- Contenido -->
                                <div class="flex-1 bg-white p-3 rounded-lg shadow-sm border">
                                  <div class="flex justify-between items-start mb-1">
                                    <span class="font-semibold text-gray-800">{{ event.label }}</span>
                                    <span class="text-xs text-gray-400">{{ event.time_display }}</span>
                                  </div>
                                  <p class="text-sm text-gray-600">{{ event.description }}</p>
                                  <p v-if="event.duration" class="text-xs text-gray-400 mt-1">
                                    <i class="fas fa-clock mr-1"></i>{{ event.duration }}
                                  </p>
                                </div>
                              </div>
                            </div>
                          </div>
                        </div>
                        
                        <!-- Info adicional -->
                        <div class="space-y-4">
                          <div>
                            <h4 class="text-sm font-semibold text-gray-700 mb-2">
                              <i class="fas fa-info-circle mr-2"></i>Detalles
                            </h4>
                            <div class="bg-white p-4 rounded-lg shadow-sm border space-y-2 text-sm">
                              <div class="flex justify-between">
                                <span class="text-gray-500">Totem:</span>
                                <span class="font-medium">{{ turn.totem?.name || 'N/A' }}</span>
                              </div>
                              <div class="flex justify-between">
                                <span class="text-gray-500">Ventanilla:</span>
                                <span class="font-medium">{{ turn.window?.name || 'N/A' }}</span>
                              </div>
                              <div class="flex justify-between">
                                <span class="text-gray-500">Tiempo total:</span>
                                <span class="font-medium">{{ formatDuration(turn.total_time) }}</span>
                              </div>
                            </div>
                          </div>
                          
                          <div v-if="turn.notes">
                            <h4 class="text-sm font-semibold text-gray-700 mb-2">
                              <i class="fas fa-comment mr-2"></i>Notas
                            </h4>
                            <div class="bg-white p-4 rounded-lg shadow-sm border text-sm text-gray-600">
                              {{ turn.notes }}
                            </div>
                          </div>
                        </div>
                      </div>
                    </td>
                  </tr>
                </template>
              </tbody>
            </table>
          </div>

          <!-- Paginación -->
          <div v-if="historyPagination.last_page > 1" class="flex items-center justify-between bg-white px-6 py-4 rounded-lg shadow">
            <div class="text-sm text-gray-600">
              Mostrando {{ ((historyPagination.current_page - 1) * historyPagination.per_page) + 1 }} 
              a {{ Math.min(historyPagination.current_page * historyPagination.per_page, historyPagination.total) }} 
              de {{ historyPagination.total }} turnos
            </div>
            <div class="flex gap-2">
              <button 
                @click="goToPage(1)" 
                :disabled="historyPagination.current_page === 1"
                class="px-3 py-1 rounded border hover:bg-gray-100 disabled:opacity-50 disabled:cursor-not-allowed"
              >
                <i class="fas fa-angle-double-left"></i>
              </button>
              <button 
                @click="goToPage(historyPagination.current_page - 1)" 
                :disabled="historyPagination.current_page === 1"
                class="px-3 py-1 rounded border hover:bg-gray-100 disabled:opacity-50 disabled:cursor-not-allowed"
              >
                <i class="fas fa-angle-left"></i>
              </button>
              
              <!-- Números de página -->
              <template v-for="page in historyPagination.last_page" :key="page">
                <button 
                  v-if="page === 1 || page === historyPagination.last_page || (page >= historyPagination.current_page - 2 && page <= historyPagination.current_page + 2)"
                  @click="goToPage(page)"
                  class="px-3 py-1 rounded border"
                  :class="page === historyPagination.current_page ? 'bg-blue-500 text-white border-blue-500' : 'hover:bg-gray-100'"
                >
                  {{ page }}
                </button>
                <span 
                  v-else-if="page === historyPagination.current_page - 3 || page === historyPagination.current_page + 3"
                  class="px-2 py-1"
                >...</span>
              </template>
              
              <button 
                @click="goToPage(historyPagination.current_page + 1)" 
                :disabled="historyPagination.current_page === historyPagination.last_page"
                class="px-3 py-1 rounded border hover:bg-gray-100 disabled:opacity-50 disabled:cursor-not-allowed"
              >
                <i class="fas fa-angle-right"></i>
              </button>
              <button 
                @click="goToPage(historyPagination.last_page)" 
                :disabled="historyPagination.current_page === historyPagination.last_page"
                class="px-3 py-1 rounded border hover:bg-gray-100 disabled:opacity-50 disabled:cursor-not-allowed"
              >
                <i class="fas fa-angle-double-right"></i>
              </button>
            </div>
          </div>
        </div>

        <!-- Usuarios -->
        <div v-if="activeTab === 'usuarios'" class="space-y-6">
          <div class="flex justify-between items-center">
            <h2 class="text-2xl font-bold text-gray-800">Gestión de Usuarios</h2>
            <button @click="openUserModal()" class="px-6 py-3 bg-blue-500 text-white rounded-lg hover:bg-blue-600 transition">
              <i class="fas fa-plus mr-2"></i>Nuevo Usuario
            </button>
          </div>
          
          <div class="bg-white rounded-lg shadow overflow-hidden">
            <table class="w-full">
              <thead class="bg-gray-50">
                <tr>
                  <th class="px-6 py-3 text-left text-xs font-medium text-gray-500 uppercase">Nombre</th>
                  <th class="px-6 py-3 text-left text-xs font-medium text-gray-500 uppercase">Email</th>
                  <th class="px-6 py-3 text-left text-xs font-medium text-gray-500 uppercase">Rol</th>
                  <th class="px-6 py-3 text-left text-xs font-medium text-gray-500 uppercase">Servicios</th>
                  <th class="px-6 py-3 text-left text-xs font-medium text-gray-500 uppercase">Acciones</th>
                </tr>
              </thead>
              <tbody class="divide-y divide-gray-200">
                <tr v-if="usersLoading" class="text-center text-gray-500">
                  <td colspan="5" class="py-8">Cargando usuarios...</td>
                </tr>
                <tr v-else-if="users.length === 0" class="text-center text-gray-500">
                  <td colspan="5" class="py-8">No hay usuarios creados</td>
                </tr>
                <tr v-else v-for="usr in users" :key="usr.id">
                  <td class="px-6 py-4 text-sm font-medium text-gray-900">{{ usr.name }}</td>
                  <td class="px-6 py-4 text-sm text-gray-600">{{ usr.email }}</td>
                  <td class="px-6 py-4">
                    <span class="px-2 py-1 text-xs rounded-full" :class="usr.role === 'admin' ? 'bg-purple-100 text-purple-800' : 'bg-blue-100 text-blue-800'">
                      {{ usr.role === 'admin' ? 'Administrador' : 'Operador' }}
                    </span>
                  </td>
                  <td class="px-6 py-4 text-sm text-gray-600">
                    <span v-if="!usr.service_names || usr.service_names.length === 0" class="text-gray-400 italic">Todos</span>
                    <span v-else>{{ usr.service_names.join(', ') }}</span>
                  </td>
                  <td class="px-6 py-4 text-sm space-x-3">
                    <button @click="openUserModal(usr)" class="text-blue-600 hover:text-blue-800" title="Editar">
                      <i class="fas fa-edit"></i>
                    </button>
                    <button @click="handleDeleteUser(usr)" class="text-red-600 hover:text-red-800" title="Eliminar">
                      <i class="fas fa-trash"></i>
                    </button>
                  </td>
                </tr>
              </tbody>
            </table>
          </div>
        </div>

        <!-- Perfiles -->
        <div v-if="activeTab === 'perfiles'" class="space-y-6">
          <div class="flex justify-between items-center">
            <h2 class="text-2xl font-bold text-gray-800">Gestión de Perfiles</h2>
            <button @click="openProfileModal()" class="px-6 py-3 bg-blue-500 text-white rounded-lg hover:bg-blue-600 transition">
              <i class="fas fa-plus mr-2"></i>Nuevo Perfil
            </button>
          </div>

          <!-- Explicación -->
          <div class="bg-blue-50 border border-blue-200 rounded-lg p-4 text-sm text-blue-800">
            <i class="fas fa-info-circle mr-2"></i>
            Un <strong>perfil</strong> es un conjunto de servicios. Ponés el <strong>código</strong> del perfil en el archivo <code class="bg-blue-100 px-1 rounded">.env</code> del totem físico y mostrará solo esos servicios.
          </div>
          
          <div class="bg-white rounded-lg shadow overflow-hidden">
            <table class="w-full">
              <thead class="bg-gray-50">
                <tr>
                  <th class="px-6 py-3 text-left text-xs font-medium text-gray-500 uppercase">Código</th>
                  <th class="px-6 py-3 text-left text-xs font-medium text-gray-500 uppercase">Nombre</th>
                  <th class="px-6 py-3 text-left text-xs font-medium text-gray-500 uppercase">Ubicación</th>
                  <th class="px-6 py-3 text-left text-xs font-medium text-gray-500 uppercase">Servicios</th>
                  <th class="px-6 py-3 text-left text-xs font-medium text-gray-500 uppercase">Pide nombre</th>
                  <th class="px-6 py-3 text-left text-xs font-medium text-gray-500 uppercase">Estado</th>
                  <th class="px-6 py-3 text-left text-xs font-medium text-gray-500 uppercase">Acciones</th>
                </tr>
              </thead>
              <tbody class="divide-y divide-gray-200">
                <tr v-if="profilesLoading" class="text-center text-gray-500">
                  <td colspan="7" class="py-8">Cargando perfiles...</td>
                </tr>
                <tr v-else-if="profiles.length === 0" class="text-center text-gray-500">
                  <td colspan="7" class="py-8">No hay perfiles creados</td>
                </tr>
                <tr v-else v-for="profile in profiles" :key="profile.id">
                  <td class="px-6 py-4">
                    <code class="bg-gray-100 px-2 py-1 rounded text-sm font-mono">{{ profile.code }}</code>
                  </td>
                  <td class="px-6 py-4 text-sm font-medium text-gray-900">{{ profile.name }}</td>
                  <td class="px-6 py-4 text-sm text-gray-600">{{ profile.location || '-' }}</td>
                  <td class="px-6 py-4 text-sm text-gray-600">
                    <span v-if="!profile.service_names || profile.service_names.length === 0" class="text-gray-400 italic">Sin servicios</span>
                    <span v-else>{{ profile.service_names.join(', ') }}</span>
                  </td>
                  <td class="px-6 py-4">
                    <span class="px-2 py-1 text-xs rounded-full" :class="profile.ask_customer_name !== false ? 'bg-blue-100 text-blue-800' : 'bg-gray-100 text-gray-800'">
                      {{ profile.ask_customer_name !== false ? 'Sí' : 'No' }}
                    </span>
                  </td>
                  <td class="px-6 py-4">
                    <span class="px-2 py-1 text-xs rounded-full" :class="profile.is_active ? 'bg-green-100 text-green-800' : 'bg-red-100 text-red-800'">
                      {{ profile.is_active ? 'Activo' : 'Inactivo' }}
                    </span>
                  </td>
                  <td class="px-6 py-4 text-sm space-x-3">
                    <button @click="openProfileModal(profile)" class="text-blue-600 hover:text-blue-800" title="Editar">
                      <i class="fas fa-edit"></i>
                    </button>
                    <button @click="handleDeleteProfile(profile)" class="text-red-600 hover:text-red-800" title="Eliminar">
                      <i class="fas fa-trash"></i>
                    </button>
                  </td>
                </tr>
              </tbody>
            </table>
          </div>
        </div>

        <!-- Ventanillas -->
        <div v-if="activeTab === 'ventanillas'" class="space-y-6">
          <div class="flex justify-between items-center">
            <h2 class="text-2xl font-bold text-gray-800">Gestión de Ventanillas</h2>
            <button @click="openWindowModal()" class="px-6 py-3 bg-blue-500 text-white rounded-lg hover:bg-blue-600 transition">
              <i class="fas fa-plus mr-2"></i>Nueva Ventanilla
            </button>
          </div>

          <!-- Explicación -->
          <div class="bg-blue-50 border border-blue-200 rounded-lg p-4 text-sm text-blue-800">
            <i class="fas fa-info-circle mr-2"></i>
            Las <strong>ventanillas</strong> se asignan automáticamente por <strong>IP</strong>. Cuando un operador abre <code class="bg-blue-100 px-1 rounded">/operator</code>, el sistema detecta su IP y lo asocia a la ventanilla correspondiente. El <strong>número</strong> de ventanilla se muestra en el display cuando se llama un turno.
          </div>
          
          <div class="bg-white rounded-lg shadow overflow-hidden">
            <table class="w-full">
              <thead class="bg-gray-50">
                <tr>
                  <th class="px-6 py-3 text-left text-xs font-medium text-gray-500 uppercase">Número</th>
                  <th class="px-6 py-3 text-left text-xs font-medium text-gray-500 uppercase">Nombre</th>
                  <th class="px-6 py-3 text-left text-xs font-medium text-gray-500 uppercase">IP Asignada</th>
                  <th class="px-6 py-3 text-left text-xs font-medium text-gray-500 uppercase">Servicios</th>
                  <th class="px-6 py-3 text-left text-xs font-medium text-gray-500 uppercase">Estado</th>
                  <th class="px-6 py-3 text-left text-xs font-medium text-gray-500 uppercase">Acciones</th>
                </tr>
              </thead>
              <tbody class="divide-y divide-gray-200">
                <tr v-if="windowsLoading" class="text-center text-gray-500">
                  <td colspan="6" class="py-8">Cargando ventanillas...</td>
                </tr>
                <tr v-else-if="windows.length === 0" class="text-center text-gray-500">
                  <td colspan="6" class="py-8">No hay ventanillas creadas</td>
                </tr>
                <tr v-else v-for="window in windows" :key="window.id">
                  <td class="px-6 py-4">
                    <span class="bg-purple-100 text-purple-800 px-3 py-1 rounded-full text-sm font-bold">{{ window.number }}</span>
                  </td>
                  <td class="px-6 py-4 text-sm font-medium text-gray-900">{{ window.name }}</td>
                  <td class="px-6 py-4">
                    <code v-if="window.ip_address" class="bg-gray-100 px-2 py-1 rounded text-sm font-mono">{{ window.ip_address }}</code>
                    <span v-else class="text-gray-400 italic text-sm">Sin IP</span>
                  </td>
                  <td class="px-6 py-4 text-sm text-gray-600">
                    <span v-if="!window.service_names || window.service_names.length === 0" class="text-gray-400 italic">Todos</span>
                    <span v-else>{{ window.service_names.join(', ') }}</span>
                  </td>
                  <td class="px-6 py-4">
                    <span class="px-2 py-1 text-xs rounded-full" :class="window.is_active ? 'bg-green-100 text-green-800' : 'bg-red-100 text-red-800'">
                      {{ window.is_active ? 'Activa' : 'Inactiva' }}
                    </span>
                  </td>
                  <td class="px-6 py-4 text-sm space-x-3">
                    <button @click="openWindowModal(window)" class="text-blue-600 hover:text-blue-800" title="Editar">
                      <i class="fas fa-edit"></i>
                    </button>
                    <button @click="handleDeleteWindow(window)" class="text-red-600 hover:text-red-800" title="Eliminar">
                      <i class="fas fa-trash"></i>
                    </button>
                  </td>
                </tr>
              </tbody>
            </table>
          </div>
        </div>

        <!-- Servicios -->
        <div v-if="activeTab === 'servicios'" class="space-y-6">
          <div class="flex justify-between items-center">
            <h2 class="text-2xl font-bold text-gray-800">Gestión de Servicios</h2>
            <button @click="openServiceModal()" class="px-6 py-3 bg-blue-500 text-white rounded-lg hover:bg-blue-600 transition">
              <i class="fas fa-plus mr-2"></i>Nuevo Servicio
            </button>
          </div>
          
          <div class="bg-white rounded-lg shadow overflow-hidden">
            <table class="w-full">
              <thead class="bg-gray-50">
                <tr>
                  <th class="px-6 py-3 text-left text-xs font-medium text-gray-500 uppercase">Código</th>
                  <th class="px-6 py-3 text-left text-xs font-medium text-gray-500 uppercase">Nombre</th>
                  <th class="px-6 py-3 text-left text-xs font-medium text-gray-500 uppercase">Descripción</th>
                  <th class="px-6 py-3 text-left text-xs font-medium text-gray-500 uppercase">Color</th>
                  <th class="px-6 py-3 text-left text-xs font-medium text-gray-500 uppercase">Estado</th>
                  <th class="px-6 py-3 text-left text-xs font-medium text-gray-500 uppercase">Acciones</th>
                </tr>
              </thead>
              <tbody class="divide-y divide-gray-200">
                <tr v-if="servicesLoading" class="text-center text-gray-500">
                  <td colspan="6" class="py-8">Cargando servicios...</td>
                </tr>
                <tr v-else-if="services.length === 0" class="text-center text-gray-500">
                  <td colspan="6" class="py-8">No hay servicios creados</td>
                </tr>
                <tr v-else v-for="service in services" :key="service.id">
                  <td class="px-6 py-4 text-sm font-medium text-gray-900">{{ service.code }}</td>
                  <td class="px-6 py-4 text-sm text-gray-900">{{ service.name }}</td>
                  <td class="px-6 py-4 text-sm text-gray-600">{{ service.description }}</td>
                  <td class="px-6 py-4">
                    <div class="w-8 h-8 rounded" :style="{ backgroundColor: service.color }"></div>
                  </td>
                  <td class="px-6 py-4">
                    <span class="px-2 py-1 text-xs rounded-full" :class="service.is_active ? 'bg-green-100 text-green-800' : 'bg-red-100 text-red-800'">
                      {{ service.is_active ? 'Activo' : 'Inactivo' }}
                    </span>
                  </td>
                  <td class="px-6 py-4 text-sm space-x-3">
                    <button @click="openServiceModal(service)" class="text-blue-600 hover:text-blue-800" title="Editar">
                      <i class="fas fa-edit"></i>
                    </button>
                    <button @click="handleDeleteService(service)" class="text-red-600 hover:text-red-800" title="Eliminar">
                      <i class="fas fa-trash"></i>
                    </button>
                  </td>
                </tr>
              </tbody>
            </table>
          </div>
        </div>
      </div>
    </div>

    <!-- Modal de Servicio -->
    <div v-if="showServiceModal" class="fixed inset-0 bg-black/50 flex items-center justify-center z-50">
      <div class="bg-white p-8 rounded-lg max-w-2xl w-full max-h-[90vh] overflow-y-auto">
        <h3 class="text-2xl font-bold mb-6">{{ editingService ? 'Editar Servicio' : 'Nuevo Servicio' }}</h3>
        
        <!-- Error en modal -->
        <div v-if="serviceError" class="mb-4 p-3 bg-red-100 text-red-700 rounded-lg">
          {{ serviceError }}
        </div>
        
        <div class="space-y-4">
          <div>
            <label class="block text-sm font-medium text-gray-700 mb-1">Nombre *</label>
            <input v-model="serviceForm.name" type="text" class="w-full px-3 py-2 border border-gray-300 rounded-lg focus:outline-none focus:border-blue-500" />
          </div>
          
          <div>
            <label class="block text-sm font-medium text-gray-700 mb-1">Código *</label>
            <input v-model="serviceForm.code" type="text" maxlength="10" class="w-full px-3 py-2 border border-gray-300 rounded-lg focus:outline-none focus:border-blue-500 uppercase" />
          </div>
          
          <div>
            <label class="block text-sm font-medium text-gray-700 mb-1">Descripción</label>
            <textarea v-model="serviceForm.description" rows="3" class="w-full px-3 py-2 border border-gray-300 rounded-lg focus:outline-none focus:border-blue-500"></textarea>
          </div>
          
          <div>
            <label class="block text-sm font-medium text-gray-700 mb-1">Color</label>
            <input v-model="serviceForm.color" type="color" class="w-20 h-10 border border-gray-300 rounded" />
          </div>
          
          <div>
            <label class="flex items-center gap-2">
              <input v-model="serviceForm.is_active" type="checkbox" class="w-4 h-4" />
              <span class="text-sm font-medium text-gray-700">Servicio activo</span>
            </label>
          </div>
        </div>
        
        <div class="flex gap-4 mt-6">
          <button @click="saveService" :disabled="serviceSaving" class="flex-1 px-6 py-3 bg-blue-500 text-white rounded-lg hover:bg-blue-600 transition disabled:opacity-50">
            <i class="fas fa-save mr-2"></i>{{ serviceSaving ? 'Guardando...' : 'Guardar' }}
          </button>
          <button @click="closeServiceModal" class="flex-1 px-6 py-3 bg-gray-200 text-gray-700 rounded-lg hover:bg-gray-300 transition">
            Cancelar
          </button>
        </div>
      </div>
    </div>

    <!-- Modal de Usuario -->
    <div v-if="showUserModal" class="fixed inset-0 bg-black/50 flex items-center justify-center z-50">
      <div class="bg-white p-8 rounded-lg max-w-2xl w-full max-h-[90vh] overflow-y-auto">
        <h3 class="text-2xl font-bold mb-6">{{ editingUser ? 'Editar Usuario' : 'Nuevo Usuario' }}</h3>
        
        <!-- Error en modal -->
        <div v-if="userError" class="mb-4 p-3 bg-red-100 text-red-700 rounded-lg">
          {{ userError }}
        </div>
        
        <div class="space-y-4">
          <div>
            <label class="block text-sm font-medium text-gray-700 mb-1">Nombre *</label>
            <input v-model="userForm.name" type="text" class="w-full px-3 py-2 border border-gray-300 rounded-lg focus:outline-none focus:border-blue-500" />
          </div>
          
          <div>
            <label class="block text-sm font-medium text-gray-700 mb-1">Email *</label>
            <input v-model="userForm.email" type="email" class="w-full px-3 py-2 border border-gray-300 rounded-lg focus:outline-none focus:border-blue-500" />
          </div>
          
          <div>
            <label class="block text-sm font-medium text-gray-700 mb-1">
              Contraseña {{ editingUser ? '(dejar vacío para no cambiar)' : '*' }}
            </label>
            <input v-model="userForm.password" type="password" class="w-full px-3 py-2 border border-gray-300 rounded-lg focus:outline-none focus:border-blue-500" />
          </div>
          
          <div>
            <label class="block text-sm font-medium text-gray-700 mb-1">Rol *</label>
            <select v-model="userForm.role" class="w-full px-3 py-2 border border-gray-300 rounded-lg focus:outline-none focus:border-blue-500">
              <option value="admin">Administrador</option>
              <option value="operator">Operador</option>
            </select>
          </div>
          
          <div v-if="userForm.role === 'operator'">
            <label class="block text-sm font-medium text-gray-700 mb-2">Servicios asignados</label>
            <div v-if="servicesLoadingForUser" class="text-gray-500 text-sm py-2">
              Cargando servicios...
            </div>
            <div v-else class="space-y-2 max-h-48 overflow-y-auto border border-gray-200 rounded p-3">
              <label v-for="service in services" :key="service.id" class="flex items-center gap-2">
                <input v-model="userForm.services" :value="service.id" type="checkbox" class="w-4 h-4" />
                <span class="text-sm">{{ service.name }}</span>
              </label>
            </div>
            <p class="text-xs text-gray-500 mt-1">Si no selecciona ninguno, verá todos los servicios</p>
          </div>
        </div>
        
        <div class="flex gap-4 mt-6">
          <button @click="saveUser" :disabled="userSaving" class="flex-1 px-6 py-3 bg-blue-500 text-white rounded-lg hover:bg-blue-600 transition disabled:opacity-50">
            <i class="fas fa-save mr-2"></i>{{ userSaving ? 'Guardando...' : 'Guardar' }}
          </button>
          <button @click="closeUserModal" class="flex-1 px-6 py-3 bg-gray-200 text-gray-700 rounded-lg hover:bg-gray-300 transition">
            Cancelar
          </button>
        </div>
      </div>
    </div>

    <!-- Modal de Perfil -->
    <div v-if="showProfileModal" class="fixed inset-0 bg-black/50 flex items-center justify-center z-50">
      <div class="bg-white p-8 rounded-lg max-w-2xl w-full max-h-[90vh] overflow-y-auto">
        <h3 class="text-2xl font-bold mb-6">{{ editingProfile ? 'Editar Perfil' : 'Nuevo Perfil' }}</h3>
        
        <!-- Error en modal -->
        <div v-if="profileError" class="mb-4 p-3 bg-red-100 text-red-700 rounded-lg">
          {{ profileError }}
        </div>
        
        <div class="space-y-4">
          <div>
            <label class="block text-sm font-medium text-gray-700 mb-1">Nombre *</label>
            <input v-model="profileForm.name" type="text" placeholder="Ej: Principal, Pediatría, Laboratorio" class="w-full px-3 py-2 border border-gray-300 rounded-lg focus:outline-none focus:border-blue-500" />
          </div>
          
          <div>
            <label class="block text-sm font-medium text-gray-700 mb-1">Código *</label>
            <input v-model="profileForm.code" type="text" maxlength="50" placeholder="Ej: TOTEM-PRINCIPAL, PEDIATRIA" class="w-full px-3 py-2 border border-gray-300 rounded-lg focus:outline-none focus:border-blue-500 uppercase font-mono" />
            <p class="text-xs text-gray-500 mt-1">Este código se usa en el archivo .env del totem físico</p>
          </div>
          
          <div>
            <label class="block text-sm font-medium text-gray-700 mb-1">Ubicación (opcional)</label>
            <input v-model="profileForm.location" type="text" placeholder="Ej: Recepción principal, Piso 2" class="w-full px-3 py-2 border border-gray-300 rounded-lg focus:outline-none focus:border-blue-500" />
          </div>
          
          <div>
            <label class="block text-sm font-medium text-gray-700 mb-2">Servicios a mostrar *</label>
            <div v-if="servicesLoadingForProfile" class="text-gray-500 text-sm py-2">
              Cargando servicios...
            </div>
            <div v-else-if="services.length === 0" class="text-gray-500 text-sm py-2">
              No hay servicios creados. <span class="text-blue-600 cursor-pointer" @click="closeProfileModal(); activeTab = 'servicios'">Crear servicios primero</span>
            </div>
            <div v-else class="space-y-2 max-h-48 overflow-y-auto border border-gray-200 rounded p-3">
              <label v-for="service in services" :key="service.id" class="flex items-center gap-2">
                <input v-model="profileForm.services" :value="service.id" type="checkbox" class="w-4 h-4" />
                <span class="text-sm">{{ service.name }}</span>
                <span class="text-xs text-gray-400">({{ service.code }})</span>
              </label>
            </div>
            <p class="text-xs text-gray-500 mt-1">Seleccione al menos un servicio</p>
          </div>
          
          <div>
            <label class="flex items-center gap-2">
              <input v-model="profileForm.is_active" type="checkbox" class="w-4 h-4" />
              <span class="text-sm font-medium text-gray-700">Perfil activo</span>
            </label>
          </div>
          
          <div class="p-4 bg-blue-50 rounded-lg border border-blue-200">
            <label class="flex items-center gap-2">
              <input v-model="profileForm.ask_customer_name" type="checkbox" class="w-4 h-4" />
              <span class="text-sm font-medium text-gray-700">Pedir nombre del cliente</span>
            </label>
            <p class="text-xs text-gray-500 mt-1 ml-6">Si está desactivado, el ticket se imprime directamente sin mostrar el modal para ingresar nombre</p>
          </div>
        </div>
        
        <div class="flex gap-4 mt-6">
          <button @click="saveProfile" :disabled="profileSaving" class="flex-1 px-6 py-3 bg-blue-500 text-white rounded-lg hover:bg-blue-600 transition disabled:opacity-50">
            <i class="fas fa-save mr-2"></i>{{ profileSaving ? 'Guardando...' : 'Guardar' }}
          </button>
          <button @click="closeProfileModal" class="flex-1 px-6 py-3 bg-gray-200 text-gray-700 rounded-lg hover:bg-gray-300 transition">
            Cancelar
          </button>
        </div>
      </div>
    </div>

    <!-- Modal de Ventanilla -->
    <div v-if="showWindowModal" class="fixed inset-0 bg-black/50 flex items-center justify-center z-50">
      <div class="bg-white p-8 rounded-lg max-w-2xl w-full max-h-[90vh] overflow-y-auto">
        <h3 class="text-2xl font-bold mb-6">{{ editingWindow ? 'Editar Ventanilla' : 'Nueva Ventanilla' }}</h3>
        
        <!-- Error en modal -->
        <div v-if="windowError" class="mb-4 p-3 bg-red-100 text-red-700 rounded-lg">
          {{ windowError }}
        </div>
        
        <div class="space-y-4">
          <div>
            <label class="block text-sm font-medium text-gray-700 mb-1">Nombre *</label>
            <input v-model="windowForm.name" type="text" placeholder="Ej: Caja 1, Recepción" class="w-full px-3 py-2 border border-gray-300 rounded-lg focus:outline-none focus:border-blue-500" />
          </div>
          
          <div>
            <label class="block text-sm font-medium text-gray-700 mb-1">Número *</label>
            <input v-model.number="windowForm.number" type="number" min="1" placeholder="Ej: 1, 2, 3..." class="w-full px-3 py-2 border border-gray-300 rounded-lg focus:outline-none focus:border-blue-500" />
            <p class="text-xs text-gray-500 mt-1">Este número se muestra en el display público</p>
          </div>
          
          <div>
            <label class="block text-sm font-medium text-gray-700 mb-1">IP Asignada (opcional)</label>
            <input v-model="windowForm.ip_address" type="text" placeholder="Ej: 192.168.0.10" class="w-full px-3 py-2 border border-gray-300 rounded-lg focus:outline-none focus:border-blue-500 font-mono" />
            <p class="text-xs text-gray-500 mt-1">La PC con esta IP será asociada automáticamente a esta ventanilla</p>
          </div>
          
          <div>
            <label class="block text-sm font-medium text-gray-700 mb-2">Servicios que puede atender (opcional)</label>
            <div v-if="servicesLoadingForWindow" class="text-gray-500 text-sm py-2">
              Cargando servicios...
            </div>
            <div v-else-if="services.length === 0" class="text-gray-500 text-sm py-2">
              No hay servicios creados. <span class="text-blue-600 cursor-pointer" @click="closeWindowModal(); activeTab = 'servicios'">Crear servicios primero</span>
            </div>
            <div v-else class="space-y-2 max-h-48 overflow-y-auto border border-gray-200 rounded p-3">
              <label v-for="service in services" :key="service.id" class="flex items-center gap-2">
                <input v-model="windowForm.services" :value="service.id" type="checkbox" class="w-4 h-4" />
                <span class="text-sm">{{ service.name }}</span>
                <span class="text-xs text-gray-400">({{ service.code }})</span>
              </label>
            </div>
            <p class="text-xs text-gray-500 mt-1">Si no selecciona ninguno, la ventanilla puede atender todos los servicios</p>
          </div>
          
          <div>
            <label class="flex items-center gap-2">
              <input v-model="windowForm.is_active" type="checkbox" class="w-4 h-4" />
              <span class="text-sm font-medium text-gray-700">Ventanilla activa</span>
            </label>
          </div>
        </div>
        
        <div class="flex gap-4 mt-6">
          <button @click="saveWindow" :disabled="windowSaving" class="flex-1 px-6 py-3 bg-blue-500 text-white rounded-lg hover:bg-blue-600 transition disabled:opacity-50">
            <i class="fas fa-save mr-2"></i>{{ windowSaving ? 'Guardando...' : 'Guardar' }}
          </button>
          <button @click="closeWindowModal" class="flex-1 px-6 py-3 bg-gray-200 text-gray-700 rounded-lg hover:bg-gray-300 transition">
            Cancelar
          </button>
        </div>
      </div>
    </div>
  </div>
</template>

<script setup>
import { ref, onMounted, watch, reactive } from 'vue';
import { useRouter } from 'vue-router';
import { 
  getUser, 
  logout as apiLogout,
  getServices,
  createService,
  updateService,
  deleteService,
  getUsers,
  createUser,
  updateUser,
  deleteUser,
  getTotems,
  createTotem,
  updateTotem,
  deleteTotem,
  getWindows,
  createWindow,
  updateWindow,
  deleteWindow,
  getDashboardStats,
  getHistory,
  getOperators,
} from '../api';

const router = useRouter();
const user = ref(null);
const activeTab = ref('dashboard');

// Estado del dashboard
const dashboardStats = ref({});
const dashboardLoading = ref(false);

// Estado del historial
const historyData = ref([]);
const historyLoading = ref(false);
const historyPagination = ref({
  current_page: 1,
  last_page: 1,
  per_page: 15,
  total: 0,
});
const historyFilters = reactive({
  search: '',
  status: '',
  operator_id: '',
  service_id: '',
  date_from: '',
  date_to: '',
});
const operators = ref([]);
const expandedTurn = ref(null);

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

const tabs = [
  { id: 'dashboard', name: 'Dashboard', icon: 'fas fa-chart-bar' },
  { id: 'historial', name: 'Historial', icon: 'fas fa-history' },
  { id: 'usuarios', name: 'Usuarios', icon: 'fas fa-users' },
  { id: 'perfiles', name: 'Perfiles', icon: 'fas fa-layer-group' },
  { id: 'ventanillas', name: 'Ventanillas', icon: 'fas fa-desktop' },
  { id: 'servicios', name: 'Servicios', icon: 'fas fa-clipboard-list' },
];

// Estado de servicios
const services = ref([]);
const servicesLoading = ref(false);
const servicesLoadingForUser = ref(false);
const showServiceModal = ref(false);
const editingService = ref(null);
const serviceSaving = ref(false);
const serviceError = ref('');
const serviceForm = ref({
  name: '',
  code: '',
  description: '',
  color: '#667eea',
  is_active: true,
});

// Estado de usuarios
const users = ref([]);
const usersLoading = ref(false);
const showUserModal = ref(false);
const editingUser = ref(null);
const userSaving = ref(false);
const userError = ref('');
const userForm = ref({
  name: '',
  email: '',
  password: '',
  role: 'operator',
  services: [],
});

// Estado de perfiles
const profiles = ref([]);
const profilesLoading = ref(false);
const servicesLoadingForProfile = ref(false);
const showProfileModal = ref(false);
const editingProfile = ref(null);
const profileSaving = ref(false);
const profileError = ref('');
const profileForm = ref({
  name: '',
  code: '',
  location: '',
  services: [],
  is_active: true,
  ask_customer_name: true,
});

// Estado de ventanillas
const windows = ref([]);
const windowsLoading = ref(false);
const servicesLoadingForWindow = ref(false);
const showWindowModal = ref(false);
const editingWindow = ref(null);
const windowSaving = ref(false);
const windowError = ref('');
const windowForm = ref({
  name: '',
  number: null,
  ip_address: '',
  services: [],
  is_active: true,
});

const loadUser = async () => {
  try {
    const response = await getUser();
    user.value = response.data;
    
    // Verificar que sea admin
    if (user.value.role !== 'admin') {
      router.push('/operator');
    }
  } catch (error) {
    console.error('Error cargando usuario:', error);
    router.push('/login');
  }
};

// Dashboard
const loadDashboardStats = async () => {
  dashboardLoading.value = true;
  try {
    const response = await getDashboardStats();
    dashboardStats.value = response.data;
  } catch (error) {
    console.error('Error cargando estadísticas:', error);
    showToast('Error al cargar estadísticas', 'error');
  } finally {
    dashboardLoading.value = false;
  }
};

// Historial
const loadHistory = async (page = 1) => {
  historyLoading.value = true;
  try {
    const params = {
      page,
      per_page: historyPagination.value.per_page,
      ...historyFilters,
    };
    
    // Limpiar parámetros vacíos
    Object.keys(params).forEach(key => {
      if (params[key] === '' || params[key] === null) {
        delete params[key];
      }
    });
    
    const response = await getHistory(params);
    historyData.value = response.data.data;
    historyPagination.value = {
      current_page: response.data.current_page,
      last_page: response.data.last_page,
      per_page: response.data.per_page,
      total: response.data.total,
    };
  } catch (error) {
    console.error('Error cargando historial:', error);
    showToast('Error al cargar historial', 'error');
  } finally {
    historyLoading.value = false;
  }
};

const loadOperators = async () => {
  try {
    const response = await getOperators();
    operators.value = response.data;
  } catch (error) {
    console.error('Error cargando operadores:', error);
  }
};

const applyHistoryFilters = () => {
  loadHistory(1);
};

const clearHistoryFilters = () => {
  historyFilters.search = '';
  historyFilters.status = '';
  historyFilters.operator_id = '';
  historyFilters.service_id = '';
  historyFilters.date_from = '';
  historyFilters.date_to = '';
  loadHistory(1);
};

const goToPage = (page) => {
  if (page >= 1 && page <= historyPagination.value.last_page) {
    loadHistory(page);
  }
};

const toggleExpandTurn = (turnId) => {
  expandedTurn.value = expandedTurn.value === turnId ? null : turnId;
};

const getStatusLabel = (status) => {
  const labels = {
    waiting: 'En espera',
    called: 'Llamado',
    in_progress: 'En atención',
    completed: 'Completado',
    cancelled: 'Cancelado',
    no_show: 'No se presentó',
  };
  return labels[status] || status;
};

const getStatusColor = (status) => {
  const colors = {
    waiting: 'bg-yellow-100 text-yellow-800',
    called: 'bg-blue-100 text-blue-800',
    in_progress: 'bg-green-100 text-green-800',
    completed: 'bg-purple-100 text-purple-800',
    cancelled: 'bg-red-100 text-red-800',
    no_show: 'bg-gray-100 text-gray-800',
  };
  return colors[status] || 'bg-gray-100 text-gray-800';
};

const getTimelineIconColor = (color) => {
  const colors = {
    blue: 'bg-blue-500',
    yellow: 'bg-yellow-500',
    green: 'bg-green-500',
    purple: 'bg-purple-500',
    red: 'bg-red-500',
    gray: 'bg-gray-500',
  };
  return colors[color] || 'bg-gray-500';
};

const formatDuration = (minutes) => {
  if (minutes === null || minutes === undefined) return '-';
  const roundedMinutes = Math.round(minutes);
  if (roundedMinutes < 60) return `${roundedMinutes} min`;
  const hours = Math.floor(roundedMinutes / 60);
  const mins = roundedMinutes % 60;
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

// Servicios
const loadServices = async () => {
  servicesLoading.value = true;
  try {
    const response = await getServices();
    services.value = response.data;
  } catch (error) {
    console.error('Error cargando servicios:', error);
    showToast('Error al cargar servicios', 'error');
  } finally {
    servicesLoading.value = false;
  }
};

const loadServicesForUserModal = async () => {
  if (services.value.length > 0) return;
  
  servicesLoadingForUser.value = true;
  try {
    const response = await getServices();
    services.value = response.data;
  } catch (error) {
    console.error('Error cargando servicios:', error);
  } finally {
    servicesLoadingForUser.value = false;
  }
};

const openServiceModal = (service = null) => {
  serviceError.value = '';
  if (service) {
    editingService.value = service;
    serviceForm.value = {
      name: service.name,
      code: service.code,
      description: service.description || '',
      color: service.color || '#667eea',
      is_active: service.is_active !== undefined ? service.is_active : true,
    };
  } else {
    editingService.value = null;
    serviceForm.value = {
      name: '',
      code: '',
      description: '',
      color: '#667eea',
      is_active: true,
    };
  }
  showServiceModal.value = true;
};

const closeServiceModal = () => {
  showServiceModal.value = false;
  editingService.value = null;
  serviceError.value = '';
};

const saveService = async () => {
  serviceError.value = '';
  
  if (!serviceForm.value.name || !serviceForm.value.code) {
    serviceError.value = 'Nombre y código son obligatorios';
    return;
  }
  
  serviceSaving.value = true;
  try {
    if (editingService.value) {
      await updateService(editingService.value.id, serviceForm.value);
      showToast('Servicio actualizado correctamente');
    } else {
      await createService(serviceForm.value);
      showToast('Servicio creado correctamente');
    }
    await loadServices();
    closeServiceModal();
  } catch (error) {
    console.error('Error guardando servicio:', error);
    serviceError.value = error.response?.data?.errors 
      ? Object.values(error.response.data.errors).flat().join(', ')
      : error.response?.data?.message || 'Error al guardar servicio';
  } finally {
    serviceSaving.value = false;
  }
};

const handleDeleteService = (service) => {
  showConfirm(
    'Eliminar Servicio',
    `¿Está seguro de eliminar el servicio "${service.name}"?`,
    async () => {
      try {
        await deleteService(service.id);
        await loadServices();
        showToast('Servicio eliminado correctamente');
      } catch (error) {
        console.error('Error eliminando servicio:', error);
        showToast(error.response?.data?.message || 'Error al eliminar servicio', 'error');
      }
    }
  );
};

// Usuarios
const loadUsers = async () => {
  usersLoading.value = true;
  try {
    const response = await getUsers();
    users.value = response.data;
  } catch (error) {
    console.error('Error cargando usuarios:', error);
    showToast('Error al cargar usuarios', 'error');
  } finally {
    usersLoading.value = false;
  }
};

const openUserModal = async (usr = null) => {
  userError.value = '';
  
  // Cargar servicios si no están cargados (para el checkbox de asignar servicios)
  await loadServicesForUserModal();
  
  if (usr) {
    editingUser.value = usr;
    userForm.value = {
      name: usr.name,
      email: usr.email,
      password: '',
      role: usr.role,
      services: Array.isArray(usr.services) ? usr.services : [],
    };
  } else {
    editingUser.value = null;
    userForm.value = {
      name: '',
      email: '',
      password: '',
      role: 'operator',
      services: [],
    };
  }
  showUserModal.value = true;
};

const closeUserModal = () => {
  showUserModal.value = false;
  editingUser.value = null;
  userError.value = '';
};

const saveUser = async () => {
  userError.value = '';
  
  if (!userForm.value.name || !userForm.value.email) {
    userError.value = 'Nombre y email son obligatorios';
    return;
  }
  
  if (!editingUser.value && !userForm.value.password) {
    userError.value = 'La contraseña es obligatoria para nuevos usuarios';
    return;
  }
  
  userSaving.value = true;
  try {
    const data = { ...userForm.value };
    
    // Si es admin, limpiar servicios
    if (data.role === 'admin') {
      data.services = [];
    }
    
    // Si está editando y no hay password, no enviar password
    if (editingUser.value && !data.password) {
      delete data.password;
    }
    
    if (editingUser.value) {
      await updateUser(editingUser.value.id, data);
      showToast('Usuario actualizado correctamente');
    } else {
      await createUser(data);
      showToast('Usuario creado correctamente');
    }
    await loadUsers();
    closeUserModal();
  } catch (error) {
    console.error('Error guardando usuario:', error);
    userError.value = error.response?.data?.errors 
      ? Object.values(error.response.data.errors).flat().join(', ')
      : error.response?.data?.message || 'Error al guardar usuario';
  } finally {
    userSaving.value = false;
  }
};

const handleDeleteUser = (usr) => {
  showConfirm(
    'Eliminar Usuario',
    `¿Está seguro de eliminar el usuario "${usr.name}"?`,
    async () => {
      try {
        await deleteUser(usr.id);
        await loadUsers();
        showToast('Usuario eliminado correctamente');
      } catch (error) {
        console.error('Error eliminando usuario:', error);
        showToast(error.response?.data?.message || 'Error al eliminar usuario', 'error');
      }
    }
  );
};

// Perfiles
const loadProfiles = async () => {
  profilesLoading.value = true;
  try {
    const response = await getTotems();
    profiles.value = response.data;
  } catch (error) {
    console.error('Error cargando perfiles:', error);
    showToast('Error al cargar perfiles', 'error');
  } finally {
    profilesLoading.value = false;
  }
};

const loadServicesForProfileModal = async () => {
  if (services.value.length > 0) return;
  
  servicesLoadingForProfile.value = true;
  try {
    const response = await getServices();
    services.value = response.data;
  } catch (error) {
    console.error('Error cargando servicios:', error);
  } finally {
    servicesLoadingForProfile.value = false;
  }
};

const openProfileModal = async (profile = null) => {
  profileError.value = '';
  
  // Cargar servicios si no están cargados
  await loadServicesForProfileModal();
  
  if (profile) {
    editingProfile.value = profile;
    profileForm.value = {
      name: profile.name,
      code: profile.code,
      location: profile.location || '',
      services: Array.isArray(profile.services) ? profile.services : [],
      is_active: profile.is_active !== undefined ? profile.is_active : true,
      ask_customer_name: profile.ask_customer_name !== undefined ? profile.ask_customer_name : true,
    };
  } else {
    editingProfile.value = null;
    profileForm.value = {
      name: '',
      code: '',
      location: '',
      services: [],
      is_active: true,
      ask_customer_name: true,
    };
  }
  showProfileModal.value = true;
};

const closeProfileModal = () => {
  showProfileModal.value = false;
  editingProfile.value = null;
  profileError.value = '';
};

const saveProfile = async () => {
  profileError.value = '';
  
  if (!profileForm.value.name || !profileForm.value.code) {
    profileError.value = 'Nombre y código son obligatorios';
    return;
  }
  
  if (!profileForm.value.services || profileForm.value.services.length === 0) {
    profileError.value = 'Debe seleccionar al menos un servicio';
    return;
  }
  
  profileSaving.value = true;
  try {
    if (editingProfile.value) {
      await updateTotem(editingProfile.value.id, profileForm.value);
      showToast('Perfil actualizado correctamente');
    } else {
      await createTotem(profileForm.value);
      showToast('Perfil creado correctamente');
    }
    await loadProfiles();
    closeProfileModal();
  } catch (error) {
    console.error('Error guardando perfil:', error);
    profileError.value = error.response?.data?.errors 
      ? Object.values(error.response.data.errors).flat().join(', ')
      : error.response?.data?.message || 'Error al guardar perfil';
  } finally {
    profileSaving.value = false;
  }
};

const handleDeleteProfile = (profile) => {
  showConfirm(
    'Eliminar Perfil',
    `¿Está seguro de eliminar el perfil "${profile.name}"?`,
    async () => {
      try {
        await deleteTotem(profile.id);
        await loadProfiles();
        showToast('Perfil eliminado correctamente');
      } catch (error) {
        console.error('Error eliminando perfil:', error);
        showToast(error.response?.data?.message || 'Error al eliminar perfil', 'error');
      }
    }
  );
};

// Ventanillas
const loadWindows = async () => {
  windowsLoading.value = true;
  try {
    const response = await getWindows();
    windows.value = response.data;
  } catch (error) {
    console.error('Error cargando ventanillas:', error);
    showToast('Error al cargar ventanillas', 'error');
  } finally {
    windowsLoading.value = false;
  }
};

const loadServicesForWindowModal = async () => {
  if (services.value.length > 0) return;
  
  servicesLoadingForWindow.value = true;
  try {
    const response = await getServices();
    services.value = response.data;
  } catch (error) {
    console.error('Error cargando servicios:', error);
  } finally {
    servicesLoadingForWindow.value = false;
  }
};

const openWindowModal = async (window = null) => {
  windowError.value = '';
  
  // Cargar servicios si no están cargados
  await loadServicesForWindowModal();
  
  if (window) {
    editingWindow.value = window;
    windowForm.value = {
      name: window.name,
      number: window.number,
      ip_address: window.ip_address || '',
      services: Array.isArray(window.services) ? window.services : [],
      is_active: window.is_active !== undefined ? window.is_active : true,
    };
  } else {
    editingWindow.value = null;
    windowForm.value = {
      name: '',
      number: null,
      ip_address: '',
      services: [],
      is_active: true,
    };
  }
  showWindowModal.value = true;
};

const closeWindowModal = () => {
  showWindowModal.value = false;
  editingWindow.value = null;
  windowError.value = '';
};

const saveWindow = async () => {
  windowError.value = '';
  
  if (!windowForm.value.name || !windowForm.value.number) {
    windowError.value = 'Nombre y número son obligatorios';
    return;
  }
  
  windowSaving.value = true;
  try {
    if (editingWindow.value) {
      await updateWindow(editingWindow.value.id, windowForm.value);
      showToast('Ventanilla actualizada correctamente');
    } else {
      await createWindow(windowForm.value);
      showToast('Ventanilla creada correctamente');
    }
    await loadWindows();
    closeWindowModal();
  } catch (error) {
    console.error('Error guardando ventanilla:', error);
    windowError.value = error.response?.data?.errors 
      ? Object.values(error.response.data.errors).flat().join(', ')
      : error.response?.data?.message || 'Error al guardar ventanilla';
  } finally {
    windowSaving.value = false;
  }
};

const handleDeleteWindow = (window) => {
  showConfirm(
    'Eliminar Ventanilla',
    `¿Está seguro de eliminar la ventanilla "${window.name}"?`,
    async () => {
      try {
        await deleteWindow(window.id);
        await loadWindows();
        showToast('Ventanilla eliminada correctamente');
      } catch (error) {
        console.error('Error eliminando ventanilla:', error);
        showToast(error.response?.data?.message || 'Error al eliminar ventanilla', 'error');
      }
    }
  );
};

// Watcher para cargar datos cuando cambia la pestaña
watch(activeTab, (newTab) => {
  if (newTab === 'dashboard') {
    loadDashboardStats();
  }
  if (newTab === 'historial') {
    if (historyData.value.length === 0) {
      loadHistory();
    }
    if (operators.value.length === 0) {
      loadOperators();
    }
    if (services.value.length === 0) {
      loadServices();
    }
  }
  if (newTab === 'servicios' && services.value.length === 0) {
    loadServices();
  }
  if (newTab === 'usuarios' && users.value.length === 0) {
    loadUsers();
  }
  if (newTab === 'perfiles' && profiles.value.length === 0) {
    loadProfiles();
  }
  if (newTab === 'ventanillas' && windows.value.length === 0) {
    loadWindows();
  }
});

onMounted(() => {
  loadUser();
  loadDashboardStats();
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
