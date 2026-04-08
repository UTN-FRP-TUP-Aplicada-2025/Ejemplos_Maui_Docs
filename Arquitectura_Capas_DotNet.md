# Arquitectura en Capas — Guía para Desarrolladores .NET

## Índice

1. [¿Qué es la arquitectura en capas?](#1-qué-es-la-arquitectura-en-capas)
2. [Las cuatro capas y sus responsabilidades](#2-las-cuatro-capas-y-sus-responsabilidades)
3. [Nomenclatura de Namespaces y Proyectos](#3-nomenclatura-de-namespaces-y-proyectos)
4. [Capa de Presentación](#4-capa-de-presentación)
5. [Capa de Aplicación](#5-capa-de-aplicación)
6. [Capa de Dominio](#6-capa-de-dominio)
7. [Capa de Infraestructura](#7-capa-de-infraestructura)
8. [Orquestadores: qué son, cuándo usarlos y ejemplos](#8-orquestadores-qué-son-cuándo-usarlos-y-ejemplos)
9. [Regla de dependencia](#9-regla-de-dependencia)
10. [Ejemplo integrado: App de Infracciones de Tránsito](#10-ejemplo-integrado-app-de-infracciones-de-tránsito)

---

## 1. ¿Qué es la arquitectura en capas?

Es una forma de organizar el código separando responsabilidades en **capas lógicas**, donde cada capa tiene un rol claro y conoce solo a las capas que están "debajo" de ella.

```
┌─────────────────────────────────────────────────────┐
│                  PRESENTACIÓN                        │
│        (lo que ve el usuario o recibe el cliente)    │
├─────────────────────────────────────────────────────┤
│                   APLICACIÓN                         │
│         (orquesta casos de uso / lógica de flujo)    │
├─────────────────────────────────────────────────────┤
│                    DOMINIO                           │
│         (reglas de negocio puras, entidades)          │
├─────────────────────────────────────────────────────┤
│                 INFRAESTRUCTURA                      │
│       (base de datos, APIs externas, email, GPS)     │
└─────────────────────────────────────────────────────┘
```

**¿Por qué separar?**

- **Mantenibilidad**: Cambiar la base de datos no afecta las reglas de negocio.
- **Testabilidad**: Podés probar la lógica de negocio sin necesitar una base de datos real.
- **Claridad**: Cada desarrollador sabe dónde poner cada cosa.
- **Reusabilidad**: La misma capa de aplicación puede servir a una app MAUI, una API REST, y un job en background.

---

## 2. Las cuatro capas y sus responsabilidades

| Capa | Responsabilidad | Conoce a | No conoce a |
|------|----------------|----------|-------------|
| **Presentación** | Mostrar datos, capturar input del usuario, delegar a aplicación | Aplicación | Infraestructura directamente |
| **Aplicación** | Orquestar casos de uso, coordinar servicios, validar input | Dominio, interfaces de Infraestructura | Implementaciones concretas de Infraestructura |
| **Dominio** | Reglas de negocio, entidades, validaciones de negocio | Nada (es el centro) | Todo lo externo |
| **Infraestructura** | Implementar acceso a datos, APIs externas, email, archivos | Dominio (implementa sus interfaces) | Presentación, Aplicación |

### Diagrama de dependencias

```
Presentación ──────► Aplicación ──────► Dominio ◄────── Infraestructura
                          │                                    ▲
                          │         (usa interfaces             │
                          └─────────  definidas en Dominio) ────┘
```

La flecha `◄──` de Infraestructura hacia Dominio significa que Infraestructura **implementa** interfaces definidas en Dominio (inversión de dependencias).

---

## 3. Nomenclatura de Namespaces y Proyectos

En .NET, cada capa suele ser un **proyecto separado** (`.csproj`) dentro de una **solución** (`.sln`). El namespace sigue el patrón:

```
NombreEmpresa.NombreApp.NombreCapa
```

### 3.1 Estructura de proyectos en la solución

```
📁 InfraccionesApp.sln
│
├── 📁 src/
│   ├── 📁 InfraccionesApp.API/                    ← Presentación (API REST)
│   │   └── InfraccionesApp.API.csproj
│   │
│   ├── 📁 InfraccionesApp.Maui/                   ← Presentación (app móvil)
│   │   └── InfraccionesApp.Maui.csproj
│   │
│   ├── 📁 InfraccionesApp.Application/            ← Capa de Aplicación
│   │   └── InfraccionesApp.Application.csproj
│   │
│   ├── 📁 InfraccionesApp.Domain/                 ← Capa de Dominio
│   │   └── InfraccionesApp.Domain.csproj
│   │
│   └── 📁 InfraccionesApp.Infrastructure/         ← Capa de Infraestructura
│       └── InfraccionesApp.Infrastructure.csproj
│
└── 📁 tests/
    ├── 📁 InfraccionesApp.Application.Tests/
    └── 📁 InfraccionesApp.Domain.Tests/
```

### 3.2 Nomenclaturas alternativas (todas significan lo mismo)

La comunidad .NET usa diferentes nombres para las mismas capas. Esta tabla traduce entre ellas:

| Capa | Nombre "clásico" | Nombre "Clean Architecture" | Nombre "DDD" | Nombre "Microsoft" |
|------|-----------------|----------------------------|--------------|-------------------|
| **Presentación** | Presentation | UI / WebUI | Interface | API / Web |
| **Aplicación** | Application | Application | Application | Services |
| **Dominio** | Domain | Core / Domain | Domain | Domain / BusinessLogic |
| **Infraestructura** | Infrastructure | Infrastructure | Infrastructure | Data / Persistence |

### 3.3 Namespaces: convenciones de nombrado

| Capa | Namespace raíz | Subcarpetas típicas |
|------|----------------|---------------------|
| Presentación (API) | `MiApp.API` | `Controllers/`, `Middleware/`, `Filters/` |
| Presentación (MAUI) | `MiApp.Maui` | `Views/`, `Pages/`, `ViewModels/` |
| Aplicación | `MiApp.Application` | `Services/`, `DTOs/`, `Interfaces/`, `Validators/` |
| Dominio | `MiApp.Domain` | `Entities/`, `ValueObjects/`, `Interfaces/`, `Enums/`, `Exceptions/` |
| Infraestructura | `MiApp.Infrastructure` | `Repositories/`, `Data/`, `ExternalServices/`, `Migrations/` |

**Ejemplo de un namespace completo:**

```csharp
namespace InfraccionesApp.Domain.Entities;       // Entidad Infraccion
namespace InfraccionesApp.Application.Services;   // InfraccionService
namespace InfraccionesApp.Infrastructure.Data;     // AppDbContext
namespace InfraccionesApp.API.Controllers;         // InfraccionesController
```

### 3.4 Convenciones de nombrado de clases

| Tipo de clase | Sufijo | Capa | Ejemplo |
|---|---|---|---|
| Entidad de dominio | (sin sufijo) | Dominio | `Infraccion`, `Inspector`, `Vehiculo` |
| Value Object | (sin sufijo) | Dominio | `Coordenada`, `Dominio`, `Patente` |
| Interfaz de repositorio | `I___Repository` | Dominio | `IInfraccionRepository` |
| Interfaz de servicio externo | `I___Service` | Dominio | `IEmailService`, `IGpsService` |
| Servicio de aplicación | `___Service` o `___UseCase` | Aplicación | `InfraccionService`, `RegistrarInfraccionUseCase` |
| DTO | `___Dto` o `___Request/Response` | Aplicación | `InfraccionDto`, `CrearInfraccionRequest` |
| Validador | `___Validator` | Aplicación | `CrearInfraccionValidator` |
| Repositorio concreto | `___Repository` | Infraestructura | `InfraccionRepository`, `SqlInfraccionRepository` |
| DbContext | `___DbContext` o `AppDbContext` | Infraestructura | `AppDbContext`, `InfraccionesDbContext` |
| Controller | `___Controller` | Presentación | `InfraccionesController` |
| Página MAUI | `___Page` | Presentación | `InfraccionPage`, `GpsPage` |
| ViewModel | `___ViewModel` | Presentación | `InfraccionViewModel` |
| BackgroundService/Job | `___Job` o `___Worker` | Host/Infraestructura | `SincronizacionJob` |

---

## 4. Capa de Presentación

### Rol

Es el **punto de entrada** del usuario o del cliente. **No tiene lógica de negocio**. Solo:

1. Recibe input (click, HTTP request, formulario)
2. Delega a la capa de aplicación
3. Muestra el resultado

### Variantes

| Tipo | Proyecto | Ejemplo |
|---|---|---|
| API REST | `MiApp.API` | ASP.NET Controller que recibe JSON |
| App móvil | `MiApp.Maui` | Página MAUI que muestra un formulario |
| App web | `MiApp.Web` | Razor Pages / Blazor |
| Worker | `MiApp.Worker` | BackgroundService que ejecuta jobs |

### Clases típicas

```csharp
// ── Presentación API (ASP.NET) ──

namespace InfraccionesApp.API.Controllers;

[ApiController]
[Route("api/[controller]")]
public class InfraccionesController : ControllerBase
{
    private readonly InfraccionService _service;

    public InfraccionesController(InfraccionService service)
    {
        _service = service;
    }

    [HttpPost]
    public async Task<IActionResult> Crear(
        CrearInfraccionRequest request,
        CancellationToken ct)                  // ← el framework lo provee
    {
        var resultado = await _service.RegistrarAsync(request, ct);
        return Ok(resultado);
    }
}
```

```csharp
// ── Presentación MAUI ──

namespace InfraccionesApp.Maui.Views;

public partial class InfraccionPage : ContentPage
{
    private readonly InfraccionService _service;
    private CancellationTokenSource? _cts;

    public InfraccionPage(InfraccionService service)
    {
        InitializeComponent();
        _service = service;
    }

    private async void OnRegistrar_Clicked(object sender, EventArgs e)
    {
        _cts?.Cancel();
        _cts?.Dispose();
        _cts = new CancellationTokenSource();

        try
        {
            var request = new CrearInfraccionRequest { /* ... */ };
            var resultado = await _service.RegistrarAsync(request, _cts.Token);
            // Mostrar resultado en UI
        }
        catch (OperationCanceledException) { /* cancelado */ }
    }
}
```

**Punto clave**: Tanto el Controller como la Page hacen lo mismo — reciben input, llaman a `_service.RegistrarAsync()`, devuelven resultado. **La misma capa de aplicación sirve a ambos.**

---

## 5. Capa de Aplicación

### Rol

**Orquesta los casos de uso**. Coordina la interacción entre dominio e infraestructura. No contiene reglas de negocio (esas van en Dominio), pero sí sabe el **flujo** de cada operación.

### ¿Qué hace?

1. Recibe un DTO o request desde Presentación
2. Valida el input (validación de formato, no de negocio)
3. Llama a servicios de Dominio o Infraestructura
4. Coordina los pasos del caso de uso
5. Devuelve un DTO o response

### ¿Qué NO hace?

- No accede directamente a la base de datos (usa interfaces de repositorios)
- No tiene reglas de negocio ("una infracción no puede tener monto negativo" → eso va en Dominio)
- No conoce HTTP, XAML, ni nada de UI
- No crea `HttpClient`, `DbContext`, etc. directamente

### Clases típicas

```
📁 InfraccionesApp.Application/
│
├── 📁 DTOs/
│   ├── CrearInfraccionRequest.cs
│   ├── InfraccionResponse.cs
│   └── UbicacionDto.cs
│
├── 📁 Interfaces/                  ← Interfaces que Infraestructura implementa
│   ├── IInfraccionRepository.cs
│   ├── IGpsService.cs
│   └── IEmailService.cs
│
├── 📁 Services/
│   └── InfraccionService.cs        ← Servicio de aplicación (orquesta)
│
└── 📁 Validators/
    └── CrearInfraccionValidator.cs
```

**Nota sobre interfaces**: Algunas arquitecturas definen las interfaces de repositorio en la capa de **Dominio** en lugar de Aplicación. Ambos enfoques son válidos. En Clean Architecture, suelen ir en Dominio. En arquitecturas más pragmáticas, van en Aplicación.

```csharp
// ── DTO de entrada ──
namespace InfraccionesApp.Application.DTOs;

public class CrearInfraccionRequest
{
    public string? Descripcion { get; set; }
    public double Latitud { get; set; }
    public double Longitud { get; set; }
    public double? Precision { get; set; }
    public int InspectorId { get; set; }
}
```

```csharp
// ── DTO de salida ──
namespace InfraccionesApp.Application.DTOs;

public class InfraccionResponse
{
    public int Id { get; set; }
    public string? Descripcion { get; set; }
    public double Latitud { get; set; }
    public double Longitud { get; set; }
    public DateTime FechaHora { get; set; }
    public string? EstadoTexto { get; set; }
}
```

```csharp
// ── Interfaz de repositorio ──
namespace InfraccionesApp.Application.Interfaces;

public interface IInfraccionRepository
{
    Task<Infraccion?> ObtenerPorIdAsync(int id, CancellationToken ct);
    Task<List<Infraccion>> ObtenerPendientesAsync(CancellationToken ct);
    Task GuardarAsync(Infraccion infraccion, CancellationToken ct);
}
```

```csharp
// ── Servicio de aplicación ──
namespace InfraccionesApp.Application.Services;

public class InfraccionService
{
    private readonly IInfraccionRepository _repo;
    private readonly IEmailService _email;

    public InfraccionService(
        IInfraccionRepository repo,
        IEmailService email)
    {
        _repo = repo;
        _email = email;
    }

    public async Task<InfraccionResponse> RegistrarAsync(
        CrearInfraccionRequest request, CancellationToken ct)
    {
        // 1. Crear entidad de dominio (las validaciones de negocio están en la entidad)
        var infraccion = new Infraccion(
            descripcion: request.Descripcion,
            latitud: request.Latitud,
            longitud: request.Longitud,
            precision: request.Precision,
            inspectorId: request.InspectorId);

        // 2. Persistir
        await _repo.GuardarAsync(infraccion, ct);

        // 3. Notificar
        await _email.EnviarNotificacionAsync(infraccion, ct);

        // 4. Mapear a response
        return new InfraccionResponse
        {
            Id = infraccion.Id,
            Descripcion = infraccion.Descripcion,
            Latitud = infraccion.Latitud,
            Longitud = infraccion.Longitud,
            FechaHora = infraccion.FechaHora,
            EstadoTexto = infraccion.Estado.ToString()
        };
    }
}
```

---

## 6. Capa de Dominio

### Rol

Contiene las **reglas de negocio puras**. No depende de ninguna otra capa. No sabe si los datos vienen de una base de datos, un archivo, o una API. No sabe si se está ejecutando en una app móvil o en un servidor.

### ¿Qué tiene?

| Elemento | Qué es | Ejemplo |
|---|---|---|
| **Entidad** | Objeto con identidad única (tiene ID) | `Infraccion`, `Inspector`, `Vehiculo` |
| **Value Object** | Objeto sin identidad, definido por sus valores | `Coordenada(lat, lng)`, `Patente("ABC123")` |
| **Enum** | Estados o tipos de negocio | `EstadoInfraccion`, `TipoInfraccion` |
| **Interfaz** | Contrato que Infraestructura debe cumplir | `IInfraccionRepository` |
| **Excepción de dominio** | Error de regla de negocio | `InfraccionInvalidaException` |

### Clases típicas

```
📁 InfraccionesApp.Domain/
│
├── 📁 Entities/
│   ├── Infraccion.cs
│   ├── Inspector.cs
│   └── Vehiculo.cs
│
├── 📁 ValueObjects/
│   └── Coordenada.cs
│
├── 📁 Enums/
│   ├── EstadoInfraccion.cs
│   └── TipoInfraccion.cs
│
├── 📁 Interfaces/               ← algunos lo ponen acá, otros en Application
│   └── IInfraccionRepository.cs
│
└── 📁 Exceptions/
    └── InfraccionInvalidaException.cs
```

```csharp
// ── Entidad de dominio ──
namespace InfraccionesApp.Domain.Entities;

public class Infraccion
{
    public int Id { get; private set; }
    public string Descripcion { get; private set; }
    public double Latitud { get; private set; }
    public double Longitud { get; private set; }
    public double? Precision { get; private set; }
    public DateTime FechaHora { get; private set; }
    public int InspectorId { get; private set; }
    public EstadoInfraccion Estado { get; private set; }

    public Infraccion(
        string? descripcion,
        double latitud,
        double longitud,
        double? precision,
        int inspectorId)
    {
        // ── Reglas de negocio (validaciones) ──
        if (string.IsNullOrWhiteSpace(descripcion))
            throw new InfraccionInvalidaException("La descripción es obligatoria.");

        if (latitud < -90 || latitud > 90)
            throw new InfraccionInvalidaException("Latitud fuera de rango.");

        if (longitud < -180 || longitud > 180)
            throw new InfraccionInvalidaException("Longitud fuera de rango.");

        if (inspectorId <= 0)
            throw new InfraccionInvalidaException("Inspector inválido.");

        Descripcion = descripcion;
        Latitud = latitud;
        Longitud = longitud;
        Precision = precision;
        InspectorId = inspectorId;
        FechaHora = DateTime.UtcNow;
        Estado = EstadoInfraccion.Pendiente;
    }

    // ── Comportamiento de negocio ──
    public void Confirmar()
    {
        if (Estado != EstadoInfraccion.Pendiente)
            throw new InfraccionInvalidaException(
                "Solo se pueden confirmar infracciones pendientes.");

        Estado = EstadoInfraccion.Confirmada;
    }

    public void Anular(string motivo)
    {
        if (Estado == EstadoInfraccion.Anulada)
            throw new InfraccionInvalidaException("Ya está anulada.");

        Estado = EstadoInfraccion.Anulada;
    }
}
```

```csharp
// ── Value Object ──
namespace InfraccionesApp.Domain.ValueObjects;

public record Coordenada(double Latitud, double Longitud)
{
    public double DistanciaA(Coordenada otra)
    {
        // Fórmula Haversine simplificada
        var dLat = (otra.Latitud - Latitud) * Math.PI / 180;
        var dLon = (otra.Longitud - Longitud) * Math.PI / 180;
        // ... cálculo ...
        return distanciaEnMetros;
    }
}
```

```csharp
// ── Enum ──
namespace InfraccionesApp.Domain.Enums;

public enum EstadoInfraccion
{
    Pendiente,
    Confirmada,
    Anulada
}
```

```csharp
// ── Excepción de dominio ──
namespace InfraccionesApp.Domain.Exceptions;

public class InfraccionInvalidaException : Exception
{
    public InfraccionInvalidaException(string mensaje) : base(mensaje) { }
}
```

**Punto clave**: La entidad `Infraccion` valida sus propias reglas en el constructor y en sus métodos. No necesita saber de base de datos, HTTP, ni GPS. Si alguien intenta crear una infracción con latitud 999, la entidad misma lo rechaza.

---

## 7. Capa de Infraestructura

### Rol

**Implementa los detalles técnicos** que las otras capas necesitan pero no quieren conocer. Acá vive todo lo que habla con el "mundo exterior": base de datos, APIs, archivos, GPS, email.

### ¿Qué tiene?

| Elemento | Qué implementa | Ejemplo |
|---|---|---|
| **Repositorio** | `IInfraccionRepository` | Accede a SQL Server con EF Core |
| **DbContext** | Configuración de EF Core | `AppDbContext` |
| **Servicio externo** | `IEmailService` | Envía email con SendGrid |
| **GPS Service** | `IGpsService` | Wrapper de `Geolocation.Default` |
| **Migraciones** | Esquema de base de datos | Archivos de migración EF Core |

### Clases típicas

```
📁 InfraccionesApp.Infrastructure/
│
├── 📁 Data/
│   ├── AppDbContext.cs
│   └── 📁 Configurations/
│       └── InfraccionConfiguration.cs     ← Fluent API de EF Core
│
├── 📁 Repositories/
│   └── InfraccionRepository.cs            ← Implementa IInfraccionRepository
│
├── 📁 ExternalServices/
│   ├── EmailService.cs                    ← Implementa IEmailService
│   └── GpsService.cs                      ← Implementa IGpsService
│
└── 📁 Migrations/
    └── (archivos generados por EF Core)
```

```csharp
// ── DbContext ──
namespace InfraccionesApp.Infrastructure.Data;

public class AppDbContext : DbContext
{
    public DbSet<Infraccion> Infracciones => Set<Infraccion>();

    public AppDbContext(DbContextOptions<AppDbContext> options)
        : base(options) { }

    protected override void OnModelCreating(ModelBuilder modelBuilder)
    {
        modelBuilder.ApplyConfigurationsFromAssembly(
            typeof(AppDbContext).Assembly);
    }
}
```

```csharp
// ── Repositorio concreto ──
namespace InfraccionesApp.Infrastructure.Repositories;

public class InfraccionRepository : IInfraccionRepository
{
    private readonly AppDbContext _db;

    public InfraccionRepository(AppDbContext db)
    {
        _db = db;
    }

    public async Task<Infraccion?> ObtenerPorIdAsync(
        int id, CancellationToken ct)
    {
        return await _db.Infracciones
            .FirstOrDefaultAsync(i => i.Id == id, ct);
    }

    public async Task<List<Infraccion>> ObtenerPendientesAsync(
        CancellationToken ct)
    {
        return await _db.Infracciones
            .Where(i => i.Estado == EstadoInfraccion.Pendiente)
            .ToListAsync(ct);
    }

    public async Task GuardarAsync(
        Infraccion infraccion, CancellationToken ct)
    {
        _db.Infracciones.Add(infraccion);
        await _db.SaveChangesAsync(ct);
    }
}
```

```csharp
// ── Servicio externo ──
namespace InfraccionesApp.Infrastructure.ExternalServices;

public class EmailService : IEmailService
{
    private readonly HttpClient _http;

    public EmailService(HttpClient http)
    {
        _http = http;
    }

    public async Task EnviarNotificacionAsync(
        Infraccion infraccion, CancellationToken ct)
    {
        var payload = new
        {
            to = "infracciones@municipio.gob.ar",
            subject = $"Nueva infracción #{infraccion.Id}",
            body = $"Ubicación: {infraccion.Latitud}, {infraccion.Longitud}"
        };

        await _http.PostAsJsonAsync("api/send", payload, ct);
    }
}
```

**Punto clave**: `InfraccionRepository` implementa `IInfraccionRepository` (definida en Dominio o Aplicación). La capa de Aplicación no sabe que se usa Entity Framework, SQL Server, ni nada. Solo conoce la interfaz.

---

## 8. Orquestadores: qué son, cuándo usarlos y ejemplos

### 8.1 ¿Qué es un orquestador?

Un **orquestador** es un componente que **coordina la ejecución de un flujo** que involucra múltiples pasos, servicios o sistemas. No contiene lógica de negocio — su responsabilidad es **decidir qué se ejecuta, en qué orden, y cuándo**.

**Analogía**: Un director de orquesta no toca ningún instrumento. Le indica a cada músico cuándo entrar, cuándo parar, y a qué velocidad. Del mismo modo, un orquestador no hace el trabajo — le dice a cada servicio cuándo actuar.

### 8.2 ¿Cuándo se necesita un orquestador?

| Situación | ¿Orquestador? | ¿Por qué? |
|---|---|---|
| CRUD simple (guardar y listo) | No | Un servicio de aplicación alcanza |
| Operación con 2 pasos simples | No | El servicio de aplicación coordina |
| Flujo con 4+ pasos, rollback, reintentos | **Sí** | Demasiada complejidad para un servicio |
| Job en background con ciclo de vida | **Sí** | Necesita controlar timing + cancelación |
| Proceso que cruza múltiples bounded contexts | **Sí** | Coordina entre subsistemas independientes |
| Saga (transacciones distribuidas) | **Sí** | Maneja compensación en caso de fallo |

### 8.3 Orquestador vs Servicio de Aplicación

```
┌──────────────────────────────────────────────────────────────────┐
│  SERVICIO DE APLICACIÓN (caso simple)                            │
│                                                                  │
│  RegistrarAsync(request):                                        │
│      1. Crear entidad                                            │
│      2. Guardar en BD                                            │
│      3. Enviar email                                             │
│      return response                                             │
│                                                                  │
│  → Flujo lineal, pocos pasos, sin lógica de control compleja     │
└──────────────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────────────┐
│  ORQUESTADOR (caso complejo)                                     │
│                                                                  │
│  EjecutarAsync(ct):                                              │
│      while (!ct.IsCancellationRequested)                         │
│          1. Obtener pendientes                                   │
│          2. Para cada uno:                                       │
│              a. Validar con servicio externo                     │
│              b. Si OK → procesar                                 │
│              c. Si falla → reintentar 3 veces                    │
│              d. Si agota reintentos → mover a cola de errores    │
│          3. Esperar 5 minutos                                    │
│                                                                  │
│  → Ciclo de vida, reintentos, manejo de errores, timing          │
└──────────────────────────────────────────────────────────────────┘
```

### 8.4 Tipos de orquestadores en .NET

#### Tipo 1: BackgroundService / Hosted Service (Job periódico)

Es el más común. Un proceso que corre en el servidor en segundo plano:

```csharp
namespace InfraccionesApp.Worker.Jobs;

/// <summary>
/// Orquestador: cada 5 minutos busca infracciones pendientes de sincronización,
/// las valida contra un servicio externo, y las marca como procesadas.
/// </summary>
public class SincronizacionJob : BackgroundService
{
    private readonly IServiceScopeFactory _scopeFactory;
    private readonly ILogger<SincronizacionJob> _logger;
    private readonly TimeSpan _intervalo = TimeSpan.FromMinutes(5);

    public SincronizacionJob(
        IServiceScopeFactory scopeFactory,
        ILogger<SincronizacionJob> logger)
    {
        _scopeFactory = scopeFactory;
        _logger = logger;
    }

    /// <summary>
    /// stoppingToken: lo crea el Host de .NET.
    /// Se cancela cuando el servidor se apaga (Ctrl+C, deploy, crash).
    /// </summary>
    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        _logger.LogInformation("SincronizacionJob iniciado.");

        while (!stoppingToken.IsCancellationRequested)
        {
            try
            {
                // Scope necesario porque BackgroundService es Singleton
                // pero los repositorios son Scoped
                using var scope = _scopeFactory.CreateScope();
                var repo = scope.ServiceProvider
                    .GetRequiredService<IInfraccionRepository>();
                var validador = scope.ServiceProvider
                    .GetRequiredService<IValidacionExternaService>();
                var email = scope.ServiceProvider
                    .GetRequiredService<IEmailService>();

                // ── Paso 1: Obtener pendientes ──
                var pendientes = await repo.ObtenerPendientesAsync(stoppingToken);
                _logger.LogInformation("Encontradas {Count} infracciones pendientes.",
                    pendientes.Count);

                // ── Paso 2: Procesar cada una ──
                foreach (var infraccion in pendientes)
                {
                    // Si el server se está apagando, no procesar más
                    stoppingToken.ThrowIfCancellationRequested();

                    try
                    {
                        var esValida = await validador.ValidarAsync(
                            infraccion, stoppingToken);

                        if (esValida)
                        {
                            infraccion.Confirmar();
                            await repo.GuardarAsync(infraccion, stoppingToken);
                            await email.EnviarNotificacionAsync(
                                infraccion, stoppingToken);
                        }
                    }
                    catch (Exception ex) when (ex is not OperationCanceledException)
                    {
                        // Error en UNA infracción no detiene las demás
                        _logger.LogError(ex,
                            "Error procesando infracción {Id}", infraccion.Id);
                    }
                }
            }
            catch (OperationCanceledException)
            {
                // Server apagándose — salir del while
                break;
            }
            catch (Exception ex)
            {
                // Error general — loguear y reintentar en el próximo ciclo
                _logger.LogError(ex, "Error en ciclo de sincronización.");
            }

            // ── Paso 3: Esperar antes del próximo ciclo ──
            // Task.Delay también recibe el token: si el server se apaga,
            // no espera los 5 minutos completos
            await Task.Delay(_intervalo, stoppingToken);
        }

        _logger.LogInformation("SincronizacionJob detenido.");
    }
}
```

Registro:
```csharp
// Program.cs del servidor
builder.Services.AddHostedService<SincronizacionJob>();
```

**¿Por qué es un orquestador?** Porque:
- Controla el **ciclo de vida** (while + delay)
- Decide **cuándo** ejecutar (cada 5 minutos)
- Coordina **múltiples servicios** (repo + validador + email)
- Maneja **errores por ítem** sin detener el flujo general
- Responde a **cancelación** del host

#### Tipo 2: Orquestador de flujo multi-paso

Un flujo complejo donde los pasos dependen entre sí y hay que manejar compensación si algo falla:

```csharp
namespace InfraccionesApp.Application.Orchestrators;

/// <summary>
/// Orquesta el flujo completo de registro de una infracción con foto:
/// 1. Validar datos
/// 2. Subir foto al storage
/// 3. Guardar infracción en BD
/// 4. Notificar por email
///
/// Si el paso 3 falla, debe eliminar la foto subida en paso 2 (compensación).
/// </summary>
public class RegistroInfraccionOrchestrator
{
    private readonly IInfraccionRepository _repo;
    private readonly IStorageService _storage;
    private readonly IEmailService _email;
    private readonly ILogger<RegistroInfraccionOrchestrator> _logger;

    public RegistroInfraccionOrchestrator(
        IInfraccionRepository repo,
        IStorageService storage,
        IEmailService email,
        ILogger<RegistroInfraccionOrchestrator> logger)
    {
        _repo = repo;
        _storage = storage;
        _email = email;
        _logger = logger;
    }

    public async Task<InfraccionResponse> EjecutarAsync(
        CrearInfraccionConFotoRequest request, CancellationToken ct)
    {
        string? fotoUrl = null;

        try
        {
            // ── Paso 1: Crear entidad (valida reglas de negocio) ──
            var infraccion = new Infraccion(
                descripcion: request.Descripcion,
                latitud: request.Latitud,
                longitud: request.Longitud,
                precision: request.Precision,
                inspectorId: request.InspectorId);

            // ── Paso 2: Subir foto al storage ──
            if (request.Foto != null)
            {
                fotoUrl = await _storage.SubirFotoAsync(
                    request.Foto, ct);
                infraccion.AgregarFoto(fotoUrl);
            }

            // ── Paso 3: Persistir en BD ──
            await _repo.GuardarAsync(infraccion, ct);

            // ── Paso 4: Notificar (no crítico — si falla, no revertimos) ──
            try
            {
                await _email.EnviarNotificacionAsync(infraccion, ct);
            }
            catch (Exception ex)
            {
                _logger.LogWarning(ex,
                    "No se pudo enviar email para infracción {Id}. " +
                    "Se procesará en el próximo ciclo de reintentos.",
                    infraccion.Id);
            }

            return MapearResponse(infraccion);
        }
        catch (Exception) when (fotoUrl != null)
        {
            // ── COMPENSACIÓN ──
            // Si el paso 3 falló pero el paso 2 ya subió la foto,
            // eliminar la foto para no dejar basura en el storage.
            _logger.LogWarning(
                "Compensación: eliminando foto {Url} por fallo posterior.",
                fotoUrl);
            await _storage.EliminarFotoAsync(fotoUrl, CancellationToken.None);
            throw; // re-lanzar la excepción original
        }
    }

    private static InfraccionResponse MapearResponse(Infraccion i) => new()
    {
        Id = i.Id,
        Descripcion = i.Descripcion,
        Latitud = i.Latitud,
        Longitud = i.Longitud,
        FechaHora = i.FechaHora,
        EstadoTexto = i.Estado.ToString()
    };
}
```

**¿Por qué es un orquestador y no un servicio?** Porque maneja **compensación** (si paso 3 falla, revierte paso 2). Un servicio de aplicación simple no tiene esa responsabilidad.

### 8.5 ¿Dónde vive el orquestador?

| Tipo de orquestador | Capa | ¿Por qué? |
|---|---|---|
| BackgroundService / Job | **Host / Worker** (proyecto propio) | Tiene ciclo de vida del servidor |
| Orquestador de flujo multi-paso | **Aplicación** | Coordina un caso de uso complejo |
| Saga (transacciones distribuidas) | **Aplicación o Infraestructura** | Depende de la complejidad |

```
📁 Solución
│
├── 📁 InfraccionesApp.Worker/                    ← BackgroundServices / Jobs
│   └── 📁 Jobs/
│       └── SincronizacionJob.cs
│
├── 📁 InfraccionesApp.Application/               ← Orquestadores de flujo
│   ├── 📁 Orchestrators/
│   │   └── RegistroInfraccionOrchestrator.cs
│   └── 📁 Services/
│       └── InfraccionService.cs                   ← Servicio simple (CRUD)
```

### 8.6 Resumen: Servicio vs Orquestador

| Aspecto | Servicio de Aplicación | Orquestador |
|---|---|---|
| **Complejidad** | Baja-media (2-3 pasos) | Alta (4+ pasos, compensación, reintentos) |
| **Ciclo de vida** | No tiene (se ejecuta y termina) | Puede tener (while + delay) |
| **Manejo de errores** | try-catch simple | Compensación, reintentos, dead-letter |
| **Naming** | `InfraccionService` | `SincronizacionJob`, `RegistroOrchestrator` |
| **Frecuencia** | Muy común (toda app lo tiene) | Poco común (apps complejas) |
| **Cuándo usarlo** | Siempre (es el default) | Solo cuando un servicio se vuelve demasiado complejo |

> **Regla práctica**: Empezá siempre con un servicio de aplicación. Si el método crece a más de 4-5 pasos con manejo de errores complejo, reintentos, o compensación, considerá extraerlo a un orquestador.

---

## 9. Regla de dependencia

La regla fundamental: **las dependencias apuntan hacia adentro** (hacia Dominio).

```
  Presentación ────► Aplicación ────► Dominio
                          │                ▲
                          │                │
                          ▼                │
                    Infraestructura ───────┘
                    (implementa interfaces
                     definidas en Dominio)
```

### ¿Cómo se logra en .NET?

Con **inyección de dependencias (DI)** en `Program.cs`:

```csharp
// Program.cs

// Dominio: no se registra (son clases que se instancian directamente)

// Aplicación
builder.Services.AddScoped<InfraccionService>();
builder.Services.AddScoped<RegistroInfraccionOrchestrator>();

// Infraestructura — registra las IMPLEMENTACIONES contra las INTERFACES
builder.Services.AddScoped<IInfraccionRepository, InfraccionRepository>();
builder.Services.AddScoped<IEmailService, EmailService>();
builder.Services.AddScoped<IGpsService, GpsService>();

// DbContext
builder.Services.AddDbContext<AppDbContext>(options =>
    options.UseSqlServer(builder.Configuration
        .GetConnectionString("DefaultConnection")));

// Jobs
builder.Services.AddHostedService<SincronizacionJob>();
```

**¿Qué logra esto?** `InfraccionService` pide `IInfraccionRepository` en su constructor. El contenedor de DI le inyecta `InfraccionRepository` (la implementación concreta). Pero `InfraccionService` **nunca sabe** que es `InfraccionRepository` — solo conoce la interfaz.

### Referencias entre proyectos (.csproj)

```
InfraccionesApp.API
    └── referencia → InfraccionesApp.Application
                         └── referencia → InfraccionesApp.Domain

InfraccionesApp.Infrastructure
    └── referencia → InfraccionesApp.Domain

InfraccionesApp.API (también)
    └── referencia → InfraccionesApp.Infrastructure  ← SOLO para registrar DI en Program.cs
```

---

## 10. Ejemplo integrado: App de Infracciones de Tránsito

### Flujo completo: el usuario toca "Registrar Infracción" en MAUI

```
┌────────────────────────────────────────────────────────────────────────┐
│ 1. PRESENTACIÓN (InfraccionPage.xaml.cs)                              │
│    Usuario toca botón → crea CancellationTokenSource                  │
│    → llama _service.RegistrarAsync(request, token)                    │
└─────────────────────────────┬──────────────────────────────────────────┘
                              │
                              ▼
┌────────────────────────────────────────────────────────────────────────┐
│ 2. APLICACIÓN (InfraccionService.cs)                                  │
│    Recibe DTO → crea entidad Infraccion (Dominio la valida)           │
│    → llama _repo.GuardarAsync(infraccion, ct)                         │
│    → llama _email.EnviarNotificacionAsync(infraccion, ct)             │
│    → devuelve InfraccionResponse (DTO)                                │
└─────────┬────────────────────────────┬─────────────────────────────────┘
          │                            │
          ▼                            ▼
┌──────────────────────┐   ┌──────────────────────────────┐
│ 3. DOMINIO           │   │ 4. INFRAESTRUCTURA           │
│ new Infraccion(...)  │   │ InfraccionRepository         │
│ • Valida latitud     │   │   → EF Core → SQL Server     │
│ • Valida descripción │   │ EmailService                 │
│ • Asigna estado      │   │   → HttpClient → SendGrid   │
│ • Asigna fecha       │   │                              │
└──────────────────────┘   └──────────────────────────────┘
```

### Si el usuario toca "Cancelar" en cualquier momento:

```
CancellationTokenSource.Cancel()
    │
    ├── Si estaba en repo.GuardarAsync()
    │   └── EF Core cancela la query SQL → OperationCanceledException
    │
    ├── Si estaba en email.EnviarAsync()
    │   └── HttpClient cancela el POST → OperationCanceledException
    │
    └── La excepción sube hasta el catch de la Page
        └── Muestra "Operación cancelada por el usuario."
```

El `CancellationToken` viaja desde la **Presentación** (donde se creó) a través de **Aplicación** hasta **Infraestructura** (donde se ejecutan las operaciones reales), sin que ninguna capa intermedia tenga que saber los detalles de las otras.
