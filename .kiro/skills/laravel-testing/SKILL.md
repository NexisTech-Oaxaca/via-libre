---
description: >
  Usar cuando se escriben tests en el backend con Pest PHP, se necesita testear validación
  de Form Requests, comportamiento de Controllers o reglas de negocio en Models de ViaLibre,
  o se crean factories para los modelos del dominio.
---

# Skill: Laravel Testing con Pest

Convención de tests para ViaLibre usando Pest PHP v3.

---

## Estructura de tests

```
tests/
├── Feature/           # Tests de integración HTTP (llaman rutas reales)
│   ├── Api/
│   │   ├── IncidenteTest.php
│   │   └── ConfirmacionTest.php
│   └── Auth/
│       └── AuthTest.php
└── Unit/              # Tests unitarios (modelos, enums, lógica pura)
    ├── IncidenteTest.php
    └── EstadoIncidenteTest.php
```

---

## Qué testear en cada capa

| Capa | Qué testear | Tipo de test |
|---|---|---|
| Form Request | Que reglas de validación fallan/pasan correctamente | Feature (HTTP) |
| Controller | Respuesta correcta, status code, estructura JSON | Feature (HTTP) |
| Model | Scopes, métodos de dominio, relaciones | Unit |
| Evento/Job | Que se disparan y ejecutan correctamente | Feature |

---

## Factories para modelos del dominio

```php
// database/factories/IncidenteFactory.php
class IncidenteFactory extends Factory
{
    protected $model = Incidente::class;

    public function definition(): array
    {
        return [
            'titulo'               => fake()->sentence(5),
            'tipo'                 => fake()->randomElement(TipoIncidente::values()),
            'estado'               => EstadoIncidente::NoVerificado,
            'latitud'              => fake()->latitude(15.5, 18.5),  // rango Oaxaca
            'longitud'             => fake()->longitude(-98.5, -93.5),
            'zona'                 => fake()->randomElement([
                'Valles Centrales', 'Sierra Norte', 'Cañada',
                'Mixteca', 'Papaloapan', 'Sierra Sur', 'Costa', 'Istmo'
            ]),
            'carretera'            => fake()->optional()->randomElement([
                'Federal 175', 'Federal 190', 'Federal 200', 'Federal 131'
            ]),
            'descripcion'          => fake()->optional()->paragraph(),
            'reportado_por'        => User::factory(),
            'confirmaciones_count' => 0,
            'denegaciones_count'   => 0,
        ];
    }

    // Estados factory para crear incidentes en estados específicos
    public function activo(): static
    {
        return $this->state(['estado' => EstadoIncidente::Activo]);
    }

    public function programado(): static
    {
        return $this->state([
            'estado'          => EstadoIncidente::Programado,
            'inicio_estimado' => now()->addHours(2),
        ]);
    }

    public function conConfirmaciones(int $cantidad = 3): static
    {
        return $this->afterCreating(function (Incidente $incidente) use ($cantidad) {
            Confirmacion::factory()->count($cantidad)->create([
                'incidente_id' => $incidente->id,
                'tipo'         => 'confirma',
            ]);
            $incidente->update(['confirmaciones_count' => $cantidad]);
        });
    }
}
```

```php
// database/factories/UserFactory.php
class UserFactory extends Factory
{
    public function definition(): array
    {
        return [
            'nombre'        => fake('es_MX')->name(),
            'email'         => fake()->unique()->safeEmail(),
            'telefono'      => fake()->optional()->numerify('+521##########'),
            'rol'           => RolUsuario::Basico,
            'verificado_en' => null,
            'password'      => bcrypt('password'),
        ];
    }

    public function verificado(): static
    {
        return $this->state([
            'rol'           => RolUsuario::Verificado,
            'verificado_en' => now()->subDays(30),
        ]);
    }

    public function moderador(): static
    {
        return $this->state(['rol' => RolUsuario::Moderador]);
    }
}
```

---

## Tests de Feature (HTTP / Controller)

```php
// tests/Feature/Api/IncidenteTest.php
use App\Models\{User, Incidente};
use App\Enums\EstadoIncidente;

describe('GET /api/incidentes', function () {
    it('devuelve lista de incidentes visibles en el mapa', function () {
        Incidente::factory()->activo()->count(3)->create();
        Incidente::factory()->state(['estado' => EstadoIncidente::Rechazado])->create();

        $response = $this->getJson('/api/incidentes');

        $response->assertOk()
                 ->assertJsonCount(3, 'data')
                 ->assertJsonStructure([
                     'data' => [['id', 'titulo', 'estado', 'latitud', 'longitud']],
                     'meta' => ['current_page', 'total'],
                 ]);
    });

    it('filtra por zona correctamente', function () {
        Incidente::factory()->activo()->create(['zona' => 'Valles Centrales']);
        Incidente::factory()->activo()->create(['zona' => 'Costa']);

        $this->getJson('/api/incidentes?zona=Valles+Centrales')
             ->assertOk()
             ->assertJsonCount(1, 'data');
    });
});

describe('POST /api/incidentes', function () {
    it('requiere autenticación', function () {
        $this->postJson('/api/incidentes', [])->assertUnauthorized();
    });

    it('crea un incidente correctamente', function () {
        $usuario = User::factory()->create();

        $datos = [
            'titulo'   => 'Bloqueo en carretera 175',
            'tipo'     => 'bloqueo_sindical',
            'latitud'  => 17.0732,
            'longitud' => -96.7266,
            'zona'     => 'Valles Centrales',
        ];

        $this->actingAs($usuario)
             ->postJson('/api/incidentes', $datos)
             ->assertCreated()
             ->assertJsonPath('data.estado', 'no_verificado')
             ->assertJsonPath('data.titulo', 'Bloqueo en carretera 175');

        $this->assertDatabaseHas('incidentes', [
            'titulo'        => 'Bloqueo en carretera 175',
            'reportado_por' => $usuario->id,
        ]);
    });

    it('falla con datos inválidos', function () {
        $usuario = User::factory()->create();

        $this->actingAs($usuario)
             ->postJson('/api/incidentes', ['titulo' => ''])
             ->assertUnprocessable()
             ->assertJsonValidationErrors(['titulo', 'tipo', 'latitud', 'longitud', 'zona']);
    });
});
```

---

## Tests de validación de Form Request

```php
describe('StoreIncidenteRequest validación', function () {
    it('falla si latitud está fuera de rango', function () {
        $usuario = User::factory()->create();

        $this->actingAs($usuario)
             ->postJson('/api/incidentes', [
                 'titulo'   => 'Test',
                 'tipo'     => 'bloqueo_sindical',
                 'latitud'  => 200,  // inválido
                 'longitud' => -96.7,
                 'zona'     => 'Costa',
             ])
             ->assertUnprocessable()
             ->assertJsonValidationErrors(['latitud']);
    });

    it('acepta tipo válido del enum', function () {
        $usuario = User::factory()->create();

        $this->actingAs($usuario)
             ->postJson('/api/incidentes', [
                 'titulo'   => 'Test',
                 'tipo'     => 'tipo_inexistente',  // inválido
                 'latitud'  => 17.0,
                 'longitud' => -96.7,
                 'zona'     => 'Costa',
             ])
             ->assertUnprocessable()
             ->assertJsonValidationErrors(['tipo']);
    });
});
```

---

## Tests de Unit (Model / Lógica de dominio)

```php
// tests/Unit/IncidenteTest.php
use App\Models\Incidente;
use App\Enums\EstadoIncidente;

describe('Incidente::deberiaActivarse()', function () {
    it('retorna true cuando tiene 3 o más votos netos positivos', function () {
        $incidente = Incidente::factory()->make([
            'estado'               => EstadoIncidente::NoVerificado,
            'confirmaciones_count' => 4,
            'denegaciones_count'   => 1,
        ]);

        expect($incidente->deberiaActivarse())->toBeTrue();
    });

    it('retorna false con menos de 3 votos netos', function () {
        $incidente = Incidente::factory()->make([
            'estado'               => EstadoIncidente::NoVerificado,
            'confirmaciones_count' => 2,
            'denegaciones_count'   => 0,
        ]);

        expect($incidente->deberiaActivarse())->toBeFalse();
    });

    it('retorna false si ya está activo', function () {
        $incidente = Incidente::factory()->make([
            'estado'               => EstadoIncidente::Activo,
            'confirmaciones_count' => 10,
            'denegaciones_count'   => 0,
        ]);

        expect($incidente->deberiaActivarse())->toBeFalse();
    });
});

describe('Scope porZona', function () {
    it('filtra incidentes por zona', function () {
        Incidente::factory()->create(['zona' => 'Costa']);
        Incidente::factory()->create(['zona' => 'Mixteca']);

        $resultado = Incidente::porZona('Costa')->get();

        expect($resultado)->toHaveCount(1)
            ->first()->zona->toBe('Costa');
    });
});
```

---

## Configuración de Pest

```php
// tests/Pest.php
uses(Tests\TestCase::class)->in('Feature');
uses(Tests\TestCase::class)->in('Unit');

// Helpers reutilizables
function usuarioAutenticado(string $rol = 'basico'): User
{
    $usuario = User::factory()->create(['rol' => $rol]);
    test()->actingAs($usuario);
    return $usuario;
}
```

## Comandos

```bash
php artisan test                          # todos los tests
php artisan test --filter=IncidenteTest   # filtrar por nombre
php artisan test tests/Feature/Api/       # directorio específico
php artisan test --coverage               # con cobertura
```
