# Permisos de GPS / Geolocalización en .NET MAUI — Guía Completa

## Índice

1. [Introducción](#1-introducción)
2. [Diferencias entre Android e iOS](#2-diferencias-entre-android-e-ios)
3. [Configuración de Manifiestos](#3-configuración-de-manifiestos)
4. [Lógica de Permisos en C#](#4-lógica-de-permisos-en-c)
5. [Manejo de Denegación de Permisos](#5-manejo-de-denegación-de-permisos)
6. [Integración con Geolocation de MAUI](#6-integración-con-geolocation-de-maui)
7. [Ejemplo Completo: Página con GPS](#7-ejemplo-completo-página-con-gps)
8. [Troubleshooting](#8-troubleshooting)

---

## 1. Introducción

En .NET MAUI, el acceso a la ubicación del dispositivo (GPS) requiere permisos en tiempo de ejecución tanto en Android como en iOS. A diferencia de otros permisos, la ubicación tiene **niveles de acceso** que el usuario puede elegir.

### ¿Qué niveles de ubicación existen?

| Nivel | Android | iOS | ¿Cuándo usarlo? |
|---|---|---|---|
| **Ubicación aproximada** | `ACCESS_COARSE_LOCATION` (API 23+) | N/A | Clima, noticias locales |
| **Ubicación precisa** | `ACCESS_FINE_LOCATION` (API 23+) | `WhenInUse` | Navegación, delivery, registro de posición |
| **En segundo plano** | `ACCESS_BACKGROUND_LOCATION` (API 29+) | `Always` | Tracking, geofencing |

> **En esta guía nos enfocamos en ubicación precisa mientras la app está en uso** (`ACCESS_FINE_LOCATION` / `WhenInUse`), que es el caso más común.

### Versiones de Android y su impacto en permisos de ubicación

| API Level | Versión Android | Cambio relevante |
|-----------|----------------|-------------------|
| 23 (M) | 6.0 | Runtime permissions obligatorios para ubicación |
| 29 (Q) | 10 | `ACCESS_BACKGROUND_LOCATION` separado (antes era implícito) |
| 31 (S) | 12 | El usuario puede elegir "ubicación aproximada" vs "precisa" |

---

## 2. Diferencias entre Android e iOS

### Android

```
┌────────────────────────────────────────────────────────┐
│                  PRIMERA SOLICITUD                      │
│                                                        │
│   "¿Permitir que [App] acceda a la                     │
│    ubicación de este dispositivo?"                      │
│                                                        │
│    ☑ Precisa    ☐ Aproximada  (Android 12+)            │
│                                                        │
│     [Mientras se usa la app]  [Solo esta vez]           │
│                 [No permitir]                           │
└────────────────────────────────────────────────────────┘
         │                              │
    ✅ Granted                     ❌ Denied
         │                              │
    Usar GPS                  Segunda solicitud posible
                                        │
                              ┌─────────┴──────────┐
                              │ Si marca "No volver │
                              │ a preguntar" →      │
                              │ Denegado permanente  │
                              └────────────────────┘
```

- Se puede pedir el permiso **varias veces** (salvo que marque "No volver a preguntar").
- A partir de **Android 12**, el usuario puede elegir entre ubicación **precisa** o **aproximada**.
- Se puede detectar la denegación permanente con `ShouldShowRationale`.
- Se puede redirigir al usuario a Configuración del sistema.

### iOS

```
┌────────────────────────────────────────────────────────┐
│               PRIMERA Y ÚNICA SOLICITUD                 │
│                                                        │
│   "[App] quiere usar tu ubicación"                      │
│   "La app necesita tu ubicación para registrar          │
│    la posición de las infracciones."                    │
│                                                        │
│   [Permitir una vez] [Al usar la app] [No permitir]     │
└────────────────────────────────────────────────────────┘
         │                              │
    ✅ Granted                     ❌ Denied
         │                              │
    Usar GPS                 NO se puede volver a pedir
                              → Redirigir a Configuración
```

- Solo se puede pedir **una vez**. Después el sistema recuerda la decisión.
- iOS ofrece 3 opciones: **una vez**, **al usar la app**, **no permitir**.
- Si el usuario deniega, la única opción es enviarlo a **Configuración > Privacidad > Localización**.
- El texto de `NSLocationWhenInUseUsageDescription` en `Info.plist` es lo que ve el usuario. Si falta, la app **crashea**.

---

## 3. Configuración de Manifiestos

### 3.1 Android — `Platforms/Android/AndroidManifest.xml`

```xml
<?xml version="1.0" encoding="utf-8"?>
<manifest xmlns:android="http://schemas.android.com/apk/res/android">

    <!-- ══════════════════════════════════════════════ -->
    <!-- PERMISOS DE UBICACIÓN                         -->
    <!-- ══════════════════════════════════════════════ -->

    <!--
        ACCESS_FINE_LOCATION: ubicación precisa (GPS).
        Necesario para obtener coordenadas exactas.
    -->
    <uses-permission android:name="android.permission.ACCESS_FINE_LOCATION" />

    <!--
        ACCESS_COARSE_LOCATION: ubicación aproximada (red/wifi).
        Requerido como acompañante de FINE_LOCATION a partir de API 31.
        Si solo pedís FINE sin COARSE, en Android 12+ el sistema
        puede mostrar la opción de "ubicación aproximada" igualmente.
    -->
    <uses-permission android:name="android.permission.ACCESS_COARSE_LOCATION" />

    <!--
        ACCESS_BACKGROUND_LOCATION: ubicación en segundo plano.
        Solo necesario si tu app obtiene ubicación cuando NO está
        visible (ej: tracking, geofencing).
        En Android 10+ se pide SEPARADO del permiso de primer plano.
        Omitilo si solo usás GPS mientras la app está abierta.
    -->
    <!-- <uses-permission android:name="android.permission.ACCESS_BACKGROUND_LOCATION" /> -->

    <!--
        uses-feature con required="false" permite que la app se instale
        en dispositivos SIN GPS (tablets wifi-only).
    -->
    <uses-feature android:name="android.hardware.location.gps" android:required="false" />

    <!-- ══════════════════════════════════════════════ -->
    <!-- APPLICATION                                    -->
    <!-- ══════════════════════════════════════════════ -->

    <application
        android:allowBackup="true"
        android:label="@string/app_name">
    </application>

</manifest>
```

#### ¿Por qué `ACCESS_COARSE_LOCATION` si quiero precisión?

A partir de **Android 12 (API 31)**, el sistema permite al usuario elegir entre "Precisa" y "Aproximada". Si declarás ambos permisos (`FINE` + `COARSE`), el diálogo muestra las dos opciones. Si solo declarás `FINE`, el sistema puede degradar a aproximada sin que lo sepas.

### 3.2 iOS — `Platforms/iOS/Info.plist`

Agregá estas claves dentro del tag `<dict>`:

```xml
<!-- ══════════════════════════════════════════════ -->
<!-- UBICACIÓN                                     -->
<!-- ══════════════════════════════════════════════ -->

<!--
    OBLIGATORIO para ubicación en primer plano.
    Sin esta clave la app crashea al pedir ubicación.
    El texto es lo que ve el usuario en el diálogo de permisos.
    Tip: Sé claro y específico sobre POR QUÉ necesitás la ubicación.
    Apple puede rechazar la app si el texto es genérico.
-->
<key>NSLocationWhenInUseUsageDescription</key>
<string>La app necesita tu ubicación para registrar la posición de las infracciones de tránsito.</string>

<!--
    Solo necesario si pedís ubicación en SEGUNDO PLANO (Always).
    Si solo usás WhenInUse, no lo agregues.
-->
<!-- <key>NSLocationAlwaysAndWhenInUseUsageDescription</key>
<string>La app necesita acceso permanente a tu ubicación para tracking en segundo plano.</string> -->
```

#### Ejemplo de diálogo en iOS generado a partir de Info.plist

```
┌─────────────────────────────────────────┐
│  "MiApp" quiere usar tu ubicación       │
│                                         │
│  La app necesita tu ubicación para      │
│  registrar la posición de las           │
│  infracciones de tránsito.              │
│                                         │
│  [Permitir una vez]                     │
│  [Permitir mientras se usa la app]      │
│  [No permitir]                          │
└─────────────────────────────────────────┘
         ↑
    Este texto viene de NSLocationWhenInUseUsageDescription
```

### 3.3 Manifiestos mínimos

#### Solo ubicación mientras la app está abierta (caso más común)

**Android:**
```xml
<?xml version="1.0" encoding="utf-8"?>
<manifest xmlns:android="http://schemas.android.com/apk/res/android">
    <uses-permission android:name="android.permission.ACCESS_FINE_LOCATION" />
    <uses-permission android:name="android.permission.ACCESS_COARSE_LOCATION" />
    <uses-feature android:name="android.hardware.location.gps" android:required="false" />

    <application android:allowBackup="true" android:label="@string/app_name" />
</manifest>
```

**iOS:**
```xml
<key>NSLocationWhenInUseUsageDescription</key>
<string>La app necesita tu ubicación para registrar la posición de las infracciones.</string>
```

> **Nota**: A diferencia de la cámara con `MediaPicker`, la geolocalización NO necesita `FileProvider` ni `file_paths.xml` porque no hay intercambio de archivos entre apps.

---

## 4. Lógica de Permisos en C#

### 4.1 Flujo recomendado (multiplataforma)

La API de permisos de MAUI (`Permissions.LocationWhenInUse`) funciona cross-platform y es suficiente para GPS. **No necesitás código Android-específico con `ActivityCompat`** como en el caso de cámara con `MediaPicker`.

Sin embargo, hay que tener en cuenta las mismas trampas de Android:
- `CheckStatusAsync` devuelve `Denied` (no `Unknown`) cuando nunca se pidió
- `ShouldShowRationale` solo es confiable **después** de llamar a `RequestAsync`

```csharp
private async Task<bool> EvaluarPermisosUbicacionAsync()
{
    // Paso 1: Verificar estado actual
    var status = await Permissions.CheckStatusAsync<Permissions.LocationWhenInUse>();

    if (status == PermissionStatus.Granted)
        return true;

    // Paso 2: Pedir permiso
    // NOTA: En Android, CheckStatusAsync devuelve Denied (no Unknown)
    // cuando el permiso nunca fue solicitado. Por eso siempre llamamos
    // a RequestAsync si no está Granted.
    status = await Permissions.RequestAsync<Permissions.LocationWhenInUse>();

    if (status == PermissionStatus.Granted)
        return true;

    // Paso 3: Restringido (control parental / MDM en iOS)
    if (status == PermissionStatus.Restricted)
    {
        await MostrarAlertaAsync("Ubicación restringida",
            "El acceso a la ubicación está restringido por una política " +
            "del dispositivo. Consultá con el administrador.");
        return false;
    }

    // Paso 4: Denegado — ¿se puede volver a pedir?
    // ShouldShowRationale es confiable ahora porque ya llamamos a RequestAsync
    bool puedeReintentar = false;

#if ANDROID
    puedeReintentar = Permissions.ShouldShowRationale<Permissions.LocationWhenInUse>();
#endif

    if (puedeReintentar)
    {
        await MostrarAlertaAsync("Permiso de ubicación necesario",
            "Para registrar la posición necesitamos acceso al GPS. " +
            "Podés intentar conceder el permiso.");
    }
    else
    {
        // Denegación permanente (Android: "No volver a preguntar" / iOS: siempre)
        bool irAConfig = await Application.Current!.MainPage!.DisplayAlert(
            "Ubicación denegada",
            "Para usar esta función, habilitá la ubicación desde " +
            "los ajustes de la aplicación.",
            "Ir a Configuración", "Cancelar");

        if (irAConfig)
            AppInfo.ShowSettingsUI();
    }

    return false;
}

private async Task MostrarAlertaAsync(string titulo, string mensaje)
{
    await Application.Current!.MainPage!.DisplayAlert(titulo, mensaje, "OK");
}
```

### 4.2 Diferencia con el permiso de cámara

| Aspecto | Cámara | GPS |
|---|---|---|
| Permiso MAUI | `Permissions.Camera` | `Permissions.LocationWhenInUse` |
| `CheckStatusAsync` devuelve en Android (nunca pedido) | `Denied` | `Denied` |
| `ShouldShowRationale` confiable | Después de `RequestAsync` | Después de `RequestAsync` |
| Código nativo necesario | Solo si usás `MediaPicker` con `ActivityCompat` | No, la API de MAUI es suficiente |

### 4.3 Verificar si el GPS está activado

Un problema diferente al permiso: el usuario puede conceder el permiso pero tener el **GPS desactivado** en el dispositivo.

```csharp
private async Task<bool> VerificarGpsActivadoAsync()
{
    try
    {
        // Intentar obtener la última ubicación conocida (no activa el GPS)
        var location = await Geolocation.Default.GetLastKnownLocationAsync();

        // Si devuelve null, puede ser que el GPS esté apagado o nunca se usó.
        // Intentar obtener ubicación actual con timeout corto:
        var request = new GeolocationRequest(GeolocationAccuracy.Medium, TimeSpan.FromSeconds(5));
        location = await Geolocation.Default.GetLocationAsync(request);

        if (location == null)
        {
            await MostrarAlertaAsync("GPS desactivado",
                "El GPS está apagado. Activalo desde los ajustes del dispositivo.");
            return false;
        }

        return true;
    }
    catch (FeatureNotEnabledException)
    {
        // El GPS está desactivado en el dispositivo
        await MostrarAlertaAsync("GPS desactivado",
            "El GPS está apagado. Activalo desde los ajustes del dispositivo.");
        return false;
    }
    catch (FeatureNotSupportedException)
    {
        await MostrarAlertaAsync("GPS no disponible",
            "Este dispositivo no tiene GPS.");
        return false;
    }
}
```

---

## 5. Manejo de Denegación de Permisos

### 5.1 Diagrama de flujo completo

```
                    ┌───────────────────┐
                    │ CheckStatusAsync  │
                    └────────┬──────────┘
                             │
                    ┌────────▼─────────┐
                    │  ¿Granted?       │
                    └────────┬─────────┘
                        Sí ──┤── No
                       │     │
                  ✅ Usar   ┌▼─────────────────┐
                   GPS      │  RequestAsync     │
                            └────────┬──────────┘
                                     │
                            ┌────────▼─────────┐
                            │  ¿Granted?       │
                            └────────┬─────────┘
                               Sí ──┤── No
                              │     │
                         ✅ Usar   ┌▼───────────────────────┐
                          GPS      │ ¿Restricted?           │
                                   └────────┬───────────────┘
                                      Sí ──┤── No
                                     │     │
                    ┌────────────────▼┐   ┌▼───────────────────────┐
                    │ "Acceso         │   │ ¿ShouldShowRationale?  │
                    │  restringido"   │   │ (solo Android)         │
                    └─────────────────┘   └────────┬───────────────┘
                                             Sí ──┤── No (permanente)
                                            │     │
                              ┌─────────────▼┐   ┌▼────────────────────┐
                              │ Mostrar msg: │   │ "Ir a Configuración" │
                              │ "Reintentar" │   │ AppInfo.ShowSettings │
                              └──────────────┘   └─────────────────────┘
```

### 5.2 Tabla resumen: Flujo de denegación por plataforma

| Escenario | Android | iOS |
|---|---|---|
| **Primera vez** | Muestra diálogo nativo con opciones Precisa/Aproximada (API 31+) | Muestra diálogo: una vez / al usar / no permitir |
| **Denegado una vez** | Vuelve a pedir (`ShouldShowRationale` = true) | No puede volver a pedir |
| **"No volver a preguntar"** | `ShouldShowRationale` = false → ir a Configuración | N/A (siempre permanente) |
| **Denegado permanente** | `AppInfo.ShowSettingsUI()` | `AppInfo.ShowSettingsUI()` |
| **Restringido (parental/MDM)** | N/A | `PermissionStatus.Restricted` → informar |
| **GPS apagado (permiso OK)** | `FeatureNotEnabledException` | `FeatureNotEnabledException` |

---

## 6. Integración con Geolocation de MAUI

MAUI incluye `Geolocation` que abstrae el acceso al GPS. **Igual necesitás los permisos configurados** en manifiestos y solicitados en runtime.

### 6.1 Obtener ubicación actual

```csharp
private async Task<Location?> ObtenerUbicacionAsync()
{
    try
    {
        // Verificar permisos primero
        var permisoConcedido = await EvaluarPermisosUbicacionAsync();
        if (!permisoConcedido) return null;

        // Configurar la solicitud
        var request = new GeolocationRequest(
            GeolocationAccuracy.Best,       // Máxima precisión (GPS)
            TimeSpan.FromSeconds(10));       // Timeout

        // Obtener ubicación
        var location = await Geolocation.Default.GetLocationAsync(request);

        if (location != null)
        {
            Console.WriteLine($"[APP] Lat: {location.Latitude}, " +
                              $"Lng: {location.Longitude}, " +
                              $"Alt: {location.Altitude}m, " +
                              $"Precisión: {location.Accuracy}m");
        }

        return location;
    }
    catch (FeatureNotSupportedException)
    {
        await MostrarAlertaAsync("GPS no disponible",
            "Este dispositivo no tiene GPS.");
        return null;
    }
    catch (FeatureNotEnabledException)
    {
        await MostrarAlertaAsync("GPS desactivado",
            "Activá el GPS desde los ajustes del dispositivo.");
        return null;
    }
    catch (PermissionException)
    {
        await MostrarAlertaAsync("Sin permisos",
            "No hay permisos de ubicación.");
        return null;
    }
    catch (Exception ex)
    {
        Console.WriteLine($"[APP] Error GPS: {ex.Message}");
        await MostrarAlertaAsync("Error", $"Error al obtener ubicación: {ex.Message}");
        return null;
    }
}
```

### 6.2 Última ubicación conocida (rápido, sin activar GPS)

```csharp
private async Task<Location?> ObtenerUltimaUbicacionAsync()
{
    try
    {
        // No activa el GPS — devuelve la última posición cacheada.
        // Puede ser null si el GPS nunca se usó.
        // Puede ser vieja si el dispositivo se movió.
        var location = await Geolocation.Default.GetLastKnownLocationAsync();

        if (location != null)
        {
            // Verificar antigüedad
            var antiguedad = DateTime.UtcNow - location.Timestamp.UtcDateTime;
            if (antiguedad.TotalMinutes > 5)
            {
                Console.WriteLine($"[APP] Ubicación cacheada tiene {antiguedad.TotalMinutes:F0} minutos de antigüedad");
            }
        }

        return location;
    }
    catch (Exception ex)
    {
        Console.WriteLine($"[APP] Error última ubicación: {ex.Message}");
        return null;
    }
}
```

### 6.3 Niveles de precisión

```csharp
// GeolocationAccuracy determina qué sensor se usa:

GeolocationAccuracy.Lowest      // ~3000m — Cell ID
GeolocationAccuracy.Low         // ~1000m — Cell ID / WiFi
GeolocationAccuracy.Medium      // ~100m  — WiFi / Cell triangulation
GeolocationAccuracy.High        // ~10m   — GPS
GeolocationAccuracy.Best        // ~1m    — GPS + GLONASS + Galileo (más batería)
```

| Precisión | Fuente | Consumo batería | Caso de uso |
|---|---|---|---|
| `Lowest` / `Low` | Red celular | Bajo | Clima, contenido regional |
| `Medium` | WiFi + red | Medio | Búsqueda de comercios cercanos |
| `High` / `Best` | GPS | Alto | Navegación, registro de posición exacta |

---

## 7. Ejemplo Completo: Página con GPS

### 7.1 XAML — `GpsPage.xaml`

```xml
<?xml version="1.0" encoding="utf-8" ?>
<ContentPage xmlns="http://schemas.microsoft.com/dotnet/2021/maui"
             xmlns:x="http://schemas.microsoft.com/winfx/2009/xaml"
             x:Class="MiApp.Views.GpsPage"
             Title="Ubicación GPS">

    <ScrollView Padding="20">
        <VerticalStackLayout Spacing="15">

            <!-- Estado de permisos -->
            <Label x:Name="GpsStatusLabel"
                   Text="GPS: verificando..."
                   FontSize="14"
                   TextColor="Gray" />

            <!-- Datos de ubicación -->
            <Frame BackgroundColor="#F5F5F5"
                   CornerRadius="10"
                   Padding="15">
                <VerticalStackLayout Spacing="8">
                    <Label Text="Coordenadas"
                           FontSize="16"
                           FontAttributes="Bold" />
                    <Label x:Name="LblLatitud"
                           Text="Latitud: —"
                           FontSize="14" />
                    <Label x:Name="LblLongitud"
                           Text="Longitud: —"
                           FontSize="14" />
                    <Label x:Name="LblAltitud"
                           Text="Altitud: —"
                           FontSize="14" />
                    <Label x:Name="LblPrecision"
                           Text="Precisión: —"
                           FontSize="14" />
                    <Label x:Name="LblTimestamp"
                           Text="Última actualización: —"
                           FontSize="12"
                           TextColor="Gray" />
                </VerticalStackLayout>
            </Frame>

            <!-- Botones -->
            <Button x:Name="BtnObtenerUbicacion"
                    Text="Obtener Ubicación"
                    BackgroundColor="#2196F3"
                    TextColor="White"
                    Clicked="OnObtenerUbicacion_Clicked" />

            <Button x:Name="BtnUltimaUbicacion"
                    Text="Última ubicación conocida"
                    BackgroundColor="#4CAF50"
                    TextColor="White"
                    Clicked="OnUltimaUbicacion_Clicked" />

            <!-- Botón de reintentar (oculto por defecto) -->
            <Button x:Name="BtnReintentarGps"
                    Text="Reintentar permisos"
                    BackgroundColor="#FF9800"
                    TextColor="White"
                    IsVisible="False"
                    Clicked="OnReintentarGps_Clicked" />

            <!-- Info adicional -->
            <Label x:Name="LblInfo"
                   Text=""
                   FontSize="12"
                   TextColor="Gray" />

        </VerticalStackLayout>
    </ScrollView>
</ContentPage>
```

### 7.2 Code-behind — `GpsPage.xaml.cs`

```csharp
namespace MiApp.Views;

public partial class GpsPage : ContentPage
{
    public GpsPage()
    {
        InitializeComponent();
    }

    protected override async void OnAppearing()
    {
        base.OnAppearing();
        await VerificarEstadoGpsAsync();
    }

    // ══════════════════════════════════════════════
    // VERIFICACIÓN INICIAL
    // ══════════════════════════════════════════════

    private async Task VerificarEstadoGpsAsync()
    {
        var status = await Permissions.CheckStatusAsync<Permissions.LocationWhenInUse>();

        if (status == PermissionStatus.Granted)
        {
            GpsStatusLabel.Text = "GPS: permiso concedido";
            GpsStatusLabel.TextColor = Colors.Green;
            return;
        }

        GpsStatusLabel.Text = "GPS: sin permisos";
        GpsStatusLabel.TextColor = Colors.Red;
    }

    // ══════════════════════════════════════════════
    // EVENTOS DE BOTONES
    // ══════════════════════════════════════════════

    private async void OnObtenerUbicacion_Clicked(object sender, EventArgs e)
    {
        BtnObtenerUbicacion.IsEnabled = false;
        GpsStatusLabel.Text = "GPS: obteniendo ubicación...";
        GpsStatusLabel.TextColor = Colors.Gray;

        try
        {
            var permiso = await EvaluarPermisosUbicacionAsync();
            if (!permiso) return;

            var request = new GeolocationRequest(
                GeolocationAccuracy.Best,
                TimeSpan.FromSeconds(10));

            var location = await Geolocation.Default.GetLocationAsync(request);

            if (location != null)
            {
                MostrarUbicacion(location);
                GpsStatusLabel.Text = "GPS: ubicación obtenida";
                GpsStatusLabel.TextColor = Colors.Green;
            }
            else
            {
                GpsStatusLabel.Text = "GPS: no se pudo obtener ubicación";
                GpsStatusLabel.TextColor = Colors.Orange;
                LblInfo.Text = "Verificá que el GPS esté activado.";
            }
        }
        catch (FeatureNotEnabledException)
        {
            GpsStatusLabel.Text = "GPS: desactivado";
            GpsStatusLabel.TextColor = Colors.Red;
            LblInfo.Text = "Activá el GPS desde los ajustes del dispositivo.";
        }
        catch (FeatureNotSupportedException)
        {
            GpsStatusLabel.Text = "GPS: no disponible";
            GpsStatusLabel.TextColor = Colors.Red;
            LblInfo.Text = "Este dispositivo no tiene GPS.";
        }
        catch (Exception ex)
        {
            GpsStatusLabel.Text = "GPS: error";
            GpsStatusLabel.TextColor = Colors.Red;
            LblInfo.Text = $"Error: {ex.Message}";
        }
        finally
        {
            BtnObtenerUbicacion.IsEnabled = true;
        }
    }

    private async void OnUltimaUbicacion_Clicked(object sender, EventArgs e)
    {
        try
        {
            var permiso = await EvaluarPermisosUbicacionAsync();
            if (!permiso) return;

            var location = await Geolocation.Default.GetLastKnownLocationAsync();

            if (location != null)
            {
                MostrarUbicacion(location);

                var antiguedad = DateTime.UtcNow - location.Timestamp.UtcDateTime;
                LblInfo.Text = $"Ubicación cacheada ({antiguedad.TotalMinutes:F0} minutos de antigüedad)";
            }
            else
            {
                LblInfo.Text = "No hay ubicación cacheada. Usá 'Obtener Ubicación'.";
            }
        }
        catch (Exception ex)
        {
            LblInfo.Text = $"Error: {ex.Message}";
        }
    }

    private async void OnReintentarGps_Clicked(object sender, EventArgs e)
    {
        BtnReintentarGps.IsVisible = false;
        await VerificarEstadoGpsAsync();
    }

    // ══════════════════════════════════════════════
    // PERMISOS
    // ══════════════════════════════════════════════

    private async Task<bool> EvaluarPermisosUbicacionAsync()
    {
        var status = await Permissions.CheckStatusAsync<Permissions.LocationWhenInUse>();

        if (status == PermissionStatus.Granted)
            return true;

        // En Android, CheckStatusAsync devuelve Denied (no Unknown) cuando
        // el permiso nunca fue solicitado. Siempre llamar a RequestAsync
        // si no está Granted.
        status = await Permissions.RequestAsync<Permissions.LocationWhenInUse>();

        if (status == PermissionStatus.Granted)
        {
            GpsStatusLabel.Text = "GPS: permiso concedido";
            GpsStatusLabel.TextColor = Colors.Green;
            return true;
        }

        if (status == PermissionStatus.Restricted)
        {
            GpsStatusLabel.Text = "GPS: restringido";
            GpsStatusLabel.TextColor = Colors.Red;
            await DisplayAlert("Ubicación restringida",
                "El acceso a la ubicación está restringido por una política " +
                "del dispositivo.", "Entendido");
            return false;
        }

        // Denegado — ¿se puede volver a pedir?
        // ShouldShowRationale es confiable DESPUÉS de RequestAsync
        bool puedeReintentar = false;

#if ANDROID
        puedeReintentar = Permissions.ShouldShowRationale<Permissions.LocationWhenInUse>();
#endif

        if (puedeReintentar)
        {
            await DisplayAlert("Permiso de ubicación necesario",
                "Para registrar la posición necesitamos acceso al GPS. " +
                "Podés intentar conceder el permiso.", "OK");
            BtnReintentarGps.IsVisible = true;
        }
        else
        {
            bool irAConfig = await DisplayAlert(
                "Ubicación denegada",
                "Denegaste el acceso a la ubicación. " +
                "Para usar esta función, habilitalo desde los ajustes.",
                "Ir a Configuración", "Cancelar");

            if (irAConfig)
                AppInfo.ShowSettingsUI();

            BtnReintentarGps.IsVisible = true;
        }

        GpsStatusLabel.Text = "GPS: permisos denegados";
        GpsStatusLabel.TextColor = Colors.Red;
        return false;
    }

    // ══════════════════════════════════════════════
    // HELPERS
    // ══════════════════════════════════════════════

    private void MostrarUbicacion(Location location)
    {
        LblLatitud.Text = $"Latitud: {location.Latitude:F6}";
        LblLongitud.Text = $"Longitud: {location.Longitude:F6}";
        LblAltitud.Text = location.Altitude.HasValue
            ? $"Altitud: {location.Altitude:F1} m"
            : "Altitud: no disponible";
        LblPrecision.Text = location.Accuracy.HasValue
            ? $"Precisión: ±{location.Accuracy:F0} m"
            : "Precisión: no disponible";
        LblTimestamp.Text = $"Última actualización: {location.Timestamp.LocalDateTime:HH:mm:ss}";
    }
}
```

---

## 8. Troubleshooting

### 8.1 Errores comunes y soluciones

| Error | Causa | Solución |
|-------|-------|----------|
| `Java.Lang.SecurityException` al pedir ubicación | Falta permiso en `AndroidManifest.xml` | Agregar `<uses-permission android:name="android.permission.ACCESS_FINE_LOCATION" />` |
| App crashea en iOS al pedir ubicación | Falta `NSLocationWhenInUseUsageDescription` en `Info.plist` | Agregar la clave con texto descriptivo |
| `FeatureNotEnabledException` | GPS desactivado en el dispositivo | Indicar al usuario que active el GPS desde ajustes |
| `FeatureNotSupportedException` | Dispositivo sin GPS (tablet wifi-only, emulador) | Verificar con `try-catch` y mostrar mensaje |
| Ubicación devuelve `null` | GPS activado pero sin señal (interior de edificio) | Reintentar, mover el dispositivo, usar `GeolocationAccuracy.Medium` |
| Permisos se muestran como denegados en primera ejecución (Android) | `CheckStatusAsync` devuelve `Denied` en vez de `Unknown` | Siempre llamar a `RequestAsync` si no está `Granted` |
| `ShouldShowRationale` devuelve `false` pero el permiso nunca se pidió | Comportamiento esperado de Android | Llamar a `ShouldShowRationale` después de `RequestAsync` |
| Precisión muy baja (>100m) | Usando `GeolocationAccuracy.Low` o señal GPS débil | Usar `GeolocationAccuracy.Best` y esperar más tiempo |
| `GetLocationAsync` tarda mucho | Timeout corto + GPS frío ("cold start") | Aumentar timeout a 15-30s. El primer fix GPS después de encender puede demorar |

### 8.2 Checklist de verificación

```
✅ AndroidManifest.xml
   ├── android.permission.ACCESS_FINE_LOCATION
   ├── android.permission.ACCESS_COARSE_LOCATION
   ├── uses-feature location.gps required="false"
   └── NO necesita FileProvider ni file_paths.xml

✅ Info.plist (iOS)
   └── NSLocationWhenInUseUsageDescription (texto descriptivo)

✅ Código C#
   ├── Permissions.LocationWhenInUse (no LocationAlways salvo segundo plano)
   ├── Siempre llamar RequestAsync si no está Granted (no filtrar por Unknown)
   ├── ShouldShowRationale DESPUÉS de RequestAsync
   ├── Manejo de FeatureNotEnabledException (GPS apagado)
   ├── Manejo de FeatureNotSupportedException (sin hardware GPS)
   ├── Timeout adecuado en GeolocationRequest (10-30 segundos)
   ├── Redirección a Configuración en denegación permanente
   └── try-catch en toda la cadena
```

### 8.3 Probar ubicación en emulador

**Android (Android Studio AVD):**
```bash
# Establecer ubicación simulada por línea de comando
adb emu geo fix -34.5986 -58.4208   # Ejemplo: Buenos Aires

# También se puede desde Extended Controls > Location en el emulador
```

**iOS (Xcode Simulator):**
- Menú **Features > Location** > elegir una ubicación predefinida o personalizada.
- O usar **Debug > Simulate Location** en Xcode con un archivo `.gpx`.

---

> **Nota sobre privacidad**: La ubicación es un dato sensible. Guardá coordenadas solo si es necesario para la funcionalidad. Si las transmitís a un servidor, usá HTTPS. Informá al usuario claramente por qué necesitás su ubicación — esto es un requisito de Apple para aprobar la app y una buena práctica de UX.
