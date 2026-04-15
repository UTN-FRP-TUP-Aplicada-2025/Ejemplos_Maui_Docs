# Permisos en .NET MAUI — Guía Completa para Desarrolladores (Cámara, GPS y PhoneDialer)

## Índice

### Parte I — Conceptos Comunes de Permisos
1. [Introducción: Permisos en .NET MAUI](#1-introducción-permisos-en-net-maui)
2. [Diferencias entre Android e iOS](#2-diferencias-entre-android-e-ios)
3. [Lógica de Permisos en C# — Patrón común](#3-lógica-de-permisos-en-c--patrón-común)
4. [Manejo de Denegación de Permisos](#4-manejo-de-denegación-de-permisos)

### Parte II — Cámara
5. [Configuración de Manifiestos (Cámara)](#5-configuración-de-manifiestos-cámara)
6. [Integración con MediaPicker de MAUI](#6-integración-con-mediapicker-de-maui)
7. [MediaPicker vs CameraView — Dos formas de usar la cámara](#7-mediapicker-vs-cameraview--dos-formas-de-usar-la-cámara)
8. [CameraView (CommunityToolkit): Consideraciones y Trampas](#8-cameraview-communitytoolkit-consideraciones-y-trampas)
9. [Ejemplo Completo: Cámara con MediaPicker](#9-ejemplo-completo-cámara-con-mediapicker)
10. [Ejemplo Completo: CameraView con permisos y overlay](#10-ejemplo-completo-cameraview-con-permisos-y-overlay)

### Parte III — GPS / Geolocalización
12. [Configuración de Manifiestos (GPS)](#12-configuración-de-manifiestos-gps)
13. [API Reference: Clases y Métodos de Geolocation](#13-api-reference-clases-y-métodos-de-geolocation)
14. [CancellationToken en GPS](#14-cancellationtoken-en-gps)
15. [Precisión: Pros, Contras y Rendimiento](#15-precisión-pros-contras-y-rendimiento)
16. [Ejemplo Completo: Página con GPS](#16-ejemplo-completo-página-con-gps)
17. [Ejemplo: GPS + Envío a Servidor con CancellationToken](#17-ejemplo-gps--envío-a-servidor-con-cancellationtoken)

### Parte IV — PhoneDialer / Llamadas / SMS
20. [¿Qué es el PhoneDialer? — No es un dispositivo, es una API](#20-qué-es-el-phonedialer--no-es-un-dispositivo-es-una-api)
21. [Esquemas URI de comunicación: tel:, sms:, mailto:](#21-esquemas-uri-de-comunicación-tel-sms-mailto)
22. [Configuración de Manifiestos (PhoneDialer y SMS)](#22-configuración-de-manifiestos-phonedialer-y-sms)
23. [PhoneDialer.Default.Open — Abrir el marcador con número precargado](#23-phonedialerdefaultopen--abrir-el-marcador-con-número-precargado)
24. [Llamada directa sin confirmación del usuario (solo Android)](#24-llamada-directa-sin-confirmación-del-usuario-solo-android)
25. [Permisos en tiempo de ejecución para CALL_PHONE](#25-permisos-en-tiempo-de-ejecución-para-call_phone)
26. [Manejo de denegación de permisos para CALL_PHONE](#26-manejo-de-denegación-de-permisos-para-call_phone)
27. [Sms.Default.ComposeAsync — Enviar SMS desde la app](#27-smsdefaultcomposeasync--enviar-sms-desde-la-app)
28. [Email: Launcher y esquema mailto](#28-email-launcher-y-esquema-mailto)
29. [Abrir WhatsApp, Telegram y otras apps de mensajería vía URI](#29-abrir-whatsapp-telegram-y-otras-apps-de-mensajería-vía-uri)
30. [API Reference: Clases y Métodos de PhoneDialer, Sms, Email y Launcher](#30-api-reference-clases-y-métodos-de-phonedialer-sms-email-y-launcher)
31. [Ejemplo Completo: Página con PhoneDialer](#31-ejemplo-completo-página-con-phonedialer)
32. [Ejemplo Completo: Página de Contacto Multicanal](#32-ejemplo-completo-página-de-contacto-multicanal)

### Parte V — Referencia
33. [Troubleshooting Unificado (Cámara + GPS + PhoneDialer + SMS)](#33-troubleshooting-unificado-cámara--gps--phonedialer--sms)
34. [Checklists de Verificación](#34-checklists-de-verificación)

---

# PARTE I — CONCEPTOS COMUNES DE PERMISOS

---

## 1. Introducción: Permisos en .NET MAUI

En .NET MAUI, el acceso a hardware del dispositivo (cámara, GPS, micrófono, etc.) requiere **permisos en tiempo de ejecución** (runtime permissions) tanto en Android como en iOS. No alcanza con declararlos en el manifiesto.

### ¿Por qué no alcanza con declarar el permiso en el manifiesto?

- **Android**: El manifiesto declara *qué* permisos puede pedir la app. Pero el usuario debe **aceptar explícitamente** en tiempo de ejecución.
- **iOS**: Los permisos se declaran en `Info.plist` con una descripción obligatoria. El sistema muestra un diálogo nativo **una sola vez**. Si el usuario lo deniega, no se puede volver a pedir desde la app.

### Versiones de Android y su impacto en permisos

| API Level | Versión Android | Cambio relevante |
|-----------|----------------|-------------------|
| 23 (M) | 6.0 | Runtime permissions obligatorios |
| 29 (Q) | 10 | Scoped Storage; `ACCESS_BACKGROUND_LOCATION` separado |
| 30 (R) | 11 | `WRITE_EXTERNAL_STORAGE` ignorado |
| 31 (S) | 12 | El usuario puede elegir ubicación "aproximada" vs "precisa" |
| 33 (Tiramisu) | 13 | `READ_MEDIA_IMAGES` reemplaza `READ_EXTERNAL_STORAGE` |

---

## 2. Diferencias entre Android e iOS

### Android

```
┌────────────────────────────────────────────────────────┐
│                  PRIMERA SOLICITUD                      │
│                                                        │
│   Sistema muestra diálogo nativo:                      │
│   "¿Permitir que [App] acceda a [recurso]?"            │
│                                                        │
│     [Mientras se usa la app]  [Solo esta vez]           │
│                 [No permitir]                           │
└────────────────────────────────────────────────────────┘
         │                              │
    ✅ Granted                     ❌ Denied
         │                              │
    Usar recurso              Segunda solicitud posible
                                        │
                              ┌─────────┴──────────┐
                              │ Si marca "No volver │
                              │ a preguntar" →      │
                              │ Denegado permanente  │
                              └────────────────────┘
```

- Se puede pedir el permiso **varias veces** (salvo que marque "No volver a preguntar").
- Se puede detectar la denegación permanente con `ShouldShowRationale` / `ShouldShowRequestPermissionRationale`.
- Se puede redirigir al usuario a Configuración del sistema.

### iOS

```
┌────────────────────────────────────────────────────────┐
│               PRIMERA Y ÚNICA SOLICITUD                 │
│                                                        │
│   "[App] quiere acceder a [recurso]"                   │
│   "[Texto descriptivo de Info.plist]"                  │
│                                                        │
│              [Permitir]  [No permitir]                  │
│    (Ubicación agrega: [Permitir una vez])               │
└────────────────────────────────────────────────────────┘
         │                              │
    ✅ Granted                     ❌ Denied
         │                              │
    Usar recurso             NO se puede volver a pedir
                              → Redirigir a Configuración
```

- Solo se puede pedir **una vez**. Después el sistema recuerda la decisión.
- Si el usuario deniega, la única opción es enviarlo a **Configuración > Privacidad**.
- El texto descriptivo en `Info.plist` es lo que ve el usuario. **Si falta, la app crashea.**
- Para ubicación, iOS ofrece 3 opciones: **una vez**, **al usar la app**, **no permitir**.

---

## 3. Lógica de Permisos en C# — Patrón común

Tanto para cámara como para GPS, la lógica de permisos sigue el mismo patrón. Hay que tener en cuenta dos trampas de Android:

- **Trampa 1**: `CheckStatusAsync` devuelve `Denied` (no `Unknown`) cuando el permiso **nunca fue solicitado**.
- **Trampa 2**: `ShouldShowRationale` solo es confiable **después** de llamar a `RequestAsync`.

### 3.1 Patrón recomendado (funciona para cualquier permiso)

```csharp
// T puede ser: Permissions.Camera, Permissions.LocationWhenInUse, etc.
private async Task<bool> EvaluarPermisoAsync<T>() where T : Permissions.BasePermission, new()
{
    // Paso 1: Verificar estado actual
    var status = await Permissions.CheckStatusAsync<T>();

    if (status == PermissionStatus.Granted)
        return true;

    // Paso 2: Pedir permiso
    // NOTA: En Android, CheckStatusAsync devuelve Denied (no Unknown) cuando
    // el permiso nunca fue solicitado. Por eso siempre llamamos a RequestAsync
    // si no está Granted.
    status = await Permissions.RequestAsync<T>();

    if (status == PermissionStatus.Granted)
        return true;

    // Paso 3: Restringido (control parental / MDM en iOS)
    if (status == PermissionStatus.Restricted)
    {
        await Application.Current!.MainPage!.DisplayAlert(
            "Acceso restringido",
            "El acceso está restringido por una política del dispositivo.",
            "Entendido");
        return false;
    }

    // Paso 4: Denegado — ¿se puede volver a pedir?
    // ShouldShowRationale es confiable ahora porque ya llamamos a RequestAsync
    bool puedeReintentar = false;

#if ANDROID
    puedeReintentar = Permissions.ShouldShowRationale<T>();
#endif

    if (puedeReintentar)
    {
        await Application.Current!.MainPage!.DisplayAlert(
            "Permiso necesario",
            "Necesitamos este permiso para la funcionalidad. " +
            "Podés intentar concederlo.",
            "OK");
    }
    else
    {
        // Denegación permanente (Android: "No volver a preguntar" / iOS: siempre)
        bool irAConfig = await Application.Current!.MainPage!.DisplayAlert(
            "Permiso denegado",
            "Para usar esta función, habilitá el permiso desde " +
            "los ajustes de la aplicación.",
            "Ir a Configuración", "Cancelar");

        if (irAConfig)
            AppInfo.ShowSettingsUI();
    }

    return false;
}
```

### 3.2 ¿Por qué `CheckStatusAsync` devuelve `Denied` y no `Unknown` en Android?

| Situación | iOS | Android |
|---|---|---|
| Permiso nunca solicitado | `Unknown` | `Denied` ⚠️ |
| Usuario dijo "No" | `Denied` | `Denied` |
| "No volver a preguntar" | N/A (siempre permanente) | `Denied` |
| Permiso concedido | `Granted` | `Granted` |
| Control parental / MDM | `Restricted` | N/A |

**Consecuencia**: No uses `if (status == PermissionStatus.Unknown)` como condición para llamar a `RequestAsync`. Siempre llamá a `RequestAsync` si no está `Granted`.

```csharp
// ❌ INCORRECTO — En Android nunca entra al if
if (status == PermissionStatus.Unknown)
    status = await Permissions.RequestAsync<T>();  // Nunca se ejecuta en Android

// ✅ CORRECTO — Funciona en ambas plataformas
if (status != PermissionStatus.Granted)
    status = await Permissions.RequestAsync<T>();
```

### 3.3 `ShouldShowRationale`: cuándo es confiable

`Permissions.ShouldShowRationale<T>()` en Android devuelve:

| Situación | Valor | ¿Podemos re-pedir? |
|---|---|---|
| Permiso **nunca** solicitado | `false` ⚠️ | Sí |
| Usuario dijo "No" (sin marcar "No volver a preguntar") | `true` | Sí |
| Usuario marcó "No volver a preguntar" | `false` | No → ir a Configuración |

**El problema**: devuelve `false` tanto cuando **nunca se pidió** como cuando es **denegación permanente**. La solución es llamarlo **después** de `RequestAsync`:

```csharp
// Primero pedir
status = await Permissions.RequestAsync<T>();

// Si fue denegado, ahora ShouldShowRationale es confiable
if (status == PermissionStatus.Denied)
{
    bool puedeReintentar = false;
#if ANDROID
    puedeReintentar = Permissions.ShouldShowRationale<T>();
#endif
    // Si false → denegación permanente (ir a Configuración)
    // Si true → puede volver a pedir
}
```

---

## 4. Manejo de Denegación de Permisos

### 4.1 Diagrama de flujo completo (aplica a cámara y GPS)

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
                  recurso   │  RequestAsync     │
                            └────────┬──────────┘
                                     │
                            ┌────────▼─────────┐
                            │  ¿Granted?       │
                            └────────┬─────────┘
                               Sí ──┤── No
                              │     │
                         ✅ Usar   ┌▼───────────────────────┐
                         recurso   │ ¿Restricted?           │
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

### 4.2 Tabla resumen: Flujo de denegación por plataforma

| Escenario | Android | iOS |
|---|---|---|
| **Primera vez** | Muestra diálogo nativo del sistema | Muestra diálogo nativo del sistema |
| **Denegado una vez** | Vuelve a pedir (`ShouldShowRationale` = true) | No puede volver a pedir |
| **"No volver a preguntar"** | `ShouldShowRationale` = false → ir a Configuración | N/A (siempre permanente) |
| **Denegado permanente** | `AppInfo.ShowSettingsUI()` | `AppInfo.ShowSettingsUI()` |
| **Restringido (parental/MDM)** | N/A | `PermissionStatus.Restricted` → informar |

### 4.3 Abrir Configuración del sistema

```csharp
// Multiplataforma (recomendado)
AppInfo.ShowSettingsUI();

// iOS específico (alternativa)
#if IOS
UIKit.UIApplication.SharedApplication.OpenUrl(
    new Foundation.NSUrl(UIKit.UIApplication.OpenSettingsUrlString));
#endif

// Android específico (abre la pantalla de la app directamente)
#if ANDROID
var intent = new Android.Content.Intent(
    Android.Provider.Settings.ActionApplicationDetailsSettings);
intent.SetData(Android.Net.Uri.FromParts(
    "package", Platform.CurrentActivity!.PackageName!, null));
intent.AddFlags(Android.Content.ActivityFlags.NewTask);
Platform.CurrentActivity.StartActivity(intent);
#endif
```

---

# PARTE II — CÁMARA

---

## 5. Configuración de Manifiestos (Cámara)

### 5.1 Android — `Platforms/Android/AndroidManifest.xml`

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

### 5.2 Android — `Platforms/Android/Resources/xml/file_paths.xml`

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

### 5.3 iOS — `Platforms/iOS/Info.plist`

Agregá estas claves dentro del tag `<dict>`:

```xml
<!-- ══════════════════════════════════════════════ -->
<!-- CÁMARA                                        -->
<!-- ══════════════════════════════════════════════ -->

<!--
    OBLIGATORIO. Sin esta clave la app crashea al intentar
    abrir la cámara. El texto es lo que ve el usuario en el
    diálogo de permisos.
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

### 5.4 Manifiestos mínimos según enfoque

#### Solo MediaPicker (captura de foto con app del sistema)

**Android:**
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

**iOS:**
```xml
<key>NSCameraUsageDescription</key>
<string>La app necesita acceso a la cámara para tomar fotos.</string>
```

#### Solo CameraView (cámara embebida — CommunityToolkit)

**Android:**
```xml
<?xml version="1.0" encoding="utf-8"?>
<manifest xmlns:android="http://schemas.android.com/apk/res/android">
    <uses-permission android:name="android.permission.CAMERA" />
    <uses-feature android:name="android.hardware.camera" android:required="false" />

    <!-- NO necesita FileProvider, file_paths.xml ni permisos de almacenamiento -->
    <application android:allowBackup="true" android:label="@string/app_name" />
</manifest>
```

**iOS:**
```xml
<key>NSCameraUsageDescription</key>
<string>La app necesita acceso a la cámara para tomar fotos.</string>
```

> **Nota**: Si solo tomás fotos y las guardás en `FileSystem.AppDataDirectory` (directorio privado de la app), NO necesitás permisos de almacenamiento ni permisos de galería.

### 5.5 Permisos runtime para cámara — Android (código nativo)

Cuando usás `MediaPicker`, puede ser necesario solicitar permisos con el API nativa de Android (`ActivityCompat`):

```csharp
#if ANDROID
using Android;
using AndroidX.Core.App;
using AndroidX.Core.Content;

private async Task<bool> RequestCameraPermissionsAsync()
{
    var activity = Platform.CurrentActivity
        ?? throw new InvalidOperationException("No hay Activity activa.");

    // Determinar qué permisos pedir según la versión
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
        return true; // Android < 6: permisos otorgados en instalación
    }

    // Verificar si ya están concedidos
    bool allGranted = requiredPermissions.All(p =>
        ContextCompat.CheckSelfPermission(activity, p)
            == (int)Android.Content.PM.Permission.Granted);

    if (allGranted)
        return true;

    // Solicitar permisos
    ActivityCompat.RequestPermissions(activity, requiredPermissions, requestCode: 100);
    await Task.Delay(5000); // Ver sección sobre TaskCompletionSource para mejor alternativa

    // Re-verificar
    allGranted = requiredPermissions.All(p =>
        ContextCompat.CheckSelfPermission(activity, p)
            == (int)Android.Content.PM.Permission.Granted);

    return allGranted;
}
#endif
```

### 5.6 Mejora: Reemplazar `Task.Delay` con `TaskCompletionSource` (Android)

El `Task.Delay` tiene problemas: si el usuario tarda más de 5 segundos, la app asume denegación. La solución correcta:

#### Paso 1 — Modificar `MainActivity.cs`

```csharp
public class MainActivity : MauiAppCompatActivity
{
    public static event Action<int, string[], Android.Content.PM.Permission[]>?
        PermissionsResultReceived;

    public override void OnRequestPermissionsResult(
        int requestCode, string[] permissions,
        Android.Content.PM.Permission[] grantResults)
    {
        base.OnRequestPermissionsResult(requestCode, permissions, grantResults);
        PermissionsResultReceived?.Invoke(requestCode, permissions, grantResults);
    }
}
```

#### Paso 2 — Método helper

```csharp
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

        var completedTask = await Task.WhenAny(tcs.Task, Task.Delay(timeoutMs));

        if (completedTask == tcs.Task)
            return tcs.Task.Result;

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

---

## 6. Integración con MediaPicker de MAUI

MAUI incluye `MediaPicker` que abstrae la captura de fotos. **Igual necesitás los permisos configurados**.

### 6.1 Capturar foto con MediaPicker

```csharp
private async Task OpenCameraAsync()
{
    try
    {
        if (!MediaPicker.Default.IsCaptureSupported)
        {
            await DisplayAlert("Error", "Este dispositivo no tiene cámara.", "OK");
            return;
        }

        var photo = await MediaPicker.Default.CapturePhotoAsync(new MediaPickerOptions
        {
            Title = "Tomar foto de infracción"  // Solo tiene efecto en Android
        });

        if (photo == null)
        {
            // El usuario canceló
            return;
        }

        // Opción A: Leer como stream y mostrar en Image
        using var stream = await photo.OpenReadAsync();
        FotoPreview.Source = ImageSource.FromStream(() =>
        {
            var ms = new MemoryStream();
            stream.CopyTo(ms);
            ms.Position = 0;
            return ms;
        });

        // Opción B: Guardar en directorio privado de la app
        var filePath = Path.Combine(
            FileSystem.AppDataDirectory,
            $"infraccion_{DateTime.Now:yyyyMMdd_HHmmss}.jpg");

        using var saveStream = File.OpenWrite(filePath);
        using var sourceStream = await photo.OpenReadAsync();
        await sourceStream.CopyToAsync(saveStream);

        Console.WriteLine($"[APP] Foto guardada en: {filePath}");
    }
    catch (FeatureNotSupportedException)
    {
        await DisplayAlert("Error", "Captura de fotos no soportada.", "OK");
    }
    catch (PermissionException)
    {
        await DisplayAlert("Error", "No hay permisos para usar la cámara.", "OK");
    }
    catch (Exception ex)
    {
        Console.WriteLine($"[APP] Error al capturar foto: {ex.Message}");
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

        FotoPreview.Source = ImageSource.FromFile(photo.FullPath);
    }
    catch (Exception ex)
    {
        await DisplayAlert("Error", $"Error al seleccionar foto: {ex.Message}", "OK");
    }
}
```

---

## 7. MediaPicker vs CameraView — Dos formas de usar la cámara

### 7.1 Comparación general

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

### 7.2 Tabla comparativa detallada

| Aspecto | MediaPicker | CameraView (CommunityToolkit) |
|---|---|---|
| **Paquete** | Incluido en .NET MAUI | NuGet: `CommunityToolkit.Maui` |
| **Cómo funciona** | Abre la app de cámara del **sistema** | Renderiza la cámara **dentro** de tu página |
| **UI** | No controlás la interfaz | 100% personalizable |
| **FileProvider** | **Obligatorio** | **No necesario** |
| **`file_paths.xml`** | **Obligatorio** | **No necesario** |
| **Permisos almacenamiento** | Puede necesitarlos | **No necesarios** si guardás en `AppDataDirectory` |
| **Permiso de cámara** | `android.permission.CAMERA` + `NSCameraUsageDescription` | Igual |
| **Flash** | No controlás | Controlable por código |
| **Cámara frontal/trasera** | No controlás | Controlable por código |
| **Galería** | `PickPhotoAsync` incluido | No incluido (solo captura) |

### 7.3 ¿Cuándo usar cada uno?

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

### 7.4 ¿Por qué MediaPicker necesita FileProvider y CameraView no?

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


CameraView (una sola app):

Tu App
  │
  │── CameraView renderiza la cámara
  │── CaptureImage() → Stream en memoria
  │── Tu código guarda donde quiera
  │
  ✅ Todo pasa dentro de tu proceso. No se necesita FileProvider.
```

---

## 8. CameraView (CommunityToolkit): Consideraciones y Trampas

### 8.1 Trampa crítica: `ConnectHandler` y el crash sin cámara

El error más común al usar `CameraView`:

```
Unhandled managed exception:
  No camera available on device (CommunityToolkit.Maui.Core.CameraException)
    at CommunityToolkit.Maui.Core.CameraManager.ConnectCamera(CancellationToken)
    at CommunityToolkit.Maui.Core.Handlers.CameraViewHandler.ConnectHandler(UIView)
```

**¿Por qué pasa?** Cuando MAUI procesa el XAML, `ConnectHandler` se ejecuta automáticamente:

```
XAML parseado
    └─ Crea instancia de CameraView
         └─ MAUI busca el Handler nativo
              └─ CameraViewHandler.ConnectHandler()
                   └─ CameraManager.ConnectCamera()
                        ├─ Hay cámara → OK
                        └─ NO hay cámara → CRASH (SIGABRT)
```

**`IsVisible="False"` NO lo evita.** Solo controla si el control se dibuja, pero el handler ya se conectó.

```
┌──────────────────────────────────────────────────────────────┐
│  INCORRECTO — IsVisible="False" NO evita ConnectHandler      │
│                                                              │
│  <toolkit:CameraView x:Name="Camera" IsVisible="False" />   │
│  Resultado: ConnectHandler() → busca cámara → CRASH          │
└──────────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────────┐
│  CORRECTO — Usar ContentView como contenedor vacío           │
│                                                              │
│  <ContentView x:Name="CameraContainer" />                    │
│                                                              │
│  // En código C#, DESPUÉS de verificar que hay cámara:       │
│  CameraContainer.Content = new CameraView { ... };           │
└──────────────────────────────────────────────────────────────┘
```

### 8.2 Verificar hardware antes de crear el CameraView

```csharp
if (MediaPicker.Default.IsCaptureSupported)
{
    // Hay cámara → crear CameraView dinámicamente
    MostrarVisorCamara();
}
else
{
    // No hay cámara (simulador, tablet sin cámara)
    MostrarOverlayPermiso("Cámara no disponible",
        "Este dispositivo no tiene cámara.",
        puedeReintentar: false);
}
```

### 8.3 Flujo completo recomendado para CameraView

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
    │               ├─ Sí → (mismo check de IsCaptureSupported)
    │               ├─ Restricted → Overlay "Acceso restringido"
    │               └─ Denied → ShouldShowRationale?
    │                            ├─ true → Overlay + botón "Pedir permiso"
    │                            └─ false → Overlay + botón "Abrir configuración"
    │
    └─ 2. OnNavigatedTo → SeleccionarCamaraAsync (trasera por defecto)
```

### 8.4 Limpieza del CameraView

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

## 9. Ejemplo Completo: Cámara con MediaPicker

### 9.1 XAML — `CameraPage.xaml`

```xml
<?xml version="1.0" encoding="utf-8" ?>
<ContentPage xmlns="http://schemas.microsoft.com/dotnet/2021/maui"
             xmlns:x="http://schemas.microsoft.com/winfx/2009/xaml"
             x:Class="MiApp.Views.CameraPage"
             Title="Captura de Foto">

    <ScrollView Padding="20">
        <VerticalStackLayout Spacing="15">

            <Label x:Name="CameraStatusLabel"
                   Text="Cámara: verificando..."
                   FontSize="14"
                   TextColor="Gray" />

            <Border StrokeShape="RoundRectangle 10"
                    Stroke="LightGray"
                    HeightRequest="300"
                    BackgroundColor="#F5F5F5">
                <Image x:Name="FotoPreview"
                       Aspect="AspectFit" />
            </Border>

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

            <Button x:Name="BtnReintentarCamara"
                    Text="Reintentar permisos"
                    BackgroundColor="#FF9800"
                    TextColor="White"
                    IsVisible="False"
                    Clicked="OnReintentarCamara_Clicked" />

            <Label x:Name="LblFotoInfo"
                   Text=""
                   FontSize="12"
                   TextColor="Gray" />

        </VerticalStackLayout>
    </ScrollView>
</ContentPage>
```

### 9.2 Code-behind — `CameraPage.xaml.cs`

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
#if ANDROID || IOS
        var granted = await RequestCameraPermissionsAsync();
        if (granted) await CapturePhotoAsync();
#else
        await CapturePhotoAsync();
#endif
    }

    private async Task CapturePhotoAsync()
    {
        try
        {
            var photo = await MediaPicker.Default.CapturePhotoAsync();
            if (photo == null) return;
            await LoadPhotoAsync(photo);
        }
        catch (Exception ex)
        {
            Console.WriteLine($"[APP] Error captura: {ex}");
            await DisplayAlert("Error", $"Error al capturar: {ex.Message}", "OK");
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
            await DisplayAlert("Error", $"Error al seleccionar: {ex.Message}", "OK");
        }
    }

    private async Task LoadPhotoAsync(FileResult photo)
    {
        var localPath = Path.Combine(
            FileSystem.AppDataDirectory,
            $"foto_{DateTime.Now:yyyyMMdd_HHmmss}.jpg");

        using (var sourceStream = await photo.OpenReadAsync())
        using (var localFile = File.OpenWrite(localPath))
        {
            await sourceStream.CopyToAsync(localFile);
        }

        FotoPreview.Source = ImageSource.FromFile(localPath);

        var fileInfo = new FileInfo(localPath);
        LblFotoInfo.Text = $"Archivo: {fileInfo.Name}\n" +
                           $"Tamaño: {fileInfo.Length / 1024} KB\n" +
                           $"Guardado en: {localPath}";
    }

#if ANDROID
    private async Task<bool> RequestCameraPermissionsAsync()
    {
        try
        {
            var activity = Platform.CurrentActivity
                ?? throw new InvalidOperationException("No hay Activity activa.");

            string[] requiredPermissions;

            if (Android.OS.Build.VERSION.SdkInt >= Android.OS.BuildVersionCodes.Tiramisu)
                requiredPermissions = new[] { Manifest.Permission.Camera, Manifest.Permission.ReadMediaImages };
            else if (Android.OS.Build.VERSION.SdkInt >= Android.OS.BuildVersionCodes.M)
                requiredPermissions = new[] { Manifest.Permission.Camera, Manifest.Permission.ReadExternalStorage, Manifest.Permission.WriteExternalStorage };
            else
                return true;

            bool allGranted = requiredPermissions.All(p =>
                ContextCompat.CheckSelfPermission(activity, p) == (int)Android.Content.PM.Permission.Granted);

            if (allGranted) return true;

            ActivityCompat.RequestPermissions(activity, requiredPermissions, 100);
            await Task.Delay(5000);

            allGranted = requiredPermissions.All(p =>
                ContextCompat.CheckSelfPermission(activity, p) == (int)Android.Content.PM.Permission.Granted);

            if (allGranted) return true;

            bool permanentlyDenied = requiredPermissions.Any(p =>
                ContextCompat.CheckSelfPermission(activity, p) != (int)Android.Content.PM.Permission.Granted
                && !ActivityCompat.ShouldShowRequestPermissionRationale(activity, p));

            if (permanentlyDenied)
            {
                bool goToSettings = await DisplayAlert("Permiso requerido",
                    "Denegaste el permiso permanentemente.\nHabilitalo desde Configuración > Permisos.",
                    "Ir a Configuración", "Cancelar");
                if (goToSettings)
                {
                    var intent = new Android.Content.Intent(Android.Provider.Settings.ActionApplicationDetailsSettings);
                    intent.SetData(Android.Net.Uri.FromParts("package", activity.PackageName!, null));
                    intent.AddFlags(Android.Content.ActivityFlags.NewTask);
                    activity.StartActivity(intent);
                }
            }

            CameraStatusLabel.Text = "Cámara: permisos denegados";
            CameraStatusLabel.TextColor = Colors.Red;
            BtnReintentarCamara.IsVisible = true;
            return false;
        }
        catch (Exception ex)
        {
            Console.WriteLine($"[APP] Camera permission error: {ex}");
            BtnReintentarCamara.IsVisible = true;
            return false;
        }
    }

#elif IOS
    private async Task<bool> RequestCameraPermissionsAsync()
    {
        var status = await Permissions.CheckStatusAsync<Permissions.Camera>();
        if (status == PermissionStatus.Granted) return true;

        if (status == PermissionStatus.Unknown)
        {
            status = await Permissions.RequestAsync<Permissions.Camera>();
            if (status == PermissionStatus.Granted) return true;
        }

        if (status == PermissionStatus.Restricted)
        {
            await DisplayAlert("Cámara restringida",
                "Acceso bloqueado por controles parentales o políticas.", "Entendido");
            return false;
        }

        bool goToSettings = await DisplayAlert("Permiso de cámara requerido",
            "Habilitá la cámara desde Configuración > Privacidad > Cámara.",
            "Ir a Configuración", "Cancelar");

        if (goToSettings)
            UIKit.UIApplication.SharedApplication.OpenUrl(
                new Foundation.NSUrl(UIKit.UIApplication.OpenSettingsUrlString));

        CameraStatusLabel.Text = "Cámara: permisos denegados";
        BtnReintentarCamara.IsVisible = true;
        return false;
    }
#endif
}
```

---

## 10. Ejemplo Completo: CameraView con permisos y overlay

Este ejemplo usa `CameraView` del CommunityToolkit con creación dinámica, overlay de permisos, flash, y navegación con callback.

### 10.1 XAML — `MyMediaPickerPage.xaml`

```xml
<?xml version="1.0" encoding="utf-8" ?>
<ContentPage xmlns="http://schemas.microsoft.com/dotnet/2021/maui"
             xmlns:x="http://schemas.microsoft.com/winfx/2009/xaml"
             x:Class="Ejemplo_Photo_MiMediaPicker_Callback.Pages.MyMediaPickerPage"
             Title="Camera Page"
             BackgroundColor="#000000">

    <!--
        NOTA: No se declara <toolkit:CameraView> en XAML.
        Se crea dinámicamente desde C# para evitar ConnectHandler crash.
    -->

    <Grid x:Name="DynamicLayout" RowDefinitions="Auto,*,Auto" ColumnDefinitions="*"
          HorizontalOptions="Fill" VerticalOptions="Fill">

        <Button x:Name="BtnFlashButton" Grid.Row="0" Grid.Column="0"
                Clicked="OnActiveFlashClicked"
                IsVisible="False"
                HorizontalOptions="Center" VerticalOptions="Center"
                Background="#aa000000" CornerRadius="8">
            <Button.ImageSource>
                <FontImageSource Glyph="{Binding FlashIcon}" FontFamily="MaterialIconsOutlined" />
            </Button.ImageSource>
        </Button>

        <!-- Contenedor vacío — CameraView se crea desde código -->
        <ContentView x:Name="CameraContainer" Grid.Row="1" Grid.Column="0"
                     IsVisible="False"
                     HorizontalOptions="Fill" VerticalOptions="Fill" />

        <!-- Overlay de permisos -->
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
                       FontSize="20" FontAttributes="Bold"
                       TextColor="White" HorizontalTextAlignment="Center"
                       HorizontalOptions="Center"/>

                <Label x:Name="LblPermissionMessage"
                       Text="Para tomar fotos necesitamos acceso a la cámara."
                       FontSize="14" TextColor="#CCCCCC"
                       HorizontalTextAlignment="Center"
                       HorizontalOptions="Center"/>

                <Button x:Name="BtnPedirPermiso"
                        Text="Pedir permiso"
                        Clicked="OnPedirPermisoClicked"
                        BackgroundColor="#512BD4" TextColor="White"
                        CornerRadius="8" Padding="20,12"
                        HorizontalOptions="Center"
                        IsVisible="False"/>

                <Button x:Name="BtnGoToSettings"
                        Text="Abrir configuración"
                        Clicked="OnGoToSettingsClicked"
                        BackgroundColor="#512BD4" TextColor="White"
                        CornerRadius="8" Padding="20,12"
                        HorizontalOptions="Center"/>

                <Button Text="Volver"
                        Clicked="OnVolverClicked"
                        BackgroundColor="Transparent"
                        TextColor="#AAAAAA" BorderColor="#555555"
                        BorderWidth="1" CornerRadius="8"
                        Padding="20,12"
                        HorizontalOptions="Center"/>
            </VerticalStackLayout>
        </Grid>

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

### 10.2 Code-behind — `MyMediaPickerPage.xaml.cs`

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
    private CameraView? _cameraView;

    public Action<Image>? OnPhotoCallback { get; set; }

    private string _flashIcon = "flash_off";
    public string FlashIcon
    {
        get => _flashIcon;
        set { if (_flashIcon != value) { _flashIcon = value; OnPropertyChanged(nameof(FlashIcon)); } }
    }

    public MyMediaPickerPage()
    {
        InitializeComponent();
        BindingContext = this;
    }

    protected override async void OnAppearing()
    {
        base.OnAppearing();
        DeviceDisplay.MainDisplayInfoChanged += OnMainDisplayInfoChanged;
        UpdateLayoutOrientation(DeviceDisplay.MainDisplayInfo.Orientation);
        await EvaluarYMostrarEstadoPermisoAsync();
    }

    protected override void OnDisappearing()
    {
        base.OnDisappearing();
        try
        {
            DeviceDisplay.MainDisplayInfoChanged -= OnMainDisplayInfoChanged;
            _captureCancellationTokenSource?.Cancel();

            if (_cameraView != null)
            {
                _cameraView.MediaCaptured -= OnMediaCaptured;
                _cameraView.MediaCaptureFailed -= OnMediaCaptureFailed;
                CameraContainer.Content = null;
                _cameraView = null;
            }
        }
        catch (Exception ex) { Debug.WriteLine($"OnDisappearing error: {ex.Message}"); }

#if ANDROID
        var activity = Platform.CurrentActivity;
        if (activity != null)
            activity.RequestedOrientation = Android.Content.PM.ScreenOrientation.Unspecified;
#endif
    }

    protected override async void OnNavigatedTo(NavigatedToEventArgs args)
    {
        base.OnNavigatedTo(args);
        var status = await Permissions.CheckStatusAsync<Permissions.Camera>();
        if (status == PermissionStatus.Granted && _cameraView != null)
            await SeleccionarCamaraAsync();
    }

    // ══════════════════════════════════════════════
    // PERMISOS
    // ══════════════════════════════════════════════

    private async Task EvaluarYMostrarEstadoPermisoAsync()
    {
        var status = await Permissions.CheckStatusAsync<Permissions.Camera>();

        if (status == PermissionStatus.Granted)
        {
            if (MediaPicker.Default.IsCaptureSupported)
                MostrarVisorCamara();
            else
                MostrarOverlayPermiso("Cámara no disponible",
                    "Este dispositivo no tiene cámara.", puedeReintentar: false);
            return;
        }

        status = await Permissions.RequestAsync<Permissions.Camera>();

        if (status == PermissionStatus.Granted)
        {
            if (MediaPicker.Default.IsCaptureSupported)
                MostrarVisorCamara();
            else
                MostrarOverlayPermiso("Cámara no disponible",
                    "Este dispositivo no tiene cámara.", puedeReintentar: false);
            return;
        }

        if (status == PermissionStatus.Restricted)
        {
            MostrarOverlayPermiso("Acceso restringido",
                "El acceso a la cámara está restringido por una política del dispositivo.",
                puedeReintentar: false);
            return;
        }

        bool puedeReintentar = false;
#if ANDROID
        puedeReintentar = Permissions.ShouldShowRationale<Permissions.Camera>();
#endif

        MostrarOverlayPermiso(
            titulo: puedeReintentar ? "Permiso de cámara necesario" : "Acceso denegado",
            mensaje: puedeReintentar
                ? "Para tomar fotos necesitamos acceso a la cámara."
                : "Habilitá la cámara desde los ajustes de la aplicación.",
            puedeReintentar: puedeReintentar);
    }

    // ══════════════════════════════════════════════
    // OVERLAY / VISOR
    // ══════════════════════════════════════════════

    private void MostrarVisorCamara()
    {
        MainThread.BeginInvokeOnMainThread(() =>
        {
            PermissionDeniedOverlay.IsVisible = false;
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
                CameraContainer.Content = _cameraView;
            }
            CameraContainer.IsVisible = true;
            BtnTomarFoto.IsVisible = true;
            BtnFlashButton.IsVisible = true;
            StatusFlashToIcons();
        });
    }

    private void MostrarOverlayPermiso(string titulo, string mensaje, bool puedeReintentar)
    {
        MainThread.BeginInvokeOnMainThread(() =>
        {
            LblPermissionTitle.Text = titulo;
            LblPermissionMessage.Text = mensaje;
            BtnPedirPermiso.IsVisible = puedeReintentar;
            BtnGoToSettings.IsVisible = !puedeReintentar;
            CameraContainer.IsVisible = false;
            BtnTomarFoto.IsVisible = false;
            BtnFlashButton.IsVisible = false;
            PermissionDeniedOverlay.IsVisible = true;
        });
    }

    // ══════════════════════════════════════════════
    // HANDLERS
    // ══════════════════════════════════════════════

    private async void OnPedirPermisoClicked(object sender, EventArgs e)
        => await EvaluarYMostrarEstadoPermisoAsync();

    private void OnGoToSettingsClicked(object sender, EventArgs e)
        => AppInfo.ShowSettingsUI();

    private async void OnVolverClicked(object sender, EventArgs e)
    {
        OnPhotoCallback?.Invoke(null!);
        await Shell.Current.GoToAsync("..");
    }

    private async Task SeleccionarCamaraAsync()
    {
        if (_cameraView == null) return;
        try
        {
            var cameras = await _cameraView.GetAvailableCameras(CancellationToken.None);
            var rear = cameras.FirstOrDefault(c => c.Position == CameraPosition.Rear)
                       ?? cameras.FirstOrDefault();
            if (rear != null)
                MainThread.BeginInvokeOnMainThread(() => _cameraView.SelectedCamera = rear);
        }
        catch (Exception ex) { Debug.WriteLine($"SeleccionarCamaraAsync error: {ex.Message}"); }
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
        catch (Exception ex) { Debug.WriteLine($"OnTomarFotoClicked error: {ex}"); }
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
            if (_cameraView?.IsAvailable == true && e.Media != null)
            {
                var image = new Image { Source = ImageSource.FromStream(() => e.Media) };
                OnPhotoCallback?.Invoke(image);
            }
            await Shell.Current.GoToAsync("..");
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
    // ORIENTACIÓN
    // ══════════════════════════════════════════════

    private void OnMainDisplayInfoChanged(object sender, DisplayInfoChangedEventArgs e)
    {
        if (e != null) UpdateLayoutOrientation(e.DisplayInfo.Orientation);
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
                Grid.SetRow(PermissionDeniedOverlay, 0); Grid.SetColumn(PermissionDeniedOverlay, 1);
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
                Grid.SetRow(PermissionDeniedOverlay, 1); Grid.SetColumn(PermissionDeniedOverlay, 0);
            }

            DynamicLayout.BatchCommit();
        }
        catch (Exception ex) { Debug.WriteLine($"UpdateLayoutOrientation error: {ex.Message}"); }
    }
}
```

### 10.3 Página que navega y recibe la foto — `MainPage.xaml.cs`

```csharp
namespace Ejemplo_Photo_MiMediaPicker_Callback.Pages;

public partial class MainPage : ContentPage
{
    public MainPage() { InitializeComponent(); }

    async private void OnAbrirCamaraClicked(object? sender, EventArgs e)
    {
        BtnPhoto.IsEnabled = false;
        try
        {
            Action<Image> resultadoCallback = async (image) =>
            {
                await this.Dispatcher.DispatchAsync(new Action(async () =>
                {
                    if (image != null) ImgPhoto.Source = image.Source;
                }));
            };

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
        finally { BtnPhoto.IsEnabled = true; }
    }
}
```

---

# PARTE III — GPS / GEOLOCALIZACIÓN

---

## 11. Niveles de Ubicación: Aproximada, Precisa y Segundo Plano

### 11.1 Ubicación Aproximada — `ACCESS_COARSE_LOCATION`

**Precisión**: ~1-3 km (triangulación WiFi / torres celulares, sin GPS).

**Permiso MAUI**: No hay un `Permissions.LocationCoarse` directo. Se controla desde el manifiesto Android: si declarás solo `ACCESS_COARSE_LOCATION` sin `ACCESS_FINE_LOCATION`, MAUI usa ubicación aproximada.

**Ejemplos de uso**:

| App | Funcionalidad | ¿Por qué alcanza aproximada? |
|-----|--------------|-------------------------------|
| App de clima | Pronóstico local | No importa la esquina exacta |
| App de noticias | Noticias regionales | Solo necesita "Tucumán" o "Buenos Aires" |
| App de tiendas | Lista de sucursales | Alcanza con el barrio/zona |
| App de idioma | Auto-detectar región | Solo necesita país o provincia |

```csharp
// AndroidManifest.xml — SOLO aproximada
// <uses-permission android:name="android.permission.ACCESS_COARSE_LOCATION" />
// NO declarar ACCESS_FINE_LOCATION

var request = new GeolocationRequest(GeolocationAccuracy.Low, TimeSpan.FromSeconds(5));
var location = await Geolocation.Default.GetLocationAsync(request);
// Resultado: coordenadas con error de ~1-3 km
```

**Ventaja**: Menor consumo de batería, menor invasividad percibida por el usuario.

### 11.2 Ubicación Precisa — `ACCESS_FINE_LOCATION`

**Precisión**: ~1-10 metros (GPS + GLONASS + Galileo).

**Permiso MAUI**: `Permissions.LocationWhenInUse`.

**Ejemplos de uso**:

| App | Funcionalidad | ¿Por qué necesita precisa? |
|-----|--------------|----------------------------|
| **App de infracciones** | Registrar posición exacta | Necesita calle + altura |
| Uber / Cabify | Punto de recogida | Diferencia "en la puerta" vs "a 2 cuadras" |
| Google Maps / Waze | Navegación turn-by-turn | Debe saber en qué carril estás |
| App de delivery | "Estoy en la puerta" | El repartidor necesita el punto exacto |
| App de running | Registrar recorrido | Trazar ruta metro a metro |

```csharp
// AndroidManifest.xml — ambos permisos (recomendado desde API 31)
// <uses-permission android:name="android.permission.ACCESS_FINE_LOCATION" />
// <uses-permission android:name="android.permission.ACCESS_COARSE_LOCATION" />

var request = new GeolocationRequest(GeolocationAccuracy.Best, TimeSpan.FromSeconds(10));
var location = await Geolocation.Default.GetLocationAsync(request);

if (location != null)
{
    var infraccion = new Infraccion
    {
        Latitud = location.Latitude,     // -26.836720
        Longitud = location.Longitude,   // -65.203684
        Precision = location.Accuracy,   // 4.2 metros
        FechaHora = DateTime.Now
    };
}
```

**Consideración Android 12+**: El usuario puede elegir "Aproximada" aunque pidas precisa. Si la app **requiere** precisión, verificá:

```csharp
if (location != null && location.Accuracy.HasValue && location.Accuracy > 100)
{
    await DisplayAlert("Precisión insuficiente",
        "Se necesita ubicación precisa. Activá 'Ubicación precisa' en permisos.", "OK");
}
```

### 11.3 Ubicación en Segundo Plano — `ACCESS_BACKGROUND_LOCATION`

**Precisión**: Igual que FINE/COARSE (depende de con cuál se combine).

**Diferencia clave**: La app recibe ubicación **cuando no está visible** (pantalla apagada, otra app en primer plano).

**Permiso MAUI**: `Permissions.LocationAlways`.

**Ejemplos de uso**:

| App | Funcionalidad | ¿Por qué segundo plano? |
|-----|--------------|--------------------------|
| Tracking vehicular | Rastrear flota 24/7 | Chofer no tiene la app abierta |
| Geofencing | Notificar al entrar/salir de zona | Detecta la zona con app cerrada |
| Strava | Grabar recorrido completo | Usuario apaga pantalla mientras corre |
| Seguridad personal | Compartir ubicación en tiempo real | Celular en el bolsillo |

```csharp
// Requiere ADEMÁS de los permisos de primer plano:
// Android: <uses-permission android:name="android.permission.ACCESS_BACKGROUND_LOCATION" />
// iOS: NSLocationAlwaysAndWhenInUseUsageDescription + UIBackgroundModes > location

var status = await Permissions.RequestAsync<Permissions.LocationAlways>();
```

**Restricciones**:

| Plataforma | Restricción |
|---|---|
| **Android 10+** | Pedir `LocationWhenInUse` PRIMERO, después `LocationAlways` separado |
| **Android 11+** | "Permitir siempre" solo aparece en Configuración, no en popup |
| **iOS** | Apple rechaza apps que piden `Always` sin justificación clara |
| **Ambos** | Mayor consumo de batería |

### 11.4 ¿Cuál elegir? — Árbol de decisión

```
¿Tu app necesita ubicación con la app cerrada/en background?
    │
    ├── SÍ → ACCESS_BACKGROUND_LOCATION + FINE o COARSE
    │         (tracking, geofencing, fitness)
    │
    └── NO → ¿Necesitás precisión de metros?
                │
                ├── SÍ → ACCESS_FINE_LOCATION + ACCESS_COARSE_LOCATION
                │         (infracciones, navegación, delivery)
                │
                └── NO → ACCESS_COARSE_LOCATION solo
                          (clima, noticias, tiendas cercanas)
```

> **Regla de oro**: Pedí el **mínimo nivel necesario**. Cada nivel extra = más batería, más fricción con el usuario, más escrutinio en Google Play / App Store.
>
> Para la app de infracciones de tránsito: **ubicación precisa en primer plano** (`FINE` + `COARSE`) es lo correcto.

---

## 12. Configuración de Manifiestos (GPS)

### 12.1 Android — `Platforms/Android/AndroidManifest.xml`

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
    -->
    <uses-permission android:name="android.permission.ACCESS_COARSE_LOCATION" />

    <!--
        ACCESS_BACKGROUND_LOCATION: solo si necesitás ubicación en segundo plano.
        En Android 10+ se pide SEPARADO del permiso de primer plano.
    -->
    <!-- <uses-permission android:name="android.permission.ACCESS_BACKGROUND_LOCATION" /> -->

    <!--
        uses-feature con required="false" permite que la app se instale
        en dispositivos SIN GPS (tablets wifi-only).
    -->
    <uses-feature android:name="android.hardware.location.gps" android:required="false" />

    <application
        android:allowBackup="true"
        android:label="@string/app_name">
    </application>

</manifest>
```

#### ¿Por qué `ACCESS_COARSE_LOCATION` si quiero precisión?

A partir de **Android 12 (API 31)**, el sistema permite al usuario elegir entre "Precisa" y "Aproximada". Si declarás ambos permisos (`FINE` + `COARSE`), el diálogo muestra las dos opciones. Si solo declarás `FINE`, el sistema puede degradar a aproximada sin que lo sepas.

### 12.2 iOS — `Platforms/iOS/Info.plist`

```xml
<!--
    OBLIGATORIO para ubicación en primer plano.
    Sin esta clave la app crashea al pedir ubicación.
-->
<key>NSLocationWhenInUseUsageDescription</key>
<string>La app necesita tu ubicación para registrar la posición de las infracciones de tránsito.</string>

<!--
    Solo necesario si pedís ubicación en SEGUNDO PLANO (Always).
-->
<!-- <key>NSLocationAlwaysAndWhenInUseUsageDescription</key>
<string>La app necesita acceso permanente a tu ubicación para tracking en segundo plano.</string> -->
```

### 12.3 Manifiestos mínimos (GPS)

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

### 12.4 Verificar si el GPS está activado

Un problema diferente al permiso: el usuario puede conceder el permiso pero tener el **GPS desactivado**.

```csharp
private async Task<bool> VerificarGpsActivadoAsync()
{
    try
    {
        var request = new GeolocationRequest(GeolocationAccuracy.Medium, TimeSpan.FromSeconds(5));
        var location = await Geolocation.Default.GetLocationAsync(request);

        if (location == null)
        {
            await DisplayAlert("GPS desactivado",
                "El GPS está apagado. Activalo desde los ajustes del dispositivo.", "OK");
            return false;
        }
        return true;
    }
    catch (FeatureNotEnabledException)
    {
        await DisplayAlert("GPS desactivado",
            "El GPS está apagado. Activalo desde los ajustes del dispositivo.", "OK");
        return false;
    }
    catch (FeatureNotSupportedException)
    {
        await DisplayAlert("GPS no disponible",
            "Este dispositivo no tiene GPS.", "OK");
        return false;
    }
}
```

---

## 13. API Reference: Clases y Métodos de Geolocation

### 13.1 `GeolocationAccuracy` (enum)

Determina qué sensores usa el dispositivo y define la relación precisión / velocidad / batería.

| Valor | Precisión aprox. | Fuente | Tiempo típico | Batería |
|---|---|---|---|---|
| `Lowest` | ~3000 m | Cell ID | < 1 seg | Mínimo |
| `Low` | ~1000 m | Cell ID / WiFi | < 1 seg | Bajo |
| `Medium` | ~100 m | WiFi + triangulación celular | 1-3 seg | Medio |
| `High` | ~10 m | GPS | 3-10 seg | Alto |
| `Best` | ~1 m | GPS + GLONASS + Galileo + BeiDou | 5-30 seg | Máximo |
| `Default` | Variable | El SO decide | Variable | Variable |

```csharp
var request = new GeolocationRequest(GeolocationAccuracy.Best);
```

> **Nota**: `Best` no siempre da 1 metro. Depende de cielo despejado, cantidad de satélites visibles, y calidad del chip GPS. En interiores, incluso `Best` puede dar >50m de error.

### 13.2 `GeolocationRequest` (clase)

Encapsula los parámetros de una solicitud de ubicación.

```csharp
// Constructores
public GeolocationRequest(GeolocationAccuracy accuracy)
public GeolocationRequest(GeolocationAccuracy accuracy, TimeSpan timeout)
```

**Propiedades**:

| Propiedad | Tipo | Default | Descripción |
|---|---|---|---|
| `DesiredAccuracy` | `GeolocationAccuracy` | `Default` | Nivel de precisión deseado |
| `Timeout` | `TimeSpan` | ~10 seg | Tiempo máximo de espera para obtener fix |

```csharp
// Máxima precisión, esperar hasta 15 segundos
var request = new GeolocationRequest(GeolocationAccuracy.Best, TimeSpan.FromSeconds(15));

// Precisión media, rápido
var request = new GeolocationRequest(GeolocationAccuracy.Medium, TimeSpan.FromSeconds(5));

// Solo con accuracy (timeout por defecto)
var request = new GeolocationRequest(GeolocationAccuracy.High);
```

**¿Qué pasa si se vence el timeout?**
- `GetLocationAsync` devuelve `null` (no lanza excepción por timeout).
- Puede devolver la **mejor ubicación obtenida** hasta ese momento (depende de la plataforma).

### 13.3 `Geolocation.Default` (singleton)

Punto de entrada principal para GPS. Es la implementación por defecto de `IGeolocation`.

#### `GetLocationAsync` — Obtener ubicación actual (activa el GPS)

```csharp
Task<Location?> GetLocationAsync(GeolocationRequest request)
Task<Location?> GetLocationAsync(GeolocationRequest request, CancellationToken token)
```

| Aspecto | Detalle |
|---|---|
| **¿Activa el GPS?** | Sí — enciende el chip GPS del dispositivo |
| **Retorno** | `Location` con coordenadas, o `null` si no pudo obtener |
| **Timeout** | Definido en `GeolocationRequest.Timeout` |
| **Cancelable** | Sí, con `CancellationToken` |
| **Excepciones** | `FeatureNotSupportedException`, `FeatureNotEnabledException`, `PermissionException` |
| **Consumo batería** | Alto (proporcional a `GeolocationAccuracy`) |

```csharp
var request = new GeolocationRequest(GeolocationAccuracy.Best, TimeSpan.FromSeconds(10));
var location = await Geolocation.Default.GetLocationAsync(request);

if (location != null)
    Console.WriteLine($"Lat: {location.Latitude}, Lng: {location.Longitude}");
else
    Console.WriteLine("No se pudo obtener ubicación (timeout o GPS sin señal)");
```

#### `GetLastKnownLocationAsync` — Última ubicación cacheada (NO activa el GPS)

```csharp
Task<Location?> GetLastKnownLocationAsync()
```

| Aspecto | Detalle |
|---|---|
| **¿Activa el GPS?** | **No** — lee del caché del sistema |
| **Retorno** | `Location` cacheada, o `null` si nunca se usó GPS |
| **Velocidad** | Instantáneo (~0 ms) |
| **Consumo batería** | Ninguno |
| **Riesgo** | Puede ser **vieja** (minutos, horas o días de antigüedad) |
| **Excepciones** | `FeatureNotSupportedException`, `PermissionException` |

```csharp
var location = await Geolocation.Default.GetLastKnownLocationAsync();

if (location != null)
{
    var antiguedad = DateTime.UtcNow - location.Timestamp.UtcDateTime;
    Console.WriteLine($"Ubicación de hace {antiguedad.TotalMinutes:F0} minutos");

    if (antiguedad.TotalMinutes > 5)
        Console.WriteLine("⚠ Ubicación posiblemente desactualizada");
}
else
{
    Console.WriteLine("No hay ubicación cacheada — usar GetLocationAsync");
}
```

#### ¿Cuándo usar cada uno?

| Escenario | Método recomendado |
|---|---|
| Mostrar posición actual del usuario | `GetLocationAsync` con `Best` |
| Pre-llenar un formulario con "última posición conocida" | `GetLastKnownLocationAsync` |
| Registrar infracción (necesito precisión) | `GetLocationAsync` con `Best` |
| Verificar rápido si el GPS funciona | `GetLastKnownLocationAsync` (si null → `GetLocationAsync`) |
| Mostrar mapa centrado en usuario | `GetLastKnownLocationAsync` primero, luego `GetLocationAsync` para actualizar |

### 13.4 `Location` (clase de retorno)

Objeto devuelto por `GetLocationAsync` y `GetLastKnownLocationAsync`.

| Propiedad | Tipo | Descripción |
|---|---|---|
| `Latitude` | `double` | Latitud en grados decimales (-90 a 90) |
| `Longitude` | `double` | Longitud en grados decimales (-180 a 180) |
| `Altitude` | `double?` | Altitud en metros sobre el nivel del mar (puede ser null) |
| `Accuracy` | `double?` | Radio de error en metros (±). Menor = más preciso |
| `VerticalAccuracy` | `double?` | Precisión vertical en metros |
| `Speed` | `double?` | Velocidad en m/s |
| `Course` | `double?` | Dirección de viaje en grados (0 = norte, 90 = este) |
| `Timestamp` | `DateTimeOffset` | Momento en que se tomó la medición |
| `IsFromMockProvider` | `bool` | `true` si es ubicación simulada (detecta GPS spoofing) |

```csharp
var location = await Geolocation.Default.GetLocationAsync(request);
if (location != null)
{
    Console.WriteLine($"Lat: {location.Latitude:F6}");
    Console.WriteLine($"Lng: {location.Longitude:F6}");
    Console.WriteLine($"Alt: {location.Altitude?.ToString("F1") ?? "N/A"} m");
    Console.WriteLine($"Precisión: ±{location.Accuracy?.ToString("F0") ?? "N/A"} m");
    Console.WriteLine($"Velocidad: {location.Speed?.ToString("F1") ?? "N/A"} m/s");
    Console.WriteLine($"Timestamp: {location.Timestamp.LocalDateTime:HH:mm:ss}");
    Console.WriteLine($"Mock: {location.IsFromMockProvider}");
}
```

### 13.5 GPS en simuladores y emuladores

A diferencia de la cámara (que **NO funciona** en simulador iOS), el GPS **sí funciona** en simuladores/emuladores porque ambas plataformas proveen ubicaciones simuladas.

| Plataforma | ¿GPS funciona? | Ubicación por defecto | Cómo cambiarla |
|---|---|---|---|
| **iOS Simulator** | ✅ Sí (simulado) | Apple Park, Cupertino | Xcode: Features > Location |
| **Android Emulator** | ✅ Sí (simulado) | Última configurada | Panel "⋮" > Location |
| **Dispositivo real con GPS** | ✅ Sí (real) | Posición real | N/A |
| **Dispositivo real SIN GPS** | ❌ No | N/A | `FeatureNotSupportedException` |

> **Nota**: En simuladores, `GetLocationAsync` devuelve coordenadas simuladas sin lanzar excepciones. `FeatureNotSupportedException` solo ocurre en dispositivos reales sin chip GPS (tablets wifi-only, algunos dispositivos industriales).

---

## 14. CancellationToken en GPS

### 14.1 ¿Por qué usar CancellationToken?

`GetLocationAsync` puede tardar mucho (especialmente con `GeolocationAccuracy.Best` en un GPS "frío"). El `CancellationToken` permite cancelar la operación si:

- El usuario sale de la página
- El usuario toca un botón de cancelar
- Se agota un timeout personalizado

### 14.2 Ejemplo: Cancelar al salir de la página

```csharp
public partial class GpsPage : ContentPage
{
    private CancellationTokenSource? _gpsCts;

    private async Task ObtenerUbicacionAsync()
    {
        // Cancelar cualquier consulta anterior
        _gpsCts?.Cancel();
        _gpsCts?.Dispose();
        _gpsCts = new CancellationTokenSource();

        try
        {
            var request = new GeolocationRequest(
                GeolocationAccuracy.Best,
                TimeSpan.FromSeconds(15));

            // Pasar el token — si se cancela, lanza OperationCanceledException
            var location = await Geolocation.Default.GetLocationAsync(
                request, _gpsCts.Token);

            if (location != null)
                MostrarUbicacion(location);
        }
        catch (OperationCanceledException)
        {
            // El usuario salió de la página o canceló — no hacer nada
            Console.WriteLine("[APP] Consulta GPS cancelada");
        }
        catch (FeatureNotEnabledException)
        {
            // GPS apagado
        }
        catch (FeatureNotSupportedException)
        {
            // Dispositivo sin GPS (tablet wifi-only, etc.)
        }
    }

    protected override void OnDisappearing()
    {
        base.OnDisappearing();
        _gpsCts?.Cancel();
        _gpsCts?.Dispose();
        _gpsCts = null;
    }
}
```

### 14.3 Ejemplo: Botón de cancelar

```csharp
private CancellationTokenSource? _gpsCts;

private async void OnObtenerUbicacion_Clicked(object sender, EventArgs e)
{
    _gpsCts?.Cancel();
    _gpsCts?.Dispose();
    _gpsCts = new CancellationTokenSource();

    BtnObtener.IsEnabled = false;
    BtnCancelar.IsVisible = true;

    try
    {
        var request = new GeolocationRequest(GeolocationAccuracy.Best, TimeSpan.FromSeconds(30));
        var location = await Geolocation.Default.GetLocationAsync(request, _gpsCts.Token);

        if (location != null)
            MostrarUbicacion(location);
    }
    catch (OperationCanceledException)
    {
        LblInfo.Text = "Búsqueda de GPS cancelada por el usuario.";
    }
    catch (FeatureNotEnabledException)
    {
        LblInfo.Text = "El GPS está desactivado. Activalo desde ajustes.";
    }
    catch (FeatureNotSupportedException)
    {
        LblInfo.Text = "Este dispositivo no tiene GPS.";
    }
    finally
    {
        BtnObtener.IsEnabled = true;
        BtnCancelar.IsVisible = false;
        _gpsCts?.Dispose();
        _gpsCts = null;
    }
}

private void OnCancelar_Clicked(object sender, EventArgs e)
{
    _gpsCts?.Cancel();
}
```

### 14.4 Timeout de GeolocationRequest vs CancellationToken

| Mecanismo | Qué hace | Resultado al vencer |
|---|---|---|
| `GeolocationRequest.Timeout` | Timeout **interno** del SO para obtener fix GPS | Devuelve `null` (no excepción) |
| `CancellationToken` | Cancelación **de tu código** | Lanza `OperationCanceledException` |

Podés combinar ambos:

```csharp
// Timeout del SO: 15 seg (deja de buscar satélites)
var request = new GeolocationRequest(GeolocationAccuracy.Best, TimeSpan.FromSeconds(15));

// Timeout de tu app: 20 seg (margen extra)
var cts = new CancellationTokenSource(TimeSpan.FromSeconds(20));

var location = await Geolocation.Default.GetLocationAsync(request, cts.Token);
// Si el SO encuentra fix en 15 seg → devuelve Location
// Si el SO no encuentra en 15 seg → devuelve null
// Si tu token se cancela en 20 seg → OperationCanceledException
```

---

## 15. Precisión: Pros, Contras y Rendimiento

### 15.1 Tabla comparativa completa

| | `Lowest` | `Low` | `Medium` | `High` | `Best` |
|---|---|---|---|---|---|
| **Precisión** | ~3 km | ~1 km | ~100 m | ~10 m | ~1 m |
| **Fuente** | Cell ID | Cell/WiFi | WiFi + red | GPS | GPS + GNSS |
| **Tiempo primer fix** | < 1 seg | < 1 seg | 1-3 seg | 3-10 seg | 5-30 seg |
| **Cold start** | N/A | N/A | N/A | 10-30 seg | 15-60 seg |
| **Funciona en interior** | ✅ Sí | ✅ Sí | ✅ Sí | ⚠️ Degradado | ❌ Muy pobre |
| **Consumo batería** | Mínimo | Bajo | Medio | Alto | Máximo |
| **Requiere cielo abierto** | No | No | No | Preferible | Necesario |

> **Cold start**: primera vez que se usa el GPS después de encender el dispositivo o después de mucho tiempo sin usarlo. El chip necesita descargar almanaque de satélites.

### 15.2 Pros y contras de cada nivel

#### `Lowest` / `Low` — Red celular
```
✅ PROS                              ❌ CONTRAS
├── Instantáneo (< 1 seg)           ├── Precisión de ~1-3 km
├── Funciona en interiores           ├── Inútil para navegación
├── Mínimo consumo de batería        ├── Requiere señal celular
└── No necesita GPS activado         └── No funciona en modo avión
```

#### `Medium` — WiFi + triangulación
```
✅ PROS                              ❌ CONTRAS
├── Rápido (1-3 seg)                ├── Precisión de ~100 m (1 cuadra)
├── Funciona en interiores           ├── Requiere WiFi activado (scan)
├── Consumo moderado                 ├── No sirve para posición exacta
└── Buen balance para búsquedas      └── Varía según densidad WiFi
```

#### `High` / `Best` — GPS
```
✅ PROS                              ❌ CONTRAS
├── Precisión de metros              ├── Lento (5-30 seg, más en cold start)
├── Funciona sin red (offline)       ├── Alto consumo de batería
├── Independiente de WiFi/celular    ├── Degradado en interiores/túneles
└── Necesario para registro legal     └── Puede devolver null si no hay señal
```

### 15.3 Recomendaciones según caso de uso

| Caso de uso | Accuracy | Timeout | Justificación |
|---|---|---|---|
| **App de infracciones** | `Best` | 15 seg | Registro legal necesita posición exacta |
| Clima / noticias | `Low` | 5 seg | Solo necesita ciudad |
| Comercios cercanos | `Medium` | 5 seg | Alcanza con el barrio |
| Navegación GPS | `Best` | 10 seg | Necesita carril exacto |
| "¿Dónde estoy?" rápido | `High` | 10 seg | Buena precisión sin espera excesiva |
| Tracking continuo | `High` | 5 seg | Balance entre precisión y batería |
| Formulario con ubicación | `High` | 10 seg | Calle + altura es suficiente |

### 15.4 Patrón: Ubicación rápida primero, precisa después

Si no querés que el usuario espere 30 segundos:

```csharp
private async Task<Location?> ObtenerUbicacionProgresiva()
{
    // Paso 1: Mostrar última ubicación cacheada inmediatamente
    var cached = await Geolocation.Default.GetLastKnownLocationAsync();
    if (cached != null)
    {
        MostrarUbicacion(cached);
        LblInfo.Text = "Ubicación aproximada — mejorando...";
    }

    // Paso 2: Obtener ubicación precisa
    var request = new GeolocationRequest(GeolocationAccuracy.Best, TimeSpan.FromSeconds(15));
    var precise = await Geolocation.Default.GetLocationAsync(request);

    if (precise != null)
    {
        MostrarUbicacion(precise);
        LblInfo.Text = $"Ubicación precisa (±{precise.Accuracy:F0} m)";
        return precise;
    }

    // Si falla la precisa pero tenemos cacheada, usar esa
    if (cached != null)
    {
        LblInfo.Text = "⚠ Usando ubicación cacheada (GPS sin señal)";
        return cached;
    }

    LblInfo.Text = "No se pudo obtener ubicación";
    return null;
}
```

---

## 16. Ejemplo Completo: Página con GPS

### 16.1 XAML — `GpsPage.xaml`

```xml
<?xml version="1.0" encoding="utf-8" ?>
<ContentPage xmlns="http://schemas.microsoft.com/dotnet/2021/maui"
             xmlns:x="http://schemas.microsoft.com/winfx/2009/xaml"
             x:Class="MiApp.Views.GpsPage"
             Title="Ubicación GPS">

    <ScrollView Padding="20">
        <VerticalStackLayout Spacing="15">

            <Label x:Name="GpsStatusLabel"
                   Text="GPS: verificando..."
                   FontSize="14"
                   TextColor="Gray" />

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

            <Button x:Name="BtnCancelar"
                    Text="Cancelar búsqueda"
                    BackgroundColor="#F44336"
                    TextColor="White"
                    IsVisible="False"
                    Clicked="OnCancelar_Clicked" />

            <Button x:Name="BtnReintentarGps"
                    Text="Reintentar permisos"
                    BackgroundColor="#FF9800"
                    TextColor="White"
                    IsVisible="False"
                    Clicked="OnReintentarGps_Clicked" />

            <Label x:Name="LblInfo"
                   Text=""
                   FontSize="12"
                   TextColor="Gray" />

        </VerticalStackLayout>
    </ScrollView>
</ContentPage>
```

### 16.2 Code-behind — `GpsPage.xaml.cs`

```csharp
namespace MiApp.Views;

public partial class GpsPage : ContentPage
{
    private CancellationTokenSource? _gpsCts;

    public GpsPage()
    {
        InitializeComponent();
    }

    protected override async void OnAppearing()
    {
        base.OnAppearing();
        await VerificarEstadoGpsAsync();
    }

    protected override void OnDisappearing()
    {
        base.OnDisappearing();
        _gpsCts?.Cancel();
        _gpsCts?.Dispose();
        _gpsCts = null;
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
        _gpsCts?.Cancel();
        _gpsCts?.Dispose();
        _gpsCts = new CancellationTokenSource();

        BtnObtenerUbicacion.IsEnabled = false;
        BtnCancelar.IsVisible = true;
        GpsStatusLabel.Text = "GPS: obteniendo ubicación...";
        GpsStatusLabel.TextColor = Colors.Gray;

        try
        {
            var permiso = await EvaluarPermisosUbicacionAsync();
            if (!permiso) return;

            var request = new GeolocationRequest(
                GeolocationAccuracy.Best,
                TimeSpan.FromSeconds(15));

            var location = await Geolocation.Default.GetLocationAsync(
                request, _gpsCts.Token);

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
        catch (OperationCanceledException)
        {
            GpsStatusLabel.Text = "GPS: búsqueda cancelada";
            GpsStatusLabel.TextColor = Colors.Orange;
            LblInfo.Text = "Búsqueda cancelada por el usuario.";
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
            BtnCancelar.IsVisible = false;
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

    private void OnCancelar_Clicked(object sender, EventArgs e)
    {
        _gpsCts?.Cancel();
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

## 17. Ejemplo: GPS + Envío a Servidor con CancellationToken

Este ejemplo muestra cómo integrar la obtención de GPS con el envío de datos a un servidor (API REST), usando un **único `CancellationToken`** que cancela ambas operaciones si el usuario cancela.

### 17.1 Arquitectura

```
InfraccionPage (UI)
    │
    ├── GpsService          → Encapsula Geolocation API
    │
    ├── InfraccionApiService → HttpClient hacia el servidor
    │
    └── CancellationToken ───┐
                             ├── Cancela GPS si el usuario cancela
                             └── Cancela HTTP si el usuario cancela
```

El mismo `CancellationToken` se pasa a `GetLocationAsync` y a `PostAsync`. Si el usuario toca "Cancelar", se aborta cualquiera de las dos operaciones que esté en curso.

### 17.2 Modelo — `InfraccionDto.cs`

```csharp
namespace MiApp.Models;

public class InfraccionDto
{
    public double Latitud { get; set; }
    public double Longitud { get; set; }
    public double? Precision { get; set; }
    public string? Descripcion { get; set; }
    public DateTime FechaHora { get; set; }
}
```

### 17.3 Servicio GPS — `GpsService.cs`

Encapsula la lógica de geolocalización. Facilita testing y reutilización.

```csharp
using Microsoft.Maui.Devices.Sensors;

namespace MiApp.Services;

public class GpsService
{
    /// <summary>
    /// Obtiene la ubicación actual con precisión máxima.
    /// Lanza OperationCanceledException si se cancela el token.
    /// Devuelve null si no hay permisos, sin señal, o timeout.
    /// Lanza OperationCanceledException si se cancela el token.
    /// </summary>
    public async Task<Location?> ObtenerUbicacionAsync(
        CancellationToken ct = default)
    {
        // 1. Verificar permiso antes de usar el GPS
        var status = await Permissions.CheckStatusAsync<Permissions.LocationWhenInUse>();
        if (status != PermissionStatus.Granted)
        {
            status = await Permissions.RequestAsync<Permissions.LocationWhenInUse>();
        }
        if (status != PermissionStatus.Granted)
            return null;  // Sin permiso → null limpio, sin excepción

        // 2. Con permiso confirmado, pedir ubicación
        var request = new GeolocationRequest(
            GeolocationAccuracy.Best,
            TimeSpan.FromSeconds(15));

        return await Geolocation.Default.GetLocationAsync(request, ct);
    }
}
```

### 17.4 Servicio API — `InfraccionApiService.cs`

Usa `HttpClient` inyectado vía DI. El mismo `CancellationToken` cancela la llamada HTTP si el usuario cancela.

```csharp
using System.Net.Http.Json;
using MiApp.Models;

namespace MiApp.Services;

public class InfraccionApiService
{
    private readonly HttpClient _http;

    public InfraccionApiService(HttpClient http)
    {
        _http = http;
    }

    /// <summary>
    /// Envía una infracción al servidor.
    /// Lanza OperationCanceledException si se cancela el token.
    /// </summary>
    public async Task<bool> EnviarInfraccionAsync(
        InfraccionDto infraccion, CancellationToken ct)
    {
        var response = await _http.PostAsJsonAsync(
            "api/infracciones", infraccion, ct);

        return response.IsSuccessStatusCode;
    }
}
```

### 17.5 Registro en DI — `MauiProgram.cs`

```csharp
using MiApp.Services;

public static class MauiProgram
{
    public static MauiApp CreateMauiApp()
    {
        var builder = MauiApp.CreateBuilder();
        builder.UseMauiApp<App>();

        // Servicios
        builder.Services.AddSingleton<GpsService>();

        builder.Services.AddHttpClient<InfraccionApiService>(client =>
        {
            client.BaseAddress = new Uri("https://mi-servidor.com/");
            client.Timeout = TimeSpan.FromSeconds(30);
        });

        // Páginas
        builder.Services.AddTransient<InfraccionPage>();

        return builder.Build();
    }
}
```

> **`AddHttpClient<T>`** registra `InfraccionApiService` y le inyecta un `HttpClient` configurado. Esto es mejor que crear `new HttpClient()` manualmente (evita socket exhaustion).

### 17.6 Página — `InfraccionPage.xaml`

```xml
<?xml version="1.0" encoding="utf-8" ?>
<ContentPage xmlns="http://schemas.microsoft.com/dotnet/2021/maui"
             xmlns:x="http://schemas.microsoft.com/winfx/2009/xaml"
             x:Class="MiApp.Views.InfraccionPage"
             Title="Registrar Infracción">

    <ScrollView Padding="20">
        <VerticalStackLayout Spacing="15">

            <Entry x:Name="EntryDescripcion"
                   Placeholder="Descripción de la infracción" />

            <Button x:Name="BtnRegistrar"
                    Text="Obtener GPS y Enviar"
                    Clicked="OnRegistrar_Clicked" />

            <Button x:Name="BtnCancelar"
                    Text="Cancelar"
                    IsVisible="False"
                    BackgroundColor="Gray"
                    Clicked="OnCancelar_Clicked" />

            <Label x:Name="LblEstado"
                   Text="Esperando acción del usuario."
                   FontSize="14" />

            <Label x:Name="LblResultado"
                   FontSize="14"
                   TextColor="Green" />

        </VerticalStackLayout>
    </ScrollView>
</ContentPage>
```

### 17.7 Code-behind — `InfraccionPage.xaml.cs`

```csharp
using MiApp.Models;
using MiApp.Services;

namespace MiApp.Views;

public partial class InfraccionPage : ContentPage
{
    private readonly GpsService _gps;
    private readonly InfraccionApiService _api;
    private CancellationTokenSource? _cts;

    public InfraccionPage(GpsService gps, InfraccionApiService api)
    {
        InitializeComponent();
        _gps = gps;
        _api = api;
    }

    private async void OnRegistrar_Clicked(object sender, EventArgs e)
    {
        // Defensivo: cancelar operación anterior si existe
        _cts?.Cancel();
        _cts?.Dispose();
        _cts = new CancellationTokenSource();

        BtnRegistrar.IsEnabled = false;
        BtnCancelar.IsVisible = true;
        LblResultado.Text = "";

        try
        {
            // ── Paso 1: Obtener GPS ──
            LblEstado.Text = "Obteniendo ubicación GPS...";
            var location = await _gps.ObtenerUbicacionAsync(_cts.Token);

            if (location == null)
            {
                LblEstado.Text = "No se pudo obtener ubicación (GPS sin señal).";
                return;
            }

            LblEstado.Text = $"GPS OK (±{location.Accuracy:F0} m). Enviando al servidor...";

            // ── Paso 2: Enviar al servidor ──
            var dto = new InfraccionDto
            {
                Latitud = location.Latitude,
                Longitud = location.Longitude,
                Precision = location.Accuracy,
                Descripcion = EntryDescripcion.Text,
                FechaHora = DateTime.Now
            };

            var ok = await _api.EnviarInfraccionAsync(dto, _cts.Token);

            if (ok)
            {
                LblEstado.Text = "Infracción registrada correctamente.";
                LblResultado.Text = $"Lat: {location.Latitude:F6}, Lng: {location.Longitude:F6}";
                LblResultado.TextColor = Colors.Green;
            }
            else
            {
                LblEstado.Text = "El servidor rechazó la solicitud.";
                LblResultado.TextColor = Colors.Red;
            }
        }
        catch (OperationCanceledException)
        {
            LblEstado.Text = "Operación cancelada por el usuario.";
        }
        catch (HttpRequestException ex)
        {
            LblEstado.Text = $"Error de red: {ex.Message}";
        }
        catch (FeatureNotEnabledException)
        {
            LblEstado.Text = "El GPS está desactivado. Activalo desde ajustes.";
        }
        catch (FeatureNotSupportedException)
        {
            LblEstado.Text = "Este dispositivo no tiene GPS.";
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

    protected override void OnDisappearing()
    {
        base.OnDisappearing();
        _cts?.Cancel();
        _cts?.Dispose();
        _cts = null;
    }
}
```

### 17.8 Flujo de cancelación

```
Usuario toca "Obtener GPS y Enviar"
    │
    ├── Crea CancellationTokenSource
    │
    ├── Paso 1: _gps.ObtenerUbicacionAsync(token)
    │       └── GetLocationAsync(request, token)  ← token cancela esto
    │
    ├── Paso 2: _api.EnviarInfraccionAsync(dto, token)
    │       └── PostAsJsonAsync(url, dto, token)   ← mismo token cancela esto
    │
    └── Si el usuario toca "Cancelar" en CUALQUIER momento:
            _cts.Cancel()
                ├── Si estaba en Paso 1 → cancela GPS → OperationCanceledException
                └── Si estaba en Paso 2 → cancela HTTP → OperationCanceledException
```

> **Punto clave**: Un único `CancellationToken` gobierna toda la cadena. No importa si la operación actual es GPS o HTTP — al cancelar, se aborta lo que esté en curso.

---

# PARTE IV — PHONEDIALER / LLAMADAS / SMS

---

## 20. ¿Qué es el PhoneDialer? — No es un dispositivo, es una API

`PhoneDialer` en .NET MAUI **no es un dispositivo físico** — es una API que permite a tu app interactuar con la aplicación de teléfono del dispositivo.

### 20.1 ¿Qué hace exactamente?

| Acción | Método | Resultado |
|--------|--------|-----------|
| Abrir el marcador con número precargado | `PhoneDialer.Default.Open("1155551234")` | Abre la app de teléfono con el número, el usuario toca Llamar |
| Verificar si el dispositivo puede hacer llamadas | `PhoneDialer.Default.IsSupported` | `true` en celulares, `false` en tablets wifi-only |

### 20.2 Analogía con GPS

```
GPS                                    PhoneDialer
─────────────────────────────          ─────────────────────────────
Geolocation.Default.GetLocationAsync   PhoneDialer.Default.Open
→ accede al hardware GPS               → accede a la app de teléfono
→ necesita permisos de ubicación        → NO necesita permisos (solo abre marcador)
→ devuelve Location                     → no devuelve nada, abre otra app
```

### 20.3 ¿PhoneDialer necesita permisos?

| Acción | ¿Necesita permiso? | Explicación |
|--------|-------------------|-------------|
| `PhoneDialer.Default.Open()` | **NO** | Solo abre el marcador — el usuario decide si llama |
| Llamada directa (`Intent.ActionCall`) | **SÍ** (`CALL_PHONE`) | Inicia la llamada sin confirmación del usuario (solo Android) |

> **Nota**: Esta distinción es clave. `PhoneDialer.Default.Open()` es como pasarle un papel con un número de teléfono al usuario. `Intent.ActionCall` es como marcar el número por él sin pedirle confirmación.

---

## 21. Esquemas URI de comunicación: tel:, sms:, mailto:

Un **esquema URI** (Uniform Resource Identifier) es una cadena con formato `esquema:datos` que el sistema operativo sabe interpretar para abrir la aplicación correcta.

### 21.1 Tabla de esquemas

| Esquema | Formato | Qué abre | Ejemplo |
|---------|---------|----------|---------|
| `tel:` | `tel:+5491155551234` | App de teléfono (marcador) | Precarga el número |
| `sms:` | `sms:+5491155551234` | App de mensajes SMS | Abre conversación con ese número |
| `mailto:` | `mailto:info@ejemplo.com` | App de correo electrónico | Abre compositor de email |
| `https://wa.me/` | `https://wa.me/5491155551234` | WhatsApp | Abre chat con ese número |
| `tg://` | `tg://resolve?domain=usuario` | Telegram | Abre chat con ese usuario |

### 21.2 ¿Cómo funciona Launcher?

`Launcher.Default.OpenAsync(uri)` le dice al SO: "abrí la app que sepa manejar este URI".

```
Tu app → Launcher.OpenAsync("tel:+5491155551234") → SO → App de Teléfono
Tu app → Launcher.OpenAsync("https://wa.me/549...") → SO → WhatsApp
Tu app → Launcher.OpenAsync("mailto:x@y.com")      → SO → Gmail / Mail
```

### 21.3 ¿Y si la app destino no está instalada?

```csharp
// ✅ CORRECTO — verificar antes de abrir
if (await Launcher.Default.CanOpenAsync("https://wa.me/5491155551234"))
{
    await Launcher.Default.OpenAsync("https://wa.me/5491155551234");
}
else
{
    await DisplayAlert("Error", "WhatsApp no está instalado", "OK");
}
```

> **Nota**: `CanOpenAsync` verifica si hay alguna app registrada para ese esquema URI. En iOS requiere configurar `LSApplicationQueriesSchemes` en `Info.plist`.

---

## 22. Configuración de Manifiestos (PhoneDialer y SMS)

### 22.1 Android — AndroidManifest.xml

```xml
<!-- Para PhoneDialer.Default.Open() → NO necesita ningún permiso -->
<!-- Solo abre el marcador, no inicia la llamada -->

<!-- Para llamada directa (Intent.ActionCall) → SÍ necesita permiso -->
<uses-permission android:name="android.permission.CALL_PHONE" />

<!-- Para verificar qué apps están instaladas (Android 11+) -->
<queries>
    <intent>
        <action android:name="android.intent.action.DIAL" />
        <data android:scheme="tel" />
    </intent>
    <intent>
        <action android:name="android.intent.action.SENDTO" />
        <data android:scheme="smsto" />
    </intent>
    <intent>
        <action android:name="android.intent.action.SENDTO" />
        <data android:scheme="mailto" />
    </intent>
    <!-- WhatsApp -->
    <package android:name="com.whatsapp" />
    <!-- Telegram -->
    <package android:name="org.telegram.messenger" />
</queries>
```

> **¿Por qué es obligatorio `<queries>`?**
> Desde **Android 11 (API 30)**, el sistema aplica **filtrado de visibilidad de paquetes** (*package visibility filtering*): una app ya no puede "ver" todas las demás apps instaladas por defecto.
> Si no declarás `<queries>`, el SO le oculta a tu app la existencia de la app de Teléfono, SMS, etc.
> La consecuencia práctica es que **`PhoneDialer.Default.IsSupported` devuelve `false`** incluso en un dispositivo físico con SIM, y `Launcher.CanOpenAsync()` también falla.
>
> **Retrocompatibilidad:** en versiones anteriores a Android 11 (API < 30), la etiqueta `<queries>` es **ignorada silenciosamente** por el SO. Podés dejarla siempre declarada sin riesgo.

### 22.2 iOS — Info.plist

```xml
<!-- Solo para apps de TERCEROS — los esquemas del sistema (tel, sms, mailto)
     NO necesitan declararse porque iOS los maneja nativamente -->
<key>LSApplicationQueriesSchemes</key>
<array>
    <string>whatsapp</string>
    <string>tg</string>
</array>
```

> **Diferencia clave con Android:** iOS **no necesita** declarar esquemas del sistema (`tel:`, `sms:`, `mailto:`) en `LSApplicationQueriesSchemes`. El SO los maneja nativamente y `PhoneDialer.Default.IsSupported` funciona sin configuración extra.
> Solo necesitás listar esquemas de **apps de terceros** (`whatsapp`, `tg`, etc.) para que `Launcher.CanOpenAsync("whatsapp://...")` devuelva `true` cuando la app está instalada.

### 22.3 Comparación Android vs iOS

| Aspecto | Android | iOS |
|---------|---------|-----|
| `PhoneDialer.Open()` | Sin permisos | Sin permisos |
| Llamada directa | `CALL_PHONE` en Manifest + permiso runtime | **No disponible** — Apple no lo permite |
| Verificar apps instaladas | `<queries>` en Manifest (obligatorio API 30+; ignorado en API < 30) | `LSApplicationQueriesSchemes` en Info.plist (solo para apps de terceros) |
| SMS | Sin permisos (abre app SMS) | Sin permisos (abre app Mensajes) |
| Email | Sin permisos (abre app Email) | Sin permisos (abre app Mail) |

---

## 23. PhoneDialer.Default.Open — Abrir el marcador con número precargado

### 23.1 Uso básico

```csharp
// ✅ CORRECTO — abre el marcador con el número precargado
// El usuario decide si toca "Llamar" o no
PhoneDialer.Default.Open("1155551234");
```

### 23.2 Verificar soporte antes de llamar

```csharp
if (PhoneDialer.Default.IsSupported)
{
    PhoneDialer.Default.Open("1155551234");
}
else
{
    await DisplayAlert("Error", "Este dispositivo no puede hacer llamadas", "OK");
}
```

### 23.3 ¿Qué pasa en cada plataforma?

| Plataforma | Comportamiento |
|------------|---------------|
| Android con SIM | Abre app Teléfono con número precargado |
| Android sin SIM (tablet wifi-only) | `IsSupported` devuelve `false` |
| iOS con SIM | Abre app Teléfono con número precargado |
| iOS sin SIM (iPod/iPad wifi-only) | `IsSupported` devuelve `false` |
| Emulador Android | `IsSupported` puede ser `true` pero la llamada no se concreta |
| Simulador iOS | `IsSupported` devuelve `false` |
| Android físico con SIM pero sin `<queries>` | `IsSupported` devuelve `false` (ver nota abajo) |

> ⚠️ **`IsSupported` devuelve `false` en un dispositivo físico con SIM?**
> En Android 11+, si falta la etiqueta `<queries>` en `AndroidManifest.xml`, el sistema le oculta a tu app la existencia del marcador telefónico. Resultado: `IsSupported` devuelve `false` aunque el dispositivo pueda hacer llamadas. Solución: agregar `<queries>` con el intent `android.intent.action.DIAL` y el esquema `tel` (ver sección 22.1).

### 23.4 Formato del número

```csharp
// ✅ Todos estos formatos funcionan
PhoneDialer.Default.Open("1155551234");          // sin código de país
PhoneDialer.Default.Open("+5491155551234");       // con código de país
PhoneDialer.Default.Open("0800-333-4444");        // con guiones

// ❌ INCORRECTO — no pasar cadena vacía
PhoneDialer.Default.Open("");   // lanza ArgumentNullException
```

---

## 24. Llamada directa sin confirmación del usuario (solo Android)

### 24.0 ¿Qué es `CALL_PHONE`?

`CALL_PHONE` (`android.permission.CALL_PHONE`) es un **permiso de Android** clasificado como **peligroso** (*dangerous permission*). Otorga a la app la capacidad de iniciar una llamada telefónica **sin que el usuario toque el botón "Llamar"** del marcador.

| Aspecto | Detalle |
|---------|--------:|
| **Nombre completo** | `android.permission.CALL_PHONE` |
| **Categoría** | Permiso peligroso (*dangerous*) → requiere solicitud en tiempo de ejecución |
| **Qué habilita** | Usar `Intent.ActionCall` para iniciar una llamada directa |
| **Sin este permiso** | Solo se puede usar `Intent.ActionDial` / `PhoneDialer.Default.Open()` (abre el marcador, no llama) |
| **¿Existe en iOS?** | **No** — Apple prohíbe que una app inicie llamadas sin confirmación del usuario |
| **¿Cuándo usarlo?** | Apps de emergencia, call centers, asistencia. La mayoría de las apps **no** lo necesitan |

> **Regla simple:**
> - `PhoneDialer.Default.Open()` → **no necesita** `CALL_PHONE` → abre el marcador y el usuario decide.
> - `Intent.ActionCall` → **sí necesita** `CALL_PHONE` → la llamada se inicia sin intervención del usuario.

### 24.1 ¿Cuándo usar esto?

En apps de **emergencia**, **call centers** o asistencia donde el usuario ya confirmó que quiere llamar. En la gran mayoría de apps, `PhoneDialer.Default.Open()` es suficiente.

### 24.2 ACTION_DIAL vs ACTION_CALL

| Aspecto | `ACTION_DIAL` | `ACTION_CALL` |
|---------|--------------|---------------|
| ¿Qué hace? | Abre marcador con número | **Inicia la llamada directamente** |
| ¿Necesita permiso? | NO | SÍ — `CALL_PHONE` |
| ¿Equivalente MAUI? | `PhoneDialer.Default.Open()` | Código nativo Android |
| ¿Disponible en iOS? | Sí (vía `tel:`) | **NO** — Apple no lo permite |

### 24.3 Código nativo Android

```csharp
#if ANDROID
using Android.Content;
using AndroidUri = Android.Net.Uri;

// ✅ Llamada directa — requiere permiso CALL_PHONE
var intent = new Intent(Intent.ActionCall);
intent.SetData(AndroidUri.Parse("tel:+5491155551234"));
intent.SetFlags(ActivityFlags.NewTask);
Platform.CurrentActivity?.StartActivity(intent);
#endif
```

### 24.4 ¿Por qué no existe en iOS?

Apple **prohíbe** que una app inicie una llamada sin intervención del usuario. Incluso usando `tel:`, iOS muestra un diálogo de confirmación ("¿Desea llamar a este número?"). Esta es una política de seguridad de la plataforma.

```
Android                              iOS
────────────────────                 ────────────────────
ACTION_DIAL  → marcador (sin perm.) tel: → marcador + diálogo confirm.
ACTION_CALL  → llama directo (perm.) ❌ No existe equivalente
```

---

## 25. Permisos en tiempo de ejecución para CALL_PHONE

> Esta sección aplica **solo si usás llamada directa** (`Intent.ActionCall`). Si usás `PhoneDialer.Default.Open()`, no necesitás nada de esto.

### 25.1 Patrón de verificación

```csharp
#if ANDROID
using Android;
using Android.Content;
using AndroidUri = Android.Net.Uri;
using AndroidX.Core.App;
using AndroidX.Core.Content;

private async Task RealizarLlamadaDirectaAsync(string numero)
{
    var activity = Platform.CurrentActivity;
    if (activity == null) return;

    // 1. Verificar si el permiso ya fue concedido
    var permiso = ContextCompat.CheckSelfPermission(activity, Manifest.Permission.CallPhone);

    if (permiso == Android.Content.PM.Permission.Granted)
    {
        // ✅ Permiso concedido → llamar directamente
        var intent = new Intent(Intent.ActionCall);
        intent.SetData(AndroidUri.Parse($"tel:{numero}"));
        intent.SetFlags(ActivityFlags.NewTask);
        activity.StartActivity(intent);
    }
    else
    {
        // 2. Pedir permiso al usuario
        ActivityCompat.RequestPermissions(activity, new[] { Manifest.Permission.CallPhone }, 1001);
    }
}
#endif
```

### 25.2 Alternativa multiplataforma con Permissions API

```csharp
private async Task RealizarLlamadaDirectaAsync(string numero)
{
    var status = await Permissions.CheckStatusAsync<Permissions.Phone>();

    if (status != PermissionStatus.Granted)
    {
        status = await Permissions.RequestAsync<Permissions.Phone>();
    }

    if (status == PermissionStatus.Granted)
    {
#if ANDROID
        var intent = new Intent(Intent.ActionCall);
        intent.SetData(AndroidUri.Parse($"tel:{numero}"));
        intent.SetFlags(ActivityFlags.NewTask);
        Platform.CurrentActivity?.StartActivity(intent);
#else
        // iOS no soporta llamada directa, usar PhoneDialer
        PhoneDialer.Default.Open(numero);
#endif
    }
    else
    {
        await DisplayAlert("Permiso denegado",
            "Se necesita el permiso de teléfono para llamar directamente", "OK");
    }
}
```

---

## 26. Manejo de denegación de permisos para CALL_PHONE

### 26.1 Flujo completo con ShouldShowRationale

```csharp
private async Task LlamarConPermisoAsync(string numero)
{
    var status = await Permissions.CheckStatusAsync<Permissions.Phone>();

    if (status == PermissionStatus.Granted)
    {
        RealizarLlamada(numero);
        return;
    }

    // Pedir permiso
    status = await Permissions.RequestAsync<Permissions.Phone>();

    if (status == PermissionStatus.Granted)
    {
        RealizarLlamada(numero);
        return;
    }

    // Permiso denegado
    if (Permissions.ShouldShowRationale<Permissions.Phone>())
    {
        // Denegación TEMPORAL — se puede volver a pedir
        bool reintentar = await DisplayAlert(
            "Permiso necesario",
            "Necesitamos acceso al teléfono para hacer la llamada directa. ¿Querés intentar de nuevo?",
            "Sí", "No");

        if (reintentar)
        {
            status = await Permissions.RequestAsync<Permissions.Phone>();
            if (status == PermissionStatus.Granted)
                RealizarLlamada(numero);
        }
    }
    else
    {
        // Denegación PERMANENTE — redirigir a Configuración
        bool abrirConfig = await DisplayAlert(
            "Permiso bloqueado",
            "El permiso de teléfono fue denegado permanentemente. Abrí Configuración para habilitarlo.",
            "Abrir Configuración", "Cancelar");

        if (abrirConfig)
            AppInfo.Current.ShowSettingsUI();
    }
}

private void RealizarLlamada(string numero)
{
#if ANDROID
    var intent = new Intent(Intent.ActionCall);
    intent.SetData(AndroidUri.Parse($"tel:{numero}"));
    intent.SetFlags(ActivityFlags.NewTask);
    Platform.CurrentActivity?.StartActivity(intent);
#else
    PhoneDialer.Default.Open(numero);
#endif
}
```

### 26.2 Diagrama de flujo de permisos CALL_PHONE

```
LlamarConPermisoAsync("1155551234")
    │
    ├── CheckStatusAsync<Permissions.Phone>()
    │       ├── Granted → RealizarLlamada() ✅
    │       └── Denied / Unknown ↓
    │
    ├── RequestAsync<Permissions.Phone>()
    │       ├── Granted → RealizarLlamada() ✅
    │       └── Denied ↓
    │
    ├── ShouldShowRationale()?
    │       ├── true → "¿Querés intentar de nuevo?"
    │       │           ├── Sí → RequestAsync (segundo intento)
    │       │           └── No → fin
    │       │
    │       └── false → Denegación permanente
    │                   └── "Abrir Configuración" → AppInfo.ShowSettingsUI()
```

---

## 27. Sms.Default.ComposeAsync — Enviar SMS desde la app

### 27.1 Uso básico

```csharp
// ✅ CORRECTO — abre la app de SMS con número y texto precargados
var mensaje = new SmsMessage("Hola, te contactamos desde la app", new[] { "+5491155551234" });
await Sms.Default.ComposeAsync(mensaje);
```

### 27.2 Con múltiples destinatarios

```csharp
var destinatarios = new[] { "+5491155551234", "+5491166662222" };
var mensaje = new SmsMessage("Recordatorio: turno mañana a las 10hs", destinatarios);
await Sms.Default.ComposeAsync(mensaje);
```

### 27.3 Verificar soporte

```csharp
if (Sms.Default.IsComposeSupported)
{
    var mensaje = new SmsMessage("Hola", new[] { "1155551234" });
    await Sms.Default.ComposeAsync(mensaje);
}
else
{
    await DisplayAlert("Error", "Este dispositivo no puede enviar SMS", "OK");
}
```

### 27.4 Comparación Android vs iOS

| Aspecto | Android | iOS |
|---------|---------|-----|
| Permisos | Ninguno (abre app SMS) | Ninguno (abre app Mensajes) |
| Múltiples destinatarios | ✅ Soportado | ✅ Soportado |
| Texto precargado | ✅ | ✅ |
| Emulador/Simulador | Abre app SMS (no envía) | Abre app Mensajes (no envía) |

> **Nota**: `Sms.Default.ComposeAsync` **no envía el SMS directamente**. Abre la app de mensajes del SO con el texto y destinatarios precargados. El usuario toca "Enviar".

---

## 28. Email: Launcher y esquema mailto

### 28.1 Usando Email.Default.ComposeAsync

```csharp
// ✅ CORRECTO — abre el compositor de email
var emailMessage = new EmailMessage
{
    Subject = "Consulta desde la app",
    Body = "Hola, quería consultar sobre...",
    To = new List<string> { "info@ejemplo.com" }
};

await Email.Default.ComposeAsync(emailMessage);
```

### 28.2 Con CC, BCC y adjuntos

```csharp
var emailMessage = new EmailMessage
{
    Subject = "Reporte de incidencia",
    Body = "Adjunto captura del error",
    To = new List<string> { "soporte@ejemplo.com" },
    Cc = new List<string> { "supervisor@ejemplo.com" },
    Bcc = new List<string> { "log@ejemplo.com" }
};

// Adjuntar archivo
var archivo = Path.Combine(FileSystem.AppDataDirectory, "reporte.txt");
emailMessage.Attachments?.Add(new EmailAttachment(archivo));

await Email.Default.ComposeAsync(emailMessage);
```

### 28.3 Alternativa con Launcher y mailto

```csharp
// Alternativa usando esquema URI mailto:
var uri = "mailto:info@ejemplo.com?subject=Consulta&body=Hola";
await Launcher.Default.OpenAsync(uri);
```

### 28.4 Verificar soporte

```csharp
if (Email.Default.IsComposeSupported)
{
    // Abrir compositor de email
}
else
{
    // Alternativa: copiar dirección email al portapapeles
    await Clipboard.Default.SetTextAsync("info@ejemplo.com");
    await DisplayAlert("Sin email", "No hay app de email instalada. La dirección fue copiada al portapapeles.", "OK");
}
```

### 28.5 Comparación Android vs iOS

| Aspecto | Android | iOS |
|---------|---------|-----|
| API MAUI | `Email.Default.ComposeAsync` | `Email.Default.ComposeAsync` |
| Permisos | Ninguno | Ninguno |
| App por defecto | Gmail (o la que configure el usuario) | Mail |
| Adjuntos | ✅ Soportado | ✅ Soportado |
| `mailto:` vía Launcher | ✅ | ✅ |

---

## 29. Abrir WhatsApp, Telegram y otras apps de mensajería vía URI

### 29.1 WhatsApp

```csharp
// ✅ CORRECTO — abrir chat de WhatsApp con número específico
// Formato: https://wa.me/{código_país}{número} (sin + ni espacios)
var numero = "5491155551234"; // Argentina, sin +
var url = $"https://wa.me/{numero}?text=Hola%2C%20te%20contacto%20desde%20la%20app";

if (await Launcher.Default.CanOpenAsync(url))
{
    await Launcher.Default.OpenAsync(url);
}
else
{
    await DisplayAlert("Error", "WhatsApp no está instalado", "OK");
}
```

### 29.2 Telegram

```csharp
// Por nombre de usuario
var url = "https://t.me/nombre_usuario";

// Por número de teléfono (requiere que el usuario tenga Telegram)
var urlTel = "https://t.me/+5491155551234";

if (await Launcher.Default.CanOpenAsync(url))
{
    await Launcher.Default.OpenAsync(url);
}
else
{
    await DisplayAlert("Error", "Telegram no está instalado", "OK");
}
```

### 29.3 Tabla de URIs de mensajería

| App | URI | Ejemplo |
|-----|-----|---------|
| WhatsApp | `https://wa.me/{número}` | `https://wa.me/5491155551234` |
| WhatsApp con texto | `https://wa.me/{número}?text={mensaje}` | `https://wa.me/5491155551234?text=Hola` |
| Telegram (usuario) | `https://t.me/{usuario}` | `https://t.me/soporte_app` |
| Telegram (teléfono) | `https://t.me/+{número}` | `https://t.me/+5491155551234` |

### 29.4 Método genérico para abrir apps externas

```csharp
private async Task AbrirAppExternaAsync(string uri, string nombreApp)
{
    try
    {
        if (await Launcher.Default.CanOpenAsync(uri))
        {
            await Launcher.Default.OpenAsync(uri);
        }
        else
        {
            await DisplayAlert("No disponible",
                $"{nombreApp} no está instalado en este dispositivo", "OK");
        }
    }
    catch (Exception ex)
    {
        await DisplayAlert("Error", $"No se pudo abrir {nombreApp}: {ex.Message}", "OK");
    }
}

// Uso:
await AbrirAppExternaAsync("https://wa.me/5491155551234", "WhatsApp");
await AbrirAppExternaAsync("https://t.me/soporte", "Telegram");
await AbrirAppExternaAsync("mailto:info@ejemplo.com", "Email");
```

---

## 30. API Reference: Clases y Métodos de PhoneDialer, Sms, Email y Launcher

### 30.1 PhoneDialer

| Miembro | Tipo | Descripción |
|---------|------|-------------|
| `PhoneDialer.Default` | `IPhoneDialer` | Instancia por defecto del servicio |
| `.IsSupported` | `bool` | `true` si el dispositivo puede hacer llamadas |
| `.Open(string number)` | `void` | Abre el marcador con el número precargado |

### 30.2 Sms

| Miembro | Tipo | Descripción |
|---------|------|-------------|
| `Sms.Default` | `ISms` | Instancia por defecto del servicio |
| `.IsComposeSupported` | `bool` | `true` si el dispositivo puede abrir compositor de SMS |
| `.ComposeAsync(SmsMessage)` | `Task` | Abre la app de SMS con mensaje precargado |
| `SmsMessage(string body, string[] recipients)` | constructor | Crea un mensaje SMS con cuerpo y destinatarios |

### 30.3 Email

| Miembro | Tipo | Descripción |
|---------|------|-------------|
| `Email.Default` | `IEmail` | Instancia por defecto del servicio |
| `.IsComposeSupported` | `bool` | `true` si el dispositivo puede abrir compositor de email |
| `.ComposeAsync(EmailMessage)` | `Task` | Abre la app de email con mensaje precargado |
| `EmailMessage` | clase | Contiene `Subject`, `Body`, `To`, `Cc`, `Bcc`, `Attachments` |
| `EmailAttachment(string fullPath)` | constructor | Crea un adjunto a partir de una ruta de archivo |

### 30.4 Launcher

| Miembro | Tipo | Descripción |
|---------|------|-------------|
| `Launcher.Default` | `ILauncher` | Instancia por defecto del servicio |
| `.OpenAsync(string uri)` | `Task<bool>` | Abre el URI en la app correspondiente |
| `.OpenAsync(Uri uri)` | `Task<bool>` | Sobrecarga que acepta `Uri` |
| `.CanOpenAsync(string uri)` | `Task<bool>` | Verifica si hay una app que pueda abrir el URI |
| `.TryOpenAsync(string uri)` | `Task<bool>` | Intenta abrir; devuelve `false` si no puede |

---

## 31. Ejemplo Completo: Página con PhoneDialer

### 31.1 XAML — PhoneDialerPage.xaml

```xml
<?xml version="1.0" encoding="utf-8" ?>
<ContentPage xmlns="http://schemas.microsoft.com/dotnet/2021/maui"
             xmlns:x="http://schemas.microsoft.com/winfx/2009/xaml"
             x:Class="MiApp.PhoneDialerPage"
             Title="PhoneDialer">

    <VerticalStackLayout Padding="20" Spacing="15">
        <Label Text="Llamar por teléfono"
               FontSize="24" FontAttributes="Bold" />

        <Entry x:Name="EntryNumero"
               Placeholder="Número de teléfono"
               Keyboard="Telephone" />

        <Button Text="📞 Abrir marcador"
                Clicked="OnAbrirMarcador_Clicked" />

        <Button x:Name="BtnLlamadaDirecta"
                Text="📱 Llamada directa (Android)"
                Clicked="OnLlamadaDirecta_Clicked"
                IsVisible="False" />

        <Label x:Name="LblEstado"
               TextColor="Gray"
               FontSize="14" />
    </VerticalStackLayout>
</ContentPage>
```

### 31.2 Code-behind — PhoneDialerPage.xaml.cs

```csharp
using Microsoft.Maui.ApplicationModel.Communication;

namespace MiApp;

public partial class PhoneDialerPage : ContentPage
{
    public PhoneDialerPage()
    {
        InitializeComponent();

        // Mostrar botón de llamada directa solo en Android
#if ANDROID
        BtnLlamadaDirecta.IsVisible = true;
#endif
    }

    private async void OnAbrirMarcador_Clicked(object sender, EventArgs e)
    {
        var numero = EntryNumero.Text?.Trim();
        if (string.IsNullOrEmpty(numero))
        {
            await DisplayAlert("Error", "Ingresá un número de teléfono", "OK");
            return;
        }

        if (PhoneDialer.Default.IsSupported)
        {
            try
            {
                PhoneDialer.Default.Open(numero);
                LblEstado.Text = $"Marcador abierto con: {numero}";
            }
            catch (Exception ex)
            {
                await DisplayAlert("Error", $"No se pudo abrir el marcador: {ex.Message}", "OK");
            }
        }
        else
        {
            await DisplayAlert("No soportado",
                "Este dispositivo no puede hacer llamadas telefónicas", "OK");
        }
    }

    private async void OnLlamadaDirecta_Clicked(object sender, EventArgs e)
    {
        var numero = EntryNumero.Text?.Trim();
        if (string.IsNullOrEmpty(numero))
        {
            await DisplayAlert("Error", "Ingresá un número de teléfono", "OK");
            return;
        }

        await LlamarConPermisoAsync(numero);
    }

    private async Task LlamarConPermisoAsync(string numero)
    {
        var status = await Permissions.CheckStatusAsync<Permissions.Phone>();

        if (status != PermissionStatus.Granted)
        {
            status = await Permissions.RequestAsync<Permissions.Phone>();
        }

        if (status == PermissionStatus.Granted)
        {
#if ANDROID
            var intent = new Android.Content.Intent(Android.Content.Intent.ActionCall);
            intent.SetData(Android.Net.Uri.Parse($"tel:{numero}"));
            intent.SetFlags(Android.Content.ActivityFlags.NewTask);
            Platform.CurrentActivity?.StartActivity(intent);
            LblEstado.Text = $"Llamando a: {numero}";
#endif
        }
        else
        {
            if (Permissions.ShouldShowRationale<Permissions.Phone>())
            {
                bool reintentar = await DisplayAlert(
                    "Permiso necesario",
                    "Necesitamos acceso al teléfono para hacer la llamada directa.",
                    "Reintentar", "Cancelar");

                if (reintentar)
                    await LlamarConPermisoAsync(numero);
            }
            else
            {
                bool abrirConfig = await DisplayAlert(
                    "Permiso bloqueado",
                    "El permiso de teléfono fue denegado. Abrí Configuración para habilitarlo.",
                    "Abrir Configuración", "Cancelar");

                if (abrirConfig)
                    AppInfo.Current.ShowSettingsUI();
            }
        }
    }
}
```

---

## 32. Ejemplo Completo: Página de Contacto Multicanal

### 32.1 XAML — ContactoPage.xaml

```xml
<?xml version="1.0" encoding="utf-8" ?>
<ContentPage xmlns="http://schemas.microsoft.com/dotnet/2021/maui"
             xmlns:x="http://schemas.microsoft.com/winfx/2009/xaml"
             x:Class="MiApp.ContactoPage"
             Title="Contacto">

    <ScrollView>
        <VerticalStackLayout Padding="20" Spacing="15">
            <Label Text="Contactanos"
                   FontSize="28" FontAttributes="Bold" />

            <Label Text="Elegí cómo querés comunicarte con nosotros:"
                   FontSize="16" TextColor="Gray" />

            <!-- Teléfono -->
            <Frame Padding="15" CornerRadius="10">
                <VerticalStackLayout Spacing="8">
                    <Label Text="📞 Teléfono" FontSize="18" FontAttributes="Bold" />
                    <Label Text="+54 9 11 5555-1234" TextColor="Gray" />
                    <Button Text="Llamar"
                            Clicked="OnLlamar_Clicked"
                            BackgroundColor="{StaticResource Primary}" />
                </VerticalStackLayout>
            </Frame>

            <!-- SMS -->
            <Frame Padding="15" CornerRadius="10">
                <VerticalStackLayout Spacing="8">
                    <Label Text="💬 SMS" FontSize="18" FontAttributes="Bold" />
                    <Label Text="+54 9 11 5555-1234" TextColor="Gray" />
                    <Button Text="Enviar SMS"
                            Clicked="OnSms_Clicked"
                            BackgroundColor="{StaticResource Primary}" />
                </VerticalStackLayout>
            </Frame>

            <!-- Email -->
            <Frame Padding="15" CornerRadius="10">
                <VerticalStackLayout Spacing="8">
                    <Label Text="📧 Email" FontSize="18" FontAttributes="Bold" />
                    <Label Text="info@ejemplo.com" TextColor="Gray" />
                    <Button Text="Enviar email"
                            Clicked="OnEmail_Clicked"
                            BackgroundColor="{StaticResource Primary}" />
                </VerticalStackLayout>
            </Frame>

            <!-- WhatsApp -->
            <Frame Padding="15" CornerRadius="10">
                <VerticalStackLayout Spacing="8">
                    <Label Text="WhatsApp" FontSize="18" FontAttributes="Bold" />
                    <Label Text="+54 9 11 5555-1234" TextColor="Gray" />
                    <Button Text="Abrir WhatsApp"
                            Clicked="OnWhatsApp_Clicked"
                            BackgroundColor="#25D366" TextColor="White" />
                </VerticalStackLayout>
            </Frame>

            <!-- Telegram -->
            <Frame Padding="15" CornerRadius="10">
                <VerticalStackLayout Spacing="8">
                    <Label Text="Telegram" FontSize="18" FontAttributes="Bold" />
                    <Label Text="@soporte_app" TextColor="Gray" />
                    <Button Text="Abrir Telegram"
                            Clicked="OnTelegram_Clicked"
                            BackgroundColor="#0088cc" TextColor="White" />
                </VerticalStackLayout>
            </Frame>

            <Label x:Name="LblEstado"
                   TextColor="Gray" FontSize="12"
                   HorizontalOptions="Center" />
        </VerticalStackLayout>
    </ScrollView>
</ContentPage>
```

### 32.2 Code-behind — ContactoPage.xaml.cs

```csharp
using Microsoft.Maui.ApplicationModel;
using Microsoft.Maui.ApplicationModel.Communication;

namespace MiApp;

public partial class ContactoPage : ContentPage
{
    private const string Telefono = "+5491155551234";
    private const string EmailContacto = "info@ejemplo.com";
    private const string NumeroWhatsApp = "5491155551234"; // sin +
    private const string UsuarioTelegram = "soporte_app";

    public ContactoPage()
    {
        InitializeComponent();
    }

    // ── Teléfono ──────────────────────────────────────────────
    private async void OnLlamar_Clicked(object sender, EventArgs e)
    {
        if (PhoneDialer.Default.IsSupported)
        {
            try
            {
                PhoneDialer.Default.Open(Telefono);
            }
            catch (Exception ex)
            {
                await DisplayAlert("Error", ex.Message, "OK");
            }
        }
        else
        {
            await MostrarNoSoportado("llamadas telefónicas");
        }
    }

    // ── SMS ───────────────────────────────────────────────────
    private async void OnSms_Clicked(object sender, EventArgs e)
    {
        if (Sms.Default.IsComposeSupported)
        {
            var mensaje = new SmsMessage("Hola, quería consultar...", new[] { Telefono });
            await Sms.Default.ComposeAsync(mensaje);
        }
        else
        {
            await MostrarNoSoportado("SMS");
        }
    }

    // ── Email ─────────────────────────────────────────────────
    private async void OnEmail_Clicked(object sender, EventArgs e)
    {
        if (Email.Default.IsComposeSupported)
        {
            var email = new EmailMessage
            {
                Subject = "Consulta desde la app",
                Body = "Hola, quería consultar sobre...",
                To = new List<string> { EmailContacto }
            };
            await Email.Default.ComposeAsync(email);
        }
        else
        {
            // Fallback: copiar email al portapapeles
            await Clipboard.Default.SetTextAsync(EmailContacto);
            await DisplayAlert("Sin app de email",
                $"La dirección {EmailContacto} fue copiada al portapapeles", "OK");
        }
    }

    // ── WhatsApp ──────────────────────────────────────────────
    private async void OnWhatsApp_Clicked(object sender, EventArgs e)
    {
        await AbrirAppExternaAsync(
            $"https://wa.me/{NumeroWhatsApp}?text=Hola%2C%20consulto%20desde%20la%20app",
            "WhatsApp");
    }

    // ── Telegram ──────────────────────────────────────────────
    private async void OnTelegram_Clicked(object sender, EventArgs e)
    {
        await AbrirAppExternaAsync($"https://t.me/{UsuarioTelegram}", "Telegram");
    }

    // ── Helpers ───────────────────────────────────────────────
    private async Task AbrirAppExternaAsync(string uri, string nombreApp)
    {
        try
        {
            if (await Launcher.Default.CanOpenAsync(uri))
            {
                await Launcher.Default.OpenAsync(uri);
            }
            else
            {
                await DisplayAlert("No disponible",
                    $"{nombreApp} no está instalado en este dispositivo", "OK");
            }
        }
        catch (Exception ex)
        {
            await DisplayAlert("Error", $"No se pudo abrir {nombreApp}: {ex.Message}", "OK");
        }
    }

    private async Task MostrarNoSoportado(string funcionalidad)
    {
        await DisplayAlert("No soportado",
            $"Este dispositivo no soporta {funcionalidad}", "OK");
    }
}
```

### 32.3 Diagrama de flujo — Contacto Multicanal

```
ContactoPage
    │
    ├── [Llamar] → PhoneDialer.IsSupported?
    │                ├── Sí → PhoneDialer.Open(número)
    │                └── No → "No soportado"
    │
    ├── [SMS] → Sms.IsComposeSupported?
    │             ├── Sí → Sms.ComposeAsync(mensaje)
    │             └── No → "No soportado"
    │
    ├── [Email] → Email.IsComposeSupported?
    │              ├── Sí → Email.ComposeAsync(email)
    │              └── No → Copiar al portapapeles
    │
    ├── [WhatsApp] → Launcher.CanOpenAsync("wa.me/...")?
    │                  ├── Sí → Launcher.OpenAsync(url)
    │                  └── No → "WhatsApp no instalado"
    │
    └── [Telegram] → Launcher.CanOpenAsync("t.me/...")?
                       ├── Sí → Launcher.OpenAsync(url)
                       └── No → "Telegram no instalado"
```

---

# PARTE V — REFERENCIA

---

## 33. Troubleshooting Unificado (Cámara + GPS + PhoneDialer + SMS)

### 33.1 Errores de Cámara

| Error | Causa | Solución |
|-------|-------|----------|
| `Java.Lang.SecurityException: Permission Denial` | Falta permiso en `AndroidManifest.xml` | Agregar `<uses-permission android:name="android.permission.CAMERA" />` |
| App crashea en iOS al abrir cámara | Falta `NSCameraUsageDescription` en `Info.plist` | Agregar la clave con texto descriptivo |
| `FileNotFoundException` al guardar foto | Falta `FileProvider` en el manifiesto | Configurar `<provider>` y `file_paths.xml` (solo para MediaPicker) |
| Permiso concedido pero cámara no abre | Emulador sin cámara configurada | Configurar cámara virtual en AVD Manager o probar en dispositivo físico |
| `READ_EXTERNAL_STORAGE` no funciona en Android 13 | Permiso deprecated en API 33 | Usar `READ_MEDIA_IMAGES` en su lugar |
| `CameraException: No camera available on device` (SIGABRT) | `CameraView` declarado en XAML en dispositivo sin cámara | No declarar `CameraView` en XAML; crearlo dinámicamente (ver sección 8.1) |
| `CameraView` crashea con `IsVisible="False"` | `IsVisible` no evita `ConnectHandler` | Usar `ContentView` como contenedor + creación dinámica (ver sección 8.1) |

### 33.2 Errores de GPS

| Error | Causa | Solución |
|-------|-------|----------|
| `Java.Lang.SecurityException` al pedir ubicación | Falta permiso en `AndroidManifest.xml` | Agregar `<uses-permission android:name="android.permission.ACCESS_FINE_LOCATION" />` |
| App crashea en iOS al pedir ubicación | Falta `NSLocationWhenInUseUsageDescription` en `Info.plist` | Agregar la clave con texto descriptivo |
| `FeatureNotEnabledException` | GPS desactivado en el dispositivo | Indicar al usuario que active el GPS desde ajustes |
| `FeatureNotSupportedException` | Dispositivo sin GPS (tablet wifi-only, emulador) | Verificar con `try-catch` y mostrar mensaje |
| Ubicación devuelve `null` | GPS activado pero sin señal (interior de edificio) | Reintentar, mover el dispositivo, usar `GeolocationAccuracy.Medium` |
| Precisión muy baja (>100m) | Usando `GeolocationAccuracy.Low` o señal GPS débil | Usar `GeolocationAccuracy.Best` y esperar más tiempo |
| `GetLocationAsync` tarda mucho | Timeout corto + GPS frío ("cold start") | Aumentar timeout a 15-30s |
| `PermissionException` al pedir ubicación | Se llamó a `GetLocationAsync` sin verificar permisos antes; el usuario canceló/denegó el diálogo del SO | Verificar permisos con `CheckStatusAsync` / `RequestAsync` **antes** de llamar a `GetLocationAsync`. No usar excepciones como control de flujo (ver sección 17.3) |

### 33.3 Errores comunes (ambos)

| Error | Causa | Solución |
|-------|-------|----------|
| Permisos se muestran como denegados en primera ejecución (Android) | `CheckStatusAsync` devuelve `Denied` en vez de `Unknown` | Siempre llamar a `RequestAsync` si no está `Granted` (sección 3.2) |
| `ShouldShowRationale` devuelve `false` pero el permiso nunca se pidió | Comportamiento esperado de Android | Llamar a `ShouldShowRationale` después de `RequestAsync` (sección 3.3) |

### 33.4 Errores de PhoneDialer / SMS / Email

| Error | Causa | Solución |
|-------|-------|----------|
| `ArgumentNullException` en `PhoneDialer.Open()` | Se pasó cadena vacía o `null` | Validar que el número no sea vacío antes de llamar |
| `FeatureNotSupportedException` en `PhoneDialer.Open()` | Dispositivo sin capacidad de llamadas (tablet wifi-only) | Verificar `PhoneDialer.Default.IsSupported` antes de llamar |
| `PhoneDialer.Default.IsSupported` devuelve `false` en dispositivo físico con SIM | Falta `<queries>` en `AndroidManifest.xml` (Android 11+ / API 30) | Agregar `<queries>` con intent `android.intent.action.DIAL` y esquema `tel` (sección 22.1) |
| `SecurityException` al usar `Intent.ActionCall` | Falta `CALL_PHONE` en `AndroidManifest.xml` o no se pidió en runtime | Agregar permiso en Manifest + pedir con `Permissions.RequestAsync<Permissions.Phone>()` |
| `ActivityNotFoundException` al abrir WhatsApp/Telegram | App no instalada en el dispositivo | Verificar con `Launcher.CanOpenAsync()` antes de abrir |
| `CanOpenAsync` devuelve `false` aunque la app está instalada (Android 11+) | Falta `<queries>` en `AndroidManifest.xml` | Agregar `<queries>` con los intents/packages necesarios (sección 22.1) |
| `CanOpenAsync` devuelve `false` en iOS | Falta esquema en `LSApplicationQueriesSchemes` | Agregar el esquema (`whatsapp`, `tg`, etc.) en `Info.plist` (sección 22.2) |
| `Sms.ComposeAsync` no hace nada en emulador | Emulador no tiene app SMS funcional | Probar en dispositivo físico |
| `Email.ComposeAsync` lanza excepción en emulador | No hay app de email configurada | Verificar `Email.Default.IsComposeSupported` antes |

---

## 34. Checklists de Verificación

### 34.1 Checklist — Cámara con MediaPicker

```
✅ AndroidManifest.xml
   ├── android.permission.CAMERA
   ├── android.permission.READ_EXTERNAL_STORAGE (maxSdkVersion="32")
   ├── android.permission.WRITE_EXTERNAL_STORAGE (maxSdkVersion="32")
   ├── android.permission.READ_MEDIA_IMAGES (API 33+)
   ├── uses-feature camera required="false"
   └── FileProvider configurado

✅ file_paths.xml
   ├── external-files-path para fotos
   ├── cache-path para temporales
   └── Ubicado en Platforms/Android/Resources/xml/

✅ Info.plist (iOS)
   ├── NSCameraUsageDescription (texto descriptivo)
   ├── NSPhotoLibraryUsageDescription
   └── NSPhotoLibraryAddUsageDescription

✅ Código C#
   ├── Verificación de versión de API (Tiramisu vs M)
   ├── Manejo de denegación temporal y permanente
   ├── Redirección a Configuración del sistema
   ├── Botón de reintentar visible al denegar
   └── try-catch en toda la cadena
```

### 34.2 Checklist — Cámara con CameraView (CommunityToolkit)

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
   ├── Siempre llamar RequestAsync si no está Granted
   ├── ShouldShowRationale DESPUÉS de RequestAsync
   ├── Null-check en _cameraView en flash, captura, etc.
   └── Limpiar CameraView en OnDisappearing
```

### 34.3 Checklist — GPS / Geolocalización

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
   ├── Siempre llamar RequestAsync si no está Granted
   ├── ShouldShowRationale DESPUÉS de RequestAsync
   ├── Verificar permisos ANTES de GetLocationAsync (no depender de PermissionException)
   ├── CancellationToken para cancelar consultas GPS (con = default para uso opcional)
   ├── Manejo de FeatureNotEnabledException (GPS apagado)
   ├── Manejo de FeatureNotSupportedException (sin hardware GPS)
   ├── Timeout adecuado en GeolocationRequest (10-30 segundos)
   ├── Redirección a Configuración en denegación permanente
   └── try-catch en toda la cadena
```

### 34.4 Checklist — PhoneDialer (marcador)

```
✅ AndroidManifest.xml
   ├── NO necesita permisos para PhoneDialer.Open()
   ├── android.permission.CALL_PHONE (solo si usás Intent.ActionCall)
   └── <queries> con intent DIAL y scheme tel (Android 11+)

✅ Info.plist (iOS)
   └── NO necesita configuración para tel:/sms: (esquemas del sistema)
   └── LSApplicationQueriesSchemes solo para apps de terceros (whatsapp, tg)

✅ Código C#
   ├── Verificar PhoneDialer.Default.IsSupported (requiere <queries> en Android 11+)
   ├── Validar número no vacío antes de Open()
   ├── try-catch para FeatureNotSupportedException
   └── #if ANDROID para código de Intent.ActionCall
```

### 34.5 Checklist — SMS

```
✅ AndroidManifest.xml
   └── <queries> con intent SENDTO y scheme smsto (Android 11+)

✅ Info.plist (iOS)
   └── LSApplicationQueriesSchemes: sms

✅ Código C#
   ├── Verificar Sms.Default.IsComposeSupported
   ├── Crear SmsMessage con body y destinatarios
   └── try-catch en ComposeAsync
```

### 34.6 Checklist — Email

```
✅ AndroidManifest.xml
   └── <queries> con intent SENDTO y scheme mailto (Android 11+)

✅ Info.plist (iOS)
   └── LSApplicationQueriesSchemes: mailto

✅ Código C#
   ├── Verificar Email.Default.IsComposeSupported
   ├── Crear EmailMessage con Subject, Body, To
   ├── Fallback: copiar email al portapapeles si no hay app
   └── try-catch en ComposeAsync
```

### 34.7 Checklist — WhatsApp / Telegram vía Launcher

```
✅ AndroidManifest.xml
   ├── <package android:name="com.whatsapp" /> en <queries>
   └── <package android:name="org.telegram.messenger" /> en <queries>

✅ Info.plist (iOS)
   └── LSApplicationQueriesSchemes: whatsapp, tg

✅ Código C#
   ├── Verificar Launcher.CanOpenAsync(uri) ANTES de abrir
   ├── Formato WhatsApp: https://wa.me/{número_sin_+}
   ├── Formato Telegram: https://t.me/{usuario}
   ├── Mostrar mensaje si app no instalada
   └── try-catch en OpenAsync
```

### 34.8 Probar en emulador

**Android (Android Studio AVD):**
```bash
# Revocar/conceder permisos de cámara
adb shell pm revoke com.tuapp.package android.permission.CAMERA
adb shell pm grant com.tuapp.package android.permission.CAMERA

# Revocar/conceder permisos de teléfono (CALL_PHONE)
adb shell pm revoke com.tuapp.package android.permission.CALL_PHONE
adb shell pm grant com.tuapp.package android.permission.CALL_PHONE

# Establecer ubicación GPS simulada
adb emu geo fix -34.5986 -58.4208   # Buenos Aires

# Verificar estado de permisos
adb shell dumpsys package com.tuapp.package | grep permission
```

**iOS (Xcode Simulator):**
- **Cámara**: Simuladores no tienen cámara real → usar dispositivo físico.
- **GPS**: Menú **Features > Location** > elegir ubicación predefinida.
- **PhoneDialer**: `IsSupported` devuelve `false` en simulador → probar en dispositivo físico.
- **SMS/Email**: Abren compositor pero no envían realmente.
- **Resetear permisos**: **Device > Erase All Content and Settings**.

### 34.9 Resumen de permisos por funcionalidad

| Funcionalidad | Android Manifest | iOS Info.plist | Permiso runtime |
|--------------|-----------------|----------------|-----------------|
| Cámara (MediaPicker) | `CAMERA`, `READ_MEDIA_IMAGES` | `NSCameraUsageDescription` | Sí |
| Cámara (CameraView) | `CAMERA` | `NSCameraUsageDescription` | Sí |
| GPS | `ACCESS_FINE_LOCATION`, `ACCESS_COARSE_LOCATION` | `NSLocationWhenInUseUsageDescription` | Sí |
| PhoneDialer (marcador) | Ninguno | Ninguno | No |
| Llamada directa | `CALL_PHONE` | ❌ No disponible | Sí (Android) |
| SMS | Ninguno | Ninguno | No |
| Email | Ninguno | Ninguno | No |
| WhatsApp/Telegram | `<queries>` (Android 11+) | `LSApplicationQueriesSchemes` | No |

---

> **Nota sobre seguridad y privacidad**:
> - Las fotos capturadas en `FileSystem.AppDataDirectory` son privadas de la app.
> - La ubicación es un dato sensible. Guardá coordenadas solo si es necesario.
> - Si transmitís datos a un servidor, usá HTTPS.
> - Informá al usuario claramente por qué necesitás cada permiso — requisito de Apple y buena práctica de UX.
> - Los números de teléfono son datos personales — no los guardes sin consentimiento del usuario.
