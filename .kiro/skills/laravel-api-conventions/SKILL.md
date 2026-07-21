---
description: >
  Usar cuando se crea un nuevo endpoint de API en Laravel, se agrega un recurso REST,
  se necesita estructurar Form Request + Controller + API Resource + ruta, o se modifica
  el flujo de validación y respuesta de cualquier endpoint del backend.
---

# Skill: Laravel API Conventions

Guía completa para construir un endpoint REST en Laravel 13 siguiendo las convenciones de ViaLibre, usando como ejemplo el módulo "reportar incidente".

---

## Flujo obligatorio

```
Form Request → Controller → Model → API Resource → Ruta registrada
```

Nunca saltarse pasos. Nunca validar en el Controller. Nunca devolver modelos crudos.

---

## Paso 1 — Form Request

Crear con artisan:
```bash
php artisan make:request StoreIncidenteRequest
```

```php
// app/Http/Requests/StoreIncidenteRequest.php
namespace App\Http\Requests;

use App\Enums\TipoIncidente;
use Illuminate\Foundation\Http\FormRequest;
use Illuminate\Validation\Rule;

class StoreIncidenteRequest extends FormRequest
{
    public function authorize(): bool
    {
        return $this->user() !== null;
    }

    public function rules(): array
    {
        return [
            'titulo'          => ['required', 'string', 'max:150'],
            'tipo'            => ['required', Rule::enum(TipoIncidente::class)],
            'latitud'         => ['required', 'numeric', 'between:-90,90'],
            'longitud'        => ['required', 'numeric', 'between:-180,180'],
            'zona'            => ['required', 'string', 'max:100'],
            'carretera'       => ['nullable', 'string', 'max:150'],
            'descripcion'     => ['nullable', 'string', 'max:1000'],
            'inicio_estimado' => ['nullable', 'date', 'after:now'],
            'fin_estimado'    => ['nullable', 'date', 'after:inicio_estimado'],
        ];
    }
}
```

## Paso 2 — Controller

Crear con artisan:
```bash
php artisan make:controller Api/IncidenteController --api
```

```php
// app/Http/Controllers/Api/IncidenteController.php
namespace App\Http\Controllers\Api;

use App\Http\Controllers\Controller;
use App\Http\Requests\StoreIncidenteRequest;
use App\Http\Resources\IncidenteResource;
use App\Models\Incidente;
use App\Enums\EstadoIncidente;
use Illuminate\Http\Request;
use Illuminate\Http\Resources\Json\AnonymousResourceCollection;
use Illuminate\Support\Facades\Auth;

class IncidenteController extends Controller
{
    public function index(Request $request): AnonymousResourceCollection
    {
        $incidentes = Incidente::query()
            ->when($request->zona, fn($q, $zona) => $q->porZona($zona))
            ->when($request->estado, fn($q, $estado) => $q->where('estado', $estado))
            ->with(['reportadoPor'])
            ->withCount(['confirmaciones', 'denegaciones'])
            ->latest()
            ->paginate($request->integer('per_page', 20));

        return IncidenteResource::collection($incidentes);
    }

    public function store(StoreIncidenteRequest $request): IncidenteResource
    {
        $incidente = Incidente::create([
            ...$request->validated(),
            'reportado_por' => Auth::id(),
            'estado'        => EstadoIncidente::NoVerificado,
        ]);

        return (new IncidenteResource($incidente))
            ->response()
            ->setStatusCode(201);
    }

    public function show(Incidente $incidente): IncidenteResource
    {
        return new IncidenteResource(
            $incidente->load(['reportadoPor', 'confirmaciones.usuario'])
        );
    }

    public function update(UpdateIncidenteRequest $request, Incidente $incidente): IncidenteResource
    {
        $this->authorize('update', $incidente);
        $incidente->update($request->validated());
        return new IncidenteResource($incidente);
    }

    public function destroy(Incidente $incidente): \Illuminate\Http\Response
    {
        $this->authorize('delete', $incidente);
        $incidente->delete();
        return response()->noContent();
    }
}
```

## Paso 3 — API Resource

Crear con artisan:
```bash
php artisan make:resource IncidenteResource
```

```php
// app/Http/Resources/IncidenteResource.php
namespace App\Http\Resources;

use Illuminate\Http\Request;
use Illuminate\Http\Resources\Json\JsonResource;

class IncidenteResource extends JsonResource
{
    public function toArray(Request $request): array
    {
        return [
            'id'                   => $this->id,
            'titulo'               => $this->titulo,
            'tipo'                 => $this->tipo,
            'estado'               => $this->estado,
            'latitud'              => (float) $this->latitud,
            'longitud'             => (float) $this->longitud,
            'zona'                 => $this->zona,
            'carretera'            => $this->carretera,
            'descripcion'          => $this->descripcion,
            'confirmaciones_count' => $this->confirmaciones_count,
            'denegaciones_count'   => $this->denegaciones_count,
            'inicio_estimado'      => $this->inicio_estimado?->toIso8601String(),
            'fin_estimado'         => $this->fin_estimado?->toIso8601String(),
            'fin_real'             => $this->fin_real?->toIso8601String(),
            'reportado_por'        => new UsuarioPublicoResource($this->whenLoaded('reportadoPor')),
            'confirmaciones'       => ConfirmacionResource::collection($this->whenLoaded('confirmaciones')),
            'created_at'           => $this->created_at->toIso8601String(),
            'updated_at'           => $this->updated_at->toIso8601String(),
        ];
    }
}
```

## Paso 4 — Registrar la ruta

```php
// routes/api.php
use App\Http\Controllers\Api\IncidenteController;

// Rutas públicas
Route::get('incidentes', [IncidenteController::class, 'index']);
Route::get('incidentes/{incidente}', [IncidenteController::class, 'show']);

// Rutas protegidas
Route::middleware('auth:sanctum')->group(function () {
    Route::post('incidentes', [IncidenteController::class, 'store']);
    Route::put('incidentes/{incidente}', [IncidenteController::class, 'update']);
    Route::delete('incidentes/{incidente}', [IncidenteController::class, 'destroy']);

    // Acción custom — confirmar un incidente
    Route::post('incidentes/{incidente}/confirmar', [ConfirmacionController::class, 'store']);
});
```

Verificar rutas registradas:
```bash
php artisan route:list --path=api/incidentes
```

---

## Checklist para un endpoint nuevo

- [ ] Form Request creado con `authorize()` y `rules()`
- [ ] Controller en `app/Http/Controllers/Api/`
- [ ] Respuestas siempre via API Resource (nunca `->toArray()` directo)
- [ ] Ruta registrada en `routes/api.php`
- [ ] Status codes correctos (201 para create, 204 para delete)
- [ ] Relaciones cargadas con `->load()` o `->with()` (nunca lazy en bucle)
- [ ] Test Feature creado (ver skill `laravel-testing`)
