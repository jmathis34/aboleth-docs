# Audio Device Management

Aboleth STT provides runtime control over which microphone is used for speech capture. You can enumerate available audio input devices, switch between them, and respond to device changes -- all without restarting the STT system.

---

## API Reference

| Function | Description |
|----------|-------------|
| `RefreshAudioDevices()` | Re-scan the system for available audio input devices |
| `GetAvailableAudioDevices()` | Get the cached list of devices (call `RefreshAudioDevices()` first) |
| `SetAudioDeviceByIndex(int32)` | Switch to a specific device by its index |
| `UseDefaultAudioDevice()` | Switch back to the OS default input device |
| `GetSelectedDeviceIndex()` | Get the index of the currently selected device (`INDEX_NONE` for default) |
| `GetSelectedDeviceInfo()` | Get full device info for the current selection |

---

## Enumerating Devices

Refresh the device list, then iterate to present options to the user:

=== "C++"

    ```cpp
    UAbolethSTTSubsystem* STT = UAbolethSTTSubsystem::GetSTTSubsystem(this);

    // Refresh the device list from the OS
    STT->RefreshAudioDevices();

    // Get all available input devices
    TArray<FAudioDeviceInfo> Devices = STT->GetAvailableAudioDevices();

    for (const FAudioDeviceInfo& Device : Devices)
    {
        UE_LOG(LogTemp, Log, TEXT("[%d] %s (Channels: %d, Rate: %d, AEC: %s)"),
            Device.DeviceIndex,
            *Device.DeviceName,
            Device.InputChannels,
            Device.PreferredSampleRate,
            Device.bSupportsHardwareAEC ? TEXT("Yes") : TEXT("No"));
    }
    ```

=== "Blueprint"

    Call **Refresh Audio Devices** on the Subsystem, then **Get Available Audio Devices** to receive an array of `FAudioDeviceInfo` structs. Each struct contains the device name, index, channel count, sample rate, and whether hardware AEC is supported.

---

## Switching Devices

=== "C++"

    ```cpp
    UAbolethSTTSubsystem* STT = UAbolethSTTSubsystem::GetSTTSubsystem(this);

    // Switch to device at index 2
    bool bSuccess = STT->SetAudioDeviceByIndex(2);

    // Switch back to OS default
    STT->UseDefaultAudioDevice();

    // Check what's currently selected
    int32 CurrentIndex = STT->GetSelectedDeviceIndex();          // INDEX_NONE = default
    FAudioDeviceInfo CurrentDevice = STT->GetSelectedDeviceInfo();
    ```

=== "Blueprint"

    Call **Set Audio Device By Index** with the desired device index. Call **Use Default Audio Device** to revert to the OS default. Use **Get Selected Device Index** and **Get Selected Device Info** to query the current selection.

!!! info "Changes take effect immediately"
    When you switch devices, the microphone capture component restarts automatically with the new device. There is no need to stop and restart listening manually. The pipeline returns to Idle and resumes listening on the new device within a few frames.

---

## Device Info Struct

Each device is described by an `FAudioDeviceInfo` struct:

| Field | Type | Description |
|-------|------|-------------|
| `DeviceName` | `FString` | Human-readable device name |
| `DeviceId` | `FString` | System-level device identifier |
| `DeviceIndex` | `int32` | Index for use with `SetAudioDeviceByIndex()` |
| `InputChannels` | `int32` | Number of input channels |
| `PreferredSampleRate` | `int32` | Device's preferred sample rate |
| `bSupportsHardwareAEC` | `bool` | Whether the device has hardware acoustic echo cancellation |

---

## Events

Two delegates fire in response to device changes:

### OnAudioDeviceChanged

Fires when the active audio device changes (after calling `SetAudioDeviceByIndex()` or `UseDefaultAudioDevice()`).

```
OnAudioDeviceChanged(FAudioDeviceInfo NewDevice)
```

### OnAudioDevicesRefreshed

Fires when the device list is refreshed (after calling `RefreshAudioDevices()`). Use this to update a device selection UI.

```
OnAudioDevicesRefreshed()
```

=== "C++"

    ```cpp
    // Bind device change events
    STT->OnAudioDeviceChanged.AddDynamic(this, &AMyActor::HandleDeviceChanged);
    STT->OnAudioDevicesRefreshed.AddDynamic(this, &AMyActor::HandleDevicesRefreshed);

    void AMyActor::HandleDeviceChanged(const FAudioDeviceInfo& NewDevice)
    {
        UE_LOG(LogTemp, Log, TEXT("Switched to: %s"), *NewDevice.DeviceName);
    }

    void AMyActor::HandleDevicesRefreshed()
    {
        // Rebuild the device dropdown in your settings UI
        RebuildDeviceList(STT->GetAvailableAudioDevices());
    }
    ```

=== "Blueprint"

    Bind **On Audio Device Changed** and **On Audio Devices Refreshed** on the Subsystem or Listener Component to respond to device changes in your UI.
