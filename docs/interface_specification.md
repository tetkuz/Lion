# API/インターフェース仕様詳細

本ドキュメントは、Lion アプリの全 Port、Repository、UseCase、ViewModel インターフェースの詳細仕様を定義する。実装時の契約（引数、戻り値、例外、前提条件、事後条件）を明示し、実装者・テスト担当者の共通理解を促進する。

**対象読者**: 実装者、テスト担当者、レビュアー

**参照ドキュメント**:
- `docs/architecture.md` - アーキテクチャ設計
- `docs/requirements_specification.md` - 要件定義
- `docs/coding_guidelines.md` - コーディング規約

---

## 1. Domain層 - Port インターフェース

### 1.1 SensorStreamPort

**責務**: BLE デバイスから加速度・角速度をリアルタイムでストリーミング配信する

**パッケージ**: `com.scrap.lion.domain.port`

```kotlin
package com.scrap.lion.domain.port

import com.scrap.lion.domain.entity.SensorSample
import kotlinx.coroutines.flow.Flow

/**
 * センサーストリーム Port
 * BLE デバイスから加速度・角速度をリアルタイム配信する
 * 
 * 実装者: Wt9011BleAdapter (data/ble/)
 */
interface SensorStreamPort {
    
    /**
     * センサーフレームのストリーム
     * 
     * - 周波数: 200 Hz 目安（WT9011DCL設定依存）
     * - 単位: accel [m/s²], gyro [rad/s], timestamp [μs]
     * - バックプレッシャ: DROP_OLDEST（最新値優先）
     * - ライフサイクル: startStreaming() 〜 stopStreaming() 間のみ emit
     */
    val frames: Flow<SensorSample>
    
    /**
     * ストリーミング開始
     * 
     * @throws BleError.BluetoothDisabled Bluetooth がオフ
     * @throws BleError.PermissionDenied 権限不足
     * @throws BleError.ConnectionFailed 未接続または接続失敗
     * @throws BleError.ServiceNotFound GATT サービス未発見
     * @throws BleError.CharacteristicNotFound Notify Characteristic 未発見
     * 
     * 前提条件:
     * - connect() 成功済み（isConnected == true）
     * - GATT サービス発見済み
     * 
     * 事後条件:
     * - frames に SensorSample が emit される（200 Hz目安）
     */
    suspend fun startStreaming()
    
    /**
     * ストリーミング停止
     * 
     * 前提条件: なし（冪等）
     * 
     * 事後条件:
     * - frames への emit が停止
     * - BLE 接続は維持（再度 startStreaming() 可能）
     */
    suspend fun stopStreaming()
}
```

---

### 1.2 ConnectionStatePort

**責務**: BLE デバイスのスキャン・接続・状態管理

**パッケージ**: `com.scrap.lion.domain.port`

```kotlin
package com.scrap.lion.domain.port

import com.scrap.lion.domain.entity.DeviceItem
import kotlinx.coroutines.flow.StateFlow

/**
 * 接続状態管理 Port
 * BLE デバイスのスキャン・接続・状態監視を提供
 * 
 * 実装者: Wt9011BleAdapter (data/ble/)
 */
interface ConnectionStatePort {
    
    /**
     * スキャン中フラグ
     * - true: startScan() 実行中
     * - false: stopScan() 済みまたは未開始
     */
    val isScanning: StateFlow<Boolean>
    
    /**
     * 接続済みフラグ
     * - true: connect() 成功、GATT 接続確立
     * - false: 未接続または切断
     */
    val isConnected: StateFlow<Boolean>
    
    /**
     * スキャン済みデバイスリスト
     * 
     * - 更新頻度: 1〜2秒毎
     * - タイムアウト: 5秒以上検出されないデバイスは削除
     * - 並び順: RSSI 降順（信号強度が強い順）
     */
    val scannedDevices: StateFlow<List<DeviceItem>>
    
    /**
     * BLE スキャン開始
     * 
     * @throws BleError.BluetoothDisabled Bluetooth がオフ
     * @throws BleError.PermissionDenied BLUETOOTH_SCAN 権限不足
     * @throws BleError.ScanFailed スキャン開始失敗（errorCode付き）
     * 
     * 前提条件:
     * - Bluetooth 有効
     * - BLUETOOTH_SCAN 権限許可済み
     * 
     * 事後条件:
     * - isScanning == true
     * - scannedDevices が更新される
     */
    suspend fun startScan()
    
    /**
     * BLE スキャン停止
     * 
     * 前提条件: なし（冪等）
     * 
     * 事後条件:
     * - isScanning == false
     * - scannedDevices は保持（クリアしない）
     */
    suspend fun stopScan()
    
    /**
     * BLE デバイスに接続
     * 
     * @param address BLE MAC アドレス（例: "AA:BB:CC:DD:EE:FF"）
     * @throws BleError.BluetoothDisabled Bluetooth がオフ
     * @throws BleError.PermissionDenied BLUETOOTH_CONNECT 権限不足
     * @throws BleError.ConnectionFailed 接続失敗（タイムアウト・デバイス不在等）
     * @throws BleError.Timeout 10秒以内に接続完了しない
     * 
     * 前提条件:
     * - Bluetooth 有効
     * - BLUETOOTH_CONNECT 権限許可済み
     * - address が有効な MAC アドレス形式
     * 
     * 事後条件:
     * - isConnected == true
     * - GATT サービス発見済み
     * - SensorStreamPort.startStreaming() 呼び出し可能
     */
    suspend fun connect(address: String)
    
    /**
     * BLE デバイスから切断
     * 
     * 前提条件: なし（冪等）
     * 
     * 事後条件:
     * - isConnected == false
     * - ストリーミング停止
     * - GATT リソース解放
     */
    suspend fun disconnect()
}
```

---

### 1.3 SwingDetectorPort

**責務**: センサーサンプルからスイング開始・終了を検出し、SwingEvent を生成

**パッケージ**: `com.scrap.lion.domain.port`

```kotlin
package com.scrap.lion.domain.port

import com.scrap.lion.domain.entity.SensorSample
import com.scrap.lion.domain.entity.SwingEvent
import com.scrap.lion.domain.entity.SwingSettings
import com.scrap.lion.domain.entity.BatProfile

/**
 * スイング検出 Port
 * 加速度・角速度から開始/終了を判定し、SwingEvent を生成
 * 
 * 実装者: SwingDetectorImpl (domain/detector/)
 */
interface SwingDetectorPort {
    
    /**
     * 検出器をリセット（新規セッション開始時）
     * 
     * 事後条件:
     * - 内部状態初期化（スイング中フラグ、バッファ、refractory タイマー）
     */
    fun reset()
    
    /**
     * 設定とバットプロファイルを更新
     * 
     * @param settings スイング検出閾値・タイミングパラメータ
     * @param bat バット幾何情報（R = d_sweet - d_hand, gain）
     * 
     * 前提条件:
     * - settings の各閾値が正の値
     * - bat.radiusMeters > 0
     * 
     * 事後条件:
     * - 以降の process() で新設定を使用
     */
    fun updateSettings(settings: SwingSettings, bat: BatProfile)
    
    /**
     * センサーサンプルを処理し、確定した SwingEvent を返す
     * 
     * @param sample 加速度・角速度・時刻
     * @return 確定した SwingEvent リスト（通常は 0件 または 1件）
     * 
     * 動作:
     * 1. 前処理: ローパスフィルタ、重力除去、スムージング
     * 2. 開始判定: A(t) > startThresholdA を T_rise 継続
     * 3. 終了判定: A(t) < endThresholdA かつ W⊥(t) < endThresholdW を T_fall 継続
     * 4. 集計: W⊥_max, v_tip = W⊥_max × R × gain
     * 
     * 前提条件:
     * - updateSettings() 実行済み
     * - sample.timestampMicros が単調増加
     * 
     * 事後条件:
     * - スイング終了時に1件の SwingEvent を返す
     * - refractory_period 内は新規検出抑止
     */
    fun process(sample: SensorSample): List<SwingEvent>
}
```

---

### 1.4 SessionRepository

**責務**: 計測セッションの開始・終了・監視

**パッケージ**: `com.scrap.lion.domain.port`

```kotlin
package com.scrap.lion.domain.port

import kotlinx.coroutines.flow.StateFlow

/**
 * セッション管理 Repository Port
 * 
 * 実装者: SessionRepositoryImpl (data/repository/)
 */
interface SessionRepository {
    
    /**
     * セッション開始
     * 
     * @param userId ユーザーID
     * @param batProfileId バットプロファイルID
     * @param note メモ（任意）
     * @return 生成されたセッションID
     * @throws IllegalArgumentException userId または batProfileId が不正
     * @throws DatabaseError.InsertFailed DB書き込み失敗
     * 
     * 前提条件:
     * - userId が存在する
     * - batProfileId が存在する
     * 
     * 事後条件:
     * - swing_sessions テーブルに新規レコード追加
     * - ended_at は NULL
     * - observeActiveSession() が新 sessionId を emit
     */
    suspend fun startSession(userId: Long, batProfileId: Long, note: String? = null): Long
    
    /**
     * セッション終了
     * 
     * @param sessionId 終了対象のセッションID
     * @param endedAtMicros 終了時刻 [μs]
     * @throws NoSuchElementException sessionId が存在しない
     * @throws DatabaseError.InsertFailed DB更新失敗
     * 
     * 前提条件:
     * - sessionId が存在する
     * - endedAtMicros >= started_at
     * 
     * 事後条件:
     * - swing_sessions.ended_at が更新
     * - observeActiveSession() が null を emit
     */
    suspend fun endSession(sessionId: Long, endedAtMicros: Long)
    
    /**
     * アクティブセッションの監視
     * 
     * @return 現在進行中のセッションID（終了済みまたは未開始なら null）
     * 
     * 動作:
     * - ended_at が NULL のセッションを監視
     * - 複数存在する場合は最新（started_at 降順）
     */
    fun observeActiveSession(): StateFlow<Long?>
}
```

---

### 1.5 SwingEventRepository

**責務**: SwingEvent の保存・取得・監視

**パッケージ**: `com.scrap.lion.domain.port`

```kotlin
package com.scrap.lion.domain.port

import com.scrap.lion.domain.entity.SwingEvent
import kotlinx.coroutines.flow.Flow

/**
 * SwingEvent Repository Port
 * 
 * 実装者: SwingEventRepositoryImpl (data/repository/)
 */
interface SwingEventRepository {
    
    /**
     * SwingEvent を保存
     * 
     * @param event 保存対象のイベント（id は null）
     * @return 生成されたイベントID
     * @throws IllegalArgumentException event の不変条件違反
     * @throws DatabaseError.InsertFailed DB書き込み失敗
     * 
     * 前提条件:
     * - event.sessionId が存在する
     * - event.startedAtMicros <= endedAtMicros
     * 
     * 事後条件:
     * - swing_events テーブルに新規レコード追加
     * - listBySession() で取得可能
     */
    suspend fun save(event: SwingEvent): Long
    
    /**
     * セッション内のイベントリストを監視
     * 
     * @param sessionId 対象セッションID
     * @return SwingEvent のリスト（started_at 昇順）
     * 
     * 動作:
     * - DB の swing_events テーブルを監視（Room の Flow）
     * - 新規 save() で自動的に emit
     */
    fun listBySession(sessionId: Long): Flow<List<SwingEvent>>
}
```

---

### 1.6 RawSampleRepository

**責務**: イベント期間内の Raw センサーサンプルを保存・取得

**パッケージ**: `com.scrap.lion.domain.port`

```kotlin
package com.scrap.lion.domain.port

import com.scrap.lion.domain.entity.SwingRawSample

/**
 * Raw サンプル Repository Port
 * 
 * 実装者: RawSampleRepositoryImpl (data/repository/)
 */
interface RawSampleRepository {
    
    /**
     * イベント期間内の Raw サンプルを一括保存
     * 
     * @param eventId 親イベントID
     * @param samples Raw サンプルリスト（t_rel_us 昇順推奨）
     * @throws IllegalArgumentException eventId が存在しない
     * @throws DatabaseError.InsertFailed DB書き込み失敗
     * 
     * 前提条件:
     * - eventId が存在する
     * - samples の各 t_rel_us が 0 以上
     * - samples の各 t_rel_us が (event.ended - event.started) * 1000 以下
     * 
     * 事後条件:
     * - swing_raw_samples テーブルに全サンプル追加
     * - listByEvent() で取得可能
     */
    suspend fun saveEventSamples(eventId: Long, samples: List<SwingRawSample>)
    
    /**
     * イベント内の Raw サンプルを取得
     * 
     * @param eventId 対象イベントID
     * @return Raw サンプルリスト（t_rel_us 昇順）
     * 
     * 用途: EventDetailScreen でのグラフ表示
     */
    suspend fun listByEvent(eventId: Long): List<SwingRawSample>
}
```

---

### 1.7 BatProfileRepository

**責務**: バットプロファイルの CRUD 操作

**パッケージ**: `com.scrap.lion.domain.port`

```kotlin
package com.scrap.lion.domain.port

import com.scrap.lion.domain.entity.BatProfile

/**
 * BatProfile Repository Port
 * 
 * 実装者: BatProfileRepositoryImpl (data/repository/)
 */
interface BatProfileRepository {
    
    /**
     * ユーザーのアクティブなプロファイルを取得
     * 
     * @param userId ユーザーID
     * @return 最後に使用したプロファイル
     * @throws NoSuchElementException プロファイルが存在しない
     * 
     * 動作:
     * - 最新のセッション（swing_sessions.started_at 降順）で使用したプロファイル
     * - セッション未実施の場合は created_at 降順で最新
     */
    suspend fun getActiveProfile(userId: Long): BatProfile
    
    /**
     * プロファイルを保存
     * 
     * @param profile 保存対象（id == null なら新規作成）
     * @return 生成または更新されたプロファイルID
     * @throws IllegalArgumentException profile の不変条件違反
     * @throws DatabaseError.InsertFailed DB書き込み失敗
     * 
     * 前提条件:
     * - profile.userId が存在する
     * - profile.lengthMeters > 0
     * - profile.radiusMeters > 0（d_sweet > d_hand）
     * - profile.gain > 0
     * 
     * 事後条件:
     * - bat_profiles テーブルに新規追加または更新
     * - list() で取得可能
     */
    suspend fun save(profile: BatProfile): Long
    
    /**
     * ユーザーのプロファイルリストを取得
     * 
     * @param userId ユーザーID
     * @return プロファイルリスト（created_at 降順）
     */
    suspend fun list(userId: Long): List<BatProfile>
    
    /**
     * プロファイルを削除
     * 
     * @param profileId 削除対象ID
     * @throws IllegalStateException プロファイルが使用中のセッションに紐付いている
     * 
     * 前提条件:
     * - profileId が存在する
     * - 未終了のセッションで使用されていない
     * 
     * 事後条件:
     * - bat_profiles テーブルから削除
     * - list() で取得不可
     */
    suspend fun delete(profileId: Long)
}
```

---

### 1.8 SettingsRepository

**責務**: スイング検出設定の読み書き（DataStore 管理）

**パッケージ**: `com.scrap.lion.domain.port`

```kotlin
package com.scrap.lion.domain.port

import com.scrap.lion.domain.entity.SwingSettings
import kotlinx.coroutines.flow.Flow

/**
 * スイング検出設定 Repository Port
 * 
 * 実装者: SettingsRepositoryImpl (data/settings/)
 * ストレージ: DataStore（Protocol Buffer）
 */
interface SettingsRepository {
    
    /**
     * スイング検出設定の監視
     * 
     * @return SwingSettings のストリーム（初期値はデフォルト設定）
     * 
     * デフォルト設定:
     * - startThresholdADyn: 3.5 m/s²
     * - endThresholdADyn: 1.8 m/s²
     * - endThresholdWPerp: 3.0 rad/s
     * - tRiseMillis: 25 ms
     * - tFallMillis: 50 ms
     * - refractoryPeriodMillis: 200 ms
     */
    fun loadSwingSettings(): Flow<SwingSettings>
    
    /**
     * スイング検出設定を更新
     * 
     * @param value 新しい設定値
     * @throws IllegalArgumentException 不正な閾値（負の値等）
     * 
     * 前提条件:
     * - value の各閾値が正の値
     * 
     * 事後条件:
     * - DataStore に永続化
     * - loadSwingSettings() が新値を emit
     */
    suspend fun updateSwingSettings(value: SwingSettings)
}
```

---

## 2. Domain層 - UseCase インターフェース

### 2.1 StartSession

**責務**: 計測セッションを開始

**パッケージ**: `com.scrap.lion.domain.usecase`

```kotlin
package com.scrap.lion.domain.usecase

import com.scrap.lion.domain.port.SessionRepository
import javax.inject.Inject

/**
 * セッション開始 UseCase
 */
class StartSession @Inject constructor(
    private val repository: SessionRepository
) {
    /**
     * @param userId ユーザーID
     * @param batProfileId バットプロファイルID
     * @param note メモ（任意）
     * @return Result<Long> 成功時は sessionId、失敗時は例外
     */
    suspend operator fun invoke(
        userId: Long,
        batProfileId: Long,
        note: String? = null
    ): Result<Long> = runCatching {
        require(userId > 0) { "userId must be positive" }
        require(batProfileId > 0) { "batProfileId must be positive" }
        repository.startSession(userId, batProfileId, note)
    }
}
```

---

### 2.2 EndSession

**責務**: 計測セッションを終了

**パッケージ**: `com.scrap.lion.domain.usecase`

```kotlin
package com.scrap.lion.domain.usecase

import com.scrap.lion.domain.port.SessionRepository
import javax.inject.Inject

/**
 * セッション終了 UseCase
 */
class EndSession @Inject constructor(
    private val repository: SessionRepository,
    private val timeSource: TimeSource
) {
    /**
     * @param sessionId 終了対象のセッションID
     * @return Result<Unit> 成功・失敗
     */
    suspend operator fun invoke(sessionId: Long): Result<Unit> = runCatching {
        require(sessionId > 0) { "sessionId must be positive" }
        val now = timeSource.currentTimeMicros()
        repository.endSession(sessionId, now)
    }
}
```

---

### 2.3 ObserveSwingEvents

**責務**: センサーストリームを監視し、スイング検出→保存→UI通知

**パッケージ**: `com.scrap.lion.domain.usecase`

```kotlin
package com.scrap.lion.domain.usecase

import com.scrap.lion.domain.entity.SwingEvent
import com.scrap.lion.domain.entity.SwingRawSample
import com.scrap.lion.domain.port.*
import kotlinx.coroutines.flow.Flow
import kotlinx.coroutines.flow.flow
import javax.inject.Inject

/**
 * スイングイベント監視 UseCase
 * 
 * データフロー:
 * SensorStreamPort.frames
 *   → SwingDetectorPort.process()
 *   → SwingEvent 確定
 *   → SwingEventRepository.save()
 *   → RawSampleRepository.saveEventSamples()
 *   → UI へ emit
 */
class ObserveSwingEvents @Inject constructor(
    private val sensorStreamPort: SensorStreamPort,
    private val swingDetectorPort: SwingDetectorPort,
    private val swingEventRepository: SwingEventRepository,
    private val rawSampleRepository: RawSampleRepository
) {
    /**
     * @param sessionId 対象セッションID
     * @return Flow<SwingEvent> 確定したスイングイベント（保存済み）
     * 
     * 動作:
     * 1. sensorStreamPort.frames を collect
     * 2. 各サンプルを swingDetectorPort.process()
     * 3. 確定イベントを swingEventRepository.save()
     * 4. イベント期間の Raw サンプルを rawSampleRepository.saveEventSamples()
     * 5. UI へ emit
     * 
     * 例外:
     * - DatabaseError: DB保存失敗時
     */
    operator fun invoke(sessionId: Long): Flow<SwingEvent> = flow {
        val rawBuffer = mutableListOf<SwingRawSample>()
        var swingStartTime = 0L
        
        sensorStreamPort.frames.collect { sample ->
            val events = swingDetectorPort.process(sample)
            
            // スイング中は Raw バッファに追加
            if (swingDetectorPort.isSwinging) {
                if (swingStartTime == 0L) swingStartTime = sample.timestampMicros
                
                rawBuffer.add(SwingRawSample(
                    eventId = 0,  // 後で上書き
                    tRelMicros = ((sample.timestampMicros - swingStartTime) / 1000).toInt(),
                    accel = sample.accel,
                    gyroRadPerSec = sample.gyroRadPerSec
                ))
            }
            
            // スイング確定時
            events.forEach { event ->
                // DB 保存
                val eventId = swingEventRepository.save(event.copy(sessionId = sessionId))
                rawSampleRepository.saveEventSamples(
                    eventId,
                    rawBuffer.map { it.copy(eventId = eventId) }
                )
                
                // UI へ通知
                emit(event.copy(id = eventId, sessionId = sessionId))
                
                // バッファクリア
                rawBuffer.clear()
                swingStartTime = 0L
            }
        }
    }
}
```

---

### 2.4 SaveBatProfile

**責務**: バットプロファイルを保存（バリデーション含む）

**パッケージ**: `com.scrap.lion.domain.usecase`

```kotlin
package com.scrap.lion.domain.usecase

import com.scrap.lion.domain.entity.BatProfile
import com.scrap.lion.domain.port.BatProfileRepository
import javax.inject.Inject

/**
 * バットプロファイル保存 UseCase
 */
class SaveBatProfile @Inject constructor(
    private val repository: BatProfileRepository
) {
    /**
     * @param profile 保存対象（id == null なら新規作成）
     * @return Result<Long> 成功時は profileId、失敗時は例外
     * 
     * バリデーション:
     * - lengthMeters > 0
     * - dHandMeters >= 0
     * - dSweetMeters > dHandMeters（radiusMeters > 0）
     * - gain > 0
     */
    suspend operator fun invoke(profile: BatProfile): Result<Long> = runCatching {
        // バリデーション
        require(profile.lengthMeters > 0) { "lengthMeters must be positive" }
        require(profile.dHandMeters >= 0) { "dHandMeters must be non-negative" }
        require(profile.dSweetMeters > profile.dHandMeters) {
            "dSweetMeters must be greater than dHandMeters"
        }
        require(profile.gain > 0) { "gain must be positive" }
        
        // 保存
        repository.save(profile)
    }
}
```

---

### 2.5 LoadBatProfiles

**責務**: ユーザーのバットプロファイルリストを取得

**パッケージ**: `com.scrap.lion.domain.usecase`

```kotlin
package com.scrap.lion.domain.usecase

import com.scrap.lion.domain.entity.BatProfile
import com.scrap.lion.domain.port.BatProfileRepository
import javax.inject.Inject

/**
 * バットプロファイルリスト取得 UseCase
 */
class LoadBatProfiles @Inject constructor(
    private val repository: BatProfileRepository
) {
    /**
     * @param userId ユーザーID
     * @return Result<List<BatProfile>> 成功時はリスト、失敗時は例外
     */
    suspend operator fun invoke(userId: Long): Result<List<BatProfile>> = runCatching {
        require(userId > 0) { "userId must be positive" }
        repository.list(userId)
    }
}
```

---

## 3. UI層 - ViewModel インターフェース

### 3.1 ConnectViewModel

**責務**: ConnectScreen の状態管理・イベントハンドリング

**パッケージ**: `com.scrap.lion.ui.feature.connect`

```kotlin
package com.scrap.lion.ui.feature.connect

import androidx.lifecycle.ViewModel
import androidx.lifecycle.viewModelScope
import com.scrap.lion.domain.port.ConnectionStatePort
import dagger.hilt.android.lifecycle.HiltViewModel
import kotlinx.coroutines.flow.*
import kotlinx.coroutines.launch
import javax.inject.Inject

/**
 * 接続画面 ViewModel
 */
@HiltViewModel
class ConnectViewModel @Inject constructor(
    private val connectionStatePort: ConnectionStatePort,
    private val permissionChecker: PermissionChecker
) : ViewModel() {
    
    // UiState
    private val _uiState = MutableStateFlow(ConnectUiState())
    val uiState: StateFlow<ConnectUiState> = combine(
        _uiState,
        connectionStatePort.isScanning,
        connectionStatePort.scannedDevices,
        connectionStatePort.isConnected
    ) { state, isScanning, devices, isConnected ->
        state.copy(
            isScanning = isScanning,
            devices = devices,
            isConnected = isConnected
        )
    }.stateIn(viewModelScope, SharingStarted.Lazily, ConnectUiState())
    
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
            
            try {
                connectionStatePort.startScan()
            } catch (e: Exception) {
                _uiState.update { it.copy(error = e.message) }
            }
        }
    }
    
    private fun handleStopScan() {
        viewModelScope.launch {
            connectionStatePort.stopScan()
        }
    }
    
    private fun handleConnect(address: String) {
        viewModelScope.launch {
            _uiState.update { it.copy(connecting = true) }
            try {
                connectionStatePort.connect(address)
                // 接続成功 → MeasureScreen へ遷移（UI側で処理）
            } catch (e: Exception) {
                _uiState.update { it.copy(error = e.message, connecting = false) }
            }
        }
    }
    
    private fun handleDisconnect() {
        viewModelScope.launch {
            connectionStatePort.disconnect()
        }
    }
    
    private fun handleRequestPermission() {
        _uiState.update { it.copy(showPermissionDialog = true) }
    }
}

/**
 * 接続画面の UI 状態
 */
data class ConnectUiState(
    val devices: List<DeviceItem> = emptyList(),
    val isScanning: Boolean = false,
    val isConnected: Boolean = false,
    val connecting: Boolean = false,
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

---

### 3.2 MeasureViewModel

**責務**: MeasureScreen の状態管理・計測制御

**パッケージ**: `com.scrap.lion.ui.feature.measure`

```kotlin
package com.scrap.lion.ui.feature.measure

import androidx.lifecycle.SavedStateHandle
import androidx.lifecycle.ViewModel
import androidx.lifecycle.viewModelScope
import com.scrap.lion.domain.entity.SwingEvent
import com.scrap.lion.domain.port.SensorStreamPort
import com.scrap.lion.domain.usecase.*
import dagger.hilt.android.lifecycle.HiltViewModel
import kotlinx.coroutines.flow.*
import kotlinx.coroutines.launch
import javax.inject.Inject

/**
 * 計測画面 ViewModel
 */
@HiltViewModel
class MeasureViewModel @Inject constructor(
    private val sensorStreamPort: SensorStreamPort,
    private val startSession: StartSession,
    private val endSession: EndSession,
    private val observeSwingEvents: ObserveSwingEvents,
    private val savedStateHandle: SavedStateHandle
) : ViewModel() {
    
    // UiState
    private val _uiState = MutableStateFlow(MeasureUiState())
    val uiState: StateFlow<MeasureUiState> = _uiState.asStateFlow()
    
    // セッションID（復帰用）
    private var sessionId: Long?
        get() = savedStateHandle["sessionId"]
        set(value) { savedStateHandle["sessionId"] = value }
    
    // イベントハンドラ
    fun onEvent(event: MeasureEvent) {
        when (event) {
            is MeasureEvent.StartSession -> handleStartSession(event.userId, event.batProfileId)
            MeasureEvent.EndSession -> handleEndSession()
            MeasureEvent.StartMeasuring -> handleStartMeasuring()
            MeasureEvent.StopMeasuring -> handleStopMeasuring()
        }
    }
    
    private fun handleStartSession(userId: Long, batProfileId: Long) {
        viewModelScope.launch {
            val result = startSession(userId, batProfileId)
            result.onSuccess { id ->
                sessionId = id
                _uiState.update { it.copy(sessionId = id) }
            }.onFailure { error ->
                _uiState.update { it.copy(error = error.message) }
            }
        }
    }
    
    private fun handleEndSession() {
        viewModelScope.launch {
            val id = sessionId ?: return@launch
            val result = endSession(id)
            result.onSuccess {
                sessionId = null
                _uiState.update { it.copy(sessionId = null) }
            }.onFailure { error ->
                _uiState.update { it.copy(error = error.message) }
            }
        }
    }
    
    private fun handleStartMeasuring() {
        val id = sessionId ?: return
        
        viewModelScope.launch {
            _uiState.update { it.copy(isStreaming = true) }
            
            try {
                sensorStreamPort.startStreaming()
                
                observeSwingEvents(id).collect { event ->
                    _uiState.update { state ->
                        state.copy(
                            latestSwing = event,
                            swingCount = state.swingCount + 1
                        )
                    }
                }
            } catch (e: Exception) {
                _uiState.update { it.copy(error = e.message, isStreaming = false) }
            }
        }
    }
    
    private fun handleStopMeasuring() {
        viewModelScope.launch {
            sensorStreamPort.stopStreaming()
            _uiState.update { it.copy(isStreaming = false) }
        }
    }
}

/**
 * 計測画面の UI 状態
 */
data class MeasureUiState(
    val sessionId: Long? = null,
    val isStreaming: Boolean = false,
    val latestSwing: SwingEvent? = null,
    val swingCount: Int = 0,
    val error: String? = null
)

/**
 * 計測画面の UI イベント
 */
sealed interface MeasureEvent {
    data class StartSession(val userId: Long, val batProfileId: Long) : MeasureEvent
    object EndSession : MeasureEvent
    object StartMeasuring : MeasureEvent
    object StopMeasuring : MeasureEvent
}
```

---

### 3.3 HistoryViewModel

**責務**: HistoryScreen の状態管理・フィルタリング

**パッケージ**: `com.scrap.lion.ui.feature.history`

```kotlin
package com.scrap.lion.ui.feature.history

import androidx.lifecycle.ViewModel
import androidx.lifecycle.viewModelScope
import com.scrap.lion.domain.entity.SwingEvent
import com.scrap.lion.domain.port.SwingEventRepository
import dagger.hilt.android.lifecycle.HiltViewModel
import kotlinx.coroutines.flow.*
import javax.inject.Inject

/**
 * 履歴画面 ViewModel
 */
@HiltViewModel
class HistoryViewModel @Inject constructor(
    private val swingEventRepository: SwingEventRepository
) : ViewModel() {
    
    // UiState
    private val _filters = MutableStateFlow(HistoryFilters())
    private val _events = MutableStateFlow<List<SwingEvent>>(emptyList())
    
    val uiState: StateFlow<HistoryUiState> = combine(
        _filters,
        _events
    ) { filters, events ->
        HistoryUiState(
            events = events.filter { event ->
                filters.matches(event)
            },
            filters = filters,
            loading = false
        )
    }.stateIn(viewModelScope, SharingStarted.Lazily, HistoryUiState(loading = true))
    
    init {
        loadEvents()
    }
    
    // イベントハンドラ
    fun onEvent(event: HistoryEvent) {
        when (event) {
            is HistoryEvent.SetUserFilter -> _filters.update { it.copy(userId = event.userId) }
            is HistoryEvent.SetDateRange -> _filters.update { it.copy(fromDate = event.from, toDate = event.to) }
            is HistoryEvent.SetSpeedRange -> _filters.update { it.copy(minSpeed = event.min, maxSpeed = event.max) }
            is HistoryEvent.ChangeSort -> _filters.update { it.copy(sortOrder = event.order) }
        }
    }
    
    private fun loadEvents() {
        // 全イベント取得（実装時は sessionId 指定）
        // swingEventRepository.listBySession(sessionId).collect { ... }
    }
}

/**
 * 履歴画面の UI 状態
 */
data class HistoryUiState(
    val events: List<SwingEvent> = emptyList(),
    val filters: HistoryFilters = HistoryFilters(),
    val loading: Boolean = false,
    val error: String? = null
)

/**
 * フィルタ条件
 */
data class HistoryFilters(
    val userId: Long? = null,
    val fromDate: Long? = null,
    val toDate: Long? = null,
    val minSpeed: Float? = null,
    val maxSpeed: Float? = null,
    val sortOrder: SortOrder = SortOrder.DATE_DESC
) {
    fun matches(event: SwingEvent): Boolean {
        // フィルタ適用ロジック
        if (minSpeed != null && event.tipSpeedMetersPerSec < minSpeed) return false
        if (maxSpeed != null && event.tipSpeedMetersPerSec > maxSpeed) return false
        // ... 他のフィルタ
        return true
    }
}

enum class SortOrder {
    DATE_DESC, DATE_ASC, SPEED_DESC, SPEED_ASC
}

/**
 * 履歴画面の UI イベント
 */
sealed interface HistoryEvent {
    data class SetUserFilter(val userId: Long?) : HistoryEvent
    data class SetDateRange(val from: Long?, val to: Long?) : HistoryEvent
    data class SetSpeedRange(val min: Float?, val max: Float?) : HistoryEvent
    data class ChangeSort(val order: SortOrder) : HistoryEvent
}
```

---

### 3.4 SettingsViewModel

**責務**: SettingsScreen の状態管理・プロファイル編集

**パッケージ**: `com.scrap.lion.ui.feature.settings`

```kotlin
package com.scrap.lion.ui.feature.settings

import androidx.lifecycle.ViewModel
import androidx.lifecycle.viewModelScope
import com.scrap.lion.domain.entity.BatProfile
import com.scrap.lion.domain.usecase.*
import dagger.hilt.android.lifecycle.HiltViewModel
import kotlinx.coroutines.flow.*
import kotlinx.coroutines.launch
import javax.inject.Inject

/**
 * 設定画面 ViewModel
 */
@HiltViewModel
class SettingsViewModel @Inject constructor(
    private val loadBatProfiles: LoadBatProfiles,
    private val saveBatProfile: SaveBatProfile
) : ViewModel() {
    
    // UiState
    private val _uiState = MutableStateFlow(SettingsUiState())
    val uiState: StateFlow<SettingsUiState> = _uiState.asStateFlow()
    
    fun loadProfiles(userId: Long) {
        viewModelScope.launch {
            _uiState.update { it.copy(loading = true) }
            
            val result = loadBatProfiles(userId)
            result.onSuccess { profiles ->
                _uiState.update { it.copy(profiles = profiles, loading = false) }
            }.onFailure { error ->
                _uiState.update { it.copy(error = error.message, loading = false) }
            }
        }
    }
    
    // イベントハンドラ
    fun onEvent(event: SettingsEvent) {
        when (event) {
            is SettingsEvent.SaveProfile -> handleSaveProfile(event.profile)
            is SettingsEvent.EditProfile -> _uiState.update { it.copy(editingProfile = event.profile) }
            SettingsEvent.CancelEdit -> _uiState.update { it.copy(editingProfile = null) }
        }
    }
    
    private fun handleSaveProfile(profile: BatProfile) {
        viewModelScope.launch {
            _uiState.update { it.copy(saving = true) }
            
            val result = saveBatProfile(profile)
            result.onSuccess { profileId ->
                _uiState.update { it.copy(saving = false, editingProfile = null) }
                loadProfiles(profile.userId)
            }.onFailure { error ->
                _uiState.update { it.copy(error = error.message, saving = false) }
            }
        }
    }
}

/**
 * 設定画面の UI 状態
 */
data class SettingsUiState(
    val profiles: List<BatProfile> = emptyList(),
    val editingProfile: BatProfile? = null,
    val loading: Boolean = false,
    val saving: Boolean = false,
    val error: String? = null
)

/**
 * 設定画面の UI イベント
 */
sealed interface SettingsEvent {
    data class SaveProfile(val profile: BatProfile) : SettingsEvent
    data class EditProfile(val profile: BatProfile) : SettingsEvent
    object CancelEdit : SettingsEvent
}
```

---

## 4. エラー型階層

### 4.1 BleError

```kotlin
package com.scrap.lion.data.ble

/**
 * BLE関連エラー
 */
sealed class BleError : Exception() {
    object BluetoothDisabled : BleError() {
        override val message = "Bluetooth is disabled"
    }
    
    object PermissionDenied : BleError() {
        override val message = "Bluetooth permission denied"
    }
    
    data class ScanFailed(val errorCode: Int) : BleError() {
        override val message = "Scan failed with code: $errorCode"
    }
    
    data class ConnectionFailed(val reason: String) : BleError() {
        override val message = "Connection failed: $reason"
    }
    
    object ServiceNotFound : BleError() {
        override val message = "GATT service not found"
    }
    
    object CharacteristicNotFound : BleError() {
        override val message = "GATT characteristic not found"
    }
    
    data class GattError(val status: Int) : BleError() {
        override val message = "GATT error: status=$status"
    }
    
    object Timeout : BleError() {
        override val message = "Operation timeout"
    }
}
```

### 4.2 DatabaseError

```kotlin
package com.scrap.lion.data.local

/**
 * データベース関連エラー
 */
sealed class DatabaseError : Exception() {
    data class InsertFailed(val cause: Throwable) : DatabaseError() {
        override val message = "Insert failed: ${cause.message}"
    }
    
    data class QueryFailed(val cause: Throwable) : DatabaseError() {
        override val message = "Query failed: ${cause.message}"
    }
    
    data class UpdateFailed(val cause: Throwable) : DatabaseError() {
        override val message = "Update failed: ${cause.message}"
    }
}
```

---

## 5. 補助型

### 5.1 TimeSource

```kotlin
package com.scrap.lion.core

/**
 * 時刻取得の抽象化（テスト用）
 */
interface TimeSource {
    /**
     * 現在時刻をマイクロ秒で取得
     * @return エポックマイクロ秒 [μs]
     */
    fun currentTimeMicros(): Long
}

class SystemTimeSource : TimeSource {
    override fun currentTimeMicros(): Long = System.currentTimeMillis() * 1000L
}
```

### 5.2 DispatcherProvider

```kotlin
package com.scrap.lion.core

import kotlinx.coroutines.CoroutineDispatcher
import kotlinx.coroutines.Dispatchers

/**
 * Coroutine Dispatcher の抽象化（テスト用）
 */
interface DispatcherProvider {
    val io: CoroutineDispatcher
    val main: CoroutineDispatcher
    val default: CoroutineDispatcher
}

class DefaultDispatcherProvider : DispatcherProvider {
    override val io = Dispatchers.IO
    override val main = Dispatchers.Main
    override val default = Dispatchers.Default
}

class TestDispatcherProvider : DispatcherProvider {
    override val io = Dispatchers.Unconfined
    override val main = Dispatchers.Unconfined
    override val default = Dispatchers.Unconfined
}
```

---

## 6. まとめ

本ドキュメントで定義した全インターフェースは、以下の原則に従う：

1. **契約の明示**: 引数、戻り値、例外、前提条件、事後条件を KDoc で明記
2. **単位の明記**: すべての物理量に単位をコメント（`[m/s²]`, `[rad/s]`, `[μs]`）
3. **不変条件の保証**: Entity/Value Object は `init` ブロックで検証
4. **エラーの明確化**: 適切な例外型を使用し、エラーメッセージを明確に
5. **テスタビリティ**: すべての Port に Fake 実装を用意

実装時は本仕様を参照し、契約を遵守すること。

---

**最終更新**: 2025-11-04
**バージョン**: 1.0.0

