

tengo el siguiente ejemplo de uso del gps

```

using Ejemplo_Maui_GPS.Services;

namespace Ejemplo_Maui_GPS.Pages;

public partial class MainPage : ContentPage
{
    GpsService _gps = default!;
    private CancellationTokenSource? _cts;

    string coordenadas;
    public string Coordenadas
    {
        get { return coordenadas; }
        set
        {
            if (coordenadas != value)
            {
                coordenadas = value;
                OnPropertyChanged();
            }
        }
    }

    public MainPage(GpsService gps)
    {
        InitializeComponent();
        BindingContext = this;
        _gps = gps;
    }

    async private void OnGetGeoLocalizacion_Clicked(object sender, EventArgs e)
    {
        _cts?.Cancel();
        _cts?.Dispose();
        _cts = new CancellationTokenSource();

        btnMostrarCoordenadas.IsEnabled = false;
        btnCancelarCoordenadas.IsVisible = true;
        Coordenadas = "";

        try
        {
            //obtener la ubicacion gps
            Coordenadas = "Obteniendo ubicación GPS...";
            var location = await _gps.ObtenerUbicacionAsync(_cts.Token);

            if (location == null)
            {
                Coordenadas = "No se pudo obtener ubicación (GPS sin señal).";
                return;
            }

            Coordenadas = $"Lat: {location.Latitude:F6}, Lng: {location.Longitude:F6}";

        }
        catch (OperationCanceledException)
        {
            Coordenadas = "Operación cancelada por el usuario.";
        }
        catch (FeatureNotEnabledException)
        {
            Coordenadas = "El GPS está desactivado. Activalo desde ajustes.";
        }
        catch (PermissionException)                                    // ← FALTA ESTE
        {
            Coordenadas = "Permiso de ubicación denegado.";
        }
        catch (FeatureNotSupportedException)                           // ← opcional
        {
            Coordenadas = "Este dispositivo no soporta GPS.";
        }
        finally
        {
            btnMostrarCoordenadas.IsEnabled = true;
            btnCancelarCoordenadas.IsVisible = false;
            _cts?.Dispose();
            _cts = null;
        }
    }
    
    private void OnCancelarGeoLocalizacion_Clicked(object sender, EventArgs e)
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

    async private void OnMostrarLocalizacionEnMapaClicked(object sender, EventArgs e)
    {
        Location? location = await _gps.ObtenerUbicacionAsync();

        if (location == null)
        {
            await DisplayAlertAsync("Advertencia", "Primero debe obtener la localización", "OK");
            return;
        }

        try
        {
            // Formato: https://maps.google.com/?q=latitude,longitude
            string url = $"https://maps.google.com/?q={location.Latitude},{location.Longitude}";
            await Browser.Default.OpenAsync(url, BrowserLaunchMode.SystemPreferred);
        }
        catch (Exception ex)
        {
            await DisplayAlertAsync("Error", $"No se pudo abrir Google Maps: {ex.Message}", "OK");
        }
    }
}
```

```
namespace Ejemplo_Maui_GPS.Services;

public class GpsService
{
    public async Task<Location?> ObtenerUbicacionAsync(CancellationToken ct = default)
    {
        // verifica si tiene permisos
        var status = await Permissions.CheckStatusAsync<Permissions.LocationWhenInUse>();

        // sino tiene los pide 
        if (status != PermissionStatus.Granted)
        {
            status = await Permissions.RequestAsync<Permissions.LocationWhenInUse>();
        }

        // si despues de pedir sigue sin permisos, salir
        if (status != PermissionStatus.Granted)
            return null;

        // con permisos confirmados, pedir ubicación
        var request = new GeolocationRequest(GeolocationAccuracy.Best, TimeSpan.FromSeconds(15));
        return await Geolocation.Default.GetLocationAsync(request, ct);
    }
}
```




tengo el siguiente ejemplo de uso de la camara

```
<?xml version="1.0" encoding="utf-8" ?>
<ContentPage xmlns="http://schemas.microsoft.com/dotnet/2021/maui"
             xmlns:x="http://schemas.microsoft.com/winfx/2009/xaml"
             x:Class="Ejemplo_Photo_MiMediaPicker_Callback.Pages.MyMediaPickerPage"
             Title="Camera Page"
             BackgroundColor="#000000">

    <Grid x:Name="DynamicLayout" RowDefinitions="Auto,*,Auto" ColumnDefinitions="*"
          HorizontalOptions="Fill" VerticalOptions="Fill">

        <!-- barra de botones -->
        <Button x:Name="BtnFlashButton" Grid.Row="0" Grid.Column="0"
                Clicked="OnActiveFlashClicked"
                IsVisible="False"
                HorizontalOptions="Center" VerticalOptions="Center"
                Background="#aa000000" CornerRadius="8">
            <Button.ImageSource>
                <FontImageSource Glyph="{Binding FlashIcon}" FontFamily="MaterialIconsOutlined" />
            </Button.ImageSource>
        </Button>

        <!-- Contenedor para CameraView (se crea desde código) -->
        <ContentView x:Name="CameraContainer" Grid.Row="1" Grid.Column="0"
                     IsVisible="False"
                     HorizontalOptions="Fill" VerticalOptions="Fill" />

        <!-- Overlay de permiso denegado -->
        <Grid x:Name="PermissionDeniedOverlay"
              Grid.Row="1" Grid.Column="0"
              IsVisible="False"
              BackgroundColor="#CC000000"
              HorizontalOptions="Fill" VerticalOptions="Fill">

            <VerticalStackLayout Spacing="20"
                                 HorizontalOptions="Center"
                                 VerticalOptions="Center"
                                 Padding="32">

                <!-- Ícono de cámara tachada -->
                <Label Text="no_photography"
                       FontFamily="MaterialIconsOutlined"
                       FontSize="80"
                       TextColor="#AAAAAA"
                       HorizontalOptions="Center"/>

                <!-- Título -->
                <Label x:Name="LblPermissionTitle"
                       Text="Sin acceso a la cámara"
                       FontSize="20"
                       FontAttributes="Bold"
                       TextColor="White"
                       HorizontalOptions="Center"
                       HorizontalTextAlignment="Center"/>

                <!-- Descripción -->
                <Label x:Name="LblPermissionMessage"
                       Text="Para tomar fotos necesitamos acceso a la cámara."
                       FontSize="14"
                       TextColor="#CCCCCC"
                       HorizontalOptions="Center"
                       HorizontalTextAlignment="Center"/>

                <!-- Botón pedir permiso (visible cuando se puede reintentar) -->
                <Button x:Name="BtnPedirPermiso"
                        Text="Pedir permiso"
                        Clicked="OnPedirPermisoClicked"
                        BackgroundColor="#512BD4"
                        TextColor="White"
                        CornerRadius="8"
                        Padding="20,12"
                        HorizontalOptions="Center"
                        IsVisible="False"/>

                <!-- Botón ir a Settings (visible cuando es denegado permanente) -->
                <Button x:Name="BtnGoToSettings"
                        Text="Abrir configuración"
                        Clicked="OnGoToSettingsClicked"
                        BackgroundColor="#512BD4"
                        TextColor="White"
                        CornerRadius="8"
                        Padding="20,12"
                        HorizontalOptions="Center"/>

                <!-- Botón volver -->
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

        <!-- barra de botones -->
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

```
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

    public Action<Stream>? OnPhotoCallback { get; set; }

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

        var status = await Permissions.CheckStatusAsync<Permissions.Camera>();
        if (status == PermissionStatus.Granted && _cameraView != null)
            await SeleccionarCamaraAsync();
    }

    // PERMISOS

    private async Task EvaluarYMostrarEstadoPermisoAsync()
    {
        var status = await Permissions.CheckStatusAsync<Permissions.Camera>();

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

        if (status == PermissionStatus.Restricted)
        {
            MostrarOverlayPermiso("Acceso restringido",
                "El acceso a la cámara está restringido por una política del dispositivo. Consultá con el administrador.",
                puedeReintentar: false);
            return;
        }

        bool puedeReintentar = false;

#if ANDROID
        puedeReintentar = Permissions.ShouldShowRationale<Permissions.Camera>();
#endif

        MostrarOverlayPermiso(
            titulo: puedeReintentar
                ? "Permiso de cámara necesario"
                : "Acceso a la cámara denegado",
            mensaje: puedeReintentar
                ? "Para tomar fotos necesitamos acceso a la cámara. Podés intentar conceder el permiso."
                : "Para tomar fotos necesitamos acceso a la cámara. Habilitalo desde los ajustes de la aplicación.",
            puedeReintentar: puedeReintentar);
    }

    // OVERLAY — mostrar / ocultar

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

    // HANDLERS DEL OVERLAY

    private async void OnPedirPermisoClicked(object sender, EventArgs e)
    {
        await EvaluarYMostrarEstadoPermisoAsync();
    }

    private void OnGoToSettingsClicked(object sender, EventArgs e)
    {
        AppInfo.ShowSettingsUI();
    }

    private async void OnVolverClicked(object sender, EventArgs e)
    {
        OnPhotoCallback?.Invoke(null!);
        await Shell.Current.GoToAsync("..");
    }

    // CÁMARA

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
                    //var image = new Image { Source = ImageSource.FromStream(() => e.Media) };
                    OnPhotoCallback?.Invoke(e.Media);
                }
                await Shell.Current.GoToAsync("..");
            }
            else
            {
                await DisplayAlert("Error del dispositivo", "El dispositivo no está activo", "OK");
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

    // FLASH Y ORIENTACIÓN

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
        catch (Exception ex)
        {
            Debug.WriteLine($"UpdateLayoutOrientation error: {ex.Message}");
        }
    }
}
```


en el ejemplo de la camara manejo la situación de que los permisos que sean denegados, dandole al usuario la oportunidad de que pueda
lanzar la configuración de la aplicación en caso de que haya sido totalmente denegada los permoss de la camara,

necesito un dialogo estandar para el gps tomando en cuenta segun la logica usada en el uso de la camara para el manejo de los permisos