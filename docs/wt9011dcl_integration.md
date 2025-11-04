# WT9011DCL BLE統合実装ガイド

## 1. 概要と目的

本ドキュメントは、WT9011DCL IMUセンサーをLionアプリに統合するための実装ガイドである。既存のアーキテクチャ設計（`docs/architecture.md`）、要件定義（`docs/requirements_specification.md`）、画面設計（`docs/screens_navigation.md`）、データモデル（`docs/er_model.md`）に完全に準拠し、実装者が直接コーディングできる技術詳細を提供する。

**対象読者**: Kotlin/Android開発者、BLE実装経験者

**実装スコープ**: v1.0（フォアグラウンド計測のみ、加速度・角速度のみ）

---

## 2. 前提と制約

### 2.1 アーキテクチャ前提

本実装は以下の設計原則に従う（`docs/architecture.md` 参照）：

- **MVVM + Ports/Adapters パターン**
- **単方向データフロー**: BLE Adapter → Repository Port → UseCase → ViewModel → UI
- **Hilt による依存性注入**
- **Coroutines/Flow によるリアクティブストリーム**
- **純粋ドメインロジック**（BLE依存を Port 背後に隔離）

### 2.2 デバイス仕様

- **センサー**: WT9011DCL（WitMotion）
- **通信**: Bluetooth 5.0 (BLE)
- **MTU**: 最大20バイト/Notification
- **デフォルト出力**: 加速度・角速度・角度（Flag=0x61）
- **出力周波数**: 0.2–200 Hz（デフォルト10 Hz、コマンドで変更可能）
- **データフォーマット**: `0x55` ヘッダ + Flag + ペイロード（11–20バイト）

### 2.3 v1 スコープ

**含む**:
- 加速度（3軸）、角速度（3軸）の取得
- BLE スキャン・接続・Notify購読
- 設定コマンド送信（レート変更、出力モード変更）
- リアルタイムストリーム配信（`Flow<SensorSample>`）
- 接続状態管理、エラーハンドリング、再接続

**含まない（v2以降）**:
- Euler角、磁気、クォータニオン、温度（プロトコル対応は可能だが、v1要件外）
- バックグラウンド計測（ForegroundService）
- キャリブレーション自動調整UI

### 2.4 Android要件

- **minSdk**: API 33（Android 13+）
- **targetSdk**: API 36
- **権限**: `BLUETOOTH_SCAN`, `BLUETOOTH_CONNECT`, `ACCESS_FINE_LOCATION`（`docs/requirements_specification.md` 6章参照）

---

## 3. BLE アダプタ設計（Port実装）

### 3.1 Port定義（`docs/architecture.md` より）

```kotlin
// domain/port/SensorStreamPort.kt
interface SensorStreamPort {
    val frames: Flow<SensorSample>
    suspend fun startStreaming()
    suspend fun stopStreaming()
}

// domain/port/ConnectionStatePort.kt
interface ConnectionStatePort {
    val isScanning: StateFlow<Boolean>
    val isConnected: StateFlow<Boolean>
    val scannedDevices: StateFlow<List<DeviceItem>>
    suspend fun startScan()
    suspend fun stopScan()
    suspend fun connect(address: String)
    suspend fun disconnect()
}
```

### 3.2 Adapter実装方針

`Wt9011BleAdapter` が両Portを実装し、BLE固有のロジックをカプセル化する：

```kotlin
// data/ble/Wt9011BleAdapter.kt
class Wt9011BleAdapter @Inject constructor(
    @ApplicationContext private val context: Context,
    private val dispatcherProvider: DispatcherProvider
) : SensorStreamPort, ConnectionStatePort {

    private val bluetoothManager = context.getSystemService<BluetoothManager>()!!
    private val bluetoothAdapter = bluetoothManager.adapter
    private val bleScanner = bluetoothAdapter.bluetoothLeScanner

    private var gatt: BluetoothGatt? = null
    private val frameParser = FrameParser()

    // ConnectionStatePort 実装
    private val _isScanning = MutableStateFlow(false)
    override val isScanning: StateFlow<Boolean> = _isScanning.asStateFlow()

    private val _isConnected = MutableStateFlow(false)
    override val isConnected: StateFlow<Boolean> = _isConnected.asStateFlow()

    private val _scannedDevices = MutableStateFlow<List<DeviceItem>>(emptyList())
    override val scannedDevices: StateFlow<List<DeviceItem>> = _scannedDevices.asStateFlow()

    // SensorStreamPort 実装
    private val _frames = MutableSharedFlow<SensorSample>(
        extraBufferCapacity = 1,
        onBufferOverflow = BufferOverflow.DROP_OLDEST
    )
    override val frames: Flow<SensorSample> = _frames.asSharedFlow()

    // 実装詳細は後述
    override suspend fun startScan() { /* ... */ }
    override suspend fun stopScan() { /* ... */ }
    override suspend fun connect(address: String) { /* ... */ }
    override suspend fun disconnect() { /* ... */ }
    override suspend fun startStreaming() { /* ... */ }
    override suspend fun stopStreaming() { /* ... */ }
}
```

**設計ポイント**:
- `_frames` は `DROP_OLDEST` で高頻度データのバックプレッシャ対応
- `scannedDevices` は UI でリアルタイム更新（タイムアウト管理含む）
- GATT コールバックは内部で Coroutine に変換

---

## 4. パッケージ構成（`data/ble/` 配下）

```
app/src/main/java/com/scrap/lion/
├── data/
│   └── ble/
│       ├── Wt9011BleAdapter.kt       // Port実装（メインクラス）
│       ├── BleScanner.kt             // スキャン処理の分離クラス
│       ├── BleGattClient.kt          // GATT接続・購読処理
│       ├── FrameParser.kt            // 0x55フレームパース・単位変換
│       ├── Wt9011Commands.kt         // FF AAコマンド生成
│       ├── BleError.kt               // エラー型定義
│       └── RingBuffer.kt             // 境界ずれ処理用バッファ
├── domain/
│   ├── entity/
│   │   ├── Vec3.kt                   // 3Dベクトル
│   │   └── SensorSample.kt           // センサーサンプル
│   └── port/
│       ├── SensorStreamPort.kt
│       └── ConnectionStatePort.kt
└── di/
    └── BleModule.kt                  // Hilt設定
```

**責務分離**:
- `Wt9011BleAdapter`: Port実装・オーケストレーション
- `BleScanner`: スキャンロジック・デバイスフィルタ
- `BleGattClient`: GATT操作・Notifyコールバック
- `FrameParser`: プロトコル解析・単位変換（**重要**）
- `Wt9011Commands`: コマンド生成（unlock/save/setRate等）
- `RingBuffer`: 20B境界のずれ対応

---

## 5. データフロー（BLE → FrameParser → SensorSample → UseCase → UI）

```
[WT9011DCL]
    ↓ BLE Notify (20 bytes, 200 Hz)
[BleGattClient]
    ↓ onCharacteristicChanged
[RingBuffer]
    ↓ 0x55 ヘッダ検出・フレーム切り出し
[FrameParser]
    ↓ 単位変換（g→m/s², deg/s→rad/s, ms→μs）
[SensorSample (timestampMicros, accel: Vec3, gyroRadPerSec: Vec3)]
    ↓ Flow<SensorSample>
[Wt9011BleAdapter._frames.emit]
    ↓
[SensorStreamPort.frames]
    ↓
[UseCase: ObserveSwingEvents]
    ↓ SwingDetectorPort.process
[SwingEvent 保存 / MeasureViewModel 更新]
    ↓
[MeasureScreen UI]
```

**キーポイント**:
1. **境界ずれ**: 20Bチャンクが中途半端に届く → `RingBuffer` で `0x55` 検出まで蓄積
2. **単位変換**: `FrameParser` で domain 仕様（m/s², rad/s, μs）に正規化
3. **バックプレッシャ**: `DROP_OLDEST` で最新値優先（検出ロジックは最新が重要）

---

## 6. プロトコル実装（受信/送信コマンド）

### 6.1 受信フレーム（Flag=0x61: 加速度+角速度+角度）

**フォーマット**:
```
0x55 | 0x61 | axL axH ayL ayH azL azH | wxL wxH wyL wyH wzL wzH | rollL rollH pitchL pitchH yawL yawH
```
- 合計20バイト
- リトルエンディアン（LoHi）

**変換式**:
- 加速度[g] = `int16_raw / 32768.0 * 16.0`
- 角速度[°/s] = `int16_raw / 32768.0 * 2000.0`
- 角度[°] = `int16_raw / 32768.0 * 180.0`（v1では未使用）

### 6.2 送信コマンド（FF AA 系）

| 目的 | コマンド (Hex) | 備考 |
|---|---|---|
| Unlock（書込前必須） | `FF AA 69 88 B5` | 設定変更前に実行 |
| Save（設定保存） | `FF AA 00 00 00` | 変更後に実行 |
| Return rate | `FF AA 03 <RATE> 00` | `0x06=10Hz`, `0x07=20Hz`, `0x08=50Hz`, `0x09=100Hz`, `0x0B=200Hz` |
| Output content | `FF AA 96 <MODE> 00` | `00:Acc+Gyro+Angle` |

**初回セットアップシーケンス**（200 Hz設定例）:
```kotlin
suspend fun initializeSensor() {
    writeCommand(Wt9011Commands.unlock())
    delay(50)
    writeCommand(Wt9011Commands.setRate(200)) // 0x0B
    delay(50)
    writeCommand(Wt9011Commands.save())
    delay(50)
}
```

### 6.3 BLE サービス/キャラクタリスティック探索

WT9011DCL のファームウェアバージョンによりUUIDが異なる可能性があるため、**探索ベース**で実装：

1. 接続後、全 GATT サービスを列挙
2. 各サービスの各キャラクタリスティックを調査
3. `Notify` プロパティを持つ Char を検出 → 購読
4. `Write` または `WriteWithoutResponse` を持つ Char を検出 → コマンド送信用

**探索ロジック例**:
```kotlin
private fun findNotifyCharacteristic(gatt: BluetoothGatt): BluetoothGattCharacteristic? {
    for (service in gatt.services) {
        for (char in service.characteristics) {
            val properties = char.properties
            if ((properties and BluetoothGattCharacteristic.PROPERTY_NOTIFY) != 0) {
                return char
            }
        }
    }
    return null
}

private fun findWriteCharacteristic(gatt: BluetoothGatt): BluetoothGattCharacteristic? {
    for (service in gatt.services) {
        for (char in service.characteristics) {
            val properties = char.properties
            if ((properties and BluetoothGattCharacteristic.PROPERTY_WRITE) != 0 ||
                (properties and BluetoothGattCharacteristic.PROPERTY_WRITE_NO_RESPONSE) != 0) {
                return char
            }
        }
    }
    return null
}
```

**フィルタ**: デバイス名に `"WT"` を含むものを優先スキャン（`ScanFilter` 使用）

---

## 7. 単位変換とデータ正規化

### 7.1 ドメインモデル（`docs/architecture.md` より）

```kotlin
// domain/entity/Vec3.kt
data class Vec3(val x: Float, val y: Float, val z: Float)

// domain/entity/SensorSample.kt
data class SensorSample(
    val timestampMicros: Long,    // μs（エポック）
    val accel: Vec3,              // m/s²（SI単位）
    val gyroRadPerSec: Vec3       // rad/s（SI単位）
)
```

### 7.2 FrameParser実装（単位変換層）

```kotlin
// data/ble/FrameParser.kt
class FrameParser {
    private val DEG2RAD = (Math.PI / 180.0).toFloat()
    private val G_TO_MPS2 = 9.80665f

    fun parse(payload: ByteArray): SensorSample? {
        if (payload.size < 20 || payload[0] != 0x55.toByte()) return null
        val flag = payload[1]
        return when (flag) {
            0x61.toByte() -> parseAccelGyroAngle(payload)
            else -> null
        }
    }

    private fun parseAccelGyroAngle(b: ByteArray): SensorSample {
        // 加速度（2–7バイト目）
        val axRaw = readInt16(b, 2)
        val ayRaw = readInt16(b, 4)
        val azRaw = readInt16(b, 6)
        val axG = axRaw / 32768f * 16f  // [g]
        val ayG = ayRaw / 32768f * 16f
        val azG = azRaw / 32768f * 16f
        // ドメイン単位へ変換: g → m/s²
        val accel = Vec3(
            x = axG * G_TO_MPS2,
            y = ayG * G_TO_MPS2,
            z = azG * G_TO_MPS2
        )

        // 角速度（8–13バイト目）
        val wxRaw = readInt16(b, 8)
        val wyRaw = readInt16(b, 10)
        val wzRaw = readInt16(b, 12)
        val wxDeg = wxRaw / 32768f * 2000f  // [°/s]
        val wyDeg = wyRaw / 32768f * 2000f
        val wzDeg = wzRaw / 32768f * 2000f
        // ドメイン単位へ変換: deg/s → rad/s
        val gyro = Vec3(
            x = wxDeg * DEG2RAD,
            y = wyDeg * DEG2RAD,
            z = wzDeg * DEG2RAD
        )

        // タイムスタンプ: 現在時刻をμsで取得
        val timestampMicros = System.currentTimeMillis() * 1000L

        return SensorSample(
            timestampMicros = timestampMicros,
            accel = accel,
            gyroRadPerSec = gyro
        )
    }

    private fun readInt16(b: ByteArray, offset: Int): Short {
        val lo = b[offset].toInt() and 0xFF
        val hi = b[offset + 1].toInt() and 0xFF
        return ((hi shl 8) or lo).toShort()
    }
}
```

**重要**:
- **加速度**: g → m/s²（`* 9.80665`）
- **角速度**: deg/s → rad/s（`* π/180`）
- **タイムスタンプ**: ms → μs（`* 1000`）
- 角度（14–19バイト目）は v1 では未使用（将来拡張用）

### 7.3 チェックサム検証（オプション）

プロトコル仕様に従い、フレーム末尾のチェックサム（下位8bit合計）を検証する場合：

```kotlin
private fun verifyChecksum(b: ByteArray): Boolean {
    if (b.size < 20) return false
    var sum = 0
    for (i in 0..18) {
        sum += (b[i].toInt() and 0xFF)
    }
    val expectedChecksum = (sum and 0xFF).toByte()
    return b[19] == expectedChecksum
}
```

---

## 8. エラーハンドリングと再接続

### 8.1 エラー型定義

```kotlin
// data/ble/BleError.kt
sealed class BleError : Exception() {
    object BluetoothDisabled : BleError()
    object PermissionDenied : BleError()
    data class ScanFailed(val errorCode: Int) : BleError()
    data class ConnectionFailed(val reason: String) : BleError()
    object ServiceNotFound : BleError()
    object CharacteristicNotFound : BleError()
    data class GattError(val status: Int) : BleError()
    object Timeout : BleError()
}
```

### 8.2 再接続ポリシー

```kotlin
class Wt9011BleAdapter @Inject constructor(/* ... */) {
    private val retryPolicy = RetryPolicy(
        maxAttempts = 3,
        initialDelayMillis = 1000,
        maxDelayMillis = 5000,
        factor = 2.0
    )

    suspend fun connectWithRetry(address: String): Result<Unit> = withContext(dispatcherProvider.io) {
        retryPolicy.retry {
            connectInternal(address)
        }
    }

    private suspend fun connectInternal(address: String) {
        // GATT接続処理
        // 失敗時は BleError をスロー
    }
}
```

**切断検知**:
```kotlin
private val gattCallback = object : BluetoothGattCallback() {
    override fun onConnectionStateChange(gatt: BluetoothGatt, status: Int, newState: Int) {
        when (newState) {
            BluetoothProfile.STATE_DISCONNECTED -> {
                _isConnected.value = false
                // 自動再接続試行（オプション）
                scope.launch {
                    delay(1000)
                    connectWithRetry(gatt.device.address)
                }
            }
            BluetoothProfile.STATE_CONNECTED -> {
                _isConnected.value = true
                gatt.discoverServices()
            }
        }
    }
}
```

### 8.3 タイムアウト管理

```kotlin
suspend fun <T> withTimeout(timeoutMillis: Long, block: suspend () -> T): T {
    return kotlinx.coroutines.withTimeout(timeoutMillis) {
        block()
    }
}

// 使用例
suspend fun connect(address: String) {
    try {
        withTimeout(10_000) { // 10秒タイムアウト
            connectInternal(address)
        }
    } catch (e: TimeoutCancellationException) {
        throw BleError.Timeout
    }
}
```

---

## 9. 既存UI統合（ConnectScreen/MeasureScreen）

### 9.1 ConnectScreen への統合

**場所**: `ui/feature/connect/ConnectScreen.kt`

**既存設計**（`docs/screens_navigation.md` 参照）:
- デバイスリスト表示（`DeviceItem`）
- スキャン開始/停止
- 接続ボタン
- 権限ガード

**統合ポイント**:
```kotlin
@Composable
fun ConnectScreen(
    viewModel: ConnectViewModel = hiltViewModel()
) {
    val uiState by viewModel.uiState.collectAsState()

    Column {
        // スキャン状態表示
        if (uiState.isScanning) {
            CircularProgressIndicator()
        }

        // デバイスリスト
        LazyColumn {
            items(uiState.devices) { device ->
                DeviceItem(
                    device = device,
                    onClick = { viewModel.onEvent(ConnectEvent.Connect(device.address)) }
                )
            }
        }

        // スキャンボタン
        Button(onClick = { viewModel.onEvent(ConnectEvent.StartScan) }) {
            Text("スキャン開始")
        }
    }
}
```

**ViewModel実装**:
```kotlin
@HiltViewModel
class ConnectViewModel @Inject constructor(
    private val connectionStatePort: ConnectionStatePort,
    private val permissionChecker: PermissionChecker
) : ViewModel() {

    val uiState: StateFlow<ConnectUiState> = combine(
        connectionStatePort.isScanning,
        connectionStatePort.scannedDevices,
        connectionStatePort.isConnected
    ) { isScanning, devices, isConnected ->
        ConnectUiState(
            isScanning = isScanning,
            devices = devices,
            isConnected = isConnected
        )
    }.stateIn(viewModelScope, SharingStarted.Lazily, ConnectUiState())

    fun onEvent(event: ConnectEvent) {
        when (event) {
            ConnectEvent.StartScan -> {
                viewModelScope.launch {
                    if (permissionChecker.hasBluetoothPermissions()) {
                        connectionStatePort.startScan()
                    } else {
                        // 権限要求
                    }
                }
            }
            is ConnectEvent.Connect -> {
                viewModelScope.launch {
                    connectionStatePort.connect(event.address)
                    // 接続成功後、MeasureScreenへ遷移
                }
            }
        }
    }
}
```

### 9.2 MeasureScreen への統合

**場所**: `ui/feature/measure/MeasureScreen.kt`

**既存設計**（`docs/screens_navigation.md` 参照）:
- リアルタイム指標表示（`A(t)`, `W⊥(t)`）
- 最新スイング速度
- 計測開始/停止ボタン

**統合ポイント**:
```kotlin
@HiltViewModel
class MeasureViewModel @Inject constructor(
    private val sensorStreamPort: SensorStreamPort,
    private val observeSwingEventsUseCase: ObserveSwingEventsUseCase
) : ViewModel() {

    private val _uiState = MutableStateFlow(MeasureUiState())
    val uiState: StateFlow<MeasureUiState> = _uiState.asStateFlow()

    fun onEvent(event: MeasureEvent) {
        when (event) {
            MeasureEvent.StartMeasuring -> {
                viewModelScope.launch {
                    sensorStreamPort.startStreaming()
                    observeSwingEventsUseCase(sessionId).collect { swingEvent ->
                        _uiState.update { it.copy(
                            latestSwing = swingEvent,
                            tipSpeed = swingEvent.tipSpeedMetersPerSec
                        )}
                    }
                }
            }
            MeasureEvent.StopMeasuring -> {
                viewModelScope.launch {
                    sensorStreamPort.stopStreaming()
                }
            }
        }
    }
}
```

**UI例**:
```kotlin
@Composable
fun MeasureScreen(viewModel: MeasureViewModel = hiltViewModel()) {
    val uiState by viewModel.uiState.collectAsState()

    Column {
        Text("最新スイング速度: ${uiState.tipSpeed?.let { "%.1f m/s".format(it) } ?: "---"}")
        
        Button(onClick = { viewModel.onEvent(MeasureEvent.StartMeasuring) }) {
            Text("計測開始")
        }
        
        Button(onClick = { viewModel.onEvent(MeasureEvent.StopMeasuring) }) {
            Text("計測停止")
        }
    }
}
```

**注意**: 独立した `Wt9011dclScreen` は作成せず、既存画面に統合する。

---

## 10. Hilt 依存性注入設定

### 10.1 BleModule

```kotlin
// di/BleModule.kt
@Module
@InstallIn(SingletonComponent::class)
object BleModule {

    @Provides
    @Singleton
    fun provideWt9011BleAdapter(
        @ApplicationContext context: Context,
        dispatcherProvider: DispatcherProvider
    ): Wt9011BleAdapter {
        return Wt9011BleAdapter(context, dispatcherProvider)
    }

    @Provides
    @Singleton
    fun provideSensorStreamPort(
        adapter: Wt9011BleAdapter
    ): SensorStreamPort = adapter

    @Provides
    @Singleton
    fun provideConnectionStatePort(
        adapter: Wt9011BleAdapter
    ): ConnectionStatePort = adapter
}
```

**ポイント**:
- `Wt9011BleAdapter` を Singleton で提供
- 同じインスタンスを `SensorStreamPort` と `ConnectionStatePort` として注入
- `DispatcherProvider` は既存の `CoreModule` で提供済み（`docs/architecture.md` 参照）

### 10.2 DispatcherProvider（既存）

```kotlin
// core/DispatcherProvider.kt
interface DispatcherProvider {
    val io: CoroutineDispatcher
    val main: CoroutineDispatcher
    val default: CoroutineDispatcher
}

@Module
@InstallIn(SingletonComponent::class)
object CoreModule {
    @Provides
    @Singleton
    fun provideDispatcherProvider(): DispatcherProvider = object : DispatcherProvider {
        override val io = Dispatchers.IO
        override val main = Dispatchers.Main
        override val default = Dispatchers.Default
    }
}
```

---

## 11. テストとデバッグ

### 11.1 Port モック化（単体テスト）

```kotlin
// test/domain/usecase/ObserveSwingEventsUseCaseTest.kt
class FakeSensorStreamPort : SensorStreamPort {
    private val _frames = MutableSharedFlow<SensorSample>()
    override val frames: Flow<SensorSample> = _frames.asSharedFlow()

    suspend fun emitSample(sample: SensorSample) {
        _frames.emit(sample)
    }

    override suspend fun startStreaming() { /* no-op */ }
    override suspend fun stopStreaming() { /* no-op */ }
}

@Test
fun `スイング検出が正しく動作する`() = runTest {
    val fakePort = FakeSensorStreamPort()
    val useCase = ObserveSwingEventsUseCase(fakePort, /* ... */)

    // テストデータ投入
    fakePort.emitSample(SensorSample(
        timestampMicros = 1000000,
        accel = Vec3(0f, 0f, 9.8f),
        gyroRadPerSec = Vec3(0f, 0f, 0f)
    ))

    // 検証
    // ...
}
```

### 11.2 FrameParser 単体テスト

```kotlin
// test/data/ble/FrameParserTest.kt
class FrameParserTest {
    private val parser = FrameParser()

    @Test
    fun `0x61フレームを正しくパースする`() {
        // WT9011DCL の実データ例（16進数）
        val frame = byteArrayOf(
            0x55.toByte(), 0x61.toByte(),  // ヘッダ + Flag
            0x00, 0x00, 0x00, 0x00, 0xE8.toByte(), 0x03,  // 加速度 (例: 0, 0, 1g)
            0x00, 0x00, 0x00, 0x00, 0x00, 0x00,  // 角速度 (例: 0, 0, 0)
            0x00, 0x00, 0x00, 0x00, 0x00, 0x00   // 角度（v1未使用）
        )

        val result = parser.parse(frame)

        assertNotNull(result)
        assertEquals(9.8f, result!!.accel.z, 0.1f)  // 1g ≈ 9.8 m/s²
        assertEquals(0f, result.gyroRadPerSec.x, 0.01f)
    }

    @Test
    fun `不正なヘッダはnullを返す`() {
        val invalidFrame = byteArrayOf(0xAA.toByte(), 0x61.toByte())
        assertNull(parser.parse(invalidFrame))
    }
}
```

### 11.3 統合テスト（Instrumented Test）

```kotlin
// androidTest/data/ble/Wt9011BleAdapterTest.kt
@RunWith(AndroidJUnit4::class)
class Wt9011BleAdapterTest {
    @Test
    fun デバイススキャンが動作する() = runTest {
        val adapter = Wt9011BleAdapter(
            ApplicationProvider.getApplicationContext(),
            TestDispatcherProvider()
        )

        adapter.startScan()
        delay(5000) // 5秒スキャン
        adapter.stopScan()

        val devices = adapter.scannedDevices.value
        assertTrue(devices.isNotEmpty())
    }
}
```

### 11.4 デバッグログ

```kotlin
// data/ble/Wt9011BleAdapter.kt
private fun logFrame(payload: ByteArray) {
    if (BuildConfig.DEBUG) {
        val hex = payload.joinToString(" ") { "%02X".format(it) }
        Log.d("WT9011DCL", "Received: $hex")
    }
}
```

**推奨ログポイント**:
- GATT 接続/切断イベント
- 受信フレーム（最初の数バイトのみ）
- パースエラー（チェックサム失敗、不明Flag）
- コマンド送信（unlock/save/setRate）

---

## 12. 実装チェックリスト

### 12.1 基本実装

- [ ] `data/ble/Wt9011BleAdapter.kt` 作成（Port実装）
- [ ] `data/ble/FrameParser.kt` 作成（単位変換含む）
- [ ] `data/ble/Wt9011Commands.kt` 作成
- [ ] `data/ble/BleError.kt` 作成
- [ ] `data/ble/RingBuffer.kt` 作成（境界ずれ対応）
- [ ] `di/BleModule.kt` に Hilt 設定追加

### 12.2 権限・設定

- [ ] `AndroidManifest.xml` に BLE 権限追加（`docs/requirements_specification.md` 参照）
- [ ] 権限チェック実装（`ConnectViewModel`）
- [ ] 権限要求ダイアログ実装（`ConnectScreen`）

### 12.3 UI統合

- [ ] `ConnectViewModel` に `ConnectionStatePort` 注入
- [ ] `ConnectScreen` でデバイスリスト表示
- [ ] `MeasureViewModel` に `SensorStreamPort` 注入
- [ ] `MeasureScreen` でリアルタイムデータ表示

### 12.4 初期化

- [ ] 接続後の初期化シーケンス実装（unlock → setRate(200) → save）
- [ ] MTU 要求（`gatt.requestMtu(247)`）
- [ ] Notify 購読設定
- [ ] `CLIENT_CHARACTERISTIC_CONFIG` Descriptor 書き込み

### 12.5 エラーハンドリング

- [ ] 切断検知と自動再接続
- [ ] タイムアウト処理（スキャン・接続）
- [ ] エラー UI フィードバック（Toast/Snackbar）

### 12.6 テスト

- [ ] `FrameParserTest` 作成（単位変換検証）
- [ ] `FakeSensorStreamPort` 作成（UseCase テスト用）
- [ ] 統合テスト（実機での BLE 接続確認）

### 12.7 パフォーマンス

- [ ] 200 Hz でドロップ無し確認（`DROP_OLDEST` 動作確認）
- [ ] メモリリーク確認（LeakCanary 使用）
- [ ] バッテリー消費確認（計測中・アイドル時）

### 12.8 ドキュメント

- [ ] コード内コメント（特に単位変換ロジック）
- [ ] README（開発環境セットアップ手順）
- [ ] トラブルシューティング（よくある問題と解決策）

---

## 13. トラブルシューティング

### 13.1 デバイスが見つからない

**原因**:
- Bluetooth がオフ
- 位置情報がオフ（Android 要件）
- 権限が許可されていない
- デバイスが近くにない

**対策**:
- 権限チェック強化
- Bluetooth 有効化リクエスト（`BluetoothAdapter.ACTION_REQUEST_ENABLE`）
- スキャンフィルタ確認（デバイス名 "WT" の大文字小文字）

### 13.2 接続できるがデータが来ない

**原因**:
- Notify が購読されていない
- Descriptor 書き込み失敗
- センサーが低周波モード（デフォルト10 Hz）

**対策**:
```kotlin
val descriptor = notifyChar.getDescriptor(UUID.fromString("00002902-0000-1000-8000-00805f9b34fb"))
descriptor.value = BluetoothGattDescriptor.ENABLE_NOTIFICATION_VALUE
gatt.writeDescriptor(descriptor)
```

### 13.3 データが途切れる・ずれる

**原因**:
- 20B 境界でフレームが分割
- `RingBuffer` 未実装

**対策**:
- `0x55` ヘッダスキャンで再同期
- バッファサイズ最小40B（2フレーム分）

### 13.4 単位がおかしい

**原因**:
- 変換係数ミス
- g/m²、deg/rad の混同

**対策**:
- `FrameParserTest` で既知の値を検証
- 実機での重力確認（静止時 `accel.z ≈ 9.8 m/s²`）

---

## 14. 参考資料

- **WT9011DCL Datasheet**: センサー仕様・軸定義
- **WT9011DCL-BT50 Communication Protocol**: BLE プロトコル詳細
- `docs/architecture.md`: アーキテクチャ設計
- `docs/requirements_specification.md`: 要件定義（スイング検出ロジック含む）
- `docs/screens_navigation.md`: UI/画面設計
- `docs/er_model.md`: データベース設計

---

## 15. 将来拡張（v2以降）

以下の機能は v1 スコープ外だが、プロトコル対応は可能：

- **Euler 角取得**: Flag=0x61 の14–19バイト目を使用（姿勢可視化）
- **磁気取得**: `FF AA 27 3A 00` → Flag=0x71 レスポンス
- **クォータニオン**: `FF AA 27 51 00` → レジスタ読み出し
- **温度**: `FF AA 27 40 00` → 温度レジスタ
- **キャリブレーション UI**: 加速度/磁気キャリブレーションコマンド
- **バックグラウンド計測**: ForegroundService 対応

これらを実装する場合は、`SensorSample` モデルの拡張が必要（Optional フィールド追加）。
