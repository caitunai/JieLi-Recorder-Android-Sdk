# JieLi Recorder Android SDK

Android BLE recorder SDK for scanning devices, connecting to peripherals, browsing recorder files, downloading and deleting files, sending custom record commands, querying device state, configuring key/touch behavior, receiving realtime PCM audio, and performing OTA upgrades.

[中文文档](docs/README.zh-CN.md) | [iOS SDK](https://github.com/caitunai/JieLi-Recorder-iOS-Sdk)

## Maven Dependency

Add the SDK dependency in your Android app module:

```kotlin
dependencies {
    implementation("com.caitun.ble:jieli-ble-recorder:1.0.0")
}
```

If your project uses a private Maven repository, configure the repository first and then add the dependency above.

## Package Name

Import the SDK from:

```kotlin
import com.jieli.sdk.ble.recorder.BLECallback
import com.jieli.sdk.ble.recorder.BLEDevice
import com.jieli.sdk.ble.recorder.BLEErrorCode
import com.jieli.sdk.ble.recorder.BLEFile
import com.jieli.sdk.ble.recorder.BLEFileDeleteEvent
import com.jieli.sdk.ble.recorder.BLEFileDownloadEvent
import com.jieli.sdk.ble.recorder.BLEKeyTouchBehavior
import com.jieli.sdk.ble.recorder.BLEManager
import com.jieli.sdk.ble.recorder.BLEOTAEvent
import com.jieli.sdk.ble.recorder.BLERecordState
import com.jieli.sdk.ble.recorder.ConnectionCode
```

## Features

- BLE device scan and stop-scan callbacks
- Device connection and disconnection
- Device information reporting during scan
- Battery status update callbacks
- Recorder file listing and refresh
- File download with progress, statistics, and output path
- File deletion for one or multiple files
- Device time synchronization
- Custom record start and stop commands
- Record state, storage size, and record file count queries
- Key/touch behavior configuration and software trigger
- Realtime PCM audio callbacks
- OTA firmware upgrade
- Generic SDK error reporting

## Required Permissions

At minimum, request these permissions:

- `android.permission.ACCESS_FINE_LOCATION`
- `android.permission.BLUETOOTH_SCAN` on Android 12+
- `android.permission.BLUETOOTH_CONNECT` on Android 12+

Example:

```kotlin
private fun requiredPermissions(): Array<String> {
    val permissions = mutableListOf(
        Manifest.permission.ACCESS_FINE_LOCATION
    )
    if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.S) {
        permissions += Manifest.permission.BLUETOOTH_SCAN
        permissions += Manifest.permission.BLUETOOTH_CONNECT
    }
    return permissions.toTypedArray()
}
```

## Core Entry Point

The SDK entry point is `BLEManager`.

Recommended usage:

- create one shared `BLEManager` instance
- register screens with `addCallback`
- remove callbacks in `onDestroy`
- keep a single manager across scan and detail screens

Example singleton holder:

```kotlin
object BleManagerHolder {
    private var manager: BLEManager? = null

    fun get(context: Context, callback: BLECallback): BLEManager {
        val appContext = context.applicationContext
        val existing = manager
        if (existing != null) {
            existing.addCallback(callback)
            return existing
        }
        return BLEManager(appContext, callback).also {
            manager = it
        }
    }

    fun removeCallback(callback: BLECallback) {
        manager?.removeCallback(callback)
    }
}
```

## Implement BLECallback

Your application layer should implement `BLECallback`.

Current callback surface:

```java
void onDiscovery(BLEDevice device);
void onScanStopped();
void onConnectionChange(BLEDevice device, ConnectionCode status);
void onBatteryChange(BLEDevice device);
void onFilesRetrieved(List<BLEFile> files);
void onTimeSynced(BLEDevice device, boolean success);
void onRecordStart(BLEDevice device, boolean started);
void onRecordStop(BLEDevice device, boolean stopped);
void onFileDownloadUpdate(BLEDevice device, BLEFileDownloadEvent event);
void onFileDeleteUpdate(BLEDevice device, BLEFileDeleteEvent event);
void onOTAUpdate(BLEDevice device, BLEOTAEvent event);
void onRecordStateUpdate(BLEDevice device, BLERecordState state);
void onStorageSizeUpdate(BLEDevice device, int available, int total);
void onRecordFilesCountUpdate(BLEDevice device, int fileCount);
void onKeyTouchBehaviorUpdate(BLEDevice device, boolean isSuccess, String errorMessage);
void onKeyTouchEmitted(BLEDevice device, boolean isSuccess);
void onError(BLEDevice device, BLEErrorCode errorCode);
void onUpgradeUnfinished(BLEDevice device);
void onRealtimeAudioStarted();
void onRealtimeAudioReceived(byte[] audio);
void onRealtimeAudioStopped();
```

Minimal skeleton:

```kotlin
class MyBleActivity : AppCompatActivity(), BLECallback {
    private var bleManager: BLEManager? = null

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        bleManager = BleManagerHolder.get(applicationContext, this)
    }

    override fun onDestroy() {
        BleManagerHolder.removeCallback(this)
        super.onDestroy()
    }

    override fun onDiscovery(device: BLEDevice) = Unit
    override fun onScanStopped() = Unit
    override fun onConnectionChange(device: BLEDevice, status: ConnectionCode) = Unit
    override fun onBatteryChange(device: BLEDevice) = Unit
    override fun onFilesRetrieved(files: List<BLEFile>) = Unit
    override fun onTimeSynced(device: BLEDevice, success: Boolean) = Unit
    override fun onRecordStart(device: BLEDevice, started: Boolean) = Unit
    override fun onRecordStop(device: BLEDevice, stopped: Boolean) = Unit
    override fun onFileDownloadUpdate(device: BLEDevice, event: BLEFileDownloadEvent) = Unit
    override fun onFileDeleteUpdate(device: BLEDevice, event: BLEFileDeleteEvent) = Unit
    override fun onOTAUpdate(device: BLEDevice, event: BLEOTAEvent) = Unit
    override fun onRecordStateUpdate(device: BLEDevice, state: BLERecordState) = Unit
    override fun onStorageSizeUpdate(device: BLEDevice, available: Int, total: Int) = Unit
    override fun onRecordFilesCountUpdate(device: BLEDevice, fileCount: Int) = Unit
    override fun onKeyTouchBehaviorUpdate(device: BLEDevice, isSuccess: Boolean, errorMessage: String) = Unit
    override fun onKeyTouchEmitted(device: BLEDevice, isSuccess: Boolean) = Unit
    override fun onError(device: BLEDevice, errorCode: BLEErrorCode) = Unit
    override fun onUpgradeUnfinished(device: BLEDevice) = Unit
    override fun onRealtimeAudioStarted() = Unit
    override fun onRealtimeAudioReceived(audio: ByteArray) = Unit
    override fun onRealtimeAudioStopped() = Unit
}
```

## BLEDevice Fields

`BLEDevice` currently exposes:

```java
public String id;
public String mac;
public String name;
public boolean charging;
public int batteryPower;
public boolean isChargedFull;
public int version;
public String versionName;
```

Typical scan UI fields:

- device name
- MAC address
- device ID
- firmware version
- battery percentage
- charging status
- fully charged status
- connection status

## Scan Devices

Start scan:

```kotlin
bleManager?.startScan()
```

Stop scan:

```kotlin
bleManager?.stopScan()
```

Handle scan callbacks:

```kotlin
private val devices = LinkedHashMap<String, BLEDevice>()

override fun onDiscovery(device: BLEDevice) {
    runOnUiThread {
        val mac = device.mac ?: return@runOnUiThread
        devices[mac] = device
        // update list UI
    }
}

override fun onScanStopped() {
    runOnUiThread {
        // restore scan button state
    }
}
```

Recommended behavior:

- clear previous results before a new scan
- deduplicate by MAC address
- re-enable the start button after `onScanStopped()`

## Connect And Disconnect

Connect:

```kotlin
bleManager?.connect(device)
```

Disconnect:

```kotlin
bleManager?.disconnect(device)
```

Monitor state:

```kotlin
override fun onConnectionChange(device: BLEDevice, status: ConnectionCode) {
    runOnUiThread {
        when (status) {
            ConnectionCode.CONNECTING -> { }
            ConnectionCode.CONNECTED -> { }
            ConnectionCode.DISCONNECTED -> { }
            ConnectionCode.FAILED -> { }
            ConnectionCode.SYSTEM_ERROR -> { }
        }
    }
}
```

## File List

Refresh from the beginning:

```kotlin
bleManager?.retrieveFilesFromStart()
```

Load more:

```kotlin
bleManager?.retrieveFiles()
```

Limit total retrieval:

```kotlin
bleManager?.setMaxRetrieveFilesCount(200)
```

Receive files:

```kotlin
override fun onFilesRetrieved(files: List<BLEFile>) {
    runOnUiThread {
        files.forEach { file ->
            val name = file.name ?: return@forEach
            // store in map keyed by name
        }
    }
}
```

## Download Files

Download by filename:

```kotlin
bleManager?.downloadFile(device, filename)
```

Download to a custom path:

```kotlin
bleManager?.downloadFileToPath(
    device = device,
    filename = filename,
    path = File(cacheDir, filename).absolutePath
)
```

Download callback events:

- `BEGIN`
- `PROGRESS`
- `FINISH`
- `CANCEL`
- `ERROR`

Example:

```kotlin
override fun onFileDownloadUpdate(device: BLEDevice, event: BLEFileDownloadEvent) {
    when (event.event) {
        BLEFileDownloadEvent.Event.BEGIN -> { }
        BLEFileDownloadEvent.Event.PROGRESS -> { }
        BLEFileDownloadEvent.Event.FINISH -> { }
        BLEFileDownloadEvent.Event.CANCEL -> { }
        BLEFileDownloadEvent.Event.ERROR -> { }
        null -> Unit
    }
}
```

Available download statistics on finish:

- `packages`
- `bytesCount`
- `duration`
- `path`

## Share Downloaded Files

Use `BLEFileDownloadEvent.path` together with `FileProvider`.

Example:

```kotlin
private fun shareDownloadedFile(event: BLEFileDownloadEvent) {
    val path = event.path ?: return
    val file = File(path)
    if (!file.exists()) return

    val contentUri = FileProvider.getUriForFile(
        this,
        "${packageName}.fileprovider",
        file
    )
    val shareIntent = Intent(Intent.ACTION_SEND).apply {
        type = URLConnection.guessContentTypeFromName(file.name) ?: "*/*"
        putExtra(Intent.EXTRA_STREAM, contentUri)
        putExtra(Intent.EXTRA_SUBJECT, file.name)
        addFlags(Intent.FLAG_GRANT_READ_URI_PERMISSION)
    }
    startActivity(Intent.createChooser(shareIntent, "Share file"))
}
```

## Delete Files

Delete one or multiple files:

```kotlin
bleManager?.deleteFile(device, listOf("A001.ogg"))
bleManager?.deleteFile(device, listOf("A001.ogg", "A002.ogg"))
```

Handle deletion:

```kotlin
override fun onFileDeleteUpdate(device: BLEDevice, event: BLEFileDeleteEvent) {
    when (event.event) {
        BLEFileDeleteEvent.Event.SUCCESS -> { }
        BLEFileDeleteEvent.Event.ERROR -> { }
        BLEFileDeleteEvent.Event.FINISH -> { }
        null -> Unit
    }
}
```

## Time Sync

```kotlin
bleManager?.syncTime(device)
```

Result:

```kotlin
override fun onTimeSynced(device: BLEDevice, success: Boolean) {
    // update UI
}
```

## Custom Record Control

Start custom record:

```kotlin
bleManager?.startCustomRecord(device)
```

Stop custom record:

```kotlin
bleManager?.stopCustomRecord(device)
```

Callbacks:

```kotlin
override fun onRecordStart(device: BLEDevice, started: Boolean) = Unit
override fun onRecordStop(device: BLEDevice, stopped: Boolean) = Unit
```

## Query Device State

Query record state:

```kotlin
bleManager?.queryRecordState(device)
```

Query storage size:

```kotlin
bleManager?.queryStorageSize(device)
```

Query record file count:

```kotlin
bleManager?.queryRecordFiles(device)
```

Callbacks:

```kotlin
override fun onRecordStateUpdate(device: BLEDevice, state: BLERecordState) = Unit
override fun onStorageSizeUpdate(device: BLEDevice, available: Int, total: Int) = Unit
override fun onRecordFilesCountUpdate(device: BLEDevice, fileCount: Int) = Unit
```

## Opus Frame Size

Set and get the Opus frame size used by the SDK download/transcoding flow:

```kotlin
bleManager?.setOpusFrameSize(40)
val frameSize = bleManager?.getOpusFrameSize()
```

Common values:

- `40`
- `80`

## Key/Touch Behavior

Configure key/touch behavior:

```kotlin
val behavior = BLEKeyTouchBehavior(
    key = "key0",
    event = BLEKeyTouchBehavior.CLICK,
    behavior = BLEKeyTouchBehavior.START_RECORD
)
bleManager?.setKeyTouchBehaviorEvent(device, behavior)
```

Emit a software event:

```kotlin
bleManager?.emitKeyTouchBehaviorEvent(device, behavior)
```

Callbacks:

```kotlin
override fun onKeyTouchBehaviorUpdate(
    device: BLEDevice,
    isSuccess: Boolean,
    errorMessage: String
) = Unit

override fun onKeyTouchEmitted(device: BLEDevice, isSuccess: Boolean) = Unit
```

## Realtime PCM Audio

The SDK now decodes Opus audio to raw PCM before delivering it to the application.

Callback flow:

- `onRealtimeAudioStarted()`
- one or more `onRealtimeAudioReceived(byte[] audio)`
- `onRealtimeAudioStopped()`

Callback definitions:

```java
void onRealtimeAudioStarted();
void onRealtimeAudioReceived(byte[] audio);
void onRealtimeAudioStopped();
```

Recommended usage:

- show stream start and stop status in UI
- append each PCM chunk to a `.pcm` cache file
- close the output stream on stop
- share the final file if it is not empty
- convert PCM to WAV if a standard playable container is required

Example:

```kotlin
private val realtimeAudioLock = Any()
private var realtimeAudioFile: File? = null
private var realtimeAudioOutputStream: FileOutputStream? = null
private var realtimeAudioBytesReceived = 0L

override fun onRealtimeAudioStarted() {
    val audioFile = File(cacheDir, "realtime_audio_${System.currentTimeMillis()}.pcm")
    synchronized(realtimeAudioLock) {
        realtimeAudioOutputStream?.close()
        realtimeAudioFile = audioFile
        realtimeAudioBytesReceived = 0L
        realtimeAudioOutputStream = FileOutputStream(audioFile, false)
    }
}

override fun onRealtimeAudioReceived(audio: ByteArray) {
    if (audio.isEmpty()) return
    synchronized(realtimeAudioLock) {
        val output = realtimeAudioOutputStream ?: return
        output.write(audio)
        realtimeAudioBytesReceived += audio.size.toLong()
    }
}

override fun onRealtimeAudioStopped() {
    val fileToShare: File?
    val totalSize: Long
    synchronized(realtimeAudioLock) {
        fileToShare = realtimeAudioFile
        totalSize = realtimeAudioBytesReceived
        realtimeAudioOutputStream?.flush()
        realtimeAudioOutputStream?.close()
        realtimeAudioOutputStream = null
        realtimeAudioFile = null
        realtimeAudioBytesReceived = 0L
    }
    if (fileToShare != null && fileToShare.exists() && totalSize > 0L) {
        // share via FileProvider
    }
}
```

## OTA Upgrade

Start OTA:

```kotlin
bleManager?.startOTA(device, firmwareBytes)
```

Typical flow:

1. choose a firmware file
2. read all bytes
3. call `startOTA(device, fileBytes)`
4. update UI from `onOTAUpdate`

Example:

```kotlin
override fun onOTAUpdate(device: BLEDevice, event: BLEOTAEvent) {
    when (event.event) {
        BLEOTAEvent.Event.START -> { }
        BLEOTAEvent.Event.RECONNECT -> { }
        BLEOTAEvent.Event.PROGRESS -> { }
        BLEOTAEvent.Event.FINISH -> { }
        BLEOTAEvent.Event.CANCEL -> { }
        BLEOTAEvent.Event.ERROR -> { }
        null -> Unit
    }
}
```

## Errors

Generic SDK errors arrive through:

```kotlin
override fun onError(device: BLEDevice, errorCode: BLEErrorCode) {
    val code = errorCode.code
    val message = errorCode.message
}
```

Upgrade reminder:

```kotlin
override fun onUpgradeUnfinished(device: BLEDevice) {
    // prompt user to continue upgrade
}
```

## Integration Notes

- Keep one shared `BLEManager` across related screens
- Remove callbacks when screens are destroyed
- Filter device-specific callbacks by MAC when needed
- Prefer `retrieveFilesFromStart()` for a full refresh
- Use `BLEFileDownloadEvent.path` after download completion
- Keep a dedicated output stream for realtime PCM audio until `onRealtimeAudioStopped()`
- Use `FileProvider` for sharing downloaded or realtime-generated files

## Chinese Documentation

For a Chinese integration guide, see [docs/README.zh-CN.md](docs/README.zh-CN.md).
