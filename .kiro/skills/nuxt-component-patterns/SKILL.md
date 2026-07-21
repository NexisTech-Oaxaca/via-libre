---
description: >
  Usar cuando se crea un componente Vue o composable en el frontend de ViaLibre, se trabaja
  con formularios de reporte de incidente usando NuxtUI, se muestra el detalle de un incidente
  en modal o sheet, o se necesita decidir entre composable vs componente.
---

# Skill: Nuxt Component Patterns

Patrones de componentes y composables para ViaLibre con Nuxt 3 + NuxtUI.

---

## Árbol de decisión: composable vs componente

```
¿Tiene template?
├── No → Composable (useXxx.ts)
└── Sí → ¿Se reutiliza la lógica en múltiples lugares SIN el template?
         ├── Sí → Extraer lógica a composable + componente usa el composable
         └── No → Solo componente con lógica interna
```

---

## Composables del dominio ViaLibre

### `useIncidentes` — fetching y estado del mapa

```ts
// composables/useIncidentes.ts
import type { Incidente, FiltrosIncidente } from '~/types'

export function useIncidentes(filtros?: Ref<FiltrosIncidente>) {
  const { data, pending, error, refresh } = useFetch<{ data: Incidente[] }>(
    '/api/incidentes',
    {
      query: filtros,
      watch: filtros ? [filtros] : [],
    }
  )

  const incidentesActivos = computed(
    () => data.value?.data.filter(i => i.estado === 'activo') ?? []
  )

  const incidentesPorZona = (zona: string) =>
    computed(() => data.value?.data.filter(i => i.zona === zona) ?? [])

  return {
    incidentes: computed(() => data.value?.data ?? []),
    incidentesActivos,
    incidentesPorZona,
    cargando: pending,
    error,
    refresh,
  }
}
```

### `useConfirmar` — acción de confirmar/negar un incidente

```ts
// composables/useConfirmar.ts
import type { TipoConfirmacion } from '~/types'

export function useConfirmar(incidenteId: string) {
  const enviando = ref(false)
  const error = ref<string | null>(null)
  const toast = useToast()

  async function confirmar(tipo: TipoConfirmacion, comentario?: string) {
    enviando.value = true
    error.value = null

    try {
      await $fetch(`/api/incidentes/${incidenteId}/confirmar`, {
        method: 'POST',
        body: { tipo, comentario },
      })
      toast.add({
        title: tipo === 'confirma' ? 'Incidente confirmado' : 'Incidente negado',
        color: tipo === 'confirma' ? 'green' : 'orange',
      })
    } catch (e: any) {
      error.value = e.data?.message ?? 'Error al procesar tu confirmación'
      toast.add({ title: error.value!, color: 'red' })
    } finally {
      enviando.value = false
    }
  }

  return { confirmar, enviando, error }
}
```

---

## Patrón: Formulario de reporte de incidente con NuxtUI

```vue
<!-- components/incidente/IncidenteFormReporte.vue -->
<script setup lang="ts">
import { z } from 'zod'
import type { FormSubmitEvent } from '#ui/types'

const emit = defineEmits<{
  exitoso: [incidenteId: string]
  cancelar: []
}>()

const schema = z.object({
  titulo:   z.string().min(5, 'Mínimo 5 caracteres').max(150),
  tipo:     z.enum(['bloqueo_sindical', 'bloqueo_comunitario', 'manifestacion',
                    'derrumbe', 'accidente', 'obra', 'otro']),
  zona:     z.string().min(1, 'Selecciona una zona'),
  carretera: z.string().optional(),
  descripcion: z.string().max(1000).optional(),
})

type FormData = z.output<typeof schema>

const form = reactive<Partial<FormData>>({
  titulo: '',
  tipo: undefined,
  zona: '',
})

const enviando = ref(false)
const toast = useToast()

const tiposIncidente = [
  { value: 'bloqueo_sindical',    label: 'Bloqueo sindical' },
  { value: 'bloqueo_comunitario', label: 'Bloqueo comunitario' },
  { value: 'manifestacion',       label: 'Manifestación' },
  { value: 'derrumbe',            label: 'Derrumbe' },
  { value: 'accidente',           label: 'Accidente' },
  { value: 'obra',                label: 'Obra vial' },
  { value: 'otro',                label: 'Otro' },
]

const zonas = [
  'Valles Centrales', 'Sierra Norte', 'Cañada', 'Mixteca',
  'Papaloapan', 'Sierra Sur', 'Costa', 'Istmo',
]

async function onSubmit(event: FormSubmitEvent<FormData>) {
  enviando.value = true
  try {
    const incidente = await $fetch<{ data: { id: string } }>('/api/incidentes', {
      method: 'POST',
      body: {
        ...event.data,
        // latitud/longitud se agregan desde el composable useMapa
      },
    })
    toast.add({ title: 'Incidente reportado. ¡Gracias!', color: 'green' })
    emit('exitoso', incidente.data.id)
  } catch (e: any) {
    toast.add({
      title: 'No se pudo reportar el incidente',
      description: e.data?.message,
      color: 'red',
    })
  } finally {
    enviando.value = false
  }
}
</script>

<template>
  <UForm :schema="schema" :state="form" @submit="onSubmit" class="space-y-4">
    <UFormGroup label="Título del incidente" name="titulo" required>
      <UInput
        v-model="form.titulo"
        placeholder="Ej: Bloqueo en km 45 carretera 175"
      />
    </UFormGroup>

    <UFormGroup label="Tipo de incidente" name="tipo" required>
      <USelect v-model="form.tipo" :options="tiposIncidente" placeholder="Selecciona..." />
    </UFormGroup>

    <UFormGroup label="Zona" name="zona" required>
      <USelectMenu v-model="form.zona" :options="zonas" searchable placeholder="Busca una zona..." />
    </UFormGroup>

    <UFormGroup label="Carretera" name="carretera">
      <UInput v-model="form.carretera" placeholder="Ej: Federal 190" />
    </UFormGroup>

    <UFormGroup label="Descripción" name="descripcion">
      <UTextarea v-model="form.descripcion" placeholder="Agrega más detalles..." rows="3" />
    </UFormGroup>

    <div class="flex justify-end gap-3 pt-2">
      <UButton variant="ghost" @click="emit('cancelar')">Cancelar</UButton>
      <UButton type="submit" :loading="enviando" color="primary">
        Reportar incidente
      </UButton>
    </div>
  </UForm>
</template>
```

---

## Patrón: Modal de detalle de incidente

```vue
<!-- components/incidente/IncidenteModal.vue -->
<script setup lang="ts">
import type { Incidente } from '~/types'
import { useEstadoIncidente } from '~/composables/useEstadoIncidente'

const props = defineProps<{
  incidente: Incidente
}>()

const emit = defineEmits<{ cerrar: [] }>()

const { colorPorEstado, etiquetaEstado } = useEstadoIncidente()
const { confirmar, enviando } = useConfirmar(props.incidente.id)

// Detalle completo con confirmaciones
const { data: detalle } = await useFetch<{ data: Incidente }>(
  `/api/incidentes/${props.incidente.id}`,
  { lazy: true }
)
</script>

<template>
  <UModal @close="emit('cerrar')">
    <UCard>
      <template #header>
        <div class="flex items-center justify-between">
          <h2 class="text-lg font-semibold">{{ incidente.titulo }}</h2>
          <UBadge :color="colorPorEstado(incidente.estado)" variant="subtle">
            {{ etiquetaEstado(incidente.estado) }}
          </UBadge>
        </div>
      </template>

      <!-- Contenido del detalle -->
      <div class="space-y-3">
        <p class="text-sm text-gray-600">{{ incidente.descripcion }}</p>

        <div class="flex gap-4 text-sm">
          <span>📍 {{ incidente.zona }}</span>
          <span v-if="incidente.carretera">🛣️ {{ incidente.carretera }}</span>
        </div>

        <div class="flex items-center gap-3 pt-2">
          <span class="text-sm text-gray-500">
            ✅ {{ incidente.confirmaciones_count }} confirmaciones
            · ❌ {{ incidente.denegaciones_count }} negaciones
          </span>
        </div>
      </div>

      <template #footer>
        <div class="flex gap-2 justify-end">
          <UButton
            color="green"
            variant="soft"
            :loading="enviando"
            @click="confirmar('confirma')"
          >
            Confirmar
          </UButton>
          <UButton
            color="orange"
            variant="soft"
            :loading="enviando"
            @click="confirmar('niega')"
          >
            Negar
          </UButton>
        </div>
      </template>
    </UCard>
  </UModal>
</template>
```

---

## Abrir el modal desde cualquier componente

```ts
// En cualquier componente o página
const modal = useModal()

function verDetalle(incidente: Incidente) {
  modal.open(IncidenteModal, {
    incidente,
    onCerrar() {
      modal.close()
    },
  })
}
```

> Para que `useModal()` funcione, el componente `<UModals />` debe estar en `layouts/default.vue`.
