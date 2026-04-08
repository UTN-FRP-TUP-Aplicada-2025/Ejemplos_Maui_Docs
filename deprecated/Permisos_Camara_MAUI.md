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
9. [MediaPicker vs CameraView — Dos formas de usar la cámara](#9-mediapicker-vs-cameraview--dos-formas-de-usar-la-cámara)
10. [CameraView (CommunityToolkit): Consideraciones y Trampas](#10-cameraview-communitytoolkit-consideraciones-y-trampas)
11. [Ejemplo Completo: CameraView con permisos y overlay](#11-ejemplo-completo-cameraview-con-permisos-y-overlay)

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
| `FileNotFoundException` al guardar foto | Falta `FileProvider` en el manifiesto | Configurar `<provider>` y `file_paths.xml` (solo para MediaPicker) |
| Permiso concedido pero cámara no abre | Emulador sin cámara configurada | Configurar cámara virtual en AVD Manager o probar en dispositivo físico |
| `PermissionException` con MediaPicker | Permisos no solicitados antes de usar MediaPicker | Solicitar permisos runtime explícitamente antes |
| `READ_EXTERNAL_STORAGE` no funciona en Android 13 | Permiso deprecated en API 33 | Usar `READ_MEDIA_IMAGES` en su lugar |
| `CameraException: No camera available on device` (SIGABRT) | `CameraView` declarado en XAML en dispositivo sin cámara | No declarar `CameraView` en XAML; crearlo dinámicamente desde código después de verificar `IsCaptureSupported` (ver sección 10.1) |
| `CameraView` crashea con `IsVisible="False"` | `IsVisible` no evita que `ConnectHandler` se ejecute | Usar `ContentView` como contenedor y crear `CameraView` por código (ver sección 10.1) |
| Permisos se muestran como denegados en primera ejecución (Android) | `CheckStatusAsync` devuelve `Denied` en vez de `Unknown` en Android | Siempre llamar a `RequestAsync` si no está `Granted`, no filtrar por `Unknown` (ver sección 10.2) |
| `ShouldShowRationale` devuelve `false` pero el permiso nunca se pidió | Es el comportamiento esperado de Android | Llamar a `ShouldShowRationale` después de `RequestAsync` para que sea confiable (ver sección 10.3) |

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

---

## 9. MediaPicker vs CameraView — Dos formas de usar la cámara

En .NET MAUI existen dos enfoques para tomar fotos, y cada uno tiene implicaciones diferentes en manifiestos, permisos y arquitectura.

### 9.1 Comparación general

```
┌─────────────────────────────────────────────────────────────────────┐
│                         MediaPicker                                 │
│                    (viene con .NET MAUI)                            │
│                                                                     │
│   Tu App ──➤ App de Cámara del Sistema ──➤ Foto devuelta a tu App  │
│                                                                     │
│   • Abre una app EXTERNA (la cámara del sistema)                   │
│   • Necesita FileProvider para compartir archivos entre apps        │
│   • La foto viaja por una URI content://                            │
│   • Interfaz que NO controlás (es la del sistema)                  │
└─────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────┐
│                    CameraView (CommunityToolkit)                    │
│              (paquete NuGet: CommunityToolkit.Maui)                │
│                                                                     │
│   Tu App ──➤ Renderiza la cámara DENTRO de tu página               │
│                                                                     │
│   • La cámara se muestra embebida en tu UI                         │
│   • NO necesita FileProvider (no hay otra app involucrada)          │
│   • La foto te llega como Stream en MediaCapturedEventArgs         │
│   • Interfaz 100% personalizable (flash, botones, overlay, etc.)   │
└─────────────────────────────────────────────────────────────────────┘
```

### 9.2 Tabla comparativa detallada

| Aspecto | MediaPicker | CameraView (CommunityToolkit) |
|---|---|---|
| **Paquete** | Incluido en .NET MAUI | NuGet: `CommunityToolkit.Maui` |
| **Cómo funciona** | Abre la app de cámara del **sistema** (otra app) | Renderiza la cámara **dentro** de tu página |
| **UI** | No controlás la interfaz | 100% personalizable |
| **FileProvider** | **Obligatorio** (intercambio entre apps) | **No necesario** |
| **`file_paths.xml`** | **Obligatorio** | **No necesario** |
| **Permisos de almacenamiento** | Puede necesitarlos (`READ/WRITE_EXTERNAL_STORAGE`, `READ_MEDIA_IMAGES`) | **No necesarios** si guardás en `AppDataDirectory` |
| **`requestLegacyExternalStorage`** | Puede necesitarlo (Android 10) | **No necesario** |
| **Permiso de cámara (manifiesto)** | `android.permission.CAMERA` + `NSCameraUsageDescription` | Igual |
| **Permiso runtime** | Sí | Sí |
| **Flash** | No controlás | Controlable por código |
| **Selección de cámara (frontal/trasera)** | No controlás | Controlable por código |
| **Galería** | `PickPhotoAsync` incluido | No incluido (solo captura) |

### 9.3 ¿Cuándo usar cada uno?

**Usá MediaPicker cuando:**
- Necesitás algo rápido y simple
- No te importa la interfaz de captura
- También necesitás seleccionar fotos de la galería
- No querés agregar dependencias extra

**Usá CameraView cuando:**
- Necesitás una interfaz de cámara personalizada
- Querés controlar flash, zoom, selección de cámara
- Necesitás un overlay sobre la cámara (ej: guías de encuadre)
- La experiencia del usuario es prioritaria

### 9.4 ¿Por qué MediaPicker necesita FileProvider y CameraView no?

```
MediaPicker (dos apps involucradas):

Tu App                          App de Cámara del Sistema
  │                                      │
  │── "Tomá una foto, guardala acá" ────▶│
  │    content://com.tuapp.fileprovider   │
  │    /cache/photo_123.jpg               │
  │                                      │
  │◀── "Listo, foto guardada" ──────────│
  │                                      │

  ⚠️ Sin FileProvider, la app de cámara NO tiene dónde escribir.
     Android no permite que una app escriba en el directorio de otra.
     FileProvider genera una URI temporal con permisos controlados.


CameraView (una sola app):

Tu App
  │
  │── CameraView renderiza la cámara
  │── CaptureImage() → Stream en memoria
  │── Tu código guarda donde quiera
  │
  ✅ Todo pasa dentro de tu proceso.
     No hay intercambio entre apps.
     No se necesita FileProvider.
```

### 9.5 Manifiestos mínimos según el enfoque

#### Solo MediaPicker (captura de foto)

**Android — `AndroidManifest.xml`:**
```xml
<?xml version="1.0" encoding="utf-8"?>
<manifest xmlns:android="http://schemas.android.com/apk/res/android">
    <uses-permission android:name="android.permission.CAMERA" />
    <uses-feature android:name="android.hardware.camera" android:required="false" />

    <application android:allowBackup="true" android:label="@string/app_name">
        <!-- FileProvider OBLIGATORIO para MediaPicker -->
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

**`file_paths.xml` OBLIGATORIO:**
```xml
<?xml version="1.0" encoding="utf-8"?>
<paths>
    <cache-path name="cache" path="." />
</paths>
```

**iOS — `Info.plist`:**
```xml
<key>NSCameraUsageDescription</key>
<string>La app necesita acceso a la cámara para tomar fotos.</string>
```

#### Solo CameraView (cámara embebida)

**Android — `AndroidManifest.xml`:**
```xml
<?xml version="1.0" encoding="utf-8"?>
<manifest xmlns:android="http://schemas.android.com/apk/res/android">
    <uses-permission android:name="android.permission.CAMERA" />
    <uses-feature android:name="android.hardware.camera" android:required="false" />

    <!-- NO necesita FileProvider, file_paths.xml ni permisos de almacenamiento -->
    <application android:allowBackup="true" android:label="@string/app_name" />
</manifest>
```

**iOS — `Info.plist`:**
```xml
<key>NSCameraUsageDescription</key>
<string>La app necesita acceso a la cámara para tomar fotos.</string>
```

> **Nota**: Si solo tomás fotos y las guardás en `FileSystem.AppDataDirectory` (directorio privado de la app), NO necesitás permisos de almacenamiento (`READ_EXTERNAL_STORAGE`, `WRITE_EXTERNAL_STORAGE`, `READ_MEDIA_IMAGES`) ni permisos de galería (`NSPhotoLibraryUsageDescription`, `NSPhotoLibraryAddUsageDescription`). Estos solo son necesarios si accedés a la galería del usuario.

---

## 10. CameraView (CommunityToolkit): Consideraciones y Trampas

### 10.1 Trampa crítica: `ConnectHandler` y el crash sin cámara

El error más común al usar `CameraView` es este crash:

```
Unhandled managed exception:
  No camera available on device (CommunityToolkit.Maui.Core.CameraException)
    at CommunityToolkit.Maui.Core.CameraManager.ConnectCamera(CancellationToken)
    at CommunityToolkit.Maui.Core.Handlers.CameraViewHandler.ConnectHandler(UIView)
```

**¿Por qué pasa?** Cuando MAUI procesa el XAML, ejecuta este flujo automáticamente:

```
XAML parseado
    │
    └─ Crea instancia de CameraView (el control .NET)
         │
         └─ MAUI busca el Handler nativo para esa plataforma
              │
              └─ CameraViewHandler.ConnectHandler()
                   │
                   └─ CameraManager.ConnectCamera()
                        │
                        └─ Busca hardware de cámara
                             │
                             ├─ Hay cámara → OK, conecta
                             └─ NO hay cámara → CameraException → CRASH (SIGABRT)
```

**Lo importante**: `ConnectHandler` NO es un evento que vos suscribís. Es un método **interno del framework** que se ejecuta automáticamente cuando el control se agrega al árbol visual.

**¿`IsVisible="False"` lo evita?** **NO.** `IsVisible` solo controla si el control se **dibuja en pantalla**, pero el control ya está en el árbol visual y su handler ya se conectó.

```
┌──────────────────────────────────────────────────────────────┐
│  INCORRECTO — IsVisible="False" NO evita ConnectHandler      │
│                                                              │
│  <!-- El handler se conecta igual, crash en simulador -->    │
│  <toolkit:CameraView x:Name="Camera"                        │
│                       IsVisible="False" />                   │
│                                                              │
│  Resultado: ConnectHandler() → busca cámara → CRASH          │
└──────────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────────┐
│  CORRECTO — Usar ContentView como contenedor vacío           │
│                                                              │
│  <!-- No hay CameraView → no hay handler → no hay crash -->  │
│  <ContentView x:Name="CameraContainer" />                    │
│                                                              │
│  // En código C#, DESPUÉS de verificar que hay cámara:       │
│  CameraContainer.Content = new CameraView { ... };           │
│                                                              │
│  Resultado: ConnectHandler() solo se ejecuta cuando vos      │
│  asignás el CameraView al contenedor.                        │
└──────────────────────────────────────────────────────────────┘
```

### 10.2 `Permissions.CheckStatusAsync` en Android: la trampa del `Denied`

En Android, `CheckStatusAsync<Permissions.Camera>()` devuelve `Denied` (no `Unknown`) cuando el permiso **nunca fue solicitado**. Esto es una diferencia clave con iOS:

| Situación | iOS | Android |
|---|---|---|
| Permiso nunca solicitado | `Unknown` | `Denied` ⚠️ |
| Usuario dijo "No" | `Denied` | `Denied` |
| Usuario marcó "No volver a preguntar" | N/A (siempre es permanente) | `Denied` |
| Permiso concedido | `Granted` | `Granted` |

**El problema con código como este:**

```csharp
// ❌ INCORRECTO — En Android nunca entra al if porque status es Denied, no Unknown
var status = await Permissions.CheckStatusAsync<Permissions.Camera>();

if (status == PermissionStatus.Unknown)
{
    status = await Permissions.RequestAsync<Permissions.Camera>();  // Nunca se ejecuta en Android
}
```

**La solución: siempre llamar a `RequestAsync` si no está Granted:**

```csharp
// ✅ CORRECTO — Funciona en ambas plataformas
var status = await Permissions.CheckStatusAsync<Permissions.Camera>();

if (status == PermissionStatus.Granted)
{
    MostrarVisorCamara();
    return;
}

// No está Granted → pedir (tanto si es Unknown como Denied sin "no volver a preguntar")
status = await Permissions.RequestAsync<Permissions.Camera>();

if (status == PermissionStatus.Granted)
{
    MostrarVisorCamara();
    return;
}

// Si llegamos acá, el usuario denegó
```

### 10.3 `ShouldShowRationale`: cuándo es confiable

`Permissions.ShouldShowRationale<Permissions.Camera>()` en Android devuelve:

| Situación | Valor | ¿Podemos re-pedir? |
|---|---|---|
| Permiso **nunca** solicitado | `false` ⚠️ | Sí |
| Usuario dijo "No" (sin marcar "No volver a preguntar") | `true` | Sí |
| Usuario marcó "No volver a preguntar" | `false` | No → ir a Configuración |

**El problema**: devuelve `false` tanto cuando **nunca se pidió** como cuando es **denegación permanente**. No sirve para distinguir el estado inicial.

**La solución**: llamar a `ShouldShowRationale` **después** de `RequestAsync`. Si el usuario ya vio el diálogo del sistema y denegó, ahora `ShouldShowRationale` es confiable:

```csharp
// Primero pedir
status = await Permissions.RequestAsync<Permissions.Camera>();

// Si fue denegado, ahora ShouldShowRationale es confiable
if (status == PermissionStatus.Denied)
{
    bool puedeReintentar = false;

#if ANDROID
    // Ahora SÍ podemos confiar: si devuelve false, es permanente
    puedeReintentar = Permissions.ShouldShowRationale<Permissions.Camera>();
#endif
}
```

### 10.4 Verificar hardware antes de crear el CameraView

Antes de instanciar el `CameraView`, hay que verificar que el dispositivo tenga cámara:

```csharp
if (MediaPicker.Default.IsCaptureSupported)
{
    // Hay cámara → crear CameraView
    MostrarVisorCamara();
}
else
{
    // No hay cámara (simulador, tablet sin cámara)
    MostrarOverlayPermiso("Cámara no disponible",
        "Este dispositivo no tiene cámara disponible.",
        puedeReintentar: false);
}
```

**¿Por qué `MediaPicker.Default.IsCaptureSupported`?** Porque verifica la presencia de hardware de cámara sin intentar conectarse. `CameraView.GetAvailableCameras()` también funciona pero requiere que el `CameraView` ya exista (y ahí ya es tarde si no hay cámara).

### 10.5 Flujo completo recomendado para CameraView

```
Página se carga (OnAppearing)
    │
    ├─ 1. CheckStatusAsync → ¿Granted?
    │      │
    │      ├─ Sí → ¿IsCaptureSupported?
    │      │         ├─ Sí → Crear CameraView dinámicamente → Mostrar
    │      │         └─ No → Overlay "Cámara no disponible"
    │      │
    │      └─ No → RequestAsync → ¿Granted?
    │               │
    │               ├─ Sí → (mismo check de IsCaptureSupported)
    │               │
    │               ├─ Restricted → Overlay "Acceso restringido"
    │               │
    │               └─ Denied → ShouldShowRationale?
    │                            │
    │                            ├─ true → Overlay + botón "Pedir permiso"
    │                            └─ false → Overlay + botón "Abrir configuración"
    │
    └─ 2. OnNavigatedTo → SeleccionarCamaraAsync (trasera por defecto)
```

### 10.6 Limpieza del CameraView

`CameraView` accede a hardware nativo. Si no lo limpiás al salir de la página, puede causar leaks o que la cámara quede bloqueada:

```csharp
protected override void OnDisappearing()
{
    base.OnDisappearing();

    if (_cameraView != null)
    {
        _cameraView.MediaCaptured -= OnMediaCaptured;
        _cameraView.MediaCaptureFailed -= OnMediaCaptureFailed;
        CameraContainer.Content = null;  // Remueve del árbol visual → DisconnectHandler
        _cameraView = null;
    }
}
```

---

## 11. Ejemplo Completo: CameraView con permisos y overlay

Este ejemplo usa `CameraView` del CommunityToolkit con:
- Creación dinámica (sin declarar en XAML) para evitar crash sin cámara
- Permisos runtime correctos para Android e iOS
- Overlay de permiso denegado con botones contextuales
- Flash y selección de cámara
- Navegación con callback para devolver la foto

### 11.1 XAML — `MyMediaPickerPage.xaml`

```xml
<?xml version="1.0" encoding="utf-8" ?>
<ContentPage xmlns="http://schemas.microsoft.com/dotnet/2021/maui"
             xmlns:x="http://schemas.microsoft.com/winfx/2009/xaml"
             x:Class="Ejemplo_Photo_MiMediaPicker_Callback.Pages.MyMediaPickerPage"
             Title="Camera Page"
             BackgroundColor="#000000">

    <!--
        NOTA: No se declara <toolkit:CameraView> en XAML.
        Se crea dinámicamente desde C# para evitar que ConnectHandler()
        crashee en dispositivos/emuladores sin cámara.
    -->

    <Grid x:Name="DynamicLayout" RowDefinitions="Auto,*,Auto" ColumnDefinitions="*"
          HorizontalOptions="Fill" VerticalOptions="Fill">

        <!-- Botón de flash (oculto hasta que la cámara esté activa) -->
        <Button x:Name="BtnFlashButton" Grid.Row="0" Grid.Column="0"
                Clicked="OnActiveFlashClicked"
                IsVisible="False"
                HorizontalOptions="Center" VerticalOptions="Center"
                Background="#aa000000" CornerRadius="8">
            <Button.ImageSource>
                <FontImageSource Glyph="{Binding FlashIcon}" FontFamily="MaterialIconsOutlined" />
            </Button.ImageSource>
        </Button>

        <!--
            Contenedor vacío para el CameraView.
            CameraView se crea desde código y se asigna a CameraContainer.Content
            DESPUÉS de verificar permisos y hardware.
        -->
        <ContentView x:Name="CameraContainer" Grid.Row="1" Grid.Column="0"
                     IsVisible="False"
                     HorizontalOptions="Fill" VerticalOptions="Fill" />

        <!--
            Overlay que se muestra cuando:
            - El usuario denegó permisos (con o sin "No volver a preguntar")
            - El dispositivo no tiene cámara
            - El acceso está restringido (control parental / MDM)
        -->
        <Grid x:Name="PermissionDeniedOverlay"
              Grid.Row="1" Grid.Column="0"
              IsVisible="False"
              BackgroundColor="#CC000000"
              HorizontalOptions="Fill" VerticalOptions="Fill">

            <VerticalStackLayout Spacing="20"
                                 HorizontalOptions="Center"
                                 VerticalOptions="Center"
                                 Padding="32">

                <Label Text="no_photography"
                       FontFamily="MaterialIconsOutlined"
                       FontSize="80"
                       TextColor="#AAAAAA"
                       HorizontalOptions="Center"/>

                <Label x:Name="LblPermissionTitle"
                       Text="Sin acceso a la cámara"
                       FontSize="20"
                       FontAttributes="Bold"
                       TextColor="White"
                       HorizontalOptions="Center"
                       HorizontalTextAlignment="Center"/>

                <Label x:Name="LblPermissionMessage"
                       Text="Para tomar fotos necesitamos acceso a la cámara."
                       FontSize="14"
                       TextColor="#CCCCCC"
                       HorizontalOptions="Center"
                       HorizontalTextAlignment="Center"/>

                <!-- Visible solo si el SO permite re-pedir el permiso -->
                <Button x:Name="BtnPedirPermiso"
                        Text="Pedir permiso"
                        Clicked="OnPedirPermisoClicked"
                        BackgroundColor="#512BD4"
                        TextColor="White"
                        CornerRadius="8"
                        Padding="20,12"
                        HorizontalOptions="Center"
                        IsVisible="False"/>

                <!-- Visible solo si la denegación es permanente -->
                <Button x:Name="BtnGoToSettings"
                        Text="Abrir configuración"
                        Clicked="OnGoToSettingsClicked"
                        BackgroundColor="#512BD4"
                        TextColor="White"
                        CornerRadius="8"
                        Padding="20,12"
                        HorizontalOptions="Center"/>

                <Button Text="Volver"
                        Clicked="OnVolverClicked"
                        BackgroundColor="Transparent"
                        TextColor="#AAAAAA"
                        BorderColor="#555555"
                        BorderWidth="1"
                        CornerRadius="8"
                        Padding="20,12"
                        HorizontalOptions="Center"/>

            </VerticalStackLayout>
        </Grid>

        <!-- Botón de captura (oculto hasta que la cámara esté activa) -->
        <Button x:Name="BtnTomarFoto" Grid.Row="2" Grid.Column="0"
                Clicked="OnTomarFotoClicked"
                IsVisible="False"
                HorizontalOptions="Center" VerticalOptions="Center"
                Background="#aa000000" CornerRadius="8">
            <Button.ImageSource>
                <FontImageSource Glyph="photo_camera" Size="50" FontFamily="MaterialIconsOutlined"/>
            </Button.ImageSource>
        </Button>

    </Grid>
</ContentPage>
```

### 11.2 Code-behind — `MyMediaPickerPage.xaml.cs`

```csharp
using CommunityToolkit.Maui.Core;
using CommunityToolkit.Maui.Views;
using System.Diagnostics;

namespace Ejemplo_Photo_MiMediaPicker_Callback.Pages;

[QueryProperty(nameof(OnPhotoCallback), "OnPhotoCallback")]
public partial class MyMediaPickerPage : ContentPage
{
    private bool _isCapturingImage = false;
    private CancellationTokenSource? _captureCancellationTokenSource;

    // Campo nullable: el CameraView se crea desde código, no desde XAML
    private CameraView? _cameraView;

    // Callback para devolver la foto a la página que navegó hasta acá
    public Action<Image>? OnPhotoCallback { get; set; }

    private string _flashIcon = "flash_off";
    public string FlashIcon
    {
        get => _flashIcon;
        set
        {
            if (_flashIcon != value)
            {
                _flashIcon = value;
                OnPropertyChanged(nameof(FlashIcon));
            }
        }
    }

    public MyMediaPickerPage()
    {
        InitializeComponent();
        BindingContext = this;
        // NO llamar a StatusFlashToIcons() acá — _cameraView es null todavía
    }

    protected override async void OnAppearing()
    {
        base.OnAppearing();

        DeviceDisplay.MainDisplayInfoChanged += OnMainDisplayInfoChanged;
        UpdateLayoutOrientation(DeviceDisplay.MainDisplayInfo.Orientation);

        // Punto de entrada principal: evaluar permisos y mostrar cámara u overlay
        await EvaluarYMostrarEstadoPermisoAsync();
    }

    protected override void OnDisappearing()
    {
        base.OnDisappearing();

        try
        {
            DeviceDisplay.MainDisplayInfoChanged -= OnMainDisplayInfoChanged;
            _captureCancellationTokenSource?.Cancel();

            // IMPORTANTE: Limpiar el CameraView para liberar hardware
            if (_cameraView != null)
            {
                _cameraView.MediaCaptured -= OnMediaCaptured;
                _cameraView.MediaCaptureFailed -= OnMediaCaptureFailed;
                CameraContainer.Content = null;  // Remueve del árbol → DisconnectHandler
                _cameraView = null;
            }
        }
        catch (Exception ex)
        {
            Debug.WriteLine($"OnDisappearing error: {ex.Message}");
        }

#if ANDROID
        var activity = Platform.CurrentActivity;
        if (activity != null)
            activity.RequestedOrientation = Android.Content.PM.ScreenOrientation.Unspecified;
#endif
    }

    protected override async void OnNavigatedTo(NavigatedToEventArgs args)
    {
        base.OnNavigatedTo(args);

        // Si ya tiene permisos y el CameraView existe, seleccionar cámara trasera
        var status = await Permissions.CheckStatusAsync<Permissions.Camera>();
        if (status == PermissionStatus.Granted && _cameraView != null)
            await SeleccionarCamaraAsync();
    }

    // ══════════════════════════════════════════════
    // PERMISOS — Flujo principal
    // ══════════════════════════════════════════════

    private async Task EvaluarYMostrarEstadoPermisoAsync()
    {
        // Paso 1: Verificar estado actual
        var status = await Permissions.CheckStatusAsync<Permissions.Camera>();

        if (status == PermissionStatus.Granted)
        {
            // Permiso OK → verificar que haya hardware de cámara
            if (MediaPicker.Default.IsCaptureSupported)
                MostrarVisorCamara();
            else
                MostrarOverlayPermiso("Cámara no disponible",
                    "Este dispositivo no tiene cámara disponible.",
                    puedeReintentar: false);
            return;
        }

        // Paso 2: Pedir permiso
        // NOTA: En Android, CheckStatusAsync devuelve Denied (no Unknown) cuando
        // el permiso nunca fue solicitado. Por eso siempre llamamos a RequestAsync
        // si no está Granted, en vez de filtrar solo por Unknown.
        status = await Permissions.RequestAsync<Permissions.Camera>();

        if (status == PermissionStatus.Granted)
        {
            if (MediaPicker.Default.IsCaptureSupported)
                MostrarVisorCamara();
            else
                MostrarOverlayPermiso("Cámara no disponible",
                    "Este dispositivo no tiene cámara disponible.",
                    puedeReintentar: false);
            return;
        }

        // Paso 3: Restringido (control parental / MDM en iOS)
        if (status == PermissionStatus.Restricted)
        {
            MostrarOverlayPermiso("Acceso restringido",
                "El acceso a la cámara está restringido por una política del dispositivo. " +
                "Consultá con el administrador.",
                puedeReintentar: false);
            return;
        }

        // Paso 4: Denegado — ¿se puede volver a pedir?
        // IMPORTANTE: ShouldShowRationale es confiable AHORA porque ya llamamos
        // a RequestAsync (el usuario ya vio el diálogo del sistema).
        bool puedeReintentar = false;

#if ANDROID
        puedeReintentar = Permissions.ShouldShowRationale<Permissions.Camera>();
#endif
        // En iOS puedeReintentar queda en false (correcto: iOS no permite re-pedir)

        MostrarOverlayPermiso(
            titulo: puedeReintentar
                ? "Permiso de cámara necesario"
                : "Acceso a la cámara denegado",
            mensaje: puedeReintentar
                ? "Para tomar fotos necesitamos acceso a la cámara. " +
                  "Podés intentar conceder el permiso."
                : "Para tomar fotos necesitamos acceso a la cámara. " +
                  "Habilitalo desde los ajustes de la aplicación.",
            puedeReintentar: puedeReintentar);
    }

    // ══════════════════════════════════════════════
    // OVERLAY — mostrar cámara / mostrar overlay
    // ══════════════════════════════════════════════

    /// <summary>
    /// Crea el CameraView dinámicamente (si no existe) y lo muestra.
    /// Solo se llama después de confirmar permisos + hardware.
    /// </summary>
    private void MostrarVisorCamara()
    {
        MainThread.BeginInvokeOnMainThread(() =>
        {
            PermissionDeniedOverlay.IsVisible = false;

            // Crear el CameraView solo si no existe
            if (_cameraView == null)
            {
                _cameraView = new CameraView
                {
                    HorizontalOptions = LayoutOptions.Fill,
                    VerticalOptions = LayoutOptions.Fill,
                    Margin = new Thickness(0)
                };
                _cameraView.MediaCaptured += OnMediaCaptured;
                _cameraView.MediaCaptureFailed += OnMediaCaptureFailed;

                // Al asignar Content, MAUI agrega el control al árbol visual
                // y ConnectHandler() se ejecuta → conecta la cámara
                CameraContainer.Content = _cameraView;
            }

            CameraContainer.IsVisible = true;
            BtnTomarFoto.IsVisible = true;
            BtnFlashButton.IsVisible = true;
            StatusFlashToIcons();
        });
    }

    /// <summary>
    /// Muestra el overlay con título, mensaje y botones contextuales.
    /// </summary>
    private void MostrarOverlayPermiso(string titulo, string mensaje, bool puedeReintentar)
    {
        MainThread.BeginInvokeOnMainThread(() =>
        {
            LblPermissionTitle.Text = titulo;
            LblPermissionMessage.Text = mensaje;

            // Solo uno de estos botones es visible a la vez
            BtnPedirPermiso.IsVisible = puedeReintentar;
            BtnGoToSettings.IsVisible = !puedeReintentar;

            CameraContainer.IsVisible = false;
            BtnTomarFoto.IsVisible = false;
            BtnFlashButton.IsVisible = false;
            PermissionDeniedOverlay.IsVisible = true;
        });
    }

    // ══════════════════════════════════════════════
    // HANDLERS DEL OVERLAY
    // ══════════════════════════════════════════════

    private async void OnPedirPermisoClicked(object sender, EventArgs e)
    {
        // Re-evaluar permisos (puede mostrar el diálogo del sistema otra vez en Android)
        await EvaluarYMostrarEstadoPermisoAsync();
    }

    private void OnGoToSettingsClicked(object sender, EventArgs e)
    {
        // Abre la pantalla de configuración de la app en el SO
        AppInfo.ShowSettingsUI();
    }

    private async void OnVolverClicked(object sender, EventArgs e)
    {
        OnPhotoCallback?.Invoke(null!);
        await Shell.Current.GoToAsync("..");
    }

    // ══════════════════════════════════════════════
    // CÁMARA — selección, captura, eventos
    // ══════════════════════════════════════════════

    private async Task SeleccionarCamaraAsync()
    {
        if (_cameraView == null) return;

        try
        {
            var cameras = await _cameraView.GetAvailableCameras(CancellationToken.None);
            var rear = cameras.FirstOrDefault(c => c.Position == CameraPosition.Rear)
                       ?? cameras.FirstOrDefault(); // fallback a cualquier cámara

            if (rear != null)
                MainThread.BeginInvokeOnMainThread(() => _cameraView.SelectedCamera = rear);
            else
                MostrarOverlayPermiso("Cámara no disponible",
                    "Este dispositivo no tiene cámara disponible.",
                    puedeReintentar: false);
        }
        catch (Exception ex)
        {
            Debug.WriteLine($"SeleccionarCamaraAsync error: {ex.Message}");
        }
    }

    private async void OnTomarFotoClicked(object sender, EventArgs e)
    {
        if (_isCapturingImage || _cameraView == null) return;

        _isCapturingImage = true;
        DynamicLayout.IsEnabled = false;

        try
        {
            _captureCancellationTokenSource?.Dispose();
            _captureCancellationTokenSource = new CancellationTokenSource();

            await MainThread.InvokeOnMainThreadAsync(async () =>
            {
                await _cameraView.CaptureImage(_captureCancellationTokenSource.Token);
            });
        }
        catch (OperationCanceledException) { }
        catch (Exception ex)
        {
            Debug.WriteLine($"OnTomarFotoClicked error: {ex}");
        }
        finally
        {
            _isCapturingImage = false;
            DynamicLayout.IsEnabled = true;
        }
    }

    private async void OnMediaCaptured(object? sender, MediaCapturedEventArgs e)
    {
        await MainThread.InvokeOnMainThreadAsync(async () =>
        {
            if (_cameraView?.IsAvailable == true)
            {
                if (e.Media != null)
                {
                    var image = new Image { Source = ImageSource.FromStream(() => e.Media) };
                    OnPhotoCallback?.Invoke(image);
                }
                await Shell.Current.GoToAsync("..");
            }
            else
            {
                await DisplayAlert("Error del dispositivo",
                    "El dispositivo no está activo", "OK");
            }
        });
    }

    private void OnMediaCaptureFailed(object sender, MediaCaptureFailedEventArgs e)
    {
        MainThread.BeginInvokeOnMainThread(() =>
        {
            _isCapturingImage = false;
            DynamicLayout.IsEnabled = true;
            _captureCancellationTokenSource?.Dispose();
            _captureCancellationTokenSource = null;
        });
    }

    // ══════════════════════════════════════════════
    // FLASH
    // ══════════════════════════════════════════════

    private void OnActiveFlashClicked(object sender, EventArgs e)
    {
        if (_cameraView == null) return;

        _cameraView.CameraFlashMode = _cameraView.CameraFlashMode switch
        {
            CameraFlashMode.Off => CameraFlashMode.On,
            CameraFlashMode.On => CameraFlashMode.Auto,
            _ => CameraFlashMode.Off
        };
        StatusFlashToIcons();
    }

    public void StatusFlashToIcons()
    {
        if (_cameraView == null) return;

        FlashIcon = _cameraView.CameraFlashMode switch
        {
            CameraFlashMode.Off => "flash_off",
            CameraFlashMode.On => "flash_on",
            CameraFlashMode.Auto => "flash_auto",
            _ => "flash_off"
        };
    }

    // ══════════════════════════════════════════════
    // ORIENTACIÓN (portrait / landscape)
    // ══════════════════════════════════════════════

    private void OnMainDisplayInfoChanged(object sender, DisplayInfoChangedEventArgs e)
    {
        if (e != null)
            UpdateLayoutOrientation(e.DisplayInfo.Orientation);
    }

    private void UpdateLayoutOrientation(DisplayOrientation orientation)
    {
        try
        {
            if (DynamicLayout == null) return;

            DynamicLayout.BatchBegin();
            DynamicLayout.RowDefinitions.Clear();
            DynamicLayout.ColumnDefinitions.Clear();

            if (orientation == DisplayOrientation.Landscape)
            {
                DynamicLayout.RowDefinitions.Add(new RowDefinition { Height = GridLength.Star });
                DynamicLayout.ColumnDefinitions.Add(new ColumnDefinition { Width = GridLength.Auto });
                DynamicLayout.ColumnDefinitions.Add(new ColumnDefinition { Width = GridLength.Star });
                DynamicLayout.ColumnDefinitions.Add(new ColumnDefinition { Width = GridLength.Auto });

                Grid.SetRow(BtnFlashButton, 0); Grid.SetColumn(BtnFlashButton, 2);
                Grid.SetRow(CameraContainer, 0); Grid.SetColumn(CameraContainer, 1);
                Grid.SetRow(BtnTomarFoto, 0); Grid.SetColumn(BtnTomarFoto, 0);
                Grid.SetRow(PermissionDeniedOverlay, 0);
                Grid.SetColumn(PermissionDeniedOverlay, 1);
            }
            else
            {
                DynamicLayout.RowDefinitions.Add(new RowDefinition { Height = GridLength.Auto });
                DynamicLayout.RowDefinitions.Add(new RowDefinition { Height = GridLength.Star });
                DynamicLayout.RowDefinitions.Add(new RowDefinition { Height = GridLength.Auto });
                DynamicLayout.ColumnDefinitions.Add(new ColumnDefinition { Width = GridLength.Star });

                Grid.SetRow(BtnFlashButton, 0); Grid.SetColumn(BtnFlashButton, 0);
                Grid.SetRow(CameraContainer, 1); Grid.SetColumn(CameraContainer, 0);
                Grid.SetRow(BtnTomarFoto, 2); Grid.SetColumn(BtnTomarFoto, 0);
                Grid.SetRow(PermissionDeniedOverlay, 1);
                Grid.SetColumn(PermissionDeniedOverlay, 0);
            }

            DynamicLayout.BatchCommit();
        }
        catch (Exception ex)
        {
            Debug.WriteLine($"UpdateLayoutOrientation error: {ex.Message}");
        }
    }
}
```

### 11.3 Página que navega y recibe la foto — `MainPage.xaml.cs`

```csharp
namespace Ejemplo_Photo_MiMediaPicker_Callback.Pages;

public partial class MainPage : ContentPage
{
    public MainPage()
    {
        InitializeComponent();
    }

    async private void OnAbrirCamaraClicked(object? sender, EventArgs e)
    {
        BtnPhoto.IsEnabled = false;

        try
        {
            // Definir callback que recibe la foto como Image
            Action<Image> resultadoCallback = async (image) =>
            {
                await this.Dispatcher.DispatchAsync(new Action(async () =>
                {
                    if (image != null) ImgPhoto.Source = image.Source;
                }));
            };

            // Pasar el callback como parámetro de navegación
            var pageParams = new ShellNavigationQueryParameters
            {
                { "OnPhotoCallback", resultadoCallback }
            };

            await Shell.Current.GoToAsync(nameof(MyMediaPickerPage), pageParams);
        }
        catch (Exception ex)
        {
            await DisplayAlert("Error", $"Ocurrió un error: {ex.Message}", "OK");
        }
        finally
        {
            BtnPhoto.IsEnabled = true;
        }
    }
}
```

### 11.4 Checklist para CameraView

```
✅ NuGet
   └── CommunityToolkit.Maui instalado y registrado en MauiProgram.cs

✅ AndroidManifest.xml (mínimo)
   ├── android.permission.CAMERA
   ├── uses-feature camera required="false"
   └── NO necesita FileProvider, file_paths.xml ni permisos de almacenamiento

✅ Info.plist (mínimo)
   └── NSCameraUsageDescription (texto descriptivo)

✅ XAML
   ├── NO declarar <toolkit:CameraView> en XAML
   ├── Usar <ContentView x:Name="CameraContainer" /> como placeholder
   ├── Botones arrancan con IsVisible="False"
   └── Overlay de permisos con botones contextuales

✅ Code-behind
   ├── CameraView? _cameraView como campo nullable
   ├── Crear CameraView solo en MostrarVisorCamara()
   ├── Verificar IsCaptureSupported ANTES de crear CameraView
   ├── Siempre llamar RequestAsync si no está Granted (no filtrar por Unknown)
   ├── ShouldShowRationale DESPUÉS de RequestAsync
   ├── Null-check en _cameraView en flash, captura, etc.
   └── Limpiar CameraView en OnDisappearing (desuscribir + null)
```
