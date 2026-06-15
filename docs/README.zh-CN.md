# JieLi Recorder Android SDK 接入文档

本文档介绍如何在 Android 应用中集成并使用 `com.jieli.sdk.ble.recorder` BLE 录音 SDK。

当前 SDK 支持：

- BLE 扫描与停止扫描
- BLE 外设连接与断开连接
- 扫描结果回调
- 电量变化回调
- 文件列表读取与刷新
- 文件下载、下载进度与下载统计
- 文件删除
- 设备时间同步
- 自定义录音开始与停止
- 录音状态、存储空间、录音文件数量查询
- 按键/触控行为设置与软触发
- 实时 PCM 音频流回调
- OTA 升级
- 错误与升级未完成提醒

## 1. Maven 依赖

SDK 依赖已经迁移到 Maven。

在应用模块中添加：

```kotlin
dependencies {
    implementation("com.caitun.ble:jieli-ble-recorder:1.0.0")
}
```

如果你的项目使用私有 Maven 仓库，请先在 Gradle 中配置对应仓库，再添加上面的依赖。

## 2. 包名与导入

SDK 使用的包名为：

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

## 3. 权限要求

至少需要申请以下权限：

- `android.permission.ACCESS_FINE_LOCATION`
- Android 12 及以上：`android.permission.BLUETOOTH_SCAN`
- Android 12 及以上：`android.permission.BLUETOOTH_CONNECT`

示例：

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

## 4. BLEManager 的使用方式

SDK 的核心入口是 `BLEManager`。

推荐做法：

- 整个相关业务页面共用一个 `BLEManager`
- 页面进入时注册回调
- 页面销毁时移除回调
- 扫描页和详情页不要各自创建独立实例

示例：

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

## 5. 实现 BLECallback

应用层需要实现 `BLECallback`。

当前回调接口包括：

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

Kotlin 空实现骨架：

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

## 6. 扫描 BLE 外设

开始扫描：

```kotlin
bleManager?.startScan()
```

停止扫描：

```kotlin
bleManager?.stopScan()
```

接收扫描结果：

```kotlin
private val devices = LinkedHashMap<String, BLEDevice>()

override fun onDiscovery(device: BLEDevice) {
    runOnUiThread {
        val mac = device.mac ?: return@runOnUiThread
        devices[mac] = device
        // 刷新列表
    }
}
```

停止扫描回调：

```kotlin
override fun onScanStopped() {
    runOnUiThread {
        // 恢复按钮状态
    }
}
```

推荐：

- 每次扫描前清空旧列表
- 通过 MAC 去重
- 在 `onScanStopped()` 中恢复“开始扫描”按钮

## 7. 扫描列表中展示的信息

`BLEDevice` 可直接使用的字段包括：

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

典型展示项：

- 设备名称
- MAC 地址
- 设备 ID
- 固件版本
- 电量百分比
- 是否充电
- 是否已充满
- 连接状态

## 8. 电量变化

当电量变化时会触发：

```kotlin
override fun onBatteryChange(device: BLEDevice) {
    runOnUiThread {
        val mac = device.mac ?: return@runOnUiThread
        device.isChargedFull = device.charging && device.batteryPower >= 100
        devices[mac] = device
        // 刷新显示
    }
}
```

## 9. 连接与断开连接

连接设备：

```kotlin
bleManager?.connect(device)
```

断开设备：

```kotlin
bleManager?.disconnect(device)
```

监听连接状态：

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

## 10. 文件列表读取

从头刷新文件列表：

```kotlin
bleManager?.retrieveFilesFromStart()
```

继续读取更多文件：

```kotlin
bleManager?.retrieveFiles()
```

限制最多读取数量：

```kotlin
bleManager?.setMaxRetrieveFilesCount(200)
```

接收文件列表：

```kotlin
override fun onFilesRetrieved(files: List<BLEFile>) {
    runOnUiThread {
        files.forEach { file ->
            val name = file.name ?: return@forEach
            // 使用文件名作为 key 保存
        }
    }
}
```

## 11. 文件下载

按文件名下载：

```kotlin
bleManager?.downloadFile(device, filename)
```

下载到指定路径：

```kotlin
bleManager?.downloadFileToPath(
    device = device,
    filename = filename,
    path = File(cacheDir, filename).absolutePath
)
```

下载回调事件：

- `BEGIN`
- `PROGRESS`
- `FINISH`
- `CANCEL`
- `ERROR`

示例：

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

下载完成后可获取：

- `packages`
- `bytesCount`
- `duration`
- `path`

## 12. 文件分享

下载完成后可使用 `BLEFileDownloadEvent.path` 配合 `FileProvider` 分享文件：

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
    startActivity(Intent.createChooser(shareIntent, "分享文件"))
}
```

## 13. 文件删除

删除一个或多个文件：

```kotlin
bleManager?.deleteFile(device, listOf("A001.ogg"))
bleManager?.deleteFile(device, listOf("A001.ogg", "A002.ogg"))
```

回调处理：

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

## 14. 同步时间

```kotlin
bleManager?.syncTime(device)
```

结果：

```kotlin
override fun onTimeSynced(device: BLEDevice, success: Boolean) {
    // 更新 UI
}
```

## 15. 自定义录音控制

开启自定义录音：

```kotlin
bleManager?.startCustomRecord(device)
```

停止自定义录音：

```kotlin
bleManager?.stopCustomRecord(device)
```

结果回调：

```kotlin
override fun onRecordStart(device: BLEDevice, started: Boolean) = Unit
override fun onRecordStop(device: BLEDevice, stopped: Boolean) = Unit
```

## 16. 查询设备状态

查询录音状态：

```kotlin
bleManager?.queryRecordState(device)
```

查询存储空间：

```kotlin
bleManager?.queryStorageSize(device)
```

查询录音文件数量：

```kotlin
bleManager?.queryRecordFiles(device)
```

结果回调：

```kotlin
override fun onRecordStateUpdate(device: BLEDevice, state: BLERecordState) = Unit
override fun onStorageSizeUpdate(device: BLEDevice, available: Int, total: Int) = Unit
override fun onRecordFilesCountUpdate(device: BLEDevice, fileCount: Int) = Unit
```

## 17. Opus 帧长设置

```kotlin
bleManager?.setOpusFrameSize(40)
val frameSize = bleManager?.getOpusFrameSize()
```

常用值：

- `40`
- `80`

## 18. 按键/触控行为

设置按键行为：

```kotlin
val behavior = BLEKeyTouchBehavior(
    key = "key0",
    event = BLEKeyTouchBehavior.CLICK,
    behavior = BLEKeyTouchBehavior.START_RECORD
)
bleManager?.setKeyTouchBehaviorEvent(device, behavior)
```

软触发按键事件：

```kotlin
bleManager?.emitKeyTouchBehaviorEvent(device, behavior)
```

回调：

```kotlin
override fun onKeyTouchBehaviorUpdate(
    device: BLEDevice,
    isSuccess: Boolean,
    errorMessage: String
) = Unit

override fun onKeyTouchEmitted(device: BLEDevice, isSuccess: Boolean) = Unit
```

## 19. 实时 PCM 音频流

SDK 现在会先把 Opus 解码成原始 PCM，再把 PCM 数据回调给应用层。

回调时序：

- `onRealtimeAudioStarted()`
- 多次 `onRealtimeAudioReceived(byte[] audio)`
- `onRealtimeAudioStopped()`

接口定义：

```java
void onRealtimeAudioStarted();
void onRealtimeAudioReceived(byte[] audio);
void onRealtimeAudioStopped();
```

推荐做法：

- 开始时显示状态
- 接收时把 PCM 数据追加写入 `.pcm` 文件
- 结束时关闭输出流
- 如果文件非空则自动分享
- 如果需要更强兼容性，可在应用层把 PCM 转成 WAV 再分享

示例：

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
        // 使用 FileProvider 分享
    }
}
```

## 20. OTA 升级

启动 OTA：

```kotlin
bleManager?.startOTA(device, firmwareBytes)
```

典型流程：

1. 选择固件文件
2. 读取完整字节数组
3. 调用 `startOTA(device, fileBytes)`
4. 通过 `onOTAUpdate` 更新 UI

示例：

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

## 21. 错误处理

通用错误回调：

```kotlin
override fun onError(device: BLEDevice, errorCode: BLEErrorCode) {
    val code = errorCode.code
    val message = errorCode.message
}
```

升级未完成提醒：

```kotlin
override fun onUpgradeUnfinished(device: BLEDevice) {
    // 提示用户继续升级
}
```

## 22. 接入建议

- 扫描页和详情页共用一个 `BLEManager`
- 页面销毁时及时移除回调
- 多设备场景下按 MAC 过滤设备相关回调
- 文件完整刷新优先使用 `retrieveFilesFromStart()`
- 下载完成后优先使用 `BLEFileDownloadEvent.path`
- 实时 PCM 音频请保持单独的输出流，并在 `onRealtimeAudioStopped()` 后统一关闭
- 分享下载文件或实时音频文件时统一使用 `FileProvider`

## 23. 英文文档

英文版说明请查看仓库根目录的 [README.md](../README.md)。
