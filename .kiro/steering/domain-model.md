---
inclusion: always
---

# Modelo de dominio — ViaLibre

Este archivo es la fuente de verdad del vocabulario y las reglas del dominio. Cualquier feature nueva debe respetar estas entidades, estados y relaciones.

---

## Entidades principales

### Usuario (`User`)

Representa a cualquier persona registrada en la plataforma.

| Atributo | Tipo | Descripción |
|---|---|---|
| `id` | UUID | Identificador único |
| `nombre` | string | Nombre o alias público |
| `email` | string | Email único |
| `telefono` | string\|null | Teléfono opcional para notificaciones SMS |
| `rol` | enum | Ver roles abajo |
| `verificado_en` | datetime\|null | Cuándo fue verificado como usuario confiable |
| `created_at` | datetime | — |

**Roles de usuario:**

| Rol | Descripción | Permisos clave |
|---|---|---|
| `basico` | Usuario registrado sin verificación especial | Reportar, confirmar, suscribirse |
| `verificado` | Usuario con identidad o trayectoria confiable | Todo lo anterior + peso extra en confirmaciones |
| `moderador` | Staff de confianza | Editar, validar, archivar cualquier incidente |
| `admin` | Administrador del sistema | Acceso total, gestión de roles |

---

### Incidente (`Incidente`)

La entidad central del sistema. Representa un bloqueo o evento vial.

| Atributo | Tipo | Descripción |
|---|---|---|
| `id` | UUID | Identificador único |
| `titulo` | string | Descripción corta del incidente |
| `descripcion` | string\|null | Detalle ampliado |
| `tipo` | enum | Ver tipos abajo |
| `estado` | enum | Ver estados abajo |
| `latitud` | decimal | Coordenada geográfica |
| `longitud` | decimal | Coordenada geográfica |
| `zona` | string | Nombre de la zona/región de Oaxaca |
| `carretera` | string\|null | Nombre de la carretera afectada |
| `inicio_estimado` | datetime\|null | Cuándo inicia (para programados) |
| `fin_estimado` | datetime\|null | Cuándo termina (estimado) |
| `fin_real` | datetime\|null | Cuándo terminó realmente |
| `reportado_por` | UUID (FK User) | Usuario que creó el reporte |
| `confirmaciones_count` | int | Cache de confirmaciones positivas |
| `denegaciones_count` | int | Cache de confirmaciones negativas |
| `created_at` | datetime | — |
| `updated_at` | datetime | — |

**Tipos de incidente:**

| Valor | Descripción |
|---|---|
| `bloqueo_sindical` | Bloqueo por sindicato o gremio |
| `bloqueo_comunitario` | Bloqueo por comunidad o ejido |
| `manifestacion` | Marcha o concentración que afecta tráfico |
| `derrumbe` | Derrumbe o deslizamiento |
| `accidente` | Accidente vial |
| `obra` | Obra en vía pública |
| `otro` | Otro tipo de evento |

**Estados del incidente:**

| Estado | Descripción | Transiciones válidas |
|---|---|---|
| `programado` | Anunciado pero no ha iniciado | → `activo`, → `cancelado` |
| `activo` | Ocurriendo en este momento | → `finalizado` |
| `finalizado` | Ya terminó | (estado terminal) |
| `cancelado` | Fue programado pero no ocurrió | (estado terminal) |
| `no_verificado` | Reportado pero sin confirmaciones suficientes | → `activo`, → `rechazado` |
| `rechazado` | La comunidad lo denegó como falso | (estado terminal) |

> **Regla de negocio:** Un incidente pasa de `no_verificado` a `activo` cuando tiene ≥ 3 confirmaciones positivas netas (confirmaciones − denegaciones ≥ 3).

---

### Confirmación (`Confirmacion`)

Votos de la comunidad para validar o negar un incidente.

| Atributo | Tipo | Descripción |
|---|---|---|
| `id` | UUID | — |
| `incidente_id` | UUID (FK) | Incidente al que pertenece |
| `usuario_id` | UUID (FK) | Usuario que confirma/niega |
| `tipo` | enum | `confirma` \| `niega` |
| `comentario` | string\|null | Comentario opcional |
| `created_at` | datetime | — |

> **Regla:** Un usuario solo puede tener una confirmación por incidente. Si vuelve a enviar, reemplaza la anterior.

---

### Suscripción (`Suscripcion`)

Permite a un usuario recibir notificaciones sobre un incidente.

| Atributo | Tipo | Descripción |
|---|---|---|
| `id` | UUID | — |
| `usuario_id` | UUID (FK) | — |
| `incidente_id` | UUID (FK) | — |
| `canal` | enum | `push` \| `email` \| `sms` |
| `created_at` | datetime | — |

---

### Notificación (`Notificacion`)

Registro de notificaciones enviadas.

| Atributo | Tipo | Descripción |
|---|---|---|
| `id` | UUID | — |
| `usuario_id` | UUID (FK) | Destinatario |
| `incidente_id` | UUID (FK)\|null | Incidente relacionado |
| `tipo` | enum | `cambio_estado` \| `nueva_confirmacion` \| `nuevo_incidente_zona` |
| `titulo` | string | Título de la notificación |
| `cuerpo` | string | Contenido |
| `leida_en` | datetime\|null | Cuándo la leyó el usuario |
| `created_at` | datetime | — |

---

## Relaciones entre entidades

```
Usuario
  ├── hasMany → Incidente (reportados)
  ├── hasMany → Confirmacion
  ├── hasMany → Suscripcion
  └── hasMany → Notificacion

Incidente
  ├── belongsTo → Usuario (reportado_por)
  ├── hasMany → Confirmacion
  ├── hasMany → Suscripcion
  └── hasMany → Notificacion
```

---

## Glosario de términos del dominio

Usar estos términos de forma consistente en código, comentarios y UI:

| Término | Significado | NO usar |
|---|---|---|
| **Incidente** | Un bloqueo o evento vial | "reporte", "evento" |
| **Confirmar** | Validar que un incidente existe | "votar", "validar" |
| **Negar** | Indicar que un incidente es falso | "rechazar", "downvote" |
| **Zona** | Región geográfica de Oaxaca | "área", "región" |
| **Moderador** | Usuario con permisos de gestión | "admin", "editor" |
| **Suscripción** | Seguir un incidente para recibir alertas | "favorito", "seguir" |
