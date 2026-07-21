---
inclusion: always
---

# Estructura del proyecto

## Monorepo

```
via-libre/
├── frontend/          # Nuxt 3 app
├── backend/           # Laravel 13 API
├── .kiro/             # Configuración de Kiro (steering, skills, agents)
└── README.md
```

---

## Frontend — `/frontend`

```
frontend/
├── assets/            # Imágenes estáticas, fonts, SVGs
├── components/        # Componentes Vue reutilizables
│   ├── incidente/     # Componentes del dominio "Incidente"
│   │   ├── IncidenteCard.vue
│   │   ├── IncidenteModal.vue
│   │   └── IncidenteFormReporte.vue
│   ├── mapa/          # Componentes del mapa (siempre con <ClientOnly>)
│   │   ├── MapaBase.vue
│   │   └── MapaMarker.vue
│   └── ui/            # Componentes UI genéricos (wrappers de NuxtUI)
├── composables/       # Lógica reutilizable con Composition API
│   ├── useIncidentes.ts
│   ├── useMapa.ts
│   └── useNotificaciones.ts
├── layouts/           # Layouts de Nuxt
│   ├── default.vue
│   └── mapa.vue
├── pages/             # Rutas del sistema (file-based routing)
│   ├── index.vue      # Vista principal del mapa
│   ├── incidentes/
│   │   └── [id].vue   # Detalle de incidente
│   └── cuenta/
│       └── index.vue
├── plugins/           # Plugins de Nuxt (ej. leaflet.client.ts)
├── public/            # Assets públicos sin procesamiento
├── server/            # Servidor Nuxt (nitro) — NO para lógica de negocio
├── stores/            # Stores de Pinia
│   ├── incidentes.ts
│   └── usuario.ts
├── types/             # Tipos TypeScript globales del dominio
│   └── index.d.ts
├── nuxt.config.ts
├── tailwind.config.ts
└── package.json
```

### Convención de nombres — Frontend

| Elemento | Convención | Ejemplo |
|---|---|---|
| Componentes Vue | PascalCase | `IncidenteCard.vue` |
| Composables | camelCase con prefijo `use` | `useIncidentes.ts` |
| Stores Pinia | camelCase, sufijo Store | `incidentesStore.ts` |
| Páginas | kebab-case o `[param]` | `[id].vue` |
| Tipos TS | PascalCase con sufijo Type o Interface | `IncidenteType` |
| Plugins cliente | sufijo `.client.ts` | `leaflet.client.ts` |
| Plugins servidor | sufijo `.server.ts` | `logger.server.ts` |

---

## Backend — `/backend`

```
backend/
├── app/
│   ├── Http/
│   │   ├── Controllers/
│   │   │   └── Api/           # Todos los controllers van en Api/
│   │   │       ├── IncidenteController.php
│   │   │       └── ConfirmacionController.php
│   │   ├── Requests/          # Form Requests de validación
│   │   │   ├── StoreIncidenteRequest.php
│   │   │   └── UpdateIncidenteRequest.php
│   │   └── Resources/         # API Resources (transformación de respuestas)
│   │       ├── IncidenteResource.php
│   │       └── IncidenteCollection.php
│   ├── Models/                # Modelos Eloquent
│   │   ├── User.php
│   │   ├── Incidente.php
│   │   ├── Confirmacion.php
│   │   └── Suscripcion.php
│   ├── Events/                # Eventos para broadcasting (tiempo real)
│   ├── Listeners/             # Listeners de eventos
│   ├── Jobs/                  # Jobs para colas (notificaciones async)
│   ├── Notifications/         # Notificaciones Laravel
│   └── Providers/
├── database/
│   ├── factories/             # Factories para testing
│   ├── migrations/            # Migraciones (una por tabla, orden por timestamp)
│   └── seeders/
├── routes/
│   ├── api.php                # TODAS las rutas del API aquí
│   └── console.php
├── tests/
│   ├── Feature/               # Tests de integración (HTTP, DB)
│   └── Unit/                  # Tests unitarios (modelos, helpers)
└── config/
```

### Convención de nombres — Backend

| Elemento | Convención | Ejemplo |
|---|---|---|
| Modelos | PascalCase singular | `Incidente.php` |
| Controllers | PascalCase + Controller, singular | `IncidenteController.php` |
| Form Requests | Acción + Modelo + Request | `StoreIncidenteRequest.php` |
| API Resources | Modelo + Resource | `IncidenteResource.php` |
| Migrations | timestamp + descripción snake_case | `2024_01_01_000000_create_incidentes_table.php` |
| Jobs | Acción en PascalCase | `EnviarNotificacionBloqueo.php` |
| Events | Pasado o presente | `IncidenteCreado.php` |

---

## Comunicación API

### Formato de respuesta estándar

```json
// Recurso único
{
  "data": { ... }
}

// Colección
{
  "data": [ ... ],
  "meta": {
    "current_page": 1,
    "last_page": 5,
    "per_page": 20,
    "total": 98
  }
}

// Error
{
  "message": "El incidente no fue encontrado.",
  "errors": {
    "campo": ["El campo es requerido."]
  }
}
```

### Convención de rutas REST

```
GET    /api/incidentes              → index
POST   /api/incidentes              → store
GET    /api/incidentes/{id}         → show
PUT    /api/incidentes/{id}         → update
DELETE /api/incidentes/{id}         → destroy
POST   /api/incidentes/{id}/confirmar → acción custom
```

- Prefijo `/api/` para todas las rutas
- Nombres en **español** y en **plural** (coincide con el dominio)
- Acciones custom como sub-rutas con verbo descriptivo
