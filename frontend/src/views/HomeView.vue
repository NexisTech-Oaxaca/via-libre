<script setup lang="ts">
import { ref } from 'vue'
import axios from 'axios'
import api from '@/lib/api'
import { Button } from '@/components/ui/button'
import {
  Card,
  CardContent,
  CardDescription,
  CardHeader,
  CardTitle,
} from '@/components/ui/card'

type Estado = 'inicial' | 'probando' | 'ok' | 'error'

const estado = ref<Estado>('inicial')
const detalle = ref('')

const stack = [
  'Vue 3 + TypeScript',
  'Vite',
  'Tailwind CSS v4',
  'shadcn-vue',
  'Axios',
  'Vue Router',
  'Pinia',
  'PWA',
]

async function probarConexion() {
  estado.value = 'probando'
  detalle.value = ''

  try {
    const { data } = await api.get('/ping')
    estado.value = 'ok'
    detalle.value = `El backend respondió: ${JSON.stringify(data)}`
  } catch (error) {
    // Cualquier respuesta HTTP —incluido un 404— significa que el backend
    // está vivo y que CORS pasó. El backend todavía no tiene rutas de API.
    if (axios.isAxiosError(error) && error.response) {
      estado.value = 'ok'
      detalle.value =
        error.response.status === 404
          ? 'Backend alcanzado (404). Falta crear las rutas en routes/api.php.'
          : `Backend alcanzado, respondió ${error.response.status}.`
    } else {
      estado.value = 'error'
      detalle.value =
        'Sin respuesta. Revisa que el backend esté corriendo en http://localhost:8000.'
    }
  }
}
</script>

<template>
  <main class="min-h-svh bg-background px-6 py-16">
    <div class="mx-auto flex max-w-2xl flex-col gap-8">
      <header class="space-y-2">
        <h1 class="text-4xl font-semibold tracking-tight text-foreground">
          Vias
        </h1>
        <p class="text-muted-foreground">
          Reporte ciudadano de incidentes viales.
        </p>
      </header>

      <Card>
        <CardHeader>
          <CardTitle>Stack del frontend</CardTitle>
          <CardDescription>Todo instalado y funcionando.</CardDescription>
        </CardHeader>
        <CardContent>
          <ul class="flex flex-wrap gap-2">
            <li
              v-for="item in stack"
              :key="item"
              class="rounded-md bg-secondary px-2.5 py-1 text-sm text-secondary-foreground"
            >
              {{ item }}
            </li>
          </ul>
        </CardContent>
      </Card>

      <Card>
        <CardHeader>
          <CardTitle>Conexión con el backend</CardTitle>
          <CardDescription>
            Llama a <code class="text-xs">GET /api/ping</code> vía Axios.
          </CardDescription>
        </CardHeader>
        <CardContent class="space-y-4">
          <Button :disabled="estado === 'probando'" @click="probarConexion">
            {{ estado === 'probando' ? 'Probando…' : 'Probar conexión' }}
          </Button>

          <p
            v-if="detalle"
            class="text-sm"
            :class="estado === 'ok' ? 'text-foreground' : 'text-destructive'"
          >
            {{ detalle }}
          </p>
        </CardContent>
      </Card>
    </div>
  </main>
</template>
