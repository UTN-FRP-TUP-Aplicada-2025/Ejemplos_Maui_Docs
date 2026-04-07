# Permisos de Cámara en .NET MAUI — Guía Completa

## Índice

1. [Introducción](#1-introducción)
2. [Diferencias entre Android e iOS](#2-diferencias-entre-android-e-ios)
3. [Configuración de Manifiestos](#3-configuración-de-manifiestos)
4. [Lógica de Permisos en C#](#4-lógica-de-permisos-en-c)
5. [Manejo de Denegación de Permisos](#5-manejo-de-denegación-de-permisos)
6. [Integración con MediaPicker de MAUI](#6-integración-con-mediapicker-de-maui)
7. [Ejemplo Completo: Página con Cámara](#7-ejemplo-completo-página-con-cámara)
8. [Troubleshooting](#8-troubleshooting)

---

## 1. Introducción

En .NET MAUI, el acceso a la cámara requiere permisos en tiempo de ejecución (runtime permissions) tanto en Android como en iOS. A diferencia de las versiones antiguas de Android donde los permisos se otorgaban al instalar la app, a partir de **Android 6.0 (API 23)** el sistema exige que la app solicite permisos al usuario en el momento que los necesita.

### ¿Por qué no alcanza con declarar el permiso en el manifiesto?

- **Android**: El manifiesto declara *qué* permisos puede pedir la app. Pero el usuario debe **aceptar explícitamente** en tiempo de ejecución.
- **iOS**: Los permisos se declaran en `Info.plist` con una descripción obligatoria. El sistema muestra un diálogo nativo **una sola vez**. Si el usuario lo deniega, no se puede volver a pedir desde la app.

### Versiones de Android y su impacto en permisos

| API Level | Versión Android | Cambio relevante |
|-----------|----------------|-------------------|
| 23 (M) | 6.0 | Runtime permissions obligatorios |
| 29 (Q) | 10 | Scoped Storage, se limita acceso a archivos |
| 30 (R) | 11 | `WRITE_EXTERNAL_STORAGE` ignorado |
| 33 (Tiramisu) | 13 | `READ_MEDIA_IMAGES` reemplaza `READ_EXTERNAL_STORAGE` |

---

## 2. Diferencias entre Android e iOS

### Android

```
┌────────────────────────────────────────────────────────┐
│                  PRIMERA SOLICITUD                      │
│                                                        │
│   Sistema muestra diálogo nativo: "¿Permitir que       │
│   [App] tome fotos y grabe video?"                     │
│                                                        │
│        [Mientras se usa la app]  [Solo esta vez]        │
│                    [No permitir]                        │
└────────────────────────────────────────────────────────┘
         │                              │
    ✅ Granted                     ❌ Denied
         │                              │
    Usar cámara               Segunda solicitud posible
                                        │
                              ┌─────────┴──────────┐
                              │ Si marca "No volver │
                              │ a preguntar" →      │
                              │ Denegado permanente  │
                              └────────────────────┘
```

- Se puede pedir el permiso **varias veces** (salvo que marque "No volver a preguntar").
- Se puede detectar la denegación permanente con `ShouldShowRequestPermissionRationale`.
- Se puede redirigir al usuario a Configuración del sistema.

### iOS

```
┌────────────────────────────────────────────────────────┐
│               PRIMERA Y ÚNICA SOLICITUD                 │
│                                                        │
│   "[App] quiere acceder a la cámara"                   │
│   "La app necesita acceso a la cámara para tomar       │
│    fotos de las infracciones."                         │
│                                                        │
│              [Permitir]  [No permitir]                  │
└────────────────────────────────────────────────────────┘
         │                              │
    ✅ Granted                     ❌ Denied
         │                              │
    Usar cámara              NO se puede volver a pedir
                              → Redirigir a Configuración
```

- Solo se puede pedir **una vez**. Después el sistema recuerda la decisión.
- Si el usuario deniega, la única opción es enviarlo a **Configuración > Privacidad > Cámara**.
- El texto de `NSCameraUsageDescription` en `Info.plist` es lo que ve el usuario. Si falta, la app **crashea**.

---

## 3. Configuración de Manifiestos

### 3.1 Android — `Platforms/Android/AndroidManifest.xml`

```xml
<?xml version="1.0" encoding="utf-8"?>
<manifest xmlns:android="http://schemas.android.com/apk/res/android">

    <!-- ══════════════════════════════════════════════ -->
    <!-- PERMISOS DE CÁMARA                            -->
    <!-- ══════════════════════════════════════════════ -->

    <!-- Permiso principal de cámara -->
    <uses-permission android:name="android.permission.CAMERA" />

    <!--
        uses-feature con required="false" permite que la app se instale
        en dispositivos SIN cámara (tablets viejas, emuladores).
        Si ponés required="true", Google Play filtra esos dispositivos.
    -->
    <uses-feature android:name="android.hardware.camera" android:required="false" />
    <uses-feature android:name="android.hardware.camera.autofocus" android:required="false" />

    <!-- ══════════════════════════════════════════════ -->
    <!-- PERMISOS DE ALMACENAMIENTO (para guardar fotos)-->
    <!-- ══════════════════════════════════════════════ -->

    <!--
        Android < 13 (API < 33): usar READ/WRITE_EXTERNAL_STORAGE.
        maxSdkVersion="32" evita que se pidan en Android 13+
        donde ya no son válidos.
    -->
    <uses-permission android:name="android.permission.READ_EXTERNAL_STORAGE"
                     android:maxSdkVersion="32" />
    <uses-permission android:name="android.permission.WRITE_EXTERNAL_STORAGE"
                     android:maxSdkVersion="32" />

    <!--
        Android 13+ (API 33): permiso granular por tipo de medio.
        Solo pedimos imágenes porque es para fotos.
    -->
    <uses-permission android:name="android.permission.READ_MEDIA_IMAGES" />

    <!-- ══════════════════════════════════════════════ -->
    <!-- APPLICATION                                    -->
    <!-- ══════════════════════════════════════════════ -->

    <application
        android:allowBackup="true"
        android:label="@string/app_name"
        android:requestLegacyExternalStorage="true">
        <!--
            requestLegacyExternalStorage="true":
            Necesario para Android 10 (API 29) para mantener acceso
            al almacenamiento tradicional. En API 30+ se ignora.
        -->

        <!--
            FileProvider: necesario para compartir archivos de fotos
            entre la app y la app de cámara del sistema.
            Sin esto, la cámara no puede escribir la foto.
        -->
        <provider
            android:name="androidx.core.content.FileProvider"
            android:authorities="${applicationId}.fileprovider"
            android:exported="false"
            android:grantUriPermissions="true">
            <meta-data
                android:name="android.support.FILE_PROVIDER_PATHS"
                android:resource="@xml/file_paths" />
        </provider>

    </application>
</manifest>
```

#### ¿Por qué `exported="false"` en el FileProvider?

El `FileProvider` solo debe ser accesible por tu propia app. Ponerlo en `true` sería una **vulnerabilidad de seguridad** — otras apps podrían leer archivos de tu app.

### 3.2 Android — `Platforms/Android/Resources/xml/file_paths.xml`

Creá este archivo si no existe:

```xml
<?xml version="1.0" encoding="utf-8"?>
<paths>
    <!--
        external-files-path: directorio privado de la app en almacenamiento externo.
        Equivale a: Android/data/[tu.package]/files/Pictures
        No necesita permisos de almacenamiento para acceder.
    -->
    <external-files-path name="camera_photos" path="Pictures" />

    <!--
        cache-path: directorio de caché de la app.
        Útil para fotos temporales que se suben a un servidor.
    -->
    <cache-path name="cache" path="." />

    <!--
        external-cache-path: caché externa (se borra si el usuario
        limpia caché desde Configuración).
    -->
    <external-cache-path name="external_cache" path="." />
</paths>
```

### 3.3 iOS — `Platforms/iOS/Info.plist`

Agregá estas claves dentro del tag `<dict>`:

```xml
<!-- ══════════════════════════════════════════════ -->
<!-- CÁMARA                                        -->
<!-- ══════════════════════════════════════════════ -->

<!--
    OBLIGATORIO. Sin esta clave la app crashea al intentar
    abrir la cámara. El texto es lo que ve el usuario en el
    diálogo de permisos.
    Tip: Sé claro y específico sobre POR QUÉ necesitás la cámara.
    Apple puede rechazar la app si el texto es genérico.
-->
<key>NSCameraUsageDescription</key>
<string>La app necesita acceso a la cámara para tomar fotos de las infracciones de tránsito.</string>

<!-- ══════════════════════════════════════════════ -->
<!-- GALERÍA DE FOTOS                              -->
<!-- ══════════════════════════════════════════════ -->

<!-- Leer fotos existentes de la galería -->
<key>NSPhotoLibraryUsageDescription</key>
<string>La app necesita acceso a la galería para seleccionar fotos existentes.</string>

<!-- Guardar fotos nuevas en la galería -->
<key>NSPhotoLibraryAddUsageDescription</key>
<string>La app necesita permiso para guardar las fotos capturadas en tu galería.</string>
```

#### Ejemplo de diálogo en iOS generado a partir de Info.plist

```
┌─────────────────────────────────────────┐
│  "MiApp" quiere acceder a la cámara     │
│                                         │
│  La app necesita acceso a la cámara     │
│  para tomar fotos de las infracciones   │
│  de tránsito.                           │
│                                         │
│          [Permitir]  [No permitir]      │
└─────────────────────────────────────────┘
         ↑
    Este texto viene de NSCameraUsageDescription
```

---

## 4. Lógica de Permisos en C#

### 4.1 Invocación principal (multiplataforma)

```csharp
private async Task InitializeCameraAsync()
{
#if ANDROID || IOS
    var granted = await RequestCameraPermissionsAsync();
    if (granted)
        await OpenCameraAsync();
#else
    // Windows, macOS: no requieren permisos runtime
    await OpenCameraAsync();
#endif
}
```

### 4.2 Permisos en Android

```csharp
#if ANDROID
using Android;
using AndroidX.Core.App;
using AndroidX.Core.Content;

private async Task<bool> RequestCameraPermissionsAsync()
{
    try
    {
        var activity = Platform.CurrentActivity
            ?? throw new InvalidOperationException("No hay Activity activa.");

        // ── Paso 1: Determinar qué permisos pedir según la versión ──

        string[] requiredPermissions;

        if (Android.OS.Build.VERSION.SdkInt >= Android.OS.BuildVersionCodes.Tiramisu)
        {
            // Android 13+ (API 33): CAMERA + READ_MEDIA_IMAGES
            requiredPermissions = new[]
            {
                Manifest.Permission.Camera,
                Manifest.Permission.ReadMediaImages
            };
        }
        else if (Android.OS.Build.VERSION.SdkInt >= Android.OS.BuildVersionCodes.M)
        {
            // Android 6-12 (API 23-32): CAMERA + almacenamiento clásico
            requiredPermissions = new[]
            {
                Manifest.Permission.Camera,
                Manifest.Permission.ReadExternalStorage,
                Manifest.Permission.WriteExternalStorage
            };
        }
        else
        {
            // Android < 6: permisos otorgados en instalación
            return true;
        }

        // ── Paso 2: Verificar si ya están concedidos ──

        bool allGranted = requiredPermissions.All(p =>
            ContextCompat.CheckSelfPermission(activity, p)
                == (int)Android.Content.PM.Permission.Granted);

        if (allGranted)
            return true;

        // ── Paso 3: Solicitar permisos ──
        // requestCode = 100 (usá un número único por tipo de permiso)

        ActivityCompat.RequestPermissions(activity, requiredPermissions, requestCode: 100);

        // Esperar a que el usuario responda.
        // NOTA: Task.Delay es una solución simple pero no ideal.
        // Ver sección 4.4 para la alternativa con TaskCompletionSource.
        await Task.Delay(5000);

        // ── Paso 4: Re-verificar después de la solicitud ──

        allGranted = requiredPermissions.All(p =>
            ContextCompat.CheckSelfPermission(activity, p)
                == (int)Android.Content.PM.Permission.Granted);

        if (allGranted)
            return true;

        // ── Paso 5: Manejar denegación ──

        await HandlePermissionDeniedAsync(activity, requiredPermissions);
        return false;
    }
    catch (Exception ex)
    {
        Console.WriteLine($"[APP] Camera permission error: {ex.Message}");
        ShowMessage($"Error al solicitar permisos de cámara: {ex.Message}");
        BtnReintentarCamara.IsVisible = true;
        return false;
    }
}
#endif
```

### 4.3 Permisos en iOS

```csharp
#if IOS
private async Task<bool> RequestCameraPermissionsAsync()
{
    try
    {
        // ── Paso 1: Verificar estado actual ──

        var status = await Permissions.CheckStatusAsync<Permissions.Camera>();

        // Si ya está concedido, salir rápido
        if (status == PermissionStatus.Granted)
            return true;

        // ── Paso 2: Si nunca se pidió, solicitarlo ──

        if (status == PermissionStatus.Unknown)
        {
            status = await Permissions.RequestAsync<Permissions.Camera>();

            if (status == PermissionStatus.Granted)
                return true;
        }

        // ── Paso 3: Denegado o Restringido ──

        /*
            PermissionStatus.Denied:
                El usuario tocó "No permitir". En iOS no se puede
                volver a mostrar el diálogo del sistema.

            PermissionStatus.Restricted:
                Control parental o MDM (Mobile Device Management)
                bloquea el acceso. El usuario NO puede cambiarlo.
        */

        if (status == PermissionStatus.Restricted)
        {
            await Application.Current!.MainPage!.DisplayAlert(
                "Cámara restringida",
                "El acceso a la cámara está restringido por controles parentales " +
                "o políticas del dispositivo. Contactá al administrador.",
                "Entendido");
            return false;
        }

        // Denegado → ofrecer ir a Configuración
        bool openSettings = await Application.Current!.MainPage!.DisplayAlert(
            "Permiso de cámara requerido",
            "Denegaste el acceso a la cámara. Para usar esta función, " +
            "habilitalo desde Configuración > Privacidad > Cámara.",
            "Ir a Configuración",
            "Cancelar");

        if (openSettings)
        {
            UIKit.UIApplication.SharedApplication.OpenUrl(
                new Foundation.NSUrl(UIKit.UIApplication.OpenSettingsUrlString));
        }

        CameraStatusLabel.Text = "Cámara: permisos denegados";
        BtnReintentarCamara.IsVisible = true;
        return false;
    }
    catch (Exception ex)
    {
        Console.WriteLine($"[APP] Camera permission error: {ex.Message}");
        ShowMessage($"Error al solicitar permisos de cámara: {ex.Message}");
        BtnReintentarCamara.IsVisible = true;
        return false;
    }
}
#endif
```

### 4.4 Mejora: Reemplazar `Task.Delay` con `TaskCompletionSource` (Android)

El `Task.Delay` tiene problemas: si el usuario tarda más de 5 segundos, la app asume denegación. Si responde en 1 segundo, la app espera 4 segundos innecesarios.

La solución correcta es interceptar `OnRequestPermissionsResult` en la Activity:

#### Paso 1 — Modificar `MainActivity.cs`

```csharp
// ── En Platforms/Android/MainActivity.cs ──

public class MainActivity : MauiAppCompatActivity
{
    // Evento estático que la lógica de permisos puede escuchar
    public static event Action<int, string[], Android.Content.PM.Permission[]>?
        PermissionsResultReceived;

    public override void OnRequestPermissionsResult(
        int requestCode,
        string[] permissions,
        Android.Content.PM.Permission[] grantResults)
    {
        base.OnRequestPermissionsResult(requestCode, permissions, grantResults);

        // Notificar a quien esté escuchando
        PermissionsResultReceived?.Invoke(requestCode, permissions, grantResults);
    }
}
```

#### Paso 2 — Método helper que espera el resultado real

```csharp
// ── En tu lógica de permisos (reemplaza el Task.Delay) ──

private async Task<bool> WaitForPermissionResultAsync(
    string[] requiredPermissions, int requestCode, int timeoutMs = 30000)
{
    var tcs = new TaskCompletionSource<bool>();

    void Handler(int code, string[] perms, Android.Content.PM.Permission[] results)
    {
        if (code != requestCode) return;

        bool granted = results.Length > 0
            && results.All(r => r == Android.Content.PM.Permission.Granted);

        tcs.TrySetResult(granted);
    }

    MainActivity.PermissionsResultReceived += Handler;

    try
    {
        var activity = Platform.CurrentActivity!;
        ActivityCompat.RequestPermissions(activity, requiredPermissions, requestCode);

        // Esperar resultado real O timeout
        var completedTask = await Task.WhenAny(
            tcs.Task,
            Task.Delay(timeoutMs));

        if (completedTask == tcs.Task)
            return tcs.Task.Result;

        // Timeout: verificar manualmente
        return requiredPermissions.All(p =>
            ContextCompat.CheckSelfPermission(activity, p)
                == (int)Android.Content.PM.Permission.Granted);
    }
    finally
    {
        MainActivity.PermissionsResultReceived -= Handler;
    }
}
```

#### Uso

```csharp
// En vez de:
//   ActivityCompat.RequestPermissions(activity, requiredPermissions, 100);
//   await Task.Delay(5000);
//   allGranted = requiredPermissions.All(p => ...);

// Usá:
bool allGranted = await WaitForPermissionResultAsync(requiredPermissions, requestCode: 100);
```

---

## 5. Manejo de Denegación de Permisos

### 5.1 Detectar denegación permanente en Android

```csharp
#if ANDROID
private async Task HandlePermissionDeniedAsync(
    Android.App.Activity activity,
    string[] permissions)
{
    /*
        ShouldShowRequestPermissionRationale devuelve:
        - true  → el usuario denegó PERO no marcó "No volver a preguntar"
        - false → el usuario marcó "No volver a preguntar"
                   O nunca se pidió el permiso (estado inicial)

        Si ya sabemos que el permiso fue denegado (paso anterior),
        y ShouldShowRationale devuelve false, significa denegación permanente.
    */

    bool permanentlyDenied = permissions.Any(p =>
        ContextCompat.CheckSelfPermission(activity, p)
            != (int)Android.Content.PM.Permission.Granted
        && !ActivityCompat.ShouldShowRequestPermissionRationale(activity, p));

    if (permanentlyDenied)
    {
        // ── Denegación permanente: solo queda ir a Configuración ──

        bool openSettings = await Application.Current!.MainPage!.DisplayAlert(
            "Permiso de cámara requerido",
            "Denegaste el permiso de cámara permanentemente. " +
            "Para usar esta función, habilitalo manualmente:\n\n" +
            "Configuración > Apps > [Tu App] > Permisos > Cámara",
            "Ir a Configuración",
            "Cancelar");

        if (openSettings)
            OpenAppSettings();
    }
    else
    {
        // ── Denegación temporal: se puede volver a pedir ──

        ShowMessage("Permisos de cámara denegados. Tocá 'Reintentar' cuando estés listo.");
    }

    CameraStatusLabel.Text = "Cámara: permisos denegados";
    BtnReintentarCamara.IsVisible = true;
}

/// <summary>
/// Abre la pantalla de configuración específica de la app.
/// El usuario puede habilitar/deshabilitar permisos desde ahí.
/// </summary>
private void OpenAppSettings()
{
    var intent = new Android.Content.Intent(
        Android.Provider.Settings.ActionApplicationDetailsSettings);

    intent.SetData(Android.Net.Uri.FromParts(
        "package",
        Platform.CurrentActivity!.PackageName!,
        null));

    intent.AddFlags(Android.Content.ActivityFlags.NewTask);
    Platform.CurrentActivity.StartActivity(intent);
}
#endif
```

### 5.2 Diagrama de flujo completo de denegación

```
                    ┌──────────────────┐
                    │ Solicitar permiso │
                    └────────┬─────────┘
                             │
                    ┌────────▼─────────┐
                    │  ¿Concedido?     │
                    └────────┬─────────┘
                        Sí ──┤── No
                       │     │
                  ✅ Usar   ┌▼───────────────────────┐
                  cámara    │ ¿Es denegación          │
                            │   permanente?           │
                            └────────┬────────────────┘
                               Sí ──┤── No
                              │     │
           ┌──────────────────▼┐   ┌▼────────────────────┐
           │ Mostrar diálogo:  │   │ Mostrar mensaje:     │
           │ "Ir a Config?"    │   │ "Tocá Reintentar"    │
           └────────┬──────────┘   │ + BtnReintentar      │
              Sí ──┤── No         └──────────────────────┘
             │     │
    ┌────────▼──┐  │
    │ Abrir     │  │
    │ Settings  │  │
    └───────────┘  │
                   │
           ┌───────▼───────────┐
           │ BtnReintentar     │
           │ visible           │
           └───────────────────┘
```

### 5.3 Botón "Reintentar" — Lógica

```csharp
// En tu XAML:
// <Button x:Name="BtnReintentarCamara"
//         Text="Reintentar cámara"
//         IsVisible="False"
//         Clicked="OnReintentarCamara_Clicked" />

private async void OnReintentarCamara_Clicked(object sender, EventArgs e)
{
    BtnReintentarCamara.IsVisible = false;
    CameraStatusLabel.Text = "Cámara: verificando permisos...";

    await InitializeCameraAsync();
}
```

### 5.4 Tabla resumen: Flujo de denegación por plataforma

| Escenario | Android | iOS |
|---|---|---|
| **Primera vez** | Muestra diálogo nativo del sistema | Muestra diálogo nativo del sistema |
| **Denegado una vez** | Vuelve a pedir + muestra explicación | No puede volver a pedir (API del sistema lo impide) |
| **"No volver a preguntar"** | Detecta con `ShouldShowRequestPermissionRationale` → ofrece ir a Configuración | N/A (siempre es permanente después de denegar) |
| **Denegado permanente** | Abre `Settings.ActionApplicationDetailsSettings` | Abre `UIApplication.OpenSettingsUrlString` |
| **Restringido (parental/MDM)** | N/A | `PermissionStatus.Restricted` → informa al usuario |

---

## 6. Integración con MediaPicker de MAUI

MAUI incluye `MediaPicker` que abstrae la captura de fotos. Sin embargo, **igual necesitás los permisos configurados**.

### 6.1 Capturar foto con MediaPicker

```csharp
private async Task OpenCameraAsync()
{
    try
    {
        // Verificar que el dispositivo tenga cámara
        if (!MediaPicker.Default.IsCaptureSupported)
        {
            ShowMessage("Este dispositivo no tiene cámara disponible.");
            return;
        }

        // Abrir la cámara del sistema
        var photo = await MediaPicker.Default.CapturePhotoAsync(new MediaPickerOptions
        {
            Title = "Tomar foto de infracción"  // Solo tiene efecto en Android
        });

        if (photo == null)
        {
            // El usuario canceló la captura
            ShowMessage("Captura cancelada.");
            return;
        }

        // ── Opción A: Leer como stream y mostrar en un Image ──

        using var stream = await photo.OpenReadAsync();

        FotoPreview.Source = ImageSource.FromStream(() =>
        {
            var ms = new MemoryStream();
            stream.CopyTo(ms);
            ms.Position = 0;
            return ms;
        });

        // ── Opción B: Guardar en directorio de la app ──

        var filePath = Path.Combine(
            FileSystem.AppDataDirectory,
            $"infraccion_{DateTime.Now:yyyyMMdd_HHmmss}.jpg");

        using var saveStream = File.OpenWrite(filePath);
        using var sourceStream = await photo.OpenReadAsync();
        await sourceStream.CopyToAsync(saveStream);

        Console.WriteLine($"[APP] Foto guardada en: {filePath}");
        ShowMessage("Foto capturada correctamente.");
    }
    catch (FeatureNotSupportedException)
    {
        ShowMessage("La captura de fotos no está soportada en este dispositivo.");
    }
    catch (PermissionException)
    {
        ShowMessage("No hay permisos para usar la cámara.");
    }
    catch (Exception ex)
    {
        Console.WriteLine($"[APP] Error al capturar foto: {ex.Message}");
        ShowMessage($"Error al capturar foto: {ex.Message}");
    }
}
```

### 6.2 Elegir foto de la galería

```csharp
private async Task PickPhotoFromGalleryAsync()
{
    try
    {
        var photo = await MediaPicker.Default.PickPhotoAsync(new MediaPickerOptions
        {
            Title = "Seleccionar foto"
        });

        if (photo == null) return;

        // Obtener ruta completa del archivo seleccionado
        var filePath = photo.FullPath;

        // Cargar en un Image control
        FotoPreview.Source = ImageSource.FromFile(filePath);

        ShowMessage($"Foto seleccionada: {photo.FileName}");
    }
    catch (Exception ex)
    {
        ShowMessage($"Error al seleccionar foto: {ex.Message}");
    }
}
```

---

## 7. Ejemplo Completo: Página con Cámara

### 7.1 XAML — `CameraPage.xaml`

```xml
<?xml version="1.0" encoding="utf-8" ?>
<ContentPage xmlns="http://schemas.microsoft.com/dotnet/2021/maui"
             xmlns:x="http://schemas.microsoft.com/winfx/2009/xaml"
             x:Class="MiApp.Views.CameraPage"
             Title="Captura de Foto">

    <ScrollView Padding="20">
        <VerticalStackLayout Spacing="15">

            <!-- Estado de permisos -->
            <Label x:Name="CameraStatusLabel"
                   Text="Cámara: verificando..."
                   FontSize="14"
                   TextColor="Gray" />

            <!-- Preview de la foto -->
            <Border StrokeShape="RoundRectangle 10"
                    Stroke="LightGray"
                    HeightRequest="300"
                    BackgroundColor="#F5F5F5">
                <Image x:Name="FotoPreview"
                       Aspect="AspectFit" />
            </Border>

            <!-- Botones de acción -->
            <Button x:Name="BtnTomarFoto"
                    Text="Tomar Foto"
                    BackgroundColor="#2196F3"
                    TextColor="White"
                    Clicked="OnTomarFoto_Clicked" />

            <Button x:Name="BtnElegirGaleria"
                    Text="Elegir de Galería"
                    BackgroundColor="#4CAF50"
                    TextColor="White"
                    Clicked="OnElegirGaleria_Clicked" />

            <!-- Botón de reintentar (oculto por defecto) -->
            <Button x:Name="BtnReintentarCamara"
                    Text="Reintentar permisos"
                    BackgroundColor="#FF9800"
                    TextColor="White"
                    IsVisible="False"
                    Clicked="OnReintentarCamara_Clicked" />

            <!-- Info de la foto capturada -->
            <Label x:Name="LblFotoInfo"
                   Text=""
                   FontSize="12"
                   TextColor="Gray" />

        </VerticalStackLayout>
    </ScrollView>
</ContentPage>
```

### 7.2 Code-behind — `CameraPage.xaml.cs`

```csharp
using Microsoft.Maui.ApplicationModel;

#if ANDROID
using Android;
using AndroidX.Core.App;
using AndroidX.Core.Content;
#endif

namespace MiApp.Views;

public partial class CameraPage : ContentPage
{
    public CameraPage()
    {
        InitializeComponent();
    }

    protected override async void OnAppearing()
    {
        base.OnAppearing();
        await CheckCameraAvailabilityAsync();
    }

    // ══════════════════════════════════════════════
    // VERIFICACIÓN INICIAL
    // ══════════════════════════════════════════════

    private async Task CheckCameraAvailabilityAsync()
    {
        if (!MediaPicker.Default.IsCaptureSupported)
        {
            CameraStatusLabel.Text = "Cámara: no disponible en este dispositivo";
            CameraStatusLabel.TextColor = Colors.Red;
            BtnTomarFoto.IsEnabled = false;
            return;
        }

        CameraStatusLabel.Text = "Cámara: disponible";
        CameraStatusLabel.TextColor = Colors.Green;
    }

    // ══════════════════════════════════════════════
    // EVENTOS DE BOTONES
    // ══════════════════════════════════════════════

    private async void OnTomarFoto_Clicked(object sender, EventArgs e)
    {
#if ANDROID || IOS
        var granted = await RequestCameraPermissionsAsync();
        if (!granted) return;
#endif
        await CapturePhotoAsync();
    }

    private async void OnElegirGaleria_Clicked(object sender, EventArgs e)
    {
        await PickPhotoFromGalleryAsync();
    }

    private async void OnReintentarCamara_Clicked(object sender, EventArgs e)
    {
        BtnReintentarCamara.IsVisible = false;
        CameraStatusLabel.Text = "Cámara: verificando permisos...";
        CameraStatusLabel.TextColor = Colors.Gray;

#if ANDROID || IOS
        var granted = await RequestCameraPermissionsAsync();
        if (granted)
            await CapturePhotoAsync();
#else
        await CapturePhotoAsync();
#endif
    }

    // ══════════════════════════════════════════════
    // CAPTURA Y SELECCIÓN DE FOTOS
    // ══════════════════════════════════════════════

    private async Task CapturePhotoAsync()
    {
        try
        {
            var photo = await MediaPicker.Default.CapturePhotoAsync();

            if (photo == null)
            {
                ShowMessage("Captura cancelada.");
                return;
            }

            await LoadPhotoAsync(photo);
        }
        catch (Exception ex)
        {
            Console.WriteLine($"[APP] Error captura: {ex}");
            ShowMessage($"Error al capturar: {ex.Message}");
        }
    }

    private async Task PickPhotoFromGalleryAsync()
    {
        try
        {
            var photo = await MediaPicker.Default.PickPhotoAsync();

            if (photo == null) return;

            await LoadPhotoAsync(photo);
        }
        catch (Exception ex)
        {
            ShowMessage($"Error al seleccionar: {ex.Message}");
        }
    }

    private async Task LoadPhotoAsync(FileResult photo)
    {
        // Guardar una copia local
        var localPath = Path.Combine(
            FileSystem.AppDataDirectory,
            $"foto_{DateTime.Now:yyyyMMdd_HHmmss}.jpg");

        using (var sourceStream = await photo.OpenReadAsync())
        using (var localFile = File.OpenWrite(localPath))
        {
            await sourceStream.CopyToAsync(localFile);
        }

        // Mostrar en el preview
        FotoPreview.Source = ImageSource.FromFile(localPath);

        // Info
        var fileInfo = new FileInfo(localPath);
        LblFotoInfo.Text = $"Archivo: {fileInfo.Name}\n" +
                           $"Tamaño: {fileInfo.Length / 1024} KB\n" +
                           $"Guardado en: {localPath}";

        ShowMessage("Foto guardada correctamente.");
    }

    // ══════════════════════════════════════════════
    // PERMISOS
    // ══════════════════════════════════════════════

#if ANDROID
    private async Task<bool> RequestCameraPermissionsAsync()
    {
        try
        {
            var activity = Platform.CurrentActivity
                ?? throw new InvalidOperationException("No hay Activity activa.");

            string[] requiredPermissions;

            if (Android.OS.Build.VERSION.SdkInt >= Android.OS.BuildVersionCodes.Tiramisu)
            {
                requiredPermissions = new[]
                {
                    Manifest.Permission.Camera,
                    Manifest.Permission.ReadMediaImages
                };
            }
            else if (Android.OS.Build.VERSION.SdkInt >= Android.OS.BuildVersionCodes.M)
            {
                requiredPermissions = new[]
                {
                    Manifest.Permission.Camera,
                    Manifest.Permission.ReadExternalStorage,
                    Manifest.Permission.WriteExternalStorage
                };
            }
            else
            {
                return true;
            }

            bool allGranted = requiredPermissions.All(p =>
                ContextCompat.CheckSelfPermission(activity, p)
                    == (int)Android.Content.PM.Permission.Granted);

            if (allGranted)
                return true;

            // Solicitar permisos
            ActivityCompat.RequestPermissions(activity, requiredPermissions, 100);
            await Task.Delay(5000);

            allGranted = requiredPermissions.All(p =>
                ContextCompat.CheckSelfPermission(activity, p)
                    == (int)Android.Content.PM.Permission.Granted);

            if (allGranted)
                return true;

            // Denegado — verificar si es permanente
            bool permanentlyDenied = requiredPermissions.Any(p =>
                ContextCompat.CheckSelfPermission(activity, p)
                    != (int)Android.Content.PM.Permission.Granted
                && !ActivityCompat.ShouldShowRequestPermissionRationale(activity, p));

            if (permanentlyDenied)
            {
                bool goToSettings = await DisplayAlert(
                    "Permiso requerido",
                    "Denegaste el permiso de cámara permanentemente.\n" +
                    "Habilitalo desde Configuración > Permisos.",
                    "Ir a Configuración", "Cancelar");

                if (goToSettings)
                {
                    var intent = new Android.Content.Intent(
                        Android.Provider.Settings.ActionApplicationDetailsSettings);
                    intent.SetData(Android.Net.Uri.FromParts(
                        "package", activity.PackageName!, null));
                    intent.AddFlags(Android.Content.ActivityFlags.NewTask);
                    activity.StartActivity(intent);
                }
            }
            else
            {
                ShowMessage("Permisos de cámara denegados. Tocá 'Reintentar' cuando estés listo.");
            }

            CameraStatusLabel.Text = "Cámara: permisos denegados";
            CameraStatusLabel.TextColor = Colors.Red;
            BtnReintentarCamara.IsVisible = true;
            return false;
        }
        catch (Exception ex)
        {
            Console.WriteLine($"[APP] Camera permission error: {ex}");
            ShowMessage($"Error permisos: {ex.Message}");
            BtnReintentarCamara.IsVisible = true;
            return false;
        }
    }

#elif IOS
    private async Task<bool> RequestCameraPermissionsAsync()
    {
        try
        {
            var status = await Permissions.CheckStatusAsync<Permissions.Camera>();

            if (status == PermissionStatus.Granted)
                return true;

            if (status == PermissionStatus.Unknown)
            {
                status = await Permissions.RequestAsync<Permissions.Camera>();
                if (status == PermissionStatus.Granted)
                    return true;
            }

            if (status == PermissionStatus.Restricted)
            {
                await DisplayAlert(
                    "Cámara restringida",
                    "El acceso a la cámara está bloqueado por controles parentales " +
                    "o políticas del dispositivo.",
                    "Entendido");
                return false;
            }

            // Denegado → ofrecer ir a Configuración
            bool goToSettings = await DisplayAlert(
                "Permiso de cámara requerido",
                "Para tomar fotos, habilitá la cámara desde " +
                "Configuración > Privacidad > Cámara.",
                "Ir a Configuración", "Cancelar");

            if (goToSettings)
            {
                UIKit.UIApplication.SharedApplication.OpenUrl(
                    new Foundation.NSUrl(UIKit.UIApplication.OpenSettingsUrlString));
            }

            CameraStatusLabel.Text = "Cámara: permisos denegados";
            CameraStatusLabel.TextColor = Colors.Red;
            BtnReintentarCamara.IsVisible = true;
            return false;
        }
        catch (Exception ex)
        {
            Console.WriteLine($"[APP] Camera permission error: {ex}");
            ShowMessage($"Error permisos: {ex.Message}");
            BtnReintentarCamara.IsVisible = true;
            return false;
        }
    }
#endif

    // ══════════════════════════════════════════════
    // HELPERS
    // ══════════════════════════════════════════════

    private void ShowMessage(string message)
    {
        MainThread.BeginInvokeOnMainThread(async () =>
        {
            await DisplayAlert("Información", message, "OK");
        });
    }
}
```

---

## 8. Troubleshooting

### 8.1 Errores comunes y soluciones

| Error | Causa | Solución |
|-------|-------|----------|
| `Java.Lang.SecurityException: Permission Denial` | Falta permiso en `AndroidManifest.xml` | Agregar `<uses-permission android:name="android.permission.CAMERA" />` |
| App crashea en iOS al abrir cámara | Falta `NSCameraUsageDescription` en `Info.plist` | Agregar la clave con un texto descriptivo |
| `FileNotFoundException` al guardar foto | Falta `FileProvider` en el manifiesto | Configurar `<provider>` y `file_paths.xml` |
| Permiso concedido pero cámara no abre | Emulador sin cámara configurada | Configurar cámara virtual en AVD Manager o probar en dispositivo físico |
| `PermissionException` con MediaPicker | Permisos no solicitados antes de usar MediaPicker | Solicitar permisos runtime explícitamente antes |
| `READ_EXTERNAL_STORAGE` no funciona en Android 13 | Permiso deprecated en API 33 | Usar `READ_MEDIA_IMAGES` en su lugar |

### 8.2 Checklist de verificación

```
✅ AndroidManifest.xml
   ├── android.permission.CAMERA
   ├── android.permission.READ_EXTERNAL_STORAGE (maxSdkVersion="32")
   ├── android.permission.WRITE_EXTERNAL_STORAGE (maxSdkVersion="32")
   ├── android.permission.READ_MEDIA_IMAGES (API 33+)
   ├── uses-feature camera required="false"
   └── FileProvider configurado

✅ Info.plist (iOS)
   ├── NSCameraUsageDescription (texto descriptivo)
   ├── NSPhotoLibraryUsageDescription
   └── NSPhotoLibraryAddUsageDescription

✅ file_paths.xml (Android)
   ├── external-files-path para fotos
   ├── cache-path para temporales
   └── Ubicado en Platforms/Android/Resources/xml/

✅ Código C#
   ├── Directivas #if ANDROID / #elif IOS
   ├── Verificación de versión de API (Tiramisu vs M)
   ├── Manejo de denegación temporal
   ├── Manejo de denegación permanente
   ├── Redirección a Configuración del sistema
   ├── Botón de reintentar visible al denegar
   └── try-catch en toda la cadena de permisos
```

### 8.3 Probar permisos en emulador

**Android (Android Studio AVD):**
```bash
# Revocar permisos por línea de comando para probar flujo de denegación
adb shell pm revoke com.tuapp.package android.permission.CAMERA
adb shell pm grant com.tuapp.package android.permission.CAMERA

# Verificar estado actual
adb shell dumpsys package com.tuapp.package | grep permission
```

**iOS (Xcode Simulator):**
- Simuladores iOS no tienen cámara real. Usá un dispositivo físico para probar `CapturePhotoAsync`.
- Para resetear permisos: **Device > Erase All Content and Settings** en el simulador.

---

> **Nota sobre seguridad**: Las fotos capturadas se guardan en `FileSystem.AppDataDirectory`, que es un directorio privado de la app. Otras apps no pueden acceder a estos archivos. Si necesitás compartir fotos, usá `FileProvider` con URIs temporales en vez de rutas absolutas.
