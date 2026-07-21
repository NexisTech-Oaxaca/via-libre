---
inclusion: fileMatch
fileMatchPattern: "frontend/**"
---

# Convenciones de Frontend — Nuxt 3

## Composables vs Componentes — cuándo usar cada uno

| Usar **composable** cuando... | Usar **componente** cuando... |
|---|---|
| La lógica se reutiliza en múltiples páginas/componentes | La estructura visual se repite |
| Necesitas encapsular estado reactivo + fetching | Solo hay presentación sin lógica compleja |
| Manejas side effects (WebSocket, eventos de mapa) | Es una pieza de UI con props bien definidas |
| La lógica no tiene template propio | Tiene un template propio que varía |

```ts
// ✅ Composable — lógica de fetching + estado
// composables/useIncidentes.ts
export function useIncidentes(zona?: Ref<string>) {
  const { data, pending, error, refresh } = useFetch('/api/incidentes', {
    query: zona ? { zona } : {},
    watch: zona ? [zona] : [],
  })

  const incidentesActivos = computed(() =>
    data.value?.data.filter(i => i.estado === 'activo') ?? []
  )

  return { incidentes: data, incidentesActivos, cargando: pending, error, refresh }
}

// ✅ Componente — presentación del marcador en el mapa
// components/mapa/MapaMarker.vue — tiene template + props visuales
```

---

## Convención de Props y Emits

Siempre definir props y emits con tipos explícitos en TypeScript:

```vue
<script setup lang="ts">
import type { Incidente } from '~/types'

// Props con defineProps tipado
const props = defineProps<{
  incidente: Incidente
  modoCompacto?: boolean
}>()

// Emits con defineEmits tipado
const emit = defineEmits<{
  confirmar: [incidenteId: string]
  cerrar: []
}>()

// Valores por defecto con withDefaults
const props = withDefaults(defineProps<{
  incidente: Incidente
  modoCompacto?: boolean
}>(), {
  modoCompacto: false,
})
</script>
```

---

## Uso de NuxtUI — componentes preferidos por caso

### Formularios
- `UForm` + `UFormGroup` para wrapping con validación
- `UInput`, `UTextarea`, `USelect` para campos
- `UButton` para acciones (no usar `<button>` nativo)
- `USelectMenu` para selects con búsqueda (ej. selector de zona)

```vue
<UForm :schema="schema" :state="form" @submit="onSubmit">
  <UFormGroup label="Título del incidente" name="titulo">
    <UInput v-model="form.titulo" placeholder="Describe brevemente el bloqueo" />
  </UFormGroup>

  <UFormGroup label="Tipo" name="tipo">
    <USelect v-model="form.tipo" :options="tiposIncidente" />
  </UFormGroup>

  <UButton type="submit" :loading="enviando" color="terracota">
    Reportar incidente
  </UButton>
</UForm>
```

### Modales
- `UModal` para detalles de incidente, confirmaciones, formularios
- `useModal()` para abrir modales de forma programática

```ts
// Abrir modal de detalle de incidente
const modal = useModal()

function abrirDetalle(incidente: Incidente) {
  modal.open(IncidenteModal, { incidente })
}
```

### Badges de estado
- `UBadge` con color semántico según estado del incidente

```vue
<UBadge :color="colorPorEstado(incidente.estado)" variant="subtle">
  {{ etiquetaEstado(incidente.estado) }}
</UBadge>
```

### Notificaciones / Toasts
- `useToast()` para feedback de acciones
- Nunca usar `alert()` nativo

```ts
const toast = useToast()
toast.add({ title: 'Incidente reportado', color: 'green', timeout: 4000 })
```

---

## Paleta de colores — Design Tokens Oaxaqueños

La paleta se define en `tailwind.config.ts` como colores personalizados:

```ts
// tailwind.config.ts
import type { Config } from 'tailwindcss'

export default {
  theme: {
    extend: {
      colors: {
        // Terracota / Rojo óxido — color principal, incidentes ACTIVOS
        terracota: {
          50:  '#fdf3f0',
          100: '#fbe3dc',
          200: '#f7c4b5',
          300: '#f29d87',
          400: '#eb6d51',
          500: '#c0392b', // base
          600: '#a93226',
          700: '#8a2820',
          800: '#6b1f19',
          900: '#4d1512',
        },
        // Verde oaxaqueño — incidentes FINALIZADOS, acciones positivas
        verde: {
          50:  '#f0f7f0',
          100: '#dceedd',
          200: '#b5ddb7',
          300: '#88c98b',
          400: '#52b057',
          500: '#27ae60', // base
          600: '#219a54',
          700: '#1a7d44',
          800: '#145f34',
          900: '#0d4024',
        },
        // Ocre / Amarillo mostaza — incidentes PROGRAMADOS, advertencias
        ocre: {
          50:  '#fefbf0',
          100: '#fdf5d9',
          200: '#faeab0',
          300: '#f5d97a',
          400: '#f0c93e',
          500: '#d4ac0d', // base
          600: '#b8950b',
          700: '#967809',
          800: '#735b07',
          900: '#4f3e05',
        },
        // Morado grana cochinilla — identidad visual, highlights especiales
        grana: {
          50:  '#f8f0f8',
          100: '#f0ddf0',
          200: '#e0b8e0',
          300: '#cc8acc',
          400: '#b557b5',
          500: '#8e44ad', // base
          600: '#7d3c99',
          700: '#663180',
          800: '#4d2460',
          900: '#33183f',
        },
        // Fondo hueso cálido
        hueso: {
          50:  '#fdfcf8',
          100: '#faf8f0',
          200: '#f5f0e4',
          300: '#ede5d2',
          400: '#e2d8be',
          500: '#d4c9a8', // base
        },
      },
    },
  },
} satisfies Config
```

### Uso semántico de colores por estado de incidente

| Estado | Color principal | Ejemplo de uso |
|---|---|---|
| `activo` | `terracota-500` | `color="terracota"` en UBadge |
| `programado` | `ocre-500` | `color="ocre"` en UBadge |
| `finalizado` | `verde-500` | `color="verde"` en UBadge |
| `no_verificado` | `gray-400` | `color="gray"` en UBadge |
| `cancelado` | `gray-500` | `color="gray"` en UBadge |
| `rechazado` | `red-700` | `color="red"` en UBadge |

```ts
// composables/useEstadoIncidente.ts
export function useEstadoIncidente() {
  const colorPorEstado = (estado: EstadoIncidente): string => {
    const mapa: Record<EstadoIncidente, string> = {
      activo:        'terracota',
      programado:    'ocre',
      finalizado:    'verde',
      no_verificado: 'gray',
      cancelado:     'gray',
      rechazado:     'red',
    }
    return mapa[estado] ?? 'gray'
  }

  const etiquetaEstado = (estado: EstadoIncidente): string => {
    const mapa: Record<EstadoIncidente, string> = {
      activo:        'Activo',
      programado:    'Programado',
      finalizado:    'Finalizado',
      no_verificado: 'Sin verificar',
      cancelado:     'Cancelado',
      rechazado:     'Rechazado',
    }
    return mapa[estado] ?? estado
  }

  return { colorPorEstado, etiquetaEstado }
}
```

### ❌ Anti-patrón — colores hardcodeados

```vue
<!-- ❌ MAL: color hardcodeado, no respeta el sistema -->
<span class="bg-red-500 text-white">Activo</span>

<!-- ✅ BIEN: usa el token semántico -->
<UBadge color="terracota" variant="subtle">Activo</UBadge>
```

---

## Patrones de fetching de datos

```ts
// En páginas — useFetch para SSR automático
const { data: incidente } = await useFetch(`/api/incidentes/${route.params.id}`)

// En composables — también useFetch
const { data, refresh } = useFetch('/api/incidentes', { query: { zona } })

// Mutaciones (POST/PUT/DELETE) — $fetch
const { execute, pending } = useAsyncData(
  'crear-incidente',
  () => $fetch('/api/incidentes', { method: 'POST', body: form }),
  { immediate: false }
)
```

---

## Configuración de NuxtUI con paleta personalizada

Para que NuxtUI use la paleta oaxaqueña en sus componentes, registrar los colores en `nuxt.config.ts`:

```ts
// nuxt.config.ts
export default defineNuxtConfig({
  ui: {
    primary: 'terracota',
    gray: 'hueso',
  },
})
```
