---
inclusion: fileMatch
fileMatchPattern: "backend/**"
---

# Convenciones de Backend — Laravel 13

## Estructura de un endpoint end-to-end

Siempre seguir este flujo en este orden:

```
HTTP Request
  → Form Request (validación)
  → Controller (orquestación, sin lógica de negocio)
  → Model / Service (lógica de negocio)
  → API Resource (transformación de respuesta)
  → JSON Response
```

---

## Form Requests

Toda validación va en un Form Request. **Nunca validar en el controller directamente.**

```php
// app/Http/Requests/StoreIncidenteRequest.php
class StoreIncidenteRequest extends FormRequest
{
    public function authorize(): bool
    {
        return Auth::check(); // o política específica
    }

    public function rules(): array
    {
        return [
            'titulo'          => ['required', 'string', 'max:150'],
            'tipo'            => ['required', Rule::in(TipoIncidente::values())],
            'latitud'         => ['required', 'numeric', 'between:-90,90'],
            'longitud'        => ['required', 'numeric', 'between:-180,180'],
            'zona'            => ['required', 'string', 'max:100'],
            'descripcion'     => ['nullable', 'string', 'max:1000'],
            'inicio_estimado' => ['nullable', 'date', 'after:now'],
        ];
    }

    public function messages(): array
    {
        return [
            'tipo.in' => 'El tipo de incidente no es válido.',
        ];
    }
}
```

---

## Controllers

- Solo orquestación: llama Form Request → Model/Service → Resource
- Sin lógica de negocio compleja (eso va en el Model o un Service)
- Respuestas siempre via API Resource
- Usar `Route::apiResource()` para los 5 métodos RESTful estándar

```php
// app/Http/Controllers/Api/IncidenteController.php
class IncidenteController extends Controller
{
    public function index(Request $request): AnonymousResourceCollection
    {
        $incidentes = Incidente::activos()
            ->porZona($request->zona)
            ->with(['reportadoPor'])
            ->paginate(20);

        return IncidenteResource::collection($incidentes);
    }

    public function store(StoreIncidenteRequest $request): IncidenteResource
    {
        $incidente = Incidente::create([
            ...$request->validated(),
            'reportado_por' => Auth::id(),
            'estado'        => EstadoIncidente::NoVerificado,
        ]);

        return new IncidenteResource($incidente);
        // HTTP 201 se controla desde el Resource o con ->response()->setStatusCode(201)
    }

    public function show(Incidente $incidente): IncidenteResource
    {
        return new IncidenteResource($incidente->load(['confirmaciones', 'reportadoPor']));
    }
}
```

---

## API Resources

Toda respuesta JSON pasa por un API Resource. **Nunca devolver `$model->toArray()` directo.**

```php
// app/Http/Resources/IncidenteResource.php
class IncidenteResource extends JsonResource
{
    public function toArray(Request $request): array
    {
        return [
            'id'                   => $this->id,
            'titulo'               => $this->titulo,
            'tipo'                 => $this->tipo,
            'estado'               => $this->estado,
            'latitud'              => $this->latitud,
            'longitud'             => $this->longitud,
            'zona'                 => $this->zona,
            'carretera'            => $this->carretera,
            'confirmaciones_count' => $this->confirmaciones_count,
            'denegaciones_count'   => $this->denegaciones_count,
            'inicio_estimado'      => $this->inicio_estimado?->toIso8601String(),
            'fin_estimado'         => $this->fin_estimado?->toIso8601String(),
            'reportado_por'        => new UsuarioPublicoResource($this->whenLoaded('reportadoPor')),
            'created_at'           => $this->created_at->toIso8601String(),
        ];
    }
}
```

---

## Rutas API

Todo en `routes/api.php`. Prefijo `/api/` es automático.

```php
// routes/api.php
Route::middleware('auth:sanctum')->group(function () {
    Route::apiResource('incidentes', IncidenteController::class);
    Route::post('incidentes/{incidente}/confirmar', [ConfirmacionController::class, 'store']);
    Route::apiResource('incidentes.suscripciones', SuscripcionController::class)
        ->only(['store', 'destroy']);
});

// Rutas públicas (sin auth)
Route::get('incidentes', [IncidenteController::class, 'index']);
Route::get('incidentes/{incidente}', [IncidenteController::class, 'show']);
```

---

## Manejo de errores estandarizado

Usar `bootstrap/app.php` para registrar los handlers globales:

```php
->withExceptions(function (Exceptions $exceptions) {
    // Modelo no encontrado → 404 consistente
    $exceptions->render(function (ModelNotFoundException $e, Request $request) {
        if ($request->is('api/*')) {
            return response()->json([
                'message' => 'El recurso solicitado no existe.',
            ], 404);
        }
    });

    // Validación → 422 con estructura estándar
    $exceptions->render(function (ValidationException $e, Request $request) {
        if ($request->is('api/*')) {
            return response()->json([
                'message' => 'Los datos enviados no son válidos.',
                'errors'  => $e->errors(),
            ], 422);
        }
    });

    // No autorizado → 403
    $exceptions->render(function (AuthorizationException $e, Request $request) {
        if ($request->is('api/*')) {
            return response()->json([
                'message' => 'No tienes permiso para realizar esta acción.',
            ], 403);
        }
    });
})
```

---

## Convenciones de Migraciones

```php
// Cada migración toca UNA tabla
// Nombre: acción_tabla (create, add, remove, rename)
// Usar UUIDs como PK
Schema::create('incidentes', function (Blueprint $table) {
    $table->uuid('id')->primary();
    $table->string('titulo', 150);
    $table->enum('tipo', TipoIncidente::values());
    $table->enum('estado', EstadoIncidente::values())->default('no_verificado');
    $table->decimal('latitud', 10, 7);
    $table->decimal('longitud', 10, 7);
    $table->string('zona', 100);
    $table->foreignUuid('reportado_por')->constrained('users')->cascadeOnDelete();
    $table->unsignedInteger('confirmaciones_count')->default(0);
    $table->unsignedInteger('denegaciones_count')->default(0);
    $table->timestamp('inicio_estimado')->nullable();
    $table->timestamp('fin_estimado')->nullable();
    $table->timestamp('fin_real')->nullable();
    $table->timestamps();

    $table->index(['estado', 'zona']); // índices para queries frecuentes del mapa
    $table->index(['latitud', 'longitud']);
});
```

---

## Seeders

- `DatabaseSeeder` solo llama a otros seeders, no crea datos directamente
- Un seeder por entidad principal
- Usar factories para datos de prueba

```php
// database/seeders/DatabaseSeeder.php
public function run(): void
{
    $this->call([
        UserSeeder::class,
        IncidenteSeeder::class,
    ]);
}
```

---

## Enums PHP 8.1+

Usar enums nativos de PHP para los campos `tipo` y `estado`:

```php
// app/Enums/EstadoIncidente.php
enum EstadoIncidente: string
{
    case NoVerificado = 'no_verificado';
    case Programado   = 'programado';
    case Activo       = 'activo';
    case Finalizado   = 'finalizado';
    case Cancelado    = 'cancelado';
    case Rechazado    = 'rechazado';

    public static function values(): array
    {
        return array_column(self::cases(), 'value');
    }

    public function esTerminal(): bool
    {
        return in_array($this, [self::Finalizado, self::Cancelado, self::Rechazado]);
    }
}
```

Castear en el modelo:

```php
protected $casts = [
    'estado' => EstadoIncidente::class,
    'tipo'   => TipoIncidente::class,
];
```
