---
description: >
  Usar cuando se aplican estilos en componentes del frontend, se define la paleta de colores
  oaxaqueña como tokens en Tailwind, se trabaja con NuxtUI y colores semánticos por estado
  de incidente, o se necesita asegurar consistencia visual en el proyecto ViaLibre.
---

# Skill: Tailwind Design Tokens — Paleta Oaxaqueña

Cómo definir y usar los colores de ViaLibre como tokens de Tailwind CSS, con mapping semántico por estado de incidente.

---

## Configuración en `tailwind.config.ts`

```ts
// frontend/tailwind.config.ts
import type { Config } from 'tailwindcss'

export default {
  content: [
    './components/**/*.{vue,ts}',
    './layouts/**/*.vue',
    './pages/**/*.vue',
    './plugins/**/*.{js,ts}',
    './nuxt.config.{js,ts}',
  ],
  theme: {
    extend: {
      colors: {
        // ── Terracota / Rojo óxido ─────────────────────────────────────
        // Uso: incidentes ACTIVOS, color primario de acción, alertas
        terracota: {
          50:  '#fdf3f0',
          100: '#fbe3dc',
          200: '#f7c4b5',
          300: '#f29d87',
          400: '#eb6d51',
          500: '#c0392b',  // ← base
          600: '#a93226',
          700: '#8a2820',
          800: '#6b1f19',
          900: '#4d1512',
          DEFAULT: '#c0392b',
        },

        // ── Verde oaxaqueño ────────────────────────────────────────────
        // Uso: incidentes FINALIZADOS, acciones de éxito, confirmaciones
        verde: {
          50:  '#f0f7f0',
          100: '#dceedd',
          200: '#b5ddb7',
          300: '#88c98b',
          400: '#52b057',
          500: '#27ae60',  // ← base
          600: '#219a54',
          700: '#1a7d44',
          800: '#145f34',
          900: '#0d4024',
          DEFAULT: '#27ae60',
        },

        // ── Ocre / Amarillo mostaza ────────────────────────────────────
        // Uso: incidentes PROGRAMADOS, advertencias, información pendiente
        ocre: {
          50:  '#fefbf0',
          100: '#fdf5d9',
          200: '#faeab0',
          300: '#f5d97a',
          400: '#f0c93e',
          500: '#d4ac0d',  // ← base
          600: '#b8950b',
          700: '#967809',
          800: '#735b07',
          900: '#4f3e05',
          DEFAULT: '#d4ac0d',
        },

        // ── Morado grana cochinilla ────────────────────────────────────
        // Uso: identidad visual, highlights, elementos de branding
        grana: {
          50:  '#f8f0f8',
          100: '#f0ddf0',
          200: '#e0b8e0',
          300: '#cc8acc',
          400: '#b557b5',
          500: '#8e44ad',  // ← base
          600: '#7d3c99',
          700: '#663180',
          800: '#4d2460',
          900: '#33183f',
          DEFAULT: '#8e44ad',
        },

        // ── Hueso cálido ──────────────────────────────────────────────
        // Uso: fondos de página, cards, superficies neutras
        hueso: {
          50:  '#fdfcf8',
          100: '#faf8f0',
          200: '#f5f0e4',
          300: '#ede5d2',
          400: '#e2d8be',
          500: '#d4c9a8',
          DEFAULT: '#faf8f0',
        },
      },

      // Sombras con tono cálido
      boxShadow: {
        'calida-sm': '0 1px 3px 0 rgba(192, 57, 43, 0.08)',
        'calida':    '0 4px 12px 0 rgba(192, 57, 43, 0.12)',
      },

      // Fuentes (si se usan fuentes custom)
      fontFamily: {
        sans: ['Inter', 'ui-sans-serif', 'system-ui'],
      },
    },
  },
  plugins: [],
} satisfies Config
```

---

## Integración con NuxtUI

```ts
// frontend/nuxt.config.ts
export default defineNuxtConfig({
  modules: ['@nuxt/ui'],

  ui: {
    primary: 'terracota',  // color principal de botones, focus rings, etc.
    gray: 'hueso',         // color de superficies neutras y grises
  },
})
```

> ⚠️ NuxtUI requiere que los colores `primary` y `gray` existan en `tailwind.config.ts` con todas las escalas (50–900).

---

## Mapping semántico — estado de incidente → color

Este es el contrato visual del sistema. **Respetar siempre este mapping.**

| Estado | Token Tailwind | Clase de fondo | Clase de texto | UBadge color |
|---|---|---|---|---|
| `activo` | `terracota` | `bg-terracota-500` | `text-terracota-700` | `color="terracota"` |
| `programado` | `ocre` | `bg-ocre-400` | `text-ocre-800` | `color="ocre"` |
| `finalizado` | `verde` | `bg-verde-500` | `text-verde-700` | `color="verde"` |
| `no_verificado` | `gray` | `bg-gray-300` | `text-gray-600` | `color="gray"` |
| `cancelado` | `gray` | `bg-gray-400` | `text-gray-700` | `color="gray"` |
| `rechazado` | `red` | `bg-red-600` | `text-red-800` | `color="red"` |

### Implementación en composable

```ts
// composables/useEstadoIncidente.ts
import type { EstadoIncidente } from '~/types'

const COLORES_ESTADO: Record<EstadoIncidente, string> = {
  activo:        'terracota',
  programado:    'ocre',
  finalizado:    'verde',
  no_verificado: 'gray',
  cancelado:     'gray',
  rechazado:     'red',
} as const

const ETIQUETAS_ESTADO: Record<EstadoIncidente, string> = {
  activo:        'Activo',
  programado:    'Programado',
  finalizado:    'Finalizado',
  no_verificado: 'Sin verificar',
  cancelado:     'Cancelado',
  rechazado:     'Rechazado',
} as const

export function useEstadoIncidente() {
  const colorPorEstado = (estado: EstadoIncidente): string =>
    COLORES_ESTADO[estado] ?? 'gray'

  const etiquetaEstado = (estado: EstadoIncidente): string =>
    ETIQUETAS_ESTADO[estado] ?? estado

  return { colorPorEstado, etiquetaEstado }
}
```

---

## Colores de marcadores del mapa

Para los marcadores de Leaflet, usar colores CSS variables que mapeen a los tokens:

```ts
// utils/marcadores.ts
export const COLORES_MARCADOR: Record<string, string> = {
  activo:        '#c0392b',  // terracota-500
  programado:    '#d4ac0d',  // ocre-500
  finalizado:    '#27ae60',  // verde-500
  no_verificado: '#9ca3af',  // gray-400
  cancelado:     '#6b7280',  // gray-500
  rechazado:     '#dc2626',  // red-600
}

export function colorMarcador(estado: string): string {
  return COLORES_MARCADOR[estado] ?? '#9ca3af'
}
```

---

## Anti-patrones a evitar

```vue
<!-- ❌ MAL: color hardcodeado que no respeta los tokens -->
<span class="bg-red-500 text-white px-2 rounded">Activo</span>
<div class="border-l-4 border-[#c0392b]">...</div>

<!-- ✅ BIEN: usa el token semántico -->
<UBadge color="terracota" variant="subtle">Activo</UBadge>
<div class="border-l-4 border-terracota-500">...</div>

<!-- ❌ MAL: lógica de color en el template -->
<UBadge :color="estado === 'activo' ? 'red' : estado === 'programado' ? 'yellow' : 'green'">

<!-- ✅ BIEN: lógica en composable -->
<UBadge :color="colorPorEstado(incidente.estado)" variant="subtle">
  {{ etiquetaEstado(incidente.estado) }}
</UBadge>
```

---

## Uso de tokens en CSS personalizado

Cuando necesitas usar los colores fuera de clases de utilidad (ej. en SVG o estilos de Leaflet):

```css
/* assets/css/mapa.css */
:root {
  --color-activo:        theme('colors.terracota.500');
  --color-programado:    theme('colors.ocre.500');
  --color-finalizado:    theme('colors.verde.500');
  --color-hueso:         theme('colors.hueso.100');
}
```

```ts
// Acceder desde JS/TS
const style = getComputedStyle(document.documentElement)
const colorActivo = style.getPropertyValue('--color-activo').trim()
```
