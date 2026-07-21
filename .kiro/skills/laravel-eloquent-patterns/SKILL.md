---
description: >
  Usar cuando se trabaja con modelos Eloquent del dominio de ViaLibre, se definen relaciones
  entre entidades (Incidente, Confirmacion, Suscripcion, Usuario), se crean scopes de query,
  o se necesita optimizar consultas del mapa para evitar problemas de N+1.
---

# Skill: Laravel Eloquent Patterns

Relaciones, scopes y optimización de queries para los modelos de ViaLibre.

---

## Relaciones entre modelos

### Usuario

```php
// app/Models/User.php
class User extends Authenticatable
{
    use HasUuids;

    protected $fillable = ['nombre', 'email', 'telefono', 'rol', 'verificado_en'];

    protected $casts = [
        'rol'           => RolUsuario::class,
        'verificado_en' => 'datetime',
    ];

    public function incidentes(): HasMany
    {
        return $this->hasMany(Incidente::class, 'reportado_por');
    }

    public function confirmaciones(): HasMany
    {
        return $this->hasMany(Confirmacion::class, 'usuario_id');
    }

    public function suscripciones(): HasMany
    {
        return $this->hasMany(Suscripcion::class, 'usuario_id');
    }

    public function notificaciones(): HasMany
    {
        return $this->hasMany(Notificacion::class, 'usuario_id');
    }

    // Scopes de usuario
    public function scopeVerificados(Builder $query): Builder
    {
        return $query->whereNotNull('verificado_en')
                     ->where('rol', RolUsuario::Verificado);
    }

    public function scopeModerador(Builder $query): Builder
    {
        return $query->whereIn('rol', [RolUsuario::Moderador, RolUsuario::Admin]);
    }
}
```

### Incidente

```php
// app/Models/Incidente.php
class Incidente extends Model
{
    use HasUuids;

    protected $fillable = [
        'titulo', 'descripcion', 'tipo', 'estado', 'latitud', 'longitud',
        'zona', 'carretera', 'reportado_por', 'inicio_estimado', 'fin_estimado', 'fin_real',
    ];

    protected $casts = [
        'tipo'            => TipoIncidente::class,
        'estado'          => EstadoIncidente::class,
        'latitud'         => 'decimal:7',
        'longitud'        => 'decimal:7',
        'inicio_estimado' => 'datetime',
        'fin_estimado'    => 'datetime',
        'fin_real'        => 'datetime',
    ];

    // Relaciones
    public function reportadoPor(): BelongsTo
    {
        return $this->belongsTo(User::class, 'reportado_por');
    }

    public function confirmaciones(): HasMany
    {
        return $this->hasMany(Confirmacion::class, 'incidente_id')
                    ->where('tipo', 'confirma');
    }

    public function denegaciones(): HasMany
    {
        return $this->hasMany(Confirmacion::class, 'incidente_id')
                    ->where('tipo', 'niega');
    }

    public function todasLasConfirmaciones(): HasMany
    {
        return $this->hasMany(Confirmacion::class, 'incidente_id');
    }

    public function suscripciones(): HasMany
    {
        return $this->hasMany(Suscripcion::class, 'incidente_id');
    }

    public function notificaciones(): HasMany
    {
        return $this->hasMany(Notificacion::class, 'incidente_id');
    }

    // ---- Scopes frecuentes ----

    /** Incidentes visibles en el mapa (activos y programados) */
    public function scopeVisiblesEnMapa(Builder $query): Builder
    {
        return $query->whereIn('estado', [
            EstadoIncidente::Activo,
            EstadoIncidente::Programado,
        ]);
    }

    /** Solo incidentes activos en este momento */
    public function scopeActivos(Builder $query): Builder
    {
        return $query->where('estado', EstadoIncidente::Activo);
    }

    /** Filtrar por zona de Oaxaca */
    public function scopePorZona(Builder $query, string $zona): Builder
    {
        return $query->where('zona', $zona);
    }

    /** Búsqueda geográfica por radio (en km) usando Haversine */
    public function scopeEnRadio(Builder $query, float $lat, float $lng, int $radioKm = 50): Builder
    {
        return $query->selectRaw("*, (
            6371 * acos(
                cos(radians(?)) * cos(radians(latitud)) *
                cos(radians(longitud) - radians(?)) +
                sin(radians(?)) * sin(radians(latitud))
            )
        ) AS distancia_km", [$lat, $lng, $lat])
        ->having('distancia_km', '<=', $radioKm)
        ->orderBy('distancia_km');
    }

    /** Incidentes que necesitan verificación (pocos votos) */
    public function scopePendientesDeVerificacion(Builder $query): Builder
    {
        return $query->where('estado', EstadoIncidente::NoVerificado)
                     ->whereColumn('confirmaciones_count', '<', 3);
    }

    // ---- Métodos de dominio ----

    /** Verifica si el incidente debería auto-activarse por confirmaciones */
    public function deberiaActivarse(): bool
    {
        $votosNetos = $this->confirmaciones_count - $this->denegaciones_count;
        return $this->estado === EstadoIncidente::NoVerificado && $votosNetos >= 3;
    }

    /** Transiciona al siguiente estado si corresponde */
    public function verificarYTransicionar(): void
    {
        if ($this->deberiaActivarse()) {
            $this->update(['estado' => EstadoIncidente::Activo]);
            // Aquí se puede disparar evento: IncidenteActivado
        }
    }
}
```

### Confirmacion

```php
// app/Models/Confirmacion.php
class Confirmacion extends Model
{
    use HasUuids;

    public $timestamps = false; // solo created_at
    const UPDATED_AT = null;

    protected $fillable = ['incidente_id', 'usuario_id', 'tipo', 'comentario'];

    protected $casts = [
        'tipo'       => TipoConfirmacion::class,
        'created_at' => 'datetime',
    ];

    public function incidente(): BelongsTo
    {
        return $this->belongsTo(Incidente::class);
    }

    public function usuario(): BelongsTo
    {
        return $this->belongsTo(User::class, 'usuario_id');
    }
}
```

---

## Evitar N+1 — queries del mapa

El endpoint de mapa devuelve muchos incidentes. Las relaciones deben cargarse en la query principal.

### ❌ Problema N+1

```php
// Esto genera 1 query por incidente para obtener reportadoPor
$incidentes = Incidente::activos()->get();
foreach ($incidentes as $incidente) {
    echo $incidente->reportadoPor->nombre; // ← N+1!
}
```

### ✅ Solución — eager loading

```php
// Una sola query con JOIN
$incidentes = Incidente::activos()
    ->with(['reportadoPor:id,nombre,rol'])  // solo columnas necesarias
    ->withCount(['confirmaciones', 'denegaciones'])
    ->paginate(50);
```

### ✅ Carga condicional en Resources

Usar `$this->whenLoaded()` para no forzar la carga si no se pidió:

```php
// IncidenteResource.php
'reportado_por' => new UsuarioPublicoResource($this->whenLoaded('reportadoPor')),
'confirmaciones' => ConfirmacionResource::collection($this->whenLoaded('confirmaciones')),
```

### ✅ Lazy eager loading cuando sea necesario

```php
// Si ya tienes la colección y necesitas cargar relaciones después
$incidentes->load(['reportadoPor:id,nombre']);
```

---

## Upsert de Confirmacion (regla de negocio)

Un usuario solo puede tener una confirmación por incidente. Usar `updateOrCreate`:

```php
// app/Http/Controllers/Api/ConfirmacionController.php
public function store(StoreConfirmacionRequest $request, Incidente $incidente): ConfirmacionResource
{
    $confirmacion = Confirmacion::updateOrCreate(
        [
            'incidente_id' => $incidente->id,
            'usuario_id'   => Auth::id(),
        ],
        [
            'tipo'       => $request->tipo,
            'comentario' => $request->comentario,
        ]
    );

    // Recalcular contadores y verificar transición de estado
    $incidente->loadCount(['confirmaciones', 'denegaciones']);
    $incidente->update([
        'confirmaciones_count' => $incidente->confirmaciones_count,
        'denegaciones_count'   => $incidente->denegaciones_count,
    ]);
    $incidente->verificarYTransicionar();

    return new ConfirmacionResource($confirmacion);
}
```
