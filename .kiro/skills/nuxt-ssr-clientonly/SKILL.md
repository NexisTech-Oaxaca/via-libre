---
description: >
  Usar cuando se integra el mapa Leaflet en Nuxt, se conecta un WebSocket o canal de tiempo
  real, se trabaja con APIs del navegador como window/navigator/localStorage, o cualquier
  componente falla con errores de hidratación SSR en el frontend de ViaLibre.
---

# Skill: Nuxt SSR y ClientOnly

Guía para manejar correctamente componentes que dependen del navegador (`window`, `document`, WebSockets) en Nuxt 3 con SSR activado.

---

## El problema

Nuxt renderiza en el servidor (SSR) antes de enviar HTML al browser. Librerías como Leaflet y código con WebSockets usan APIs que solo existen en el browser (`window`, `navigator`, `document`). Si se importan directamente, rompen en el servidor.

**Error típico:**
```
ReferenceError: window is not defined
```

---

## Solución 1 — `<ClientOnly>` (la más común)

Envolver el componente que usa `window` en `<ClientOnly>`. El servidor renderiza el fallback (o nada), y el cliente monta el componente real.

```vue
<!-- pages/index.vue — vista principal del mapa -->
<template>
  <div class="h-screen">
    <!-- ✅ Leaflet solo se monta en el cliente -->
    <ClientOnly>
      <MapaBase :incidentes="incidentes" />

      <!-- Fallback visible mientras carga en el cliente -->
      <template #fallback>
        <div class="flex items-center justify-center h-full bg-hueso-100">
          <UIcon name="i-heroicons-map" class="w-12 h-12 text-terracota-300 animate-pulse" />
          <span class="ml-2 text-gray-500">Cargando mapa...</span>
        </div>
      </template>
    </ClientOnly>
  </div>
</template>
```

---

## Solución 2 — Plugin con sufijo `.client.ts`

Para librerías que deben importarse globalmente pero solo en el cliente. Nuxt detecta el sufijo `.client.ts` y no lo ejecuta en el servidor.

```ts
// plugins/leaflet.client.ts
import L from 'leaflet'
import 'leaflet/dist/leaflet.css'

// Corregir el bug de iconos de Leaflet con Vite
import iconUrl from 'leaflet/dist/images/marker-icon.png'
import iconRetinaUrl from 'leaflet/dist/images/marker-icon-2x.png'
import shadowUrl from 'leaflet/dist/images/marker-shadow.png'

delete (L.Icon.Default.prototype as any)._getIconUrl
L.Icon.Default.mergeOptions({ iconUrl, iconRetinaUrl, shadowUrl })

export default defineNuxtPlugin(() => {
  return {
    provide: {
      L, // disponible como $L en componentes si se necesita
    },
  }
})
```

---

## Solución 3 — `onMounted` para lógica puntual

Cuando solo necesitas ejecutar código del browser en un componente, sin necesidad de envolver todo en `<ClientOnly>`:

```ts
// Acceder a window solo en el cliente
onMounted(() => {
  // Aquí sí existe window
  window.addEventListener('resize', handleResize)
})

onUnmounted(() => {
  window.removeEventListener('resize', handleResize)
})
```

---

## Ejemplo completo — Componente del mapa con Leaflet

```vue
<!-- components/mapa/MapaBase.vue -->
<script setup lang="ts">
import type { Incidente } from '~/types'

// Leaflet solo se importa en el cliente gracias al plugin leaflet.client.ts
// Si se necesita importar directamente, usar import dinámico:
// const L = import.meta.client ? (await import('leaflet')).default : null

const props = defineProps<{
  incidentes: Incidente[]
}>()

const emit = defineEmits<{
  seleccionarIncidente: [incidente: Incidente]
}>()

const mapaRef = ref<HTMLElement | null>(null)
let mapa: any = null // tipo L.Map, pero L no existe en SSR

onMounted(async () => {
  // Importación dinámica — garantiza que solo corre en el cliente
  const L = (await import('leaflet')).default

  mapa = L.map(mapaRef.value!, {
    center: [17.0732, -96.7266], // Centro de Oaxaca
    zoom: 9,
  })

  L.tileLayer('https://{s}.tile.openstreetmap.org/{z}/{x}/{y}.png', {
    attribution: '© OpenStreetMap',
  }).addTo(mapa)

  // Agregar marcadores iniciales
  agregarMarcadores(L, props.incidentes)
})

onUnmounted(() => {
  mapa?.remove()
})

// Reaccionar a cambios en incidentes
watch(() => props.incidentes, async (nuevosIncidentes) => {
  if (!mapa) return
  const L = (await import('leaflet')).default
  // Limpiar y re-agregar marcadores
  mapa.eachLayer((layer: any) => {
    if (layer instanceof L.Marker) mapa.removeLayer(layer)
  })
  agregarMarcadores(L, nuevosIncidentes)
})

function agregarMarcadores(L: any, incidentes: Incidente[]) {
  incidentes.forEach(incidente => {
    const marker = L.marker([incidente.latitud, incidente.longitud])
    marker.on('click', () => emit('seleccionarIncidente', incidente))
    marker.addTo(mapa)
  })
}
</script>

<template>
  <!-- El div del mapa necesita altura explícita -->
  <div ref="mapaRef" class="w-full h-full" />
</template>
```

**Uso en la página:**

```vue
<!-- pages/index.vue -->
<template>
  <div class="h-screen flex flex-col">
    <ClientOnly>
      <MapaBase
        :incidentes="incidentesActivos"
        @seleccionar-incidente="abrirDetalle"
      />
      <template #fallback>
        <div class="flex-1 bg-hueso-100 flex items-center justify-center">
          <span class="text-gray-400">Cargando mapa...</span>
        </div>
      </template>
    </ClientOnly>
  </div>
</template>
```

---

## WebSockets con Laravel Echo — también requiere ClientOnly

```ts
// plugins/echo.client.ts
import Echo from 'laravel-echo'
import Pusher from 'pusher-js'

export default defineNuxtPlugin(() => {
  window.Pusher = Pusher // Pusher necesita window

  const echo = new Echo({
    broadcaster: 'reverb',
    key: useRuntimeConfig().public.reverbKey,
    wsHost: useRuntimeConfig().public.reverbHost,
    wsPort: useRuntimeConfig().public.reverbPort,
    forceTLS: false,
    enabledTransports: ['ws', 'wss'],
  })

  return {
    provide: { echo },
  }
})
```

```ts
// composables/useIncidentesRealtime.ts
export function useIncidentesRealtime(onNuevoIncidente: (i: Incidente) => void) {
  // Solo en el cliente — en SSR no existe $echo
  if (!import.meta.client) return

  const { $echo } = useNuxtApp()

  onMounted(() => {
    $echo.channel('incidentes')
      .listen('IncidenteCreado', (e: { incidente: Incidente }) => {
        onNuevoIncidente(e.incidente)
      })
  })

  onUnmounted(() => {
    $echo.leaveChannel('incidentes')
  })
}
```

---

## Reglas rápidas

| ¿Cuándo? | ¿Cómo? |
|---|---|
| Componente entero usa window/DOM | `<ClientOnly>` wrapping en el template |
| Librería JS del browser (Leaflet, Chart.js) | Plugin `.client.ts` o import dinámico en `onMounted` |
| Lógica puntual con window | `if (import.meta.client)` o dentro de `onMounted` |
| WebSockets / SSE | Plugin `.client.ts` + composable con `onMounted` |
| localStorage / sessionStorage | `if (import.meta.client)` guard |

---

## Configurar rutas como SPA (sin SSR) cuando sea necesario

Si una página entera depende del cliente (ej. `/mapa`), puedes desactivar SSR solo para esa ruta:

```ts
// nuxt.config.ts
export default defineNuxtConfig({
  routeRules: {
    '/mapa': { ssr: false }, // renderiza como SPA en esta ruta
  },
})
```
