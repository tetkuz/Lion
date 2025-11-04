# コーディング規約（Coding Guidelines）

本ドキュメントは、Lion アプリの開発における Kotlin コーディング規約を定義する。一貫性のあるコードベースを維持し、保守性・可読性を高めることを目的とする。

**対象読者**: 本プロジェクトの開発者全員

**参照ドキュメント**:
- `docs/architecture.md` - アーキテクチャ設計
- `docs/requirements_specification.md` - 要件定義
- `docs/testing_strategy.md` - テスト戦略
- [Kotlin Coding Conventions](https://kotlinlang.org/docs/coding-conventions.html)
- [Android Kotlin Style Guide](https://developer.android.com/kotlin/style-guide)

---

## 1. 基本原則

### 1.1 言語とフォーマット

- **言語**: Kotlin 2.0.21、Java 11 互換
- **フォーマッタ**: Android Studio デフォルト（`ktlint` 互換）
- **自動フォーマット**: コミット前に実行（`Ctrl+Alt+L` / `Cmd+Option+L`）
- **行の長さ**: 最大 120 文字（推奨 100 文字）
- **インデント**: スペース 4 つ（タブ禁止）

### 1.2 コメントとドキュメント

- **KDoc**: すべての public クラス・関数に記述
- **TODO コメント**: 未実装・改善箇所に `// TODO: [担当者] 説明` 形式で記述
- **実装コメント**: 複雑なロジック・単位変換・前提条件には必ずコメント
- **言語**: コメント・変数名は日本語も可（ただしクラス名・関数名は英語推奨）

---

## 2. 命名規則

### 2.1 一般規則

| 対象 | 規則 | 例 |
|------|------|-----|
| クラス | PascalCase | `BatProfile`, `SwingEvent` |
| 関数・変数 | camelCase | `startSession`, `tipSpeed` |
| 定数 | UPPER_SNAKE_CASE | `MAX_RETRY_COUNT`, `DEG_TO_RAD` |
| パッケージ | lowercase | `com.scrap.lion.domain.entity` |
| リソースID | snake_case | `btn_start_scan`, `tv_tip_speed` |

### 2.2 レイヤー別命名規則

#### Domain層

| 種類 | 命名規則 | 例 |
|------|----------|-----|
| Entity | 名詞（単数形） | `SwingSession`, `BatProfile`, `SensorSample` |
| Value Object | 名詞（単数形） | `Vec3`, `SwingSettings` |
| Port | `Port` サフィックス | `SensorStreamPort`, `ConnectionStatePort`, `SwingDetectorPort` |
| UseCase | 動詞句 | `StartSession`, `ObserveSwingEvents`, `SaveBatProfile` |

#### Data層

| 種類 | 命名規則 | 例 |
|------|----------|-----|
| Repository実装 | `Repository` サフィックス | `SessionRepositoryImpl`, `BatProfileRepositoryImpl` |
| DAO | `Dao` サフィックス | `SwingEventDao`, `BatProfileDao` |
| RoomEntity | `Entity` サフィックス | `SwingEventEntity`, `BatProfileEntity` |
| BLE Adapter | `Adapter` サフィックス | `Wt9011BleAdapter` |
| Parser | `Parser` サフィックス | `FrameParser` |

#### UI層

| 種類 | 命名規則 | 例 |
|------|----------|-----|
| Screen | `Screen` サフィックス | `ConnectScreen`, `MeasureScreen`, `HistoryScreen` |
| ViewModel | `ViewModel` サフィックス | `ConnectViewModel`, `MeasureViewModel` |
| UiState | `UiState` サフィックス | `ConnectUiState`, `MeasureUiState` |
| UiEvent | `Event` サフィックス | `ConnectEvent`, `MeasureEvent` |
| Composable部品 | 名詞 | `DeviceItem`, `SwingSpeedGauge` |

---

## 3. アーキテクチャ層ごとの規約

### 3.1 Domain層（純粋Kotlin）

#### 原則
- Android SDK 依存禁止（`Context`, `Bundle` 等は使用不可）
- 外部ライブラリ依存最小化（Coroutines/Flow のみ許可）
- すべてのロジックをテスト可能にする

#### Entity/Value Object
```kotlin
/**
 * センサーサンプル（加速度・角速度の1フレーム）
 *
 * @property timestampMicros エポックマイクロ秒 [μs]
 * @property accel 加速度ベクトル [m/s²]
 * @property gyroRadPerSec 角速度ベクトル [rad/s]
 */
data class SensorSample(
    val timestampMicros: Long,
    val accel: Vec3,
    val gyroRadPerSec: Vec3
) {
    init {
        require(timestampMicros > 0) { "timestampMicros must be positive" }
    }
}
```

**規約**:
- すべてのプロパティは `val`（不変）
- 不変条件は `init` ブロックで検証
- 単位はコメントで明記（`[m/s²]`, `[rad/s]`, `[μs]`）

#### Port（Repository/Service インターフェース）
```kotlin
/**
 * センサーストリーム Port
 * BLE デバイスから加速度・角速度をリアルタイム配信する
 */
interface SensorStreamPort {
    /**
     * センサーフレームのストリーム（200 Hz 目安）
     * バックプレッシャ: DROP_OLDEST（最新値優先）
     */
    val frames: Flow<SensorSample>
    
    /**
     * ストリーミング開始
     * @throws BleError Bluetooth無効・権限不足・接続失敗時
     */
    suspend fun startStreaming()
    
    /**
     * ストリーミング停止（接続は維持）
     */
    suspend fun stopStreaming()
}
```

**規約**:
- KDoc で責務・例外・バックプレッシャ戦略を明記
- 戻り値は `Flow` または `suspend fun`
- エラーは例外スロー（戻り値で `Result` 型は UseCase 層で使用）

#### UseCase
```kotlin
/**
 * スイングイベント監視 UseCase
 * SensorStream を購読し、スイング検出→保存→UI通知を実行
 *
 * @param sessionId 計測セッションID
 * @return 確定したスイングイベントのストリーム
 */
class ObserveSwingEvents @Inject constructor(
    private val sensorStreamPort: SensorStreamPort,
    private val swingDetectorPort: SwingDetectorPort,
    private val swingEventRepository: SwingEventRepository,
    private val rawSampleRepository: RawSampleRepository
) {
    operator fun invoke(sessionId: Long): Flow<SwingEvent> = flow {
        sensorStreamPort.frames.collect { sample ->
            val events = swingDetectorPort.process(sample)
            events.forEach { event ->
                val eventId = swingEventRepository.save(event.copy(sessionId = sessionId))
                // Raw サンプル保存
                // ...
                emit(event.copy(id = eventId))
            }
        }
    }
}
```

**規約**:
- クラス名は動詞句（`StartSession`, `SaveBatProfile`）
- `operator fun invoke()` で呼び出し可能にする
- 依存は Constructor Injection（`@Inject constructor`）
- 戻り値は `Flow<T>` または `suspend fun`

---

### 3.2 Data層

#### Repository実装
```kotlin
/**
 * BatProfile Repository 実装（Room DB使用）
 */
class BatProfileRepositoryImpl @Inject constructor(
    private val dao: BatProfileDao,
    private val dispatcherProvider: DispatcherProvider
) : BatProfileRepository {
    
    override suspend fun getActiveProfile(userId: Long): BatProfile = 
        withContext(dispatcherProvider.io) {
            val entity = dao.getActiveByUserId(userId)
                ?: throw NoSuchElementException("No active profile for user $userId")
            entity.toDomain()
        }
    
    override suspend fun save(profile: BatProfile): Long = 
        withContext(dispatcherProvider.io) {
            dao.insert(profile.toEntity())
        }
}
```

**規約**:
- `Port` インターフェースを実装
- DB/Network 呼び出しは `DispatcherProvider.io` で実行
- Entity → Domain 変換は拡張関数（`toEntity()`, `toDomain()`）
- エラーは適切な例外にラップ（`NoSuchElementException`, `IOException`）

#### Room Entity
```kotlin
@Entity(
    tableName = "bat_profiles",
    foreignKeys = [
        ForeignKey(
            entity = UserEntity::class,
            parentColumns = ["id"],
            childColumns = ["user_id"],
            onDelete = ForeignKey.CASCADE
        )
    ],
    indices = [Index("user_id")]
)
data class BatProfileEntity(
    @PrimaryKey(autoGenerate = true)
    val id: Long = 0,
    
    @ColumnInfo(name = "user_id")
    val userId: Long,
    
    val name: String,
    
    @ColumnInfo(name = "length_m")
    val lengthMeters: Float,
    
    @ColumnInfo(name = "d_hand_m")
    val dHandMeters: Float,
    
    @ColumnInfo(name = "d_sweet_m")
    val dSweetMeters: Float,
    
    val gain: Float,
    
    @ColumnInfo(name = "created_at")
    val createdAtMillis: Long
)

fun BatProfileEntity.toDomain() = BatProfile(
    id = if (id == 0L) null else id,
    userId = userId,
    name = name,
    lengthMeters = lengthMeters,
    dHandMeters = dHandMeters,
    dSweetMeters = dSweetMeters,
    gain = gain
)

fun BatProfile.toEntity() = BatProfileEntity(
    id = id ?: 0,
    userId = userId,
    name = name,
    lengthMeters = lengthMeters,
    dHandMeters = dHandMeters,
    dSweetMeters = dSweetMeters,
    gain = gain,
    createdAtMillis = System.currentTimeMillis()
)
```

**規約**:
- テーブル名は snake_case（`bat_profiles`, `swing_events`）
- カラム名は snake_case（`user_id`, `created_at`）
- `@ColumnInfo` でカラム名を明示
- FK/Index は Entity に定義
- 変換関数は同一ファイルに定義

#### DAO
```kotlin
@Dao
interface BatProfileDao {
    @Query("SELECT * FROM bat_profiles WHERE user_id = :userId ORDER BY created_at DESC")
    suspend fun listByUserId(userId: Long): List<BatProfileEntity>
    
    @Query("SELECT * FROM bat_profiles WHERE user_id = :userId AND id = (SELECT bat_profile_id FROM swing_sessions WHERE user_id = :userId ORDER BY started_at DESC LIMIT 1)")
    suspend fun getActiveByUserId(userId: Long): BatProfileEntity?
    
    @Insert(onConflict = OnConflictStrategy.REPLACE)
    suspend fun insert(profile: BatProfileEntity): Long
    
    @Delete
    suspend fun delete(profile: BatProfileEntity)
}
```

**規約**:
- すべての関数は `suspend`
- `@Query` は複数行の場合は改行して可読性を確保
- 戻り値は nullable（`?`）または `List`（空の場合 `emptyList()`）

---

### 3.3 UI層（Compose）

#### Screen
```kotlin
/**
 * 接続画面
 * BLE デバイスのスキャン・接続を行う
 */
@Composable
fun ConnectScreen(
    viewModel: ConnectViewModel = hiltViewModel(),
    onNavigateToMeasure: () -> Unit
) {
    val uiState by viewModel.uiState.collectAsState()
    
    ConnectScreenContent(
        uiState = uiState,
        onEvent = viewModel::onEvent,
        onNavigateToMeasure = onNavigateToMeasure
    )
}

@Composable
private fun ConnectScreenContent(
    uiState: ConnectUiState,
    onEvent: (ConnectEvent) -> Unit,
    onNavigateToMeasure: () -> Unit
) {
    // UI実装
}

@Preview(showBackground = true)
@Composable
private fun ConnectScreenPreview() {
    ConnectScreenContent(
        uiState = ConnectUiState(
            devices = listOf(
                DeviceItem(
                    address = "AA:BB:CC:DD:EE:FF",
                    name = "WT9011DCL",
                    rssi = -60,
                    txPower = null,
                    isConnectable = true,
                    lastSeenAtMillis = System.currentTimeMillis()
                )
            ),
            isScanning = false,
            isConnected = false
        ),
        onEvent = {},
        onNavigateToMeasure = {}
    )
}
```

**規約**:
- Screen 関数は ViewModel を受け取る（Hilt注入）
- 実装は `*Content` 関数に分離（Preview用）
- Preview 関数を必ず定義
- Navigation は callback で受け取る（`onNavigateTo*`）

#### ViewModel
```kotlin
@HiltViewModel
class ConnectViewModel @Inject constructor(
    private val connectionStatePort: ConnectionStatePort,
    private val permissionChecker: PermissionChecker
) : ViewModel() {
    
    // UiState: StateFlow で公開
    private val _uiState = MutableStateFlow(ConnectUiState())
    val uiState: StateFlow<ConnectUiState> = _uiState.asStateFlow()
    
    // イベントハンドラ
    fun onEvent(event: ConnectEvent) {
        when (event) {
            ConnectEvent.StartScan -> handleStartScan()
            ConnectEvent.StopScan -> handleStopScan()
            is ConnectEvent.Connect -> handleConnect(event.address)
            ConnectEvent.Disconnect -> handleDisconnect()
            ConnectEvent.RequestPermission -> handleRequestPermission()
        }
    }
    
    private fun handleStartScan() {
        viewModelScope.launch {
            if (!permissionChecker.hasBluetoothPermissions()) {
                _uiState.update { it.copy(showPermissionDialog = true) }
                return@launch
            }
            connectionStatePort.startScan()
        }
    }
    
    // ... 他のハンドラ
}
```

**規約**:
- `@HiltViewModel` 必須
- UiState は `StateFlow` で公開（`private val _uiState` + `val uiState`）
- イベントハンドラは `onEvent(event: Event)` で一本化
- 内部ハンドラは `private fun handle*()`
- Coroutine は `viewModelScope.launch`

#### UiState/UiEvent
```kotlin
/**
 * 接続画面の UI 状態
 */
data class ConnectUiState(
    val devices: List<DeviceItem> = emptyList(),
    val isScanning: Boolean = false,
    val isConnected: Boolean = false,
    val showPermissionDialog: Boolean = false,
    val error: String? = null
)

/**
 * 接続画面の UI イベント
 */
sealed interface ConnectEvent {
    object StartScan : ConnectEvent
    object StopScan : ConnectEvent
    data class Connect(val address: String) : ConnectEvent
    object Disconnect : ConnectEvent
    object RequestPermission : ConnectEvent
}
```

**規約**:
- UiState は `data class`（デフォルト値必須）
- UiEvent は `sealed interface`（`object` または `data class`）
- プロパティはすべて `val`（不変）

---

## 4. エラーハンドリング

### 4.1 エラー型階層

```kotlin
// Domain層エラー
sealed class DomainError : Exception() {
    data class ValidationError(override val message: String) : DomainError()
    data class NotFoundError(val entityType: String, val id: Any) : DomainError()
    data class ConflictError(override val message: String) : DomainError()
}

// Data層エラー（BLE）
sealed class BleError : Exception() {
    object BluetoothDisabled : BleError()
    object PermissionDenied : BleError()
    data class ScanFailed(val errorCode: Int) : BleError()
    data class ConnectionFailed(val reason: String) : BleError()
    object Timeout : BleError()
}

// Data層エラー（DB）
sealed class DatabaseError : Exception() {
    data class InsertFailed(val cause: Throwable) : DatabaseError()
    data class QueryFailed(val cause: Throwable) : DatabaseError()
}
```

### 4.2 エラー伝搬

| 層 | エラー処理 |
|-----|-----------|
| Domain | 例外をスロー（`throw DomainError.*`） |
| Data | Port 呼び出し時は例外スロー。Repository 実装内では `Result<T>` でラップも可 |
| UseCase | 例外を `Result<T>` でラップして返す（`Result.success()` / `Result.failure()`） |
| ViewModel | `Result` を受け取り、UiState に反映（`error: String?`） |
| UI | エラーメッセージを Toast/Snackbar で表示 |

### 4.3 UseCase でのエラーハンドリング例

```kotlin
class SaveBatProfile @Inject constructor(
    private val repository: BatProfileRepository
) {
    suspend operator fun invoke(profile: BatProfile): Result<Long> = runCatching {
        // バリデーション
        require(profile.lengthMeters > 0) { "Length must be positive" }
        require(profile.radiusMeters > 0) { "Radius must be positive" }
        
        // 保存
        repository.save(profile)
    }.onFailure { error ->
        // ログ記録（Analytics等）
        Log.e("SaveBatProfile", "Failed to save profile", error)
    }
}
```

---

## 5. 非同期処理

### 5.1 Coroutine

**原則**:
- UI層: `viewModelScope.launch`
- Repository層: `withContext(dispatcherProvider.io)`
- UseCase層: `suspend fun`（呼び出し側がスコープ管理）

**DispatcherProvider 使用**:
```kotlin
class SessionRepositoryImpl @Inject constructor(
    private val dao: SessionDao,
    private val dispatcherProvider: DispatcherProvider
) : SessionRepository {
    override suspend fun startSession(userId: Long, batProfileId: Long): Long = 
        withContext(dispatcherProvider.io) {
            dao.insert(SessionEntity(userId = userId, batProfileId = batProfileId, ...))
        }
}
```

### 5.2 Flow

**命名規則**:
- Observable な状態: `isScanning`, `devices`, `events`（`Flow` サフィックス不要）
- 単発イベント: `events`, `errors`

**バックプレッシャ戦略**:
| 用途 | 戦略 | 実装 |
|------|------|------|
| センサーストリーム（高頻度） | 最新値優先 | `buffer(onBufferOverflow = BufferOverflow.DROP_OLDEST)` |
| DB監視（低頻度） | すべて処理 | `buffer()` デフォルト |
| UI状態（1値） | 最新値 | `StateFlow` |

---

## 6. 単位とコメント

### 6.1 物理量の単位

すべての物理量は**SI単位**で統一し、コメントで明記する：

| 物理量 | 単位 | 表記例 |
|--------|------|--------|
| 時刻（ドメイン） | マイクロ秒 [μs] | `timestampMicros: Long` |
| 時刻（DB） | ミリ秒 [ms] | `createdAtMillis: Long` |
| 加速度 | m/s² | `accel: Vec3  // [m/s²]` |
| 角速度 | rad/s | `gyroRadPerSec: Vec3  // [rad/s]` |
| 長さ | メートル [m] | `lengthMeters: Float` |
| 速度 | m/s | `tipSpeedMetersPerSec: Float` |
| 角度 | ラジアン [rad] | `impactAngleRad: Float` |

### 6.2 単位変換

変換処理には必ずコメントを記述：

```kotlin
// data/ble/FrameParser.kt
class FrameParser {
    private val DEG_TO_RAD = (Math.PI / 180.0).toFloat()  // deg → rad 変換係数
    private val G_TO_MPS2 = 9.80665f  // g → m/s² 変換係数
    
    private fun parseAccelGyroAngle(b: ByteArray): SensorSample {
        // 加速度: g → m/s²
        val axG = readInt16(b, 2) / 32768f * 16f  // [g]
        val accel = Vec3(
            x = axG * G_TO_MPS2,  // [m/s²]
            // ...
        )
        
        // 角速度: deg/s → rad/s
        val wxDeg = readInt16(b, 8) / 32768f * 2000f  // [deg/s]
        val gyro = Vec3(
            x = wxDeg * DEG_TO_RAD,  // [rad/s]
            // ...
        )
        
        // タイムスタンプ: ms → μs
        val timestampMicros = System.currentTimeMillis() * 1000L  // [μs]
        
        return SensorSample(timestampMicros, accel, gyro)
    }
}
```

---

## 7. Hilt 依存性注入

### 7.1 Module 構成

| Module | スコープ | 提供内容 |
|--------|----------|----------|
| `CoreModule` | `SingletonComponent` | `DispatcherProvider`, `TimeSource` |
| `BleModule` | `SingletonComponent` | `Wt9011BleAdapter`, `SensorStreamPort`, `ConnectionStatePort` |
| `DatabaseModule` | `SingletonComponent` | `AppDatabase`, 各 DAO |
| `RepositoryModule` | `SingletonComponent` | 各 Repository 実装 |
| `UseCaseModule` | `ViewModelComponent` | 各 UseCase |

### 7.2 Module 例

```kotlin
@Module
@InstallIn(SingletonComponent::class)
object RepositoryModule {
    
    @Provides
    @Singleton
    fun provideBatProfileRepository(
        dao: BatProfileDao,
        dispatcherProvider: DispatcherProvider
    ): BatProfileRepository {
        return BatProfileRepositoryImpl(dao, dispatcherProvider)
    }
    
    // ... 他の Repository
}
```

### 7.3 Constructor Injection（推奨）

```kotlin
@HiltViewModel
class MeasureViewModel @Inject constructor(
    private val startMeasuringUseCase: StartMeasuring,
    private val stopMeasuringUseCase: StopMeasuring,
    private val observeSwingEventsUseCase: ObserveSwingEvents
) : ViewModel()
```

---

## 8. テストコード

### 8.1 命名規則

| 種類 | 規則 | 例 |
|------|------|-----|
| テストクラス | `*Test` | `FrameParserTest`, `SaveBatProfileTest` |
| テスト関数 | バッククォート日本語可 | `` `0x61フレームを正しくパースする` `` |
| Fake実装 | `Fake*` | `FakeSensorStreamPort`, `FakeBatProfileRepository` |

### 8.2 テスト構造

```kotlin
class SaveBatProfileTest {
    
    private lateinit var repository: FakeBatProfileRepository
    private lateinit var useCase: SaveBatProfile
    
    @Before
    fun setUp() {
        repository = FakeBatProfileRepository()
        useCase = SaveBatProfile(repository)
    }
    
    @Test
    fun `正常なプロファイルを保存できる`() = runTest {
        // Given
        val profile = BatProfile(
            id = null,
            userId = 1,
            name = "Test Bat",
            lengthMeters = 0.84f,
            dHandMeters = 0.15f,
            dSweetMeters = 0.78f,
            gain = 1.1f
        )
        
        // When
        val result = useCase(profile)
        
        // Then
        assertTrue(result.isSuccess)
        assertEquals(1, repository.savedProfiles.size)
    }
    
    @Test
    fun `長さが負の場合はエラー`() = runTest {
        // Given
        val invalidProfile = BatProfile(/* lengthMeters = -0.84f */)
        
        // When
        val result = useCase(invalidProfile)
        
        // Then
        assertTrue(result.isFailure)
        assertTrue(result.exceptionOrNull() is IllegalArgumentException)
    }
}
```

---

## 9. リソース命名規則

### 9.1 Layout/Composable

Compose 使用のため、XML Layout は最小限：

| 種類 | 規則 | 例 |
|------|------|-----|
| Activity | `activity_*` | `activity_main.xml` |
| Fragment | `fragment_*` | （未使用） |

### 9.2 Drawable

| 種類 | 規則 | 例 |
|------|------|-----|
| アイコン | `ic_*` | `ic_bluetooth.xml`, `ic_check.xml` |
| 背景 | `bg_*` | `bg_button_rounded.xml` |
| 画像 | `img_*` | `img_bat_placeholder.png` |

### 9.3 Strings/Colors

```xml
<!-- strings.xml -->
<resources>
    <!-- Screen: Connect -->
    <string name="connect_title">デバイス接続</string>
    <string name="connect_btn_scan">スキャン開始</string>
    <string name="connect_error_permission">Bluetooth権限が必要です</string>
    
    <!-- Screen: Measure -->
    <string name="measure_title">計測中</string>
    <string name="measure_tip_speed_label">スイングスピード</string>
</resources>

<!-- colors.xml -->
<resources>
    <color name="primary">#6200EE</color>
    <color name="primary_variant">#3700B3</color>
    <color name="secondary">#03DAC6</color>
    
    <color name="swing_speed_high">#4CAF50</color>
    <color name="swing_speed_medium">#FFC107</color>
    <color name="swing_speed_low">#F44336</color>
</resources>
```

**命名規則**:
- `<screen>_<element>_<property>`
- 色は機能別（`swing_speed_high`）またはテーマ別（`primary`）

---

## 10. Git コミット規約

### 10.1 コミットメッセージ

```
<type>(<scope>): <subject>

<body>

<footer>
```

**Type**:
- `feat`: 新機能
- `fix`: バグ修正
- `refactor`: リファクタリング
- `docs`: ドキュメント変更
- `test`: テスト追加・修正
- `style`: フォーマット変更（機能変更なし）
- `chore`: ビルド設定・依存関係更新

**Scope**:
- `domain`, `data`, `ui`, `ble`, `db`, `usecase`, `viewmodel`

**例**:
```
feat(ble): WT9011DCL BLE Adapter 実装

- Wt9011BleAdapter クラス追加（SensorStreamPort実装）
- FrameParser でg→m/s², deg/s→rad/s 変換
- スキャン・接続・Notify購読機能

Refs: #12
```

### 10.2 ブランチ戦略

- `master`: 本番リリース版
- `develop`: 開発統合ブランチ
- `feature/*`: 機能開発（例: `feature/ble-adapter`）
- `fix/*`: バグ修正（例: `fix/connection-timeout`）

---

## 11. コードレビューチェックリスト

- [ ] アーキテクチャ層が正しく分離されているか
- [ ] Port/Repository を経由せず直接 BLE/DB にアクセスしていないか
- [ ] すべての物理量に単位コメントがあるか
- [ ] 単位変換ロジックにコメントがあるか
- [ ] public 関数に KDoc があるか
- [ ] エラーハンドリングが適切か（例外型、`Result` 型）
- [ ] Coroutine の Dispatcher が適切か（`viewModelScope`, `withContext(io)`）
- [ ] Room Entity と Domain Entity の変換が正しいか（`toEntity()`, `toDomain()`）
- [ ] Compose Preview が定義されているか
- [ ] テストが追加されているか（最低限 UseCase テスト）
- [ ] 命名規則に従っているか
- [ ] フォーマットが統一されているか（`Ctrl+Alt+L` 実行済み）

---

## 12. 禁止事項

### 12.1 アーキテクチャ違反

❌ **禁止**:
```kotlin
// ViewModel から直接 BLE にアクセス
class MeasureViewModel @Inject constructor(
    private val bluetoothAdapter: BluetoothAdapter  // NG!
) : ViewModel()
```

✅ **推奨**:
```kotlin
// Port を経由
class MeasureViewModel @Inject constructor(
    private val sensorStreamPort: SensorStreamPort  // OK
) : ViewModel()
```

### 12.2 Android依存の混入

❌ **禁止**:
```kotlin
// Domain 層で Context 使用
class StartSession @Inject constructor(
    private val context: Context  // NG! Domain層にAndroid依存
) {
    // ...
}
```

✅ **推奨**:
```kotlin
// Port で抽象化
interface TimeSource {
    fun currentTimeMillis(): Long
}

class StartSession @Inject constructor(
    private val timeSource: TimeSource  // OK
)
```

### 12.3 Magic Number

❌ **禁止**:
```kotlin
val accel = rawValue / 32768f * 16f  // 何の数値か不明
```

✅ **推奨**:
```kotlin
private const val INT16_MAX = 32768f
private const val ACCEL_RANGE_G = 16f
val accel = rawValue / INT16_MAX * ACCEL_RANGE_G  // [g]
```

---

## 13. まとめ

本規約に従うことで、以下を実現する：

1. **一貫性**: コードスタイル・命名規則の統一
2. **保守性**: レイヤー分離・依存性注入による疎結合
3. **テスタビリティ**: Port/Fake を活用した単体テスト
4. **可読性**: KDoc・単位コメント・構造化されたエラーハンドリング

不明点・改善提案は Issue で議論し、本ドキュメントを随時更新する。

---

**最終更新**: 2025-11-04
**バージョン**: 1.0.0

