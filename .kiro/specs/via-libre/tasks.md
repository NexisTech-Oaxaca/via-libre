# Implementation Plan: ViaLibre — MVP Frontend

## Overview

Implementación incremental del MVP frontend-only de ViaLibre: una app Nuxt 3 que visualiza bloqueos carreteros en Oaxaca con datos estáticos (seed data). El orden prioriza: scaffolding → tipos → lógica pura testeable → PBT → seed data → stores → composables → UI. Cada tarea es ejecutable por un agente de código sin intervención manual.

## Tasks

- [ ] 1. Scaffolding del proyecto Nuxt 3 y configuración base
  - [ ] 1.1 Crear proyecto Nuxt 3, instalar dependencias y configurar TypeScript estricto
    - Inicializar proyecto en `frontend/` con `nuxi init`
    - Instalar dependencias: `@nuxt/ui`, `pinia`, `@pinia/nuxt`, `leaflet`, `@types/leaflet`, `vitest`, `@vue/test-utils`, `fast-check`, `@vitest/coverage-v8`
    - Configurar `nuxt.config.ts` con módulos: `@nuxt/ui`, `@pinia/nuxt`
    - Configurar `tsconfig.json` con `strict: true`
    - Configurar `vitest.config.ts` con environment `happy-dom` y paths aliases
    - _Requirements: 9.1_

  - [ ] 1.2 Configurar Tailwind CSS con paleta oaxaqueña y design tokens
    - Crear `tailwind.config.ts` con colores: terracota, verde-oax, ocre, grana, hueso, carbon
    - Incluir fontFamily: display (Playfair Display) y sans (Inter)
    - Incluir spacing safe-area-inset para mobile
    - Configurar `content` para todos los paths del proyecto
    - _Requirements: 1.7, 1.8_

- [ ] 2. Tipos TypeScript del dominio
  - [ ] 2.1 Crear archivo de tipos `frontend/types/index.ts` con todas las interfaces del dominio
    - Definir `IncidenteType`, `ConfirmacionType`, `SuscripcionType`, `UsuarioType`, `NotificacionType`
    - Definir enums/types: `EstadoIncidente`, `TipoIncidente`
    - Definir interfaces auxiliares: `FiltrosIncidente`, `NuevoIncidentePayload`, `ValidationResult`
    - Definir constante `TRANSICIONES_VALIDAS`
    - _Requirements: 3.1, 8.1_

- [ ] 3. Funciones puras del dominio (lógica de negocio testeable)
  - [ ] 3.1 Implementar `filtrarIncidentes` en `frontend/utils/domain/filtros.ts`
    - Filtrar por estados (array vacío = sin filtro) y por zonas (array vacío = sin filtro)
    - Retornar subconjunto exacto que cumple TODOS los criterios activos
    - _Requirements: 2.5, 2.6_

  - [ ] 3.2 Implementar `esTransicionValida` y `TRANSICIONES_VALIDAS` en `frontend/utils/domain/transiciones.ts`
    - Implementar tabla de transiciones: programado→[activo, cancelado], activo→[finalizado], no_verificado→[activo, rechazado], finalizado→[], cancelado→[], rechazado→[]
    - _Requirements: 8.1, 8.2_

  - [ ] 3.3 Implementar `calcularConfirmacionNeta` y `evaluarEstadoPorConfirmaciones` en `frontend/utils/domain/confirmaciones.ts`
    - `calcularConfirmacionNeta`: retorna `confirmaciones_count - denegaciones_count`
    - `evaluarEstadoPorConfirmaciones`: retorna `'activo'` si neta >= 3, `'rechazado'` si neta <= -3, `null` si no aplica o estado !== 'no_verificado'
    - _Requirements: 5.2, 5.5, 5.6, 5.8_

  - [ ] 3.4 Implementar `getColorByEstado` y `tieneRutaAlternativa` en `frontend/utils/domain/marcadores.ts`
    - Mapeo determinista: activo→#C0392B, programado→#D4A017, finalizado→#1E7A5F, cancelado→#999999, no_verificado→#7B2D8B, rechazado→#999999
    - `tieneRutaAlternativa`: true si y solo si `ruta_alternativa` no es null ni string vacío
    - _Requirements: 2.3, 10.1, 10.3_

  - [ ] 3.5 Implementar `validarFormularioReporte` en `frontend/utils/domain/validacion.ts`
    - Campos obligatorios: tipo, latitud, longitud
    - Validar imagen: max 5MB, solo JPEG/PNG
    - Retornar `ValidationResult` con errores solo en campos inválidos
    - _Requirements: 4.1, 4.6_

  - [ ] 3.6 Implementar `deduplicarDestinatarios` en `frontend/utils/domain/notificaciones.ts`
    - Recibir array de suscripciones y un incidente
    - Filtrar suscripciones que aplican (por incidente_id o por zona)
    - Retornar array de usuario_ids únicos (sin duplicados)
    - _Requirements: 6.7_

- [ ] 4. Property-based tests con fast-check
  - [ ]* 4.1 Crear generadores (arbitraries) base en `frontend/tests/properties/arbitraries.ts`
    - Implementar: `incidenteArbitrary`, `estadoArbitrary`, `tipoArbitrary`, `zonaArbitrary`, `filtrosArbitrary`
    - Usar rangos geográficos reales de Oaxaca (lat 15.5-18.5, lng -98.5 a -94.0)
    - _Requirements: 3.1_

  - [ ]* 4.2 Write property test: filtrado produce subconjunto exacto
    - **Property 1: Filtrado produce subconjunto exacto**
    - **Validates: Requirements 2.6**
    - Archivo: `frontend/tests/properties/filtrado.property.spec.ts`
    - Verificar que todo resultado satisface los filtros Y que no se omiten incidentes válidos

  - [ ]* 4.3 Write property test: color de marcador determinista
    - **Property 2: Color de marcador es función determinista del estado**
    - **Validates: Requirements 2.3**
    - Archivo: `frontend/tests/properties/marcadores.property.spec.ts`
    - Verificar mismo input → mismo output, output dentro de paleta definida

  - [ ]* 4.4 Write property test: confirmación neta es diferencia exacta
    - **Property 3: Confirmación neta es diferencia exacta**
    - **Validates: Requirements 5.2, 5.8**
    - Archivo: `frontend/tests/properties/confirmaciones.property.spec.ts`
    - Verificar que para todo (c, d) naturales, resultado === c - d

  - [ ]* 4.5 Write property test: transiciones de estado solo pares válidos
    - **Property 4: Transiciones de estado solo permiten pares válidos**
    - **Validates: Requirements 8.1, 8.2**
    - Archivo: `frontend/tests/properties/transiciones.property.spec.ts`
    - Verificar exhaustivamente los 36 pares posibles (6×6 estados)

  - [ ]* 4.6 Write property test: promoción automática no_verificado→activo
    - **Property 5: Promoción automática de no_verificado a activo**
    - **Validates: Requirements 5.5**
    - Archivo: `frontend/tests/properties/confirmaciones.property.spec.ts` (segunda propiedad)
    - Generar incidentes no_verificado con neta >= 3, verificar retorno 'activo'

  - [ ]* 4.7 Write property test: rechazo automático no_verificado→rechazado
    - **Property 6: Rechazo automático de no_verificado**
    - **Validates: Requirements 5.6**
    - Archivo: `frontend/tests/properties/confirmaciones.property.spec.ts` (tercera propiedad)
    - Generar incidentes no_verificado con neta <= -3, verificar retorno 'rechazado'

  - [ ]* 4.8 Write property test: idempotencia de confirmación por usuario
    - **Property 7: Idempotencia de confirmación por usuario**
    - **Validates: Requirements 5.4**
    - Archivo: `frontend/tests/properties/confirmaciones.property.spec.ts` (cuarta propiedad)
    - Verificar que doble confirmación resulta en exactamente un registro

  - [ ]* 4.9 Write property test: validación de formulario señala campos faltantes
    - **Property 8: Validación de formulario señala campos faltantes sin destruir datos**
    - **Validates: Requirements 4.6**
    - Archivo: `frontend/tests/properties/validacion.property.spec.ts`
    - Generar payloads parciales, verificar errores solo en campos vacíos

  - [ ]* 4.10 Write property test: ruta alternativa visible si y solo si existe
    - **Property 9: Ruta alternativa visible si y solo si existe**
    - **Validates: Requirements 10.1, 10.3**
    - Archivo: `frontend/tests/properties/marcadores.property.spec.ts` (segunda propiedad)
    - Verificar bicondicional: tieneRutaAlternativa ↔ campo no null ni vacío

  - [ ]* 4.11 Write property test: deduplicación de notificaciones
    - **Property 10: Deduplicación de notificaciones**
    - **Validates: Requirements 6.7**
    - Archivo: `frontend/tests/properties/notificaciones.property.spec.ts`
    - Generar suscripciones con overlap zona+incidente, verificar unicidad en resultado

- [ ] 5. Checkpoint — Verificar que funciones puras y PBTs pasan
  - Ensure all tests pass, ask the user if questions arise.

- [ ] 6. Seed data y stores Pinia
  - [ ] 6.1 Crear seed data en `frontend/data/incidentes.ts` con los 11 incidentes reales de Oaxaca
    - Importar tipo `IncidenteType`
    - Incluir los 11 incidentes exactos definidos en el design.md
    - Verificar: 6 activos, 2 programados, 2 finalizados, 1 no_verificado, 8 con ruta alternativa
    - Al menos 1 de cada tipo: bloqueo_sindical, bloqueo_comunitario, manifestacion, derrumbe, obra
    - _Requirements: 3.1, 3.2, 3.3, 3.4, 3.5_

  - [ ] 6.2 Implementar store `frontend/stores/incidentes.ts`
    - State: incidentes (desde seed), filtros, incidenteSeleccionado, cargando
    - Getters: incidentesFiltrados, incidenteById, zonasDisponibles, estadosPresentes, totalConfirmaciones
    - Actions: setFiltros, seleccionarIncidente, agregarConfirmacion, evaluarEstadoAutomatico, agregarIncidente
    - _Requirements: 2.5, 5.3, 5.5, 5.6_

  - [ ] 6.3 Implementar store `frontend/stores/usuario.ts`
    - State: usuario, autenticado
    - Getters: esModerador, esVerificado
    - Actions: setUsuario, logout
    - _Requirements: 7.3_

- [ ] 7. Composables
  - [ ] 7.1 Implementar `frontend/composables/useIncidentes.ts`
    - Encapsular acceso al store de incidentes
    - Exponer: incidentesFiltrados, buscarPorId, aplicarFiltros, limpiarFiltros, confirmar, negar, totalConfirmaciones, totalIncidentesActivos
    - _Requirements: 2.5, 5.1, 5.3_

  - [ ] 7.2 Implementar `frontend/composables/useMapa.ts`
    - Import dinámico de Leaflet (SSR-safe)
    - Exponer: mapaRef, inicializarMapa, centrarEn, getColorMarcador, getIconoMarcador, destruirMapa
    - Centro default: [17.0732, -96.7266], zoom default: 8
    - _Requirements: 2.1, 2.7_

  - [ ] 7.3 Implementar `frontend/composables/useNotificaciones.ts`
    - MVP stub: almacenamiento local en reactive state
    - Exponer: suscribirseIncidente, suscribirseZona, cancelarSuscripcion, notificaciones, noLeidas, marcarLeida
    - _Requirements: 6.1, 6.2, 6.6_

- [ ] 8. Plugin Leaflet y layouts
  - [ ] 8.1 Crear plugin `frontend/plugins/leaflet.client.ts`
    - Importar CSS de Leaflet globalmente en cliente
    - No importar la librería JS aquí (se hace dinámicamente en useMapa)
    - _Requirements: 2.7_

  - [ ] 8.2 Crear layout `frontend/layouts/default.vue`
    - Header: logo ViaLibre, navegación (Mapa, Reportar, Mi cuenta)
    - Slot principal
    - Footer con info del proyecto
    - Montar `<UModal>` para modales globales
    - Paleta oaxaqueña aplicada (bg-hueso, text-carbon)
    - _Requirements: 1.7_

  - [ ] 8.3 Crear layout `frontend/layouts/mapa.vue`
    - Header mínimo (logo + botón "Salir del mapa")
    - Slot fullscreen (100vh)
    - Sin footer
    - Panel lateral colapsable
    - _Requirements: 2.1_

- [ ] 9. Landing page
  - [ ] 9.1 Crear componentes landing: LandingHero, LandingProblema, LandingPasos, LandingMetricas, LandingCTA en `frontend/components/ui/`
    - LandingHero: tagline + botón "Ver bloqueos ahora" (→ /mapa) + botón "Reportar un bloqueo" (→ /reportar)
    - LandingProblema: descripción empática (max 3 líneas) sobre situación de bloqueos en Oaxaca
    - LandingPasos: 4 pasos (ver mapa, confirmar/reportar, suscribirse, recibir alertas)
    - LandingMetricas: total confirmaciones, moderadores activos, usuarios verificados
    - LandingCTA: repetición de CTAs antes del footer
    - _Requirements: 1.1, 1.2, 1.3, 1.5, 1.6_

  - [ ] 9.2 Crear página `frontend/pages/index.vue` con layout default
    - Componer los 5 componentes landing en orden
    - Tipografía display para títulos, sans para datos
    - Colores de paleta oaxaqueña
    - _Requirements: 1.1, 1.2, 1.3, 1.4, 1.5, 1.6, 1.7, 1.8_

- [ ] 10. Mapa interactivo
  - [ ] 10.1 Crear componentes mapa: MapaBase, MapaMarker, MapaPopup, MapaFiltros en `frontend/components/mapa/`
    - MapaBase: props (incidentes, centro, zoom), emit (marker-click, map-ready), wrapper `<ClientOnly>` con fallback
    - MapaMarker: props (incidente, activo), emit (click), color dinámico por estado, badge ↗ si tiene ruta alternativa
    - MapaPopup: props (incidente), emit (cerrar, ver-detalle), muestra título, tipo, zona, carretera, hora, confirmación neta, ruta alternativa si existe
    - MapaFiltros: props (estadosDisponibles, zonasDisponibles, filtrosActivos), emit (update:filtros)
    - _Requirements: 2.1, 2.2, 2.3, 2.4, 2.5, 2.7, 10.1, 10.3_

  - [ ] 10.2 Crear página `frontend/pages/mapa.vue` con layout mapa
    - Usar composable useIncidentes para datos filtrados
    - Integrar MapaBase + MapaFiltros
    - Panel lateral con detalle al hacer clic en marcador
    - Actualización reactiva de marcadores al cambiar filtros (< 500ms)
    - _Requirements: 2.1, 2.2, 2.4, 2.5, 2.6_

- [ ] 11. Formulario de reporte
  - [ ] 11.1 Crear componente `frontend/components/incidente/IncidenteFormReporte.vue`
    - Campos obligatorios: ubicación (pin en mapa mini o texto), tipo de incidente, es_programado
    - Campo opcional: descripción (max 500 chars)
    - Campo opcional: imagen (JPEG/PNG, max 5MB)
    - Validación inline con `validarFormularioReporte` — errores por campo sin limpiar otros
    - Indicación visual si no autenticado: "Tu reporte quedará como no verificado"
    - Mensaje de éxito post-envío con enlace al mapa
    - Usable en 320px mínimo (mobile-first)
    - _Requirements: 4.1, 4.2, 4.3, 4.4, 4.5, 4.6, 4.7, 4.8_

  - [ ] 11.2 Crear página `frontend/pages/reportar.vue`
    - Montar IncidenteFormReporte
    - Pasar coordenadas iniciales si vienen del mapa (query param)
    - Wiring del emit `submit` → store `agregarIncidente`
    - _Requirements: 4.1, 4.7_

- [ ] 12. Sistema de confirmaciones y detalle de incidente
  - [ ] 12.1 Crear componente `frontend/components/incidente/IncidenteConfirmaciones.vue`
    - Botones: "Confirmo que sigue así" / "Ya no está"
    - Mostrar confirmación neta actualizada
    - Si no autenticado: mostrar invitación a registrarse
    - Si ya confirmó: mostrar estado actual de su confirmación
    - _Requirements: 5.1, 5.2, 5.3, 5.7_

  - [ ] 12.2 Crear componentes `frontend/components/incidente/IncidenteCard.vue` e `IncidenteModal.vue`
    - IncidenteCard: props (incidente, compacto), emit (ver-detalle, confirmar, negar)
    - IncidenteModal: detalle completo con confirmaciones y ruta alternativa
    - _Requirements: 2.4, 10.1, 10.2_

  - [ ] 12.3 Crear página `frontend/pages/incidentes/[id].vue`
    - Obtener incidente por ID del store
    - Mostrar IncidenteCard completo + IncidenteConfirmaciones
    - Mostrar ruta alternativa si existe
    - Estado 404 amigable si no se encuentra
    - _Requirements: 2.4, 5.1, 10.1, 10.2_

- [ ] 13. Suscripciones, notificaciones y cuenta de usuario
  - [ ] 13.1 Crear UI de suscripciones y notificaciones (stub MVP)
    - Botón "Suscribirme a este incidente" en detalle
    - Botón "Suscribirme a esta zona" en filtros del mapa
    - Panel de notificaciones en header (badge con no leídas)
    - Marcar como leída al abrir
    - Todo local (Pinia), sin backend real
    - _Requirements: 6.1, 6.2, 6.4, 6.5, 6.6_

  - [ ] 13.2 Crear página `frontend/pages/cuenta/index.vue`
    - Vista de perfil simulada (nombre, email, rol)
    - Lista de suscripciones activas con opción de cancelar
    - Indicación visual de rol (básico/verificado)
    - Login/registro simulado para MVP (setUsuario directo)
    - _Requirements: 7.1, 7.3_

- [ ] 14. Checkpoint final — Verificar integración completa
  - Ensure all tests pass, ask the user if questions arise.

## Notes

- Tasks marked with `*` are optional and can be skipped for faster MVP
- Each task references specific requirements for traceability
- Property tests validate universal correctness properties using fast-check
- Unit tests validate specific examples and edge cases
- Seed data contains exactly 11 real Oaxaca incidents as defined in design.md
- All Leaflet usage is wrapped in `<ClientOnly>` for SSR safety
- The implementation language is TypeScript (as specified in the design document)
- Domain pure functions are implemented early to enable TDD with property tests
- Stores consume seed data directly — no HTTP requests in MVP

## Task Dependency Graph

```json
{
  "waves": [
    { "id": 0, "tasks": ["1.1"] },
    { "id": 1, "tasks": ["1.2", "2.1"] },
    { "id": 2, "tasks": ["3.1", "3.2", "3.3", "3.4", "3.5", "3.6"] },
    { "id": 3, "tasks": ["4.1"] },
    { "id": 4, "tasks": ["4.2", "4.3", "4.4", "4.5", "4.6", "4.7", "4.8", "4.9", "4.10", "4.11"] },
    { "id": 5, "tasks": ["6.1"] },
    { "id": 6, "tasks": ["6.2", "6.3"] },
    { "id": 7, "tasks": ["7.1", "7.2", "7.3"] },
    { "id": 8, "tasks": ["8.1", "8.2", "8.3"] },
    { "id": 9, "tasks": ["9.1", "10.1", "11.1"] },
    { "id": 10, "tasks": ["9.2", "10.2", "11.2", "12.1", "12.2"] },
    { "id": 11, "tasks": ["12.3", "13.1", "13.2"] }
  ]
}
```
