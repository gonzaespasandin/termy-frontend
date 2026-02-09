<template>
  <div class="w-screen h-screen flex items-center justify-center m-0" style="background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);">
    <div class="bg-white p-12 rounded-3xl max-w-md w-11/12 shadow-2xl">
      <h1 class="text-4xl font-bold text-center mb-8 text-gray-800">Iniciar Sesión</h1>
      
      <form @submit.prevent="handleLogin" class="space-y-6">
        <div>
          <label class="block text-gray-700 text-lg font-semibold mb-2">Email</label>
          <input
            v-model="email"
            type="email"
            required
            class="w-full p-4 border-2 border-gray-300 rounded-lg text-lg focus:outline-none focus:border-blue-500"
            placeholder="operador@turnero.com"
          />
        </div>

        <div>
          <label class="block text-gray-700 text-lg font-semibold mb-2">Contraseña</label>
          <input
            v-model="password"
            type="password"
            required
            class="w-full p-4 border-2 border-gray-300 rounded-lg text-lg focus:outline-none focus:border-blue-500"
            placeholder="••••••••"
          />
        </div>

        <div v-if="error" class="bg-red-100 border border-red-400 text-red-700 px-4 py-3 rounded">
          {{ error }}
        </div>

        <button 
          type="submit" 
          :disabled="loading"
          class="w-full py-4 text-white text-xl font-semibold rounded-lg hover:opacity-90 transition-opacity disabled:opacity-50"
          style="background-color: #667eea;"
        >
          {{ loading ? 'Iniciando sesión...' : 'Iniciar Sesión' }}
        </button>
      </form>

      <div class="mt-8 text-center">
        <router-link to="/" class="text-gray-600 hover:text-gray-800 underline">
          Volver al totem
        </router-link>
      </div>
    </div>
  </div>
</template>

<script setup>
import { ref } from 'vue';
import { useRouter } from 'vue-router';
import { login } from '../api';

const router = useRouter();
const email = ref('');
const password = ref('');
const loading = ref(false);
const error = ref('');

const handleLogin = async () => {
  loading.value = true;
  error.value = '';

  try {
    const response = await login({
      email: email.value,
      password: password.value,
    });

    // Guardar token y usuario
    localStorage.setItem('token', response.data.token);
    localStorage.setItem('user', JSON.stringify(response.data.user));

    // Redirigir según el rol
    if (response.data.user.role === 'admin') {
      router.push('/admin');
    } else {
      router.push('/operator');
    }
  } catch (err) {
    console.error('Error en login:', err);
    error.value = err.response?.data?.message || 'Credenciales incorrectas';
  } finally {
    loading.value = false;
  }
};
</script>

