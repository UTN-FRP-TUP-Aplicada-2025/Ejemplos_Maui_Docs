# CancellationToken en .NET — Guía Completa

## Índice

1. [¿Qué es un CancellationToken?](#1-qué-es-un-cancellationtoken)
2. [CancellationTokenSource vs CancellationToken](#2-cancellationtokensource-vs-cancellationtoken)
3. [¿Quién crea y quién recibe?](#3-quién-crea-y-quién-recibe)
4. [Cómo funciona la cancelación](#4-cómo-funciona-la-cancelación)
5. [Contextos de uso](#5-contextos-de-uso)
6. [Patrones y ejemplos](#6-patrones-y-ejemplos)
7. [Errores comunes y trampas](#7-errores-comunes-y-trampas)
8. [CancellationToken y la arquitectura en capas](#8-cancellationtoken-y-la-arquitectura-en-capas)

---

## 1. ¿Qué es un CancellationToken?

Un `CancellationToken` es un mecanismo de .NET para **cancelar operaciones asíncronas de forma cooperativa**. "Cooperativa" significa que la operación no se mata a la fuerza — se le **pide amablemente** que se detenga, y ella decide cómo hacerlo de manera limpia.

### ¿Por qué es necesario?

Sin cancelación, una operación asíncrona que tarda mucho (GPS buscando satélites, una llamada HTTP a un servidor lento) sigue corriendo aunque ya nadie necesite su resultado:

```
Sin CancellationToken:
    Usuario toca "Obtener GPS"
        └── GetLocationAsync() empieza a buscar satélites (30 seg)
    Usuario sale de la página a los 2 seg
        └── GetLocationAsync() SIGUE corriendo 28 seg más
            └── Resultado desperdiciado, batería gastada, posible crash
```

```
Con CancellationToken:
    Usuario toca "Obtener GPS"
        └── GetLocationAsync(request, token) empieza a buscar satélites
    Usuario sale de la página a los 2 seg
        └── token.Cancel()
            └── GetLocationAsync detecta la cancelación → se detiene
                └── Lanza OperationCanceledException → catch limpio
```

---

## 2. CancellationTokenSource vs CancellationToken

Son dos clases distintas con roles opuestos:

| | `CancellationTokenSource` | `CancellationToken` |
|---|---|---|
| **Tipo** | `class` (referencia) | `struct` (valor) |
| **Quién lo tiene** | El que **controla** la cancelación | El que **observa** la cancelación |
| **Puede cancelar** | Sí (`.Cancel()`) | No (solo lectura) |
| **Disposable** | Sí (`IDisposable`) — hay que hacer `.Dispose()` | No |
| **Se pasa a servicios** | **Nunca** | **Siempre** |

```csharp
// El CancellationTokenSource es el "control remoto"
var cts = new CancellationTokenSource();

// El CancellationToken es la "señal" que observan los servicios
CancellationToken token = cts.Token;

// Quien tiene el CTS puede cancelar
cts.Cancel();  // ← activa la señal

// Quien tiene solo el token puede verificar, pero NO puede cancelar
bool cancelado = token.IsCancellationRequested;  // → true
```

**Analogía**: `CancellationTokenSource` es el botón de incendio que solo el encargado puede presionar. `CancellationToken` es la alarma que suena en todo el edificio — todos la escuchan, pero nadie puede activarla desde su escritorio.

---

## 3. ¿Quién crea y quién recibe?

### Regla de oro

> **El CancellationTokenSource lo crea quien sabe CUÁNDO cancelar.**
> Los servicios intermedios solo reciben el `CancellationToken` y lo propagan.

### Tabla de contextos

| Contexto | ¿Quién crea el CTS? | ¿Cuándo cancela? |
|---|---|---|
| **Página MAUI** | El code-behind de la página | Usuario toca "Cancelar" o sale de la página |
| **Controller ASP.NET** | El framework (automático) | El cliente HTTP desconecta (cierra browser, timeout) |
| **BackgroundService / Job** | El Host de .NET (automático) | El servidor se apaga (Ctrl+C, deploy, crash) |
| **Test unitario** | El test | Quiere simular un timeout o cancelación |

### Flujo visual

```
┌─────────────────────────────────────────────────────────────────┐
│  PUNTO FINAL (crea CancellationTokenSource)                     │
│                                                                 │
│  var cts = new CancellationTokenSource();                       │
│                                                                 │
│  cts.Token ──► servicio1.MetodoAsync(datos, cts.Token)          │
│                    │                                            │
│                    └──► servicio2.OtroMetodoAsync(x, token)     │
│                              │                                  │
│                              └──► httpClient.PostAsync(url, ct) │
│                                                                 │
│  // En cualquier momento:                                       │
│  cts.Cancel();  ← cancela TODA la cadena                        │
└─────────────────────────────────────────────────────────────────┘
```

Los servicios intermedios **nunca** crean un `CancellationTokenSource`. Solo reciben el `CancellationToken` (struct de solo lectura) y lo pasan al siguiente. Esto evita que un servicio intermedio cancele accidentalmente algo que no le corresponde.

---

## 4. Cómo funciona la cancelación

### 4.1 ¿Qué pasa cuando se llama a `.Cancel()`?

1. El `CancellationTokenSource` marca su token como cancelado
2. Todas las operaciones que están escuchando ese token **reciben la señal**
3. Las operaciones cooperativas (las de .NET: `GetLocationAsync`, `PostAsync`, `ToListAsync`, `Task.Delay`, etc.) lanzan `OperationCanceledException`
4. La excepción sube por la cadena de `await` hasta el `catch` más cercano

```csharp
_cts.Cancel();
//      │
//      ▼
// cts.Token.IsCancellationRequested == true
//      │
//      ▼
// GetLocationAsync detecta que el token está cancelado
//      │
//      ▼
// Lanza OperationCanceledException
//      │
//      ▼
// Sube por todos los await hasta el catch
```

### 4.2 Dos formas de detectar la cancelación

#### Forma 1: Excepción automática (la más común)

Las APIs de .NET (`HttpClient`, `EF Core`, `Geolocation`, `Task.Delay`) verifican el token internamente y lanzan la excepción automáticamente:

```csharp
try
{
    // GetLocationAsync verifica el token cada cierto tiempo internamente.
    // Si está cancelado, lanza OperationCanceledException.
    var location = await Geolocation.Default.GetLocationAsync(request, token);
}
catch (OperationCanceledException)
{
    // Cancelado — limpiar y salir
}
```

#### Forma 2: Verificación manual (en loops propios)

Si tu código tiene un loop largo, verificás el token manualmente:

```csharp
public async Task ProcesarLoteAsync(
    List<Infraccion> items, CancellationToken ct)
{
    foreach (var item in items)
    {
        // Verificar antes de cada iteración
        ct.ThrowIfCancellationRequested();

        await ProcesarUnoAsync(item, ct);
    }
}
```

`ThrowIfCancellationRequested()` es equivalente a:
```csharp
if (ct.IsCancellationRequested)
    throw new OperationCanceledException(ct);
```

### 4.3 ¿`OperationCanceledException` o `TaskCanceledException`?

`TaskCanceledException` hereda de `OperationCanceledException`. Catchear `OperationCanceledException` atrapa ambas:

```csharp
// ✅ Correcto — atrapa ambas
catch (OperationCanceledException) { }

// ⚠️ Incompleto — no atrapa OperationCanceledException directa
catch (TaskCanceledException) { }
```

| Método | Lanza |
|---|---|
| `Geolocation.GetLocationAsync` | `OperationCanceledException` |
| `HttpClient.PostAsync` | `TaskCanceledException` |
| `EF Core (ToListAsync, SaveChangesAsync)` | `OperationCanceledException` |
| `Task.Delay` | `TaskCanceledException` |

**Recomendación**: Siempre catchear `OperationCanceledException`.

---

## 5. Contextos de uso

### 5.1 Página MAUI — Cancelación por el usuario

La página crea el CTS porque **sabe** cuándo el usuario quiere cancelar (botón, salir de la página):

```csharp
public partial class GpsPage : ContentPage
{
    private CancellationTokenSource? _gpsCts;

    private async void OnObtenerUbicacion_Clicked(object sender, EventArgs e)
    {
        // Defensivo: cancelar operación anterior si existe
        _gpsCts?.Cancel();
        _gpsCts?.Dispose();
        _gpsCts = new CancellationTokenSource();

        BtnObtener.IsEnabled = false;
        BtnCancelar.IsVisible = true;

        try
        {
            var request = new GeolocationRequest(
                GeolocationAccuracy.Best, TimeSpan.FromSeconds(30));
            var location = await Geolocation.Default.GetLocationAsync(
                request, _gpsCts.Token);

            if (location != null)
                MostrarUbicacion(location);
        }
        catch (OperationCanceledException)
        {
            LblInfo.Text = "Búsqueda cancelada por el usuario.";
        }
        finally
        {
            BtnObtener.IsEnabled = true;
            BtnCancelar.IsVisible = false;
            _gpsCts?.Dispose();
            _gpsCts = null;
        }
    }

    // El usuario toca "Cancelar" — activa la señal
    private void OnCancelar_Clicked(object sender, EventArgs e)
    {
        _gpsCts?.Cancel();
    }

    // El usuario sale de la página — cancelar lo que esté en curso
    protected override void OnDisappearing()
    {
        base.OnDisappearing();
        _gpsCts?.Cancel();
        _gpsCts?.Dispose();
        _gpsCts = null;
    }
}
```

### 5.2 Controller ASP.NET — Cancelación automática por el framework

En ASP.NET, el framework te **provee** el `CancellationToken` como parámetro del action. Se cancela automáticamente si el cliente desconecta:

```csharp
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
        CancellationToken ct)              // ← el framework lo inyecta
    {
        // Si el cliente cierra el browser o hace timeout,
        // ct se cancela automáticamente.
        var resultado = await _service.RegistrarAsync(request, ct);
        return Ok(resultado);
    }
}
```

**¿Cuándo se cancela automáticamente?**

| Situación | ¿Se cancela el token? |
|---|---|
| Cliente cierra el browser | Sí |
| Cliente aborta el fetch/request | Sí |
| Timeout del HttpClient del cliente | Sí |
| El servidor tarda pero el cliente espera | No |
| Error 500 en el servidor | No (el token no se cancela, la excepción sube normal) |

### 5.3 BackgroundService — Cancelación por shutdown del host

El `stoppingToken` se cancela cuando el servidor se apaga:

```csharp
public class SincronizacionJob : BackgroundService
{
    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        // stoppingToken lo crea el Host de .NET
        // Se cancela con: Ctrl+C, docker stop, deploy, crash

        while (!stoppingToken.IsCancellationRequested)
        {
            await ProcesarPendientesAsync(stoppingToken);
            await Task.Delay(TimeSpan.FromMinutes(5), stoppingToken);
        }
    }
}
```

### 5.4 Timeout automático con CancellationTokenSource

Podés crear un CTS que se cancela solo después de un tiempo:

```csharp
// Se cancela automáticamente después de 20 segundos
var cts = new CancellationTokenSource(TimeSpan.FromSeconds(20));

try
{
    var location = await Geolocation.Default.GetLocationAsync(request, cts.Token);
}
catch (OperationCanceledException)
{
    // Pasaron 20 segundos sin resultado
    LblInfo.Text = "Timeout: no se pudo obtener ubicación.";
}
```

### 5.5 Combinar timeout del SO + timeout de tu app

`GeolocationRequest.Timeout` y `CancellationToken` no son lo mismo:

| Mecanismo | Qué hace | Resultado al vencer |
|---|---|---|
| `GeolocationRequest.Timeout` | Timeout **interno** del SO para obtener fix GPS | Devuelve `null` (no excepción) |
| `CancellationToken` | Cancelación **de tu código** | Lanza `OperationCanceledException` |

Podés combinar ambos:

```csharp
// Timeout del SO: 15 seg (deja de buscar satélites)
var request = new GeolocationRequest(
    GeolocationAccuracy.Best, TimeSpan.FromSeconds(15));

// Timeout de tu app: 20 seg (margen extra para UI)
var cts = new CancellationTokenSource(TimeSpan.FromSeconds(20));

var location = await Geolocation.Default.GetLocationAsync(
    request, cts.Token);

// Caso 1: GPS encuentra fix en 10 seg → devuelve Location ✅
// Caso 2: GPS no encuentra en 15 seg → devuelve null (timeout del SO)
// Caso 3: Token se cancela en 20 seg → OperationCanceledException
```

---

## 6. Patrones y ejemplos

### 6.1 Patrón defensivo: cancelar y disponer antes de crear nuevo CTS

Si el usuario puede tocar el botón dos veces rápido (race condition), el primer CTS queda huérfano:

```csharp
// ❌ PROBLEMA — Si el usuario toca dos veces rápido:
private async void OnObtener_Clicked(object sender, EventArgs e)
{
    _cts = new CancellationTokenSource();  // CTS_1 se pierde
    // ...
}
// Primera pulsación: _cts = CTS_1 → empieza consulta GPS con CTS_1
// Segunda pulsación: _cts = CTS_2 → CTS_1 queda huérfano, nunca se cancela
// Botón cancelar: _cts.Cancel() → solo cancela CTS_2
// CTS_1 sigue corriendo y nunca se dispone (leak)
```

```csharp
// ✅ SOLUCIÓN — Cancelar y disponer antes de crear
private async void OnObtener_Clicked(object sender, EventArgs e)
{
    _cts?.Cancel();     // si hay uno anterior, cancelarlo
    _cts?.Dispose();    // liberar recursos
    _cts = new CancellationTokenSource();
    // ...
}
```

**`_cts?.Cancel()` cuando `_cts` es null**: El operador `?.` (null-conditional) hace que la expresión **no se ejecute** y devuelva `null`. No lanza `NullReferenceException`. Es seguro.

### 6.2 Patrón completo: GPS + servidor con un solo token

Un único `CancellationToken` cancela toda la cadena (GPS + HTTP):

```csharp
private CancellationTokenSource? _cts;

private async void OnRegistrar_Clicked(object sender, EventArgs e)
{
    _cts?.Cancel();
    _cts?.Dispose();
    _cts = new CancellationTokenSource();

    BtnRegistrar.IsEnabled = false;
    BtnCancelar.IsVisible = true;

    try
    {
        // Paso 1: GPS (cancelable)
        LblEstado.Text = "Obteniendo ubicación GPS...";
        var location = await _gps.ObtenerUbicacionAsync(_cts.Token);

        if (location == null)
        {
            LblEstado.Text = "GPS sin señal.";
            return;
        }

        // Paso 2: Enviar al servidor (mismo token — cancelable)
        LblEstado.Text = "Enviando al servidor...";
        var dto = new InfraccionDto
        {
            Latitud = location.Latitude,
            Longitud = location.Longitude,
            Descripcion = EntryDescripcion.Text,
            FechaHora = DateTime.Now
        };

        var ok = await _api.EnviarInfraccionAsync(dto, _cts.Token);

        LblEstado.Text = ok
            ? "Registrado correctamente."
            : "El servidor rechazó la solicitud.";
    }
    catch (OperationCanceledException)
    {
        LblEstado.Text = "Operación cancelada.";
    }
    catch (HttpRequestException ex)
    {
        LblEstado.Text = $"Error de red: {ex.Message}";
    }
    finally
    {
        BtnRegistrar.IsEnabled = true;
        BtnCancelar.IsVisible = false;
        _cts?.Dispose();
        _cts = null;
    }
}

private void OnCancelar_Clicked(object sender, EventArgs e)
{
    _cts?.Cancel();
}
```

Flujo de cancelación:
```
Usuario toca "Cancelar"
    │
    └── _cts.Cancel()
            │
            ├── Si estaba en Paso 1 (GPS)
            │   └── GetLocationAsync detecta token → OperationCanceledException
            │
            └── Si estaba en Paso 2 (HTTP)
                └── PostAsync detecta token → TaskCanceledException
                    (que hereda de OperationCanceledException)

    En ambos casos → catch (OperationCanceledException) → "Operación cancelada."
```

### 6.3 Patrón: Cleanup en OnDisappearing

Siempre cancelar operaciones pendientes al salir de la página:

```csharp
protected override void OnDisappearing()
{
    base.OnDisappearing();
    _cts?.Cancel();
    _cts?.Dispose();
    _cts = null;
}
```

**¿Por qué?** Si el usuario navega a otra página mientras el GPS está buscando, `GetLocationAsync` seguiría corriendo. Al volver, el resultado ya no tiene sentido y podría intentar actualizar controles que ya no existen (crash).

### 6.4 Patrón: Linked tokens (combinar dos fuentes de cancelación)

Podés combinar un timeout automático con cancelación manual:

```csharp
private async void OnObtener_Clicked(object sender, EventArgs e)
{
    // Token manual (botón cancelar)
    _cts?.Cancel();
    _cts?.Dispose();
    _cts = new CancellationTokenSource();

    // Token automático (timeout de 30 seg)
    var timeoutCts = new CancellationTokenSource(TimeSpan.FromSeconds(30));

    // Linked: se cancela si CUALQUIERA de los dos se cancela
    var linkedCts = CancellationTokenSource.CreateLinkedTokenSource(
        _cts.Token, timeoutCts.Token);

    try
    {
        var location = await Geolocation.Default.GetLocationAsync(
            request, linkedCts.Token);
        // ...
    }
    catch (OperationCanceledException)
    {
        if (timeoutCts.IsCancellationRequested)
            LblInfo.Text = "Timeout: no se pudo obtener ubicación en 30 seg.";
        else
            LblInfo.Text = "Cancelado por el usuario.";
    }
    finally
    {
        linkedCts.Dispose();
        timeoutCts.Dispose();
    }
}
```

---

## 7. Errores comunes y trampas

### 7.1 No disponer el CancellationTokenSource

```csharp
// ❌ PROBLEMA — Leak de recursos
_cts = new CancellationTokenSource();
// ... usar ...
_cts = null;  // El CTS anterior nunca se dispone

// ✅ CORRECTO
_cts?.Dispose();
_cts = new CancellationTokenSource();
```

`CancellationTokenSource` implementa `IDisposable`. Si no lo disponés, mantiene referencias internas (callbacks registrados) que no se liberan.

### 7.2 Pasar CancellationTokenSource en vez de CancellationToken

```csharp
// ❌ INCORRECTO — El servicio podría llamar a .Cancel()
public async Task ProcesarAsync(CancellationTokenSource cts)
{
    cts.Cancel();  // ¡El servicio no debería poder cancelar!
}

// ✅ CORRECTO — Solo lectura, no puede cancelar
public async Task ProcesarAsync(CancellationToken ct)
{
    // ct.Cancel() no existe — solo puede observar
}
```

### 7.3 Olvidar pasar el token a métodos async

```csharp
// ❌ PROBLEMA — El token se pierde en la cadena
public async Task RegistrarAsync(InfraccionDto dto, CancellationToken ct)
{
    await _repo.GuardarAsync(dto);                    // ← falta ct
    await _http.PostAsJsonAsync("api/enviar", dto);    // ← falta ct
}

// ✅ CORRECTO — El token viaja por toda la cadena
public async Task RegistrarAsync(InfraccionDto dto, CancellationToken ct)
{
    await _repo.GuardarAsync(dto, ct);
    await _http.PostAsJsonAsync("api/enviar", dto, ct);
}
```

Si no pasás el token, la operación no se cancela aunque el usuario haya tocado "Cancelar".

### 7.4 Usar CancellationToken después de Dispose

```csharp
// ❌ PROBLEMA POTENCIAL
_cts?.Dispose();
// ... más adelante ...
var token = _cts.Token;  // ObjectDisposedException

// ✅ CORRECTO — Poner en null después de dispose
_cts?.Dispose();
_cts = null;
```

### 7.5 Ignorar OperationCanceledException

```csharp
// ❌ PROBLEMA — Atrapa cancelación como error genérico
catch (Exception ex)
{
    LblInfo.Text = $"Error: {ex.Message}";
    // Muestra "The operation was canceled" como si fuera un error
}

// ✅ CORRECTO — Catchear cancelación primero (es más específica)
catch (OperationCanceledException)
{
    LblInfo.Text = "Cancelado por el usuario.";
}
catch (Exception ex)
{
    LblInfo.Text = $"Error: {ex.Message}";
}
```

**Orden de los catch**: Siempre de más específico a más general. `OperationCanceledException` antes de `Exception`.

### 7.6 No cancelar al salir de la página (MAUI)

```csharp
// ❌ PROBLEMA — Si el usuario sale mientras busca GPS
// GetLocationAsync sigue corriendo e intenta actualizar la UI → crash
protected override void OnDisappearing()
{
    base.OnDisappearing();
    // No cancela nada
}

// ✅ CORRECTO
protected override void OnDisappearing()
{
    base.OnDisappearing();
    _cts?.Cancel();
    _cts?.Dispose();
    _cts = null;
}
```

---

## 8. CancellationToken y la arquitectura en capas

### 8.1 ¿Dónde se crea y dónde se consume?

```
┌─────────────────────────────────────────────────────────────────┐
│  PRESENTACIÓN (crea el CancellationTokenSource)                 │
│                                                                 │
│  Page / Controller / BackgroundService                          │
│  var cts = new CancellationTokenSource();                       │
│                                                                 │
│  El "punto final" sabe CUÁNDO cancelar:                         │
│  • Page.OnDisappearing()                                       │
│  • Controller: client disconnect                                │
│  • Job: server shutdown                                         │
├─────────────────────────────────────────────────────────────────┤
│  APLICACIÓN (recibe CancellationToken, lo propaga)              │
│                                                                 │
│  public async Task<Result> RegistrarAsync(                      │
│      Request request, CancellationToken ct)                     │
│  {                                                              │
│      await _repo.GuardarAsync(entity, ct);  ← pasa el token    │
│      await _email.EnviarAsync(entity, ct);  ← pasa el token    │
│  }                                                              │
│                                                                 │
│  NO crea CTS, NO cancela — solo propaga                         │
├─────────────────────────────────────────────────────────────────┤
│  INFRAESTRUCTURA (ejecuta la operación cancelable)              │
│                                                                 │
│  await _db.SaveChangesAsync(ct);   ← EF Core verifica el token │
│  await _http.PostAsync(url, ct);   ← HttpClient verifica       │
│  await Geolocation.GetLocationAsync(req, ct);  ← MAUI verifica │
│                                                                 │
│  Acá es donde realmente se EJECUTA y se CANCELA la operación   │
└─────────────────────────────────────────────────────────────────┘
```

### 8.2 Regla de diseño para las interfaces

Las interfaces de repositorio y servicio **siempre** deben incluir `CancellationToken` en sus métodos async:

```csharp
// ✅ Interfaz bien diseñada
public interface IInfraccionRepository
{
    Task<Infraccion?> ObtenerPorIdAsync(int id, CancellationToken ct);
    Task GuardarAsync(Infraccion infraccion, CancellationToken ct);
    Task<List<Infraccion>> ObtenerPendientesAsync(CancellationToken ct);
}

// ❌ Interfaz mal diseñada — sin token
public interface IInfraccionRepository
{
    Task<Infraccion?> ObtenerPorIdAsync(int id);
    Task GuardarAsync(Infraccion infraccion);
}
```

Si la interfaz no tiene el parámetro `CancellationToken`, las implementaciones no pueden honrar la cancelación aunque quieran.

### 8.3 Resumen visual

```
QUIÉN         │ CREA CTS │ RECIBE CT │ CANCELA │ EJECUTA
──────────────┼──────────┼───────────┼─────────┼──────────
Presentación  │    ✅    │           │   ✅     │
Aplicación    │          │    ✅     │          │
Dominio       │          │           │          │
Infraestructura│         │    ✅     │          │    ✅
```

- **Presentación** crea y cancela (tiene contexto del usuario/sistema)
- **Aplicación** propaga (no le corresponde decidir cuándo cancelar)
- **Dominio** no participa (es lógica pura, sincrónica en general)
- **Infraestructura** ejecuta y honra la cancelación (EF Core, HttpClient, GPS)

---

> **Resumen final**: `CancellationToken` es el mecanismo estándar de .NET para cancelar operaciones asíncronas de forma limpia. Lo crea el **punto final** (la capa que sabe cuándo cancelar), viaja como parámetro a través de las capas intermedias, y es honrado por las operaciones de infraestructura. Siempre usá el patrón defensivo (`_cts?.Cancel(); _cts?.Dispose();` antes de crear uno nuevo) y siempre cancelá en `OnDisappearing` o equivalente.
