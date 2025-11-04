# テスト戦略（Testing Strategy）

本ドキュメントは、Lion アプリの品質保証戦略とテスト方針を定義する。アーキテクチャ設計（`docs/architecture.md`）に基づき、各層のテスト方法・ツール・カバレッジ目標を明示する。

**対象読者**: 本プロジェクトの開発者・QA担当者

**参照ドキュメント**:
- `docs/architecture.md` - アーキテクチャ設計
- `docs/coding_guidelines.md` - コーディング規約
- `docs/wt9011dcl_integration.md` - BLE統合実装ガイド

---

## 1. テストピラミッド

本プロジェクトは以下のテストピラミッドに従う：

```
        ┌─────────────┐
        │  E2E Test   │  ← 少数（重要フロー）
        │   (5件)     │
        ├─────────────┤
        │Integration  │  ← 中程度（重要な統合ポイント）
        │  Test (30件)│
        ├─────────────┤
        │ Unit Test   │  ← 多数（すべてのドメインロジック）
        │  (200件)    │
        └─────────────┘
```

**理由**:
- ユニットテストは高速・安定・保守容易
- 統合テストは実際のDB/BLE動作を検証
- E2Eテストは重要なユーザーシナリオのみ（コスト大）

---

## 2. テスト種別とツール

| 種別 | 対象 | ツール | 実行環境 | 目標カバレッジ |
|------|------|--------|----------|----------------|
| 単体テスト | Domain層（UseCase/Entity/純粋ロジック） | JUnit4, Kotlin Test | JVM | 90%以上 |
| 統合テスト | Data層（Repository/DAO/BLE） | JUnit4, AndroidX Test, Robolectric | Android/実機 | 70%以上 |
| UIテスト | Compose Screen | Compose Testing, AndroidX Test | Android/実機 | 60%以上（主要画面） |
| E2Eテスト | アプリ全体フロー | Espresso, UI Automator | 実機 | 主要5シナリオ |
| パフォーマンステスト | BLE ストリーム、DB書き込み | Macrobenchmark, Profiler | 実機 | 閾値監視 |

---

## 3. レイヤー別テスト方針

### 3.1 Domain層（純粋Kotlin）

**テスト方針**:
- すべての UseCase を単体テスト
- Entity の不変条件を検証
- Port は Fake 実装でモック化
- 外部依存なし（高速・安定）

**テスト対象**:
- ✅ UseCase（入出力、エラーハンドリング）
- ✅ Entity の不変条件（`init` ブロック）
- ✅ Value Object の計算ロジック（`BatProfile.radiusMeters`）
- ✅ スイング検出ロジック（`SwingDetectorPort` 実装）

**テスト不要**:
- Port インターフェース（実装がないため）

#### 例1: UseCase テスト（SaveBatProfile）

```kotlin
// test/domain/usecase/SaveBatProfileTest.kt
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
        val savedProfile = repository.savedProfiles.first()
        assertEquals("Test Bat", savedProfile.name)
        assertEquals(0.63f, savedProfile.radiusMeters, 0.01f)  // 0.78 - 0.15
    }
    
    @Test
    fun `長さが負の場合はエラー`() = runTest {
        // Given
        val invalidProfile = BatProfile(
            id = null, userId = 1, name = "Invalid",
            lengthMeters = -0.84f,  // 不正値
            dHandMeters = 0.15f, dSweetMeters = 0.78f, gain = 1.1f
        )
        
        // When
        val result = useCase(invalidProfile)
        
        // Then
        assertTrue(result.isFailure)
        assertIs<IllegalArgumentException>(result.exceptionOrNull())
    }
    
    @Test
    fun `半径が負の場合はエラー`() = runTest {
        // Given: d_sweet < d_hand → radius < 0
        val invalidProfile = BatProfile(
            id = null, userId = 1, name = "Invalid",
            lengthMeters = 0.84f,
            dHandMeters = 0.50f,
            dSweetMeters = 0.30f,  // 不正: 手元よりスイートが根元側
            gain = 1.1f
        )
        
        // When
        val result = useCase(invalidProfile)
        
        // Then
        assertTrue(result.isFailure)
    }
    
    @Test
    fun `Repository 失敗時は Result_failure を返す`() = runTest {
        // Given
        repository.shouldThrowError = true
        val profile = BatProfile(/* 正常値 */)
        
        // When
        val result = useCase(profile)
        
        // Then
        assertTrue(result.isFailure)
    }
}
```

#### 例2: Entity 不変条件テスト

```kotlin
// test/domain/entity/SensorSampleTest.kt
class SensorSampleTest {
    
    @Test
    fun `正常なSensorSampleを生成できる`() {
        // Given/When
        val sample = SensorSample(
            timestampMicros = 1000000L,
            accel = Vec3(0f, 0f, 9.8f),
            gyroRadPerSec = Vec3(0f, 0f, 0f)
        )
        
        // Then
        assertEquals(1000000L, sample.timestampMicros)
        assertEquals(9.8f, sample.accel.z, 0.01f)
    }
    
    @Test
    fun `timestampMicros が負の場合は例外`() {
        // When/Then
        assertThrows<IllegalArgumentException> {
            SensorSample(
                timestampMicros = -1L,  // 不正値
                accel = Vec3(0f, 0f, 9.8f),
                gyroRadPerSec = Vec3(0f, 0f, 0f)
            )
        }
    }
}
```

#### 例3: スイング検出ロジックテスト

```kotlin
// test/domain/detector/SwingDetectorImplTest.kt
class SwingDetectorImplTest {
    
    private lateinit var detector: SwingDetectorImpl
    private val settings = SwingSettings(
        startThresholdADyn = 3.5f,
        endThresholdADyn = 1.8f,
        endThresholdWPerp = 3.0f,
        tRiseMillis = 25,
        tFallMillis = 50,
        refractoryPeriodMillis = 200
    )
    private val bat = BatProfile(
        id = 1, userId = 1, name = "Test",
        lengthMeters = 0.84f, dHandMeters = 0.15f, dSweetMeters = 0.78f, gain = 1.1f
    )
    
    @Before
    fun setUp() {
        detector = SwingDetectorImpl()
        detector.updateSettings(settings, bat)
    }
    
    @Test
    fun `閾値超過でスイング開始を検出`() {
        // Given: 30ms間、閾値超過
        val samples = generateHighAccelSamples(
            count = 7,  // 200Hz → 35ms
            aDyn = 4.0f,  // > 3.5f
            intervalMicros = 5000L
        )
        
        // When
        var detectedStart = false
        samples.forEach { sample ->
            val events = detector.process(sample)
            if (detector.isSwinging) detectedStart = true
        }
        
        // Then
        assertTrue(detectedStart, "Should detect swing start")
    }
    
    @Test
    fun `加速度とジャイロが両方沈静化するまで終了しない`() {
        // Given: スイング開始後、加速度のみ低下
        detector.reset()
        // 開始シーケンス
        repeat(10) { detector.process(highAccelSample()) }
        assertTrue(detector.isSwinging)
        
        // When: 加速度低下、ジャイロ高いまま
        repeat(15) {
            detector.process(SensorSample(
                timestampMicros = System.currentTimeMillis() * 1000L,
                accel = Vec3(0f, 0f, 1.0f),  // 低い
                gyroRadPerSec = Vec3(5.0f, 5.0f, 0f)  // 高い → Wperp = 7.07
            ))
        }
        
        // Then: まだ終了していない
        assertTrue(detector.isSwinging, "Should not end while gyro is high")
    }
    
    @Test
    fun `両方沈静化でスイング終了を検出`() {
        // Given: スイング中
        detector.reset()
        repeat(10) { detector.process(highAccelSample()) }
        
        // When: 両方低下
        var event: SwingEvent? = null
        repeat(15) {
            val events = detector.process(SensorSample(
                timestampMicros = System.currentTimeMillis() * 1000L,
                accel = Vec3(0f, 0f, 1.0f),  // 低い
                gyroRadPerSec = Vec3(0.5f, 0.5f, 0f)  // 低い
            ))
            if (events.isNotEmpty()) event = events.first()
        }
        
        // Then
        assertNotNull(event, "Should detect swing end")
        assertTrue(event!!.tipSpeedMetersPerSec > 0f)
    }
    
    @Test
    fun `refractory期間内は多重検出しない`() {
        // Given: 1回目のスイング完了
        detector.reset()
        repeat(10) { detector.process(highAccelSample()) }
        repeat(15) { detector.process(lowSample()) }
        
        // When: 200ms以内に再度閾値超過
        Thread.sleep(50)  // 50ms待機
        repeat(10) { detector.process(highAccelSample()) }
        
        // Then
        assertFalse(detector.isSwinging, "Should suppress detection during refractory")
    }
    
    private fun highAccelSample() = SensorSample(
        timestampMicros = System.currentTimeMillis() * 1000L,
        accel = Vec3(0f, 0f, 15.0f),  // 高い
        gyroRadPerSec = Vec3(10.0f, 10.0f, 0f)
    )
    
    private fun lowSample() = SensorSample(
        timestampMicros = System.currentTimeMillis() * 1000L,
        accel = Vec3(0f, 0f, 1.0f),
        gyroRadPerSec = Vec3(0.5f, 0.5f, 0f)
    )
}
```

---

### 3.2 Data層

**テスト方針**:
- Repository 実装を統合テスト（実際のDAOを使用）
- DAO は Room In-Memory DB でテスト
- BLE Adapter は実機統合テスト（Fake 可能な部分は単体テスト）

**テスト対象**:
- ✅ Repository 実装（CRUD操作、エラーハンドリング）
- ✅ DAO クエリ（正確性、インデックス効果）
- ✅ Entity ⇔ Domain 変換（`toEntity()`, `toDomain()`）
- ✅ FrameParser（単位変換、チェックサム）
- ✅ BLE Adapter（スキャン、接続、パース統合）

#### 例1: Repository統合テスト

```kotlin
// androidTest/data/repository/BatProfileRepositoryImplTest.kt
@RunWith(AndroidJUnit4::class)
class BatProfileRepositoryImplTest {
    
    private lateinit var database: AppDatabase
    private lateinit var repository: BatProfileRepository
    
    @Before
    fun setUp() {
        val context = ApplicationProvider.getApplicationContext<Context>()
        database = Room.inMemoryDatabaseBuilder(context, AppDatabase::class.java)
            .allowMainThreadQueries()
            .build()
        
        repository = BatProfileRepositoryImpl(
            dao = database.batProfileDao(),
            dispatcherProvider = TestDispatcherProvider()
        )
    }
    
    @After
    fun tearDown() {
        database.close()
    }
    
    @Test
    fun saveAndGetActiveProfile() = runTest {
        // Given: User作成
        val userId = database.userDao().insert(UserEntity(name = "Test User"))
        
        // When: Profile保存
        val profile = BatProfile(
            id = null, userId = userId, name = "Test Bat",
            lengthMeters = 0.84f, dHandMeters = 0.15f, dSweetMeters = 0.78f, gain = 1.1f
        )
        val profileId = repository.save(profile)
        
        // Then: 取得できる
        val retrieved = repository.getActiveProfile(userId)
        assertEquals("Test Bat", retrieved.name)
        assertEquals(profileId, retrieved.id)
    }
    
    @Test
    fun listByUserId() = runTest {
        // Given: 2ユーザー × 各2プロファイル
        val user1Id = database.userDao().insert(UserEntity(name = "User1"))
        val user2Id = database.userDao().insert(UserEntity(name = "User2"))
        
        repository.save(BatProfile(null, user1Id, "Bat1-1", 0.84f, 0.15f, 0.78f, 1.1f))
        repository.save(BatProfile(null, user1Id, "Bat1-2", 0.80f, 0.10f, 0.70f, 1.0f))
        repository.save(BatProfile(null, user2Id, "Bat2-1", 0.85f, 0.20f, 0.75f, 1.2f))
        
        // When
        val user1Profiles = repository.list(user1Id)
        val user2Profiles = repository.list(user2Id)
        
        // Then
        assertEquals(2, user1Profiles.size)
        assertEquals(1, user2Profiles.size)
        assertTrue(user1Profiles.any { it.name == "Bat1-1" })
    }
    
    @Test
    fun deleteProfile() = runTest {
        // Given
        val userId = database.userDao().insert(UserEntity(name = "User"))
        val profileId = repository.save(BatProfile(null, userId, "Bat", 0.84f, 0.15f, 0.78f, 1.1f))
        
        // When
        repository.delete(profileId)
        
        // Then
        val profiles = repository.list(userId)
        assertTrue(profiles.isEmpty())
    }
}
```

#### 例2: FrameParser 単体テスト

```kotlin
// test/data/ble/FrameParserTest.kt
class FrameParserTest {
    
    private val parser = FrameParser()
    
    @Test
    fun `0x61フレーム（加速度+角速度+角度）を正しくパースする`() {
        // Given: 実機データに基づくフレーム
        // 加速度: (0, 0, 1g) → (0, 0, 0x2000) in int16
        // 角速度: (0, 0, 0) → (0, 0, 0)
        val frame = byteArrayOf(
            0x55.toByte(), 0x61.toByte(),  // ヘッダ + Flag
            0x00, 0x00,  // ax = 0
            0x00, 0x00,  // ay = 0
            0x00, 0x20,  // az = 0x2000 → 1g
            0x00, 0x00,  // wx = 0
            0x00, 0x00,  // wy = 0
            0x00, 0x00,  // wz = 0
            0x00, 0x00, 0x00, 0x00, 0x00, 0x00  // 角度（未使用）
        )
        
        // When
        val result = parser.parse(frame)
        
        // Then
        assertNotNull(result)
        assertEquals(0f, result!!.accel.x, 0.01f)
        assertEquals(0f, result.accel.y, 0.01f)
        assertEquals(9.8f, result.accel.z, 0.1f)  // 1g ≈ 9.8 m/s²
        assertEquals(0f, result.gyroRadPerSec.x, 0.01f)
    }
    
    @Test
    fun `角速度の単位変換（deg_s to rad_s）が正しい`() {
        // Given: ωx = 2000 deg/s → 0x7FFF (max positive)
        val frame = byteArrayOf(
            0x55.toByte(), 0x61.toByte(),
            0x00, 0x00, 0x00, 0x00, 0x00, 0x00,  // accel = 0
            0xFF.toByte(), 0x7F.toByte(),  // wx = 32767 → 2000 deg/s
            0x00, 0x00, 0x00, 0x00,  // wy, wz = 0
            0x00, 0x00, 0x00, 0x00, 0x00, 0x00
        )
        
        // When
        val result = parser.parse(frame)
        
        // Then
        assertNotNull(result)
        val expectedRadPerSec = 2000f * (Math.PI.toFloat() / 180f)  // ≈ 34.9 rad/s
        assertEquals(expectedRadPerSec, result!!.gyroRadPerSec.x, 0.1f)
    }
    
    @Test
    fun `不正なヘッダはnullを返す`() {
        // Given
        val invalidFrame = byteArrayOf(0xAA.toByte(), 0x61.toByte())
        
        // When
        val result = parser.parse(invalidFrame)
        
        // Then
        assertNull(result)
    }
    
    @Test
    fun `長さ不足のフレームはnullを返す`() {
        // Given
        val shortFrame = byteArrayOf(0x55.toByte(), 0x61.toByte(), 0x00)
        
        // When
        val result = parser.parse(shortFrame)
        
        // Then
        assertNull(result)
    }
}
```

#### 例3: BLE Adapter 統合テスト（実機）

```kotlin
// androidTest/data/ble/Wt9011BleAdapterTest.kt
@RunWith(AndroidJUnit4::class)
class Wt9011BleAdapterTest {
    
    private lateinit var adapter: Wt9011BleAdapter
    
    @Before
    fun setUp() {
        val context = ApplicationProvider.getApplicationContext<Context>()
        adapter = Wt9011BleAdapter(context, TestDispatcherProvider())
    }
    
    @After
    fun tearDown() {
        runBlocking {
            adapter.disconnect()
        }
    }
    
    @Test
    @RequiresDevice  // 実機必須
    fun scanFindsDevices() = runTest {
        // Given
        val devices = mutableListOf<DeviceItem>()
        val job = launch {
            adapter.scannedDevices.collect {
                devices.addAll(it)
            }
        }
        
        // When
        adapter.startScan()
        delay(5000)  // 5秒スキャン
        adapter.stopScan()
        job.cancel()
        
        // Then
        assertTrue(devices.isNotEmpty(), "Should find at least one device")
        assertTrue(devices.any { it.name?.contains("WT") == true }, "Should find WT9011DCL")
    }
    
    @Test
    @RequiresDevice
    fun connectAndReceiveFrames() = runTest(timeout = 30.seconds) {
        // Given: スキャンでデバイス取得
        adapter.startScan()
        delay(5000)
        val device = adapter.scannedDevices.value.firstOrNull { it.name?.contains("WT") == true }
        assertNotNull(device, "WT9011DCL not found")
        adapter.stopScan()
        
        // When: 接続→ストリーミング
        adapter.connect(device!!.address)
        delay(2000)  // 接続待機
        adapter.startStreaming()
        
        val samples = mutableListOf<SensorSample>()
        val job = launch {
            adapter.frames.take(100).collect { samples.add(it) }  // 100サンプル取得
        }
        job.join()
        
        // Then
        assertEquals(100, samples.size)
        assertTrue(samples.all { it.timestampMicros > 0 }, "All timestamps should be positive")
        assertTrue(samples.any { it.accel.z > 5f }, "Should detect gravity (~9.8 m/s²)")
    }
}
```

---

### 3.3 UI層（Compose）

**テスト方針**:
- 画面の表示・操作を Compose Testing でテスト
- ViewModel は Fake UseCase でテスト（単体テスト扱い）
- E2E は Espresso で統合

**テスト対象**:
- ✅ Screen の表示内容（テキスト、ボタン、リスト）
- ✅ ユーザー操作（クリック、入力、スクロール）
- ✅ ViewModel の State 更新（Fake UseCase 使用）

#### 例1: Compose Testing（ConnectScreen）

```kotlin
// androidTest/ui/feature/connect/ConnectScreenTest.kt
@RunWith(AndroidJUnit4::class)
class ConnectScreenTest {
    
    @get:Rule
    val composeTestRule = createComposeRule()
    
    @Test
    fun displaysDeviceList() {
        // Given
        val uiState = ConnectUiState(
            devices = listOf(
                DeviceItem("AA:BB:CC:DD:EE:FF", "WT9011DCL", -60, null, true, System.currentTimeMillis()),
                DeviceItem("11:22:33:44:55:66", "Device2", -70, null, true, System.currentTimeMillis())
            ),
            isScanning = false,
            isConnected = false
        )
        
        // When
        composeTestRule.setContent {
            ConnectScreenContent(
                uiState = uiState,
                onEvent = {},
                onNavigateToMeasure = {}
            )
        }
        
        // Then
        composeTestRule.onNodeWithText("WT9011DCL").assertExists()
        composeTestRule.onNodeWithText("Device2").assertExists()
        composeTestRule.onNodeWithText("AA:BB:CC:DD:EE:FF").assertExists()
    }
    
    @Test
    fun clickScanButtonTriggersEvent() {
        // Given
        var eventReceived: ConnectEvent? = null
        composeTestRule.setContent {
            ConnectScreenContent(
                uiState = ConnectUiState(),
                onEvent = { eventReceived = it },
                onNavigateToMeasure = {}
            )
        }
        
        // When
        composeTestRule.onNodeWithText("スキャン開始").performClick()
        
        // Then
        assertEquals(ConnectEvent.StartScan, eventReceived)
    }
    
    @Test
    fun clickDeviceTriggersConnectEvent() {
        // Given
        val device = DeviceItem("AA:BB:CC:DD:EE:FF", "WT9011DCL", -60, null, true, System.currentTimeMillis())
        var eventReceived: ConnectEvent? = null
        
        composeTestRule.setContent {
            ConnectScreenContent(
                uiState = ConnectUiState(devices = listOf(device)),
                onEvent = { eventReceived = it },
                onNavigateToMeasure = {}
            )
        }
        
        // When
        composeTestRule.onNodeWithText("WT9011DCL").performClick()
        
        // Then
        assertTrue(eventReceived is ConnectEvent.Connect)
        assertEquals("AA:BB:CC:DD:EE:FF", (eventReceived as ConnectEvent.Connect).address)
    }
}
```

#### 例2: ViewModel 単体テスト（Fake UseCase）

```kotlin
// test/ui/feature/measure/MeasureViewModelTest.kt
class MeasureViewModelTest {
    
    @get:Rule
    val mainDispatcherRule = MainDispatcherRule()
    
    private lateinit var fakeSensorStream: FakeSensorStreamPort
    private lateinit var fakeObserveSwingEvents: FakeObserveSwingEvents
    private lateinit var viewModel: MeasureViewModel
    
    @Before
    fun setUp() {
        fakeSensorStream = FakeSensorStreamPort()
        fakeObserveSwingEvents = FakeObserveSwingEvents(fakeSensorStream)
        viewModel = MeasureViewModel(
            sensorStreamPort = fakeSensorStream,
            observeSwingEventsUseCase = fakeObserveSwingEvents
        )
    }
    
    @Test
    fun startMeasuringUpdatesState() = runTest {
        // Given
        val initialState = viewModel.uiState.value
        assertFalse(initialState.isStreaming)
        
        // When
        viewModel.onEvent(MeasureEvent.StartMeasuring)
        advanceUntilIdle()
        
        // Then
        assertTrue(viewModel.uiState.value.isStreaming)
    }
    
    @Test
    fun receivingSwingEventUpdatesLatestSwing() = runTest {
        // Given
        viewModel.onEvent(MeasureEvent.StartMeasuring)
        
        // When: センサーサンプル投入 → スイング検出
        val swingEvent = SwingEvent(
            id = 1, sessionId = 1,
            startedAtMicros = 1000000, endedAtMicros = 2000000,
            wPerpMaxRadPerSec = 12.5f, tipSpeedMetersPerSec = 10.5f,
            impactAngleRad = null, sampleRateHz = 200f
        )
        fakeObserveSwingEvents.emitEvent(swingEvent)
        advanceUntilIdle()
        
        // Then
        val state = viewModel.uiState.value
        assertEquals(10.5f, state.latestSwing?.tipSpeedMetersPerSec)
    }
}
```

---

### 3.4 E2E テスト

**テスト方針**:
- 重要なユーザーシナリオのみ（5件程度）
- 実機 + 実際の WT9011DCL デバイス
- 実行頻度: リリース前のみ（コスト大）

**テストシナリオ**:
1. **初回起動 → 権限許可 → スキャン → 接続 → 計測**
2. **計測中 → スイング検出 → 履歴保存 → 履歴画面で確認**
3. **設定画面 → バットプロファイル作成 → 計測で使用**
4. **計測中 → アプリ切替 → 復帰 → 再計測**
5. **履歴画面 → イベント詳細 → グラフ表示**

#### 例: E2E テスト（シナリオ1）

```kotlin
// androidTest/e2e/FirstTimeUserFlowTest.kt
@RunWith(AndroidJUnit4::class)
@LargeTest
class FirstTimeUserFlowTest {
    
    @get:Rule
    val activityRule = ActivityScenarioRule(MainActivity::class.java)
    
    @Test
    @RequiresDevice
    fun firstTimeUserCanConnectAndMeasure() {
        // 1. 権限ダイアログ → 許可
        onView(withText("Bluetooth権限が必要です")).check(matches(isDisplayed()))
        onView(withId(R.id.btn_grant_permission)).perform(click())
        // システムダイアログ → 許可（UiAutomator使用）
        val device = UiDevice.getInstance(InstrumentationRegistry.getInstrumentation())
        val allowButton = device.findObject(UiSelector().text("許可"))
        if (allowButton.exists()) allowButton.click()
        
        // 2. スキャン開始
        onView(withId(R.id.btn_start_scan)).perform(click())
        Thread.sleep(5000)  // スキャン待機
        
        // 3. デバイスリストから WT9011DCL を選択
        onView(withText(containsString("WT"))).perform(click())
        Thread.sleep(3000)  // 接続待機
        
        // 4. 計測画面へ遷移確認
        onView(withId(R.id.measure_screen)).check(matches(isDisplayed()))
        
        // 5. 計測開始
        onView(withId(R.id.btn_start_measuring)).perform(click())
        
        // 6. スイング実施（手動）→ 速度表示確認
        Thread.sleep(10000)  // ユーザーがバットを振る時間
        onView(withId(R.id.tv_latest_swing_speed)).check(matches(not(withText("---"))))
    }
}
```

---

## 4. Fake実装

### 4.1 Fake Port 実装パターン

すべての Port に対して Fake 実装を用意し、単体テストで使用する。

#### 例: FakeSensorStreamPort

```kotlin
// test/domain/port/FakeSensorStreamPort.kt
class FakeSensorStreamPort : SensorStreamPort {
    
    private val _frames = MutableSharedFlow<SensorSample>()
    override val frames: Flow<SensorSample> = _frames.asSharedFlow()
    
    var isStreaming = false
        private set
    
    suspend fun emitSample(sample: SensorSample) {
        _frames.emit(sample)
    }
    
    override suspend fun startStreaming() {
        isStreaming = true
    }
    
    override suspend fun stopStreaming() {
        isStreaming = false
    }
}
```

#### 例: FakeBatProfileRepository

```kotlin
// test/data/repository/FakeBatProfileRepository.kt
class FakeBatProfileRepository : BatProfileRepository {
    
    val savedProfiles = mutableListOf<BatProfile>()
    var shouldThrowError = false
    
    override suspend fun save(profile: BatProfile): Long {
        if (shouldThrowError) throw IOException("Fake error")
        
        val newProfile = profile.copy(id = savedProfiles.size.toLong() + 1)
        savedProfiles.add(newProfile)
        return newProfile.id!!
    }
    
    override suspend fun list(userId: Long): List<BatProfile> {
        return savedProfiles.filter { it.userId == userId }
    }
    
    override suspend fun getActiveProfile(userId: Long): BatProfile {
        return savedProfiles.firstOrNull { it.userId == userId }
            ?: throw NoSuchElementException("No profile for user $userId")
    }
    
    override suspend fun delete(profileId: Long) {
        savedProfiles.removeIf { it.id == profileId }
    }
}
```

---

## 5. パフォーマンステスト

### 5.1 BLE ストリーム性能

**目標**:
- 200 Hz でドロップ率 < 1%
- レイテンシ < 50ms（受信→UI反映）

**測定方法**:
```kotlin
@Test
@RequiresDevice
fun measureBleStreamPerformance() = runTest(timeout = 60.seconds) {
    // Given: 接続済み
    adapter.connect(deviceAddress)
    adapter.startStreaming()
    
    // When: 10秒間サンプル収集
    val startTime = System.currentTimeMillis()
    val samples = mutableListOf<SensorSample>()
    val job = launch {
        adapter.frames.collect { sample ->
            samples.add(sample)
            if (System.currentTimeMillis() - startTime > 10_000) cancel()
        }
    }
    job.join()
    
    // Then: 200 Hz × 10s = 2000 サンプル期待
    val expectedSamples = 2000
    val actualSamples = samples.size
    val dropRate = (expectedSamples - actualSamples).toFloat() / expectedSamples * 100
    
    println("Expected: $expectedSamples, Actual: $actualSamples, Drop rate: $dropRate%")
    assertTrue(dropRate < 1f, "Drop rate should be < 1%")
}
```

### 5.2 DB 書き込み性能

**目標**:
- 1 SwingEvent + 400 RawSample の保存 < 100ms

**測定方法**:
```kotlin
@Test
fun measureDatabaseWritePerformance() = runTest {
    // Given
    val event = SwingEvent(/* ... */)
    val samples = List(400) { index ->
        SwingRawSample(
            eventId = 1,
            tRelMicros = index * 5000,  // 5ms間隔
            accel = Vec3(0f, 0f, 9.8f),
            gyroRadPerSec = Vec3(0f, 0f, 0f)
        )
    }
    
    // When
    val startTime = System.nanoTime()
    val eventId = eventRepository.save(event)
    rawSampleRepository.saveEventSamples(eventId, samples)
    val durationMs = (System.nanoTime() - startTime) / 1_000_000
    
    // Then
    println("DB write time: ${durationMs}ms")
    assertTrue(durationMs < 100, "Should complete within 100ms")
}
```

---

## 6. テストデータ管理

### 6.1 テストフィクスチャ

```kotlin
// test/fixtures/TestData.kt
object TestData {
    
    fun createSensorSample(
        timestampMicros: Long = System.currentTimeMillis() * 1000L,
        accel: Vec3 = Vec3(0f, 0f, 9.8f),
        gyro: Vec3 = Vec3(0f, 0f, 0f)
    ) = SensorSample(timestampMicros, accel, gyro)
    
    fun createBatProfile(
        id: Long? = null,
        userId: Long = 1L,
        name: String = "Test Bat",
        lengthMeters: Float = 0.84f,
        dHandMeters: Float = 0.15f,
        dSweetMeters: Float = 0.78f,
        gain: Float = 1.1f
    ) = BatProfile(id, userId, name, lengthMeters, dHandMeters, dSweetMeters, gain)
    
    fun createSwingEvent(
        id: Long? = null,
        sessionId: Long = 1L,
        startedAtMicros: Long = 1000000L,
        endedAtMicros: Long = 2000000L,
        wPerpMaxRadPerSec: Float = 12.5f,
        tipSpeedMetersPerSec: Float = 10.5f
    ) = SwingEvent(id, sessionId, startedAtMicros, endedAtMicros, wPerpMaxRadPerSec, tipSpeedMetersPerSec, null, 200f)
}
```

### 6.2 テスト用DB

```kotlin
// androidTest/TestDatabase.kt
@Before
fun createDb() {
    val context = ApplicationProvider.getApplicationContext<Context>()
    database = Room.inMemoryDatabaseBuilder(context, AppDatabase::class.java)
        .allowMainThreadQueries()
        .build()
}

@After
fun closeDb() {
    database.close()
}
```

---

## 7. CI/CD 統合

### 7.1 GitHub Actions ワークフロー例

```yaml
name: Android CI

on: [push, pull_request]

jobs:
  unit-test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - name: Set up JDK 11
        uses: actions/setup-java@v3
        with:
          java-version: '11'
      - name: Run unit tests
        run: ./gradlew test --stacktrace
      - name: Upload test results
        uses: actions/upload-artifact@v3
        with:
          name: test-results
          path: app/build/test-results/
  
  instrumented-test:
    runs-on: macos-latest
    steps:
      - uses: actions/checkout@v3
      - name: Set up JDK 11
        uses: actions/setup-java@v3
        with:
          java-version: '11'
      - name: Run instrumented tests
        uses: reactivecircus/android-emulator-runner@v2
        with:
          api-level: 33
          target: google_apis
          arch: x86_64
          script: ./gradlew connectedAndroidTest
```

---

## 8. カバレッジ測定

### 8.1 JaCoCo 設定

```kotlin
// app/build.gradle.kts
plugins {
    id("jacoco")
}

jacoco {
    toolVersion = "0.8.10"
}

tasks.register<JacocoReport>("jacocoTestReport") {
    dependsOn("testDebugUnitTest")
    
    reports {
        xml.required.set(true)
        html.required.set(true)
    }
    
    sourceDirectories.setFrom(files("src/main/java"))
    classDirectories.setFrom(files("build/intermediates/javac/debug/classes"))
    executionData.setFrom(files("build/jacoco/testDebugUnitTest.exec"))
}
```

### 8.2 カバレッジ目標

| レイヤー | 目標 | 優先度 |
|---------|------|--------|
| Domain（UseCase） | 90%以上 | 最優先 |
| Domain（Entity） | 80%以上 | 高 |
| Data（Repository） | 70%以上 | 中 |
| UI（ViewModel） | 70%以上 | 中 |
| UI（Compose） | 60%以上 | 低（手動テストで補完） |

---

## 9. テスト実行戦略

### 9.1 開発中

- コミット前: 単体テスト全実行（`./gradlew test`）
- プルリクエスト: CI で単体テスト + Lint

### 9.2 統合テスト

- 週次: 実機で統合テスト全実行
- リリース前: E2Eテスト + パフォーマンステスト

### 9.3 リグレッションテスト

- バグ修正時: 同一バグを検出するテストを追加
- リファクタリング後: カバレッジ維持を確認

---

## 10. まとめ

本テスト戦略により、以下を実現する：

1. **高速フィードバック**: 単体テストで即座にバグ検出
2. **高い信頼性**: 統合テストで実際の動作を検証
3. **保守性**: Fake 実装で外部依存を排除
4. **継続的改善**: CI/CD でリグレッション防止

**重要な原則**:
- Domain 層は 90% カバレッジ必須（ビジネスロジック保護）
- BLE/DB は実機統合テストで検証
- E2E は最小限（コスト大）

---

**最終更新**: 2025-11-04
**バージョン**: 1.0.0

