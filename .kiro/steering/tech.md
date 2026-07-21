---
inclusion: always
---

# Stack y tecnologías

## Frontend — `/frontend`

- **Framework:** Nuxt 3 (modo SSR + CSR híbrido según la ruta)
- **API Style:** Composition API exclusivamente — no Options API
- **Lenguaje:** TypeScript estricto (`strict: true` en tsconfig)
- **UI Library:** NuxtUI v4.10.0
- **Estilos:** Tailwind CSS v3 con design tokens personalizados (paleta oaxaqueña)
- **Mapas:** Amazon Location Service
- **Notificaciones push:** Amazon SNS
- **HTTP Client:** `$fetch` / `useFetch` de Nuxt (nunca axios directamente)
- **Estado global:** Pinia
- **Testing:** Vitest + Vue Test Utils
- **Gestor de paquetes:** npm

### Restricciones técnicas conocidas

| Tecnología | Restricción |
|---|---|
| Leaflet | Usa `window` al importar → siempre dentro de `<ClientOnly>` o `onMounted` |
| WebSockets | Conexión solo en cliente → siempre dentro de `<ClientOnly>` |
| NuxtUI modals | Requieren `<UModal>` montado en el layout raíz via `useModal()` |
| SSR + Pinia | Usar `useNuxtApp().$pinia` para hidratación correcta |

---

## Backend — `/backend`

- **Framework:** Laravel 13
- **Lenguaje:** PHP 8.3+
- **ORM:** Eloquent
- **Autenticación:** Laravel Sanctum (API tokens para SPA)
- **Colas:** Laravel Queues con driver Redis (para notificaciones y procesamiento async)
- **Cache:** Redis
- **WebSockets:** Laravel Reverb (servidor propio) o Pusher (hosted)
- **Testing:** Pest PHP v3
- **Gestor de dependencias:** Composer

### Comandos frecuentes

```bash
# Backend
cd backend
composer install
php artisan migrate
php artisan db:seed
php artisan test           # corre tests con Pest
php artisan route:list     # ver rutas registradas
php artisan make:model Incidente -mfr   # modelo + migración + factory + resource

# Frontend
cd frontend
pnpm install
pnpm dev
pnpm build
pnpm test                  # Vitest en modo watch
pnpm test --run            # Vitest single run (para CI)
```

---

## Infraestructura (AWS)

- **Cómputo:** AWS Lambda (funciones serverless) o ECS Fargate
- **Autenticación:** AWS Cognito (gestión de usuarios)
- **Mapas/Geocoding:** AWS Location Service
- **Notificaciones push:** AWS SNS
- **Storage:** AWS S3 (imágenes de incidentes)
- **Base de datos:** Amazon RDS (PostgreSQL)
- **CDN:** CloudFront

---

## Comunicación Frontend ↔ Backend

- Protocolo: REST + JSON
- Base URL del API: `NUXT_PUBLIC_API_BASE_URL` (env var)
- Tiempo real: WebSocket via Laravel Echo + Reverb
- Autenticación: Bearer token (Sanctum) en header `Authorization`
