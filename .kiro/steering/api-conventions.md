---
inclusion: always
---

# API Conventions

## Formato de URL

- Base: `/api/`
- Recursos en **español** y **plural**: `/api/incidentes`, `/api/confirmaciones`
- Sin trailing slash
- Parámetros de ruta en snake_case: `/api/incidentes/{incidente_id}`

## Versioning

No se usa versioning de URL (`/api/v1/`) en esta etapa. Si se introduce, se hará via header `Accept: application/vnd.vialibre.v2+json`.

## Autenticación

- Header: `Authorization: Bearer {token}`
- Tokens emitidos por Laravel Sanctum
- Las rutas públicas (lectura de mapa) no requieren token

## Respuestas

### Éxito — recurso único
```json
{
  "data": {
    "id": "uuid",
    "titulo": "Bloqueo en carretera 175",
    "estado": "activo"
  }
}
```

### Éxito — colección paginada
```json
{
  "data": [ ... ],
  "meta": {
    "current_page": 1,
    "last_page": 5,
    "per_page": 20,
    "total": 98
  },
  "links": {
    "first": "...",
    "last": "...",
    "prev": null,
    "next": "..."
  }
}
```

### Error de validación — 422
```json
{
  "message": "Los datos enviados no son válidos.",
  "errors": {
    "titulo": ["El título es obligatorio."],
    "tipo": ["El tipo de incidente no es válido."]
  }
}
```

### Error genérico
```json
{
  "message": "Descripción del error en español."
}
```

## Códigos HTTP usados

| Código | Cuándo |
|---|---|
| `200` | GET exitoso, PUT/PATCH exitoso |
| `201` | POST exitoso (recurso creado) |
| `204` | DELETE exitoso (sin cuerpo) |
| `401` | No autenticado |
| `403` | Autenticado pero sin permiso |
| `404` | Recurso no encontrado |
| `422` | Error de validación |
| `500` | Error del servidor |

## Query Parameters estándar

| Parámetro | Tipo | Descripción |
|---|---|---|
| `zona` | string | Filtrar por zona de Oaxaca |
| `estado` | string | Filtrar por estado del incidente |
| `tipo` | string | Filtrar por tipo de incidente |
| `page` | int | Número de página (paginación) |
| `per_page` | int | Resultados por página (máx 100) |
| `lat` | decimal | Latitud para búsqueda por proximidad |
| `lng` | decimal | Longitud para búsqueda por proximidad |
| `radio_km` | int | Radio en km para búsqueda geográfica |

## Fechas y timestamps

- Formato: ISO 8601 con timezone → `2024-11-01T14:30:00-06:00`
- Timezone del servidor: `America/Mexico_City`
- El frontend convierte a hora local del usuario

## Eventos de tiempo real (WebSocket)

Los canales de Laravel Echo siguen esta convención:

| Canal | Evento | Descripción |
|---|---|---|
| `incidentes` | `IncidenteCreado` | Nuevo incidente reportado |
| `incidentes.{id}` | `IncidenteActualizado` | Cambio de estado o datos |
| `incidentes.{id}` | `ConfirmacionAgregada` | Nueva confirmación |
| `zona.{zona}` | `IncidenteEnZona` | Incidente nuevo en una zona específica |
