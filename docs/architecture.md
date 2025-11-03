# アーキテクチャ設計（Compose／Foreground優先）

本ドキュメントは、`docs/requirements_specification.md` に定義された要件を満たすための実装指針を示す。初期バージョンは Jetpack Compose を用いたフォアグラウンド計測に限定し、将来的な ForegroundService 対応や他センサー拡張の方針も併記する。

- 参照: `docs/requirements_specification.md`
- アーキテクチャ原則: MVVM + Ports/Adapters、単方向データフロー、Hilt による依存性注入、Coroutines/Flow、中核ロジックの純粋化による高いテスタビリティ

---

## 全体像（MVVM x Ports/Adapters）

- レイヤ構成
  - UI（Compose）: 宣言的UIと画面状態の表示
  - ViewModel: `UiState` と `UiEvent` の仲介、UseCase をオーケストレーション
  - Domain: UseCase、ドメインモデル、Repository Port（インターフェース）
  - Data: BLE/DB/設定などの具体実装（Adapter）

```
[UI(Compose)] → [ViewModel] → [UseCases] → [Repository Port]
                                        ↑                 ↓
                                  [BLE Adapter]     [DB Adapter]
```

- 単方向データフロー
  - 下り（ユーザー意図）: UI → ViewModel → UseCase → Port 呼び出し
  - 上り（状態/データ）: Adapter/Repository 実装 → UseCase → ViewModel(StateFlow) → UI

- 設計意図
  - デバイス依存（BLE）や I/O（DB）は Port の背後に隔離し、ドメインを純粋化
  - 時刻・スレッド・乱数・センサー入出力は抽象化してテスト容易性を確保

---

## パッケージ構成（単一モジュール内の論理分割）

`app/src/main/java/com/scrap/lion/`

- `core/` 共通ユーティリティ（`Result` 型、`DispatcherProvider`、`TimeSource` 等）
- `domain/`
  - `entity/` ドメインエンティティ・値オブジェクト
  - `port/` Repository Port・サービス Port（インターフェース）
  - `usecase/` アプリケーションユースケース
- `data/`
  - `ble/` WT9011DCL アダプタ（スキャン/接続/ストリーム）
  - `local/` Room 実装（DAO/Entity/Database）
    - Entity: User, BatProfile, SwingSession, SwingEvent, SwingRawSample
    - DAO: 各テーブルのアクセス
  - `settings/` DataStore 実装（プロトコルバッファ推奨）
    - SwingSettings（検出用閾値）のシリアライズ
  - `repository/` Port を満たす具象リポジトリ
    - `BatProfileRepository` 実装（Room使用）
    - `SwingSettingsRepository` 実装（DataStore使用）
    - その他 Session/Event/RawSample Repository
- `ui/`
  - `feature/connect/`
  - `feature/measure/`
  - `feature/history/`
  - `feature/settings/`
- `di/` Hilt Modules（BLE、Repository、UseCase、Coroutine）
- `analytics/` ログ/トレース
- `config/` しきい値・係数・ビルド時設定

---

## 主要ドメインモデルとポート

- ドメインモデル（確定）
```kotlin
// ベクトル（単位は利用側で明示）
data class Vec3(val x: Float, val y: Float, val z: Float)

// μs時刻、角速度はrad/s、加速度はm/s^2
data class SensorSample(
    val timestampMicros: Long,
    val accel: Vec3,        // m/s^2
    val gyroRadPerSec: Vec3 // rad/s
)

data class SwingSession(
    val id: Long?,
    val userId: Long,
    val batProfileId: Long,
    val startedAtMicros: Long,
    val endedAtMicros: Long?,
    val note: String?
)

data class SwingEvent(
    val id: Long?,
    val sessionId: Long,
    val startedAtMicros: Long,      // μs
    val endedAtMicros: Long,        // μs
    val wPerpMaxRadPerSec: Float,   // rad/s（Z軸除外）
    val tipSpeedMetersPerSec: Float, // m/s
    val impactAngleRad: Float?,     // Optional: インパクト付近の角度推定値
    val sampleRateHz: Float         // イベント中のサンプリングレート [Hz]
)

// イベント先頭からの相対μs
data class SwingRawSample(
    val eventId: Long,
    val tRelMicros: Int,
    val accel: Vec3,        // m/s^2
    val gyroRadPerSec: Vec3 // rad/s
)

data class BatProfile(
    val id: Long?,
    val userId: Long,
    val name: String,
    val lengthMeters: Float,
    val dHandMeters: Float,
    val dSweetMeters: Float,
    val gain: Float
) { val radiusMeters: Float get() = dSweetMeters - dHandMeters }

data class SwingSettings(
    val startThresholdADyn: Float,     // m/s^2
    val endThresholdADyn: Float,       // m/s^2
    val endThresholdWPerp: Float,      // rad/s
    val tRiseMillis: Int,
    val tFallMillis: Int,
    val refractoryPeriodMillis: Int
)

// BLE デバイス情報（スキャン結果）
data class DeviceItem(
    val address: String,               // BLE MAC address （例: "AA:BB:CC:DD:EE:FF"）
    val name: String?,                 // BLE advertisement name（null の場合もある）
    val rssi: Int,                     // 信号強度 [dBm] （-100～-30 目安）
    val txPower: Int?,                 // TX Power (Optional)
    val isConnectable: Boolean,        // 接続可能フラグ
    val lastSeenAtMillis: Long        // 最後に検出された時刻 (epoch millis)
)
```

- 不変条件
  - `SwingEvent.startedAtMicros <= endedAtMicros`
  - `0 <= SwingRawSample.tRelMicros <= (ended - started) * 1000`
  - `BatProfile.radiusMeters > 0`、`gain > 0`
  - `SwingEvent.sampleRateHz > 0`

- Ports（Repository/Service インターフェース）
```kotlin
interface SensorStreamPort {
    val frames: kotlinx.coroutines.flow.Flow<SensorSample>
    suspend fun startStreaming()
    suspend fun stopStreaming()
}

interface ConnectionStatePort {
    val isScanning: kotlinx.coroutines.flow.StateFlow<Boolean>
    val isConnected: kotlinx.coroutines.flow.StateFlow<Boolean>
}

interface SwingDetectorPort {
    fun reset()
    fun updateSettings(settings: SwingSettings, bat: BatProfile)
    /** 1サンプル入力で0..N件の確定SwingEventを返す（終了時1件想定） */
    fun process(sample: SensorSample): List<SwingEvent>
}

interface SessionRepository {
    suspend fun startSession(userId: Long, batProfileId: Long, note: String? = null): Long
    suspend fun endSession(sessionId: Long, endedAtMicros: Long)
    fun observeActiveSession(): kotlinx.coroutines.flow.StateFlow<Long?>
}

interface SwingEventRepository {
    suspend fun save(event: SwingEvent): Long
    fun listBySession(sessionId: Long): kotlinx.coroutines.flow.Flow<List<SwingEvent>>
}

interface RawSampleRepository {
    /** イベント期間内のRawを一括保存 */
    suspend fun saveEventSamples(eventId: Long, samples: List<SwingRawSample>)
}

interface SettingsRepository {
    /**
     * DataStore 管理：検出用の閾値・フィルタパラメータ
     * ユーザー単位ではなく、アプリ単位で1セット保持
     * 変更頻度低、高速アクセス必要 → DataStore適合
     */
    fun loadSwingSettings(): kotlinx.coroutines.flow.Flow<SwingSettings>
    suspend fun updateSwingSettings(value: SwingSettings)
}

interface BatProfileRepository {
    /**
     * Room DB 管理：ユーザー毎のバット幾何・補正設定
     * 複数個管理、ユーザー別に複数保持、歴史管理 → DB適合
     */
    suspend fun getActiveProfile(userId: Long): BatProfile
    suspend fun save(profile: BatProfile): Long
    suspend fun list(userId: Long): List<BatProfile>
    suspend fun delete(profileId: Long)
}
```

- UseCase（責務例）
  - `StartSession(userId, batProfileId, note?)` / `EndSession(sessionId)`
  - `StartMeasuring(sessionId)` / `StopMeasuring`: ストリーミング開始/停止、検出器リセット
  - `ObserveConnection`: 接続状態の監視
  - `ObserveSwingEvents(sessionId)`: `frames`→検出→`SwingEvent`/`Raw` を保存しUIへ通知

---

## BLEアブストラクション（WT9011DCL）

- 実装クラス例: `Wt9011BleAdapter` が `SensorStreamPort` と `ConnectionStatePort` を実装
- スキャン機能
  - `startScan()` で BLE デバイスをスキャン開始
  - 各デバイスを `DeviceItem` にマッピング（address, name, rssi, txPower, isConnectable, lastSeenAtMillis）
  - `scannedDevices: StateFlow<List<DeviceItem>>` で UI へリアルタイム配信
  - タイムアウト: 5 秒以上検出されないデバイスはリストから削除
- 接続機能
  - `connect(address: String)` で指定 MAC の接続開始
  - 接続成功後、自動で GATT サービス発見 → 購読開始
- ストリーミング機能
  - GATT 購読（200 Hz 目安、実測は可変）→ バイト列デコード → `SensorSample`
  - 単位変換: 角速度 deg/s → rad/s、加速度 g → m/s²（必要時）
  - バックプレッシャ: `buffer(capacity = Channel.CONFLATED)` で最新値優先
- 状態管理
  - 例外と切断: 再試行ポリシ、ユーザー誘導（権限/設定）
  - `SensorStreamPort.startStreaming()/stopStreaming()` を提供し明示的に制御

- 権限/設定
  - Bluetooth 有効化、位置情報権限（OSバージョンごとの要件を UI で誘導）
  - フォアグラウンド版では画面表示中のみ購読（ライフサイクルに追従）

---

## アプリケーションライフサイクル（v1 制約）

### フォアグラウンド限定ポリシー

本 v1 実装は以下の制約に基づく（詳細は `requirements_specification.md` 5.5 参照）：

- **計測の開始**: ConnectScreen 〜 MeasureScreen で明示的にユーザーが開始
- **フォアグラウンド中**: BLE ストリーミング・センサ購読・スイング検出が実行
- **一時停止イベント**
  - 画面ロック
  - 別アプリへ切替（onPause 検知）
  - ホームボタン押下
  → 自動でストリーミング停止、計測セッション一時保存
- **再開**: ユーザーがアプリに戻ると、前回の状態を復帰し再開可能

### 実装上の考慮

- **Activity/Fragment ライフサイクル**: `onPause()` で `StopMeasuring`、`onResume()` で自動復帰判定
- **ViewModel の保持**: `SavedStateHandle` で計測中のセッション情報を保存
- **バッテリー配慮**: 計測中断時、接続解放（再接続時に再スキャン）

### 将来拡張（v2以降）

- **ForegroundService**: 画面ロック後も通知チャネルで計測継続
- **WorkManager**: 再起動時の自動セッション復旧
- **バックグラウンド最適化**: 一定時間スイング非検出で自動停止

---

## 権限管理（Bluetooth・位置情報）

### 必要な権限（API 33+）

詳細は `requirements_specification.md` 6 を参照。実装の考慮点：

- **BLUETOOTH_SCAN**: BLE スキャン用（コア権限）
- **BLUETOOTH_CONNECT**: BLE 接続用（コア権限）
- **ACCESS_FINE_LOCATION**: 位置情報（一部デバイスの BLE スキャン補助）

### 実装パターン

**Hilt Module で Permission Check UseCase を提供**
```kotlin
@Module
@InstallIn(SingletonComponent::class)
object PermissionModule {
    @Provides
    @Singleton
    fun providePermissionChecker(context: Context): PermissionChecker {
        return AndroidPermissionChecker(context)
    }
}

interface PermissionChecker {
    suspend fun hasBluetoothPermissions(): Boolean
    suspend fun requestBluetoothPermissions(): Boolean  // Activity 結果返却
}
```

**ConnectViewModel で権限ガード**
```kotlin
// ユーザーが「スキャン開始」をタップ
onEvent(StartScan) {
    if (!permissionChecker.hasBluetoothPermissions()) {
        state.update { it.copy(showPermissionDialog = true) }
    } else {
        // スキャン開始
    }
}
```

---

## 時刻・単位ポリシー

**ドメイン層（高精度）**
- タイムスタンプは `timestampMicros: Long`（μs）。サンプルレートは可変を許容（200–1000 Hz）
- 角速度は rad/s、加速度は m/s² へ正規化（Adapter層で変換）
- 持続条件（`tRise`/`tFall`）は時間ベースで判定（ms→μs換算）

**データ永続化層（ストレージ効率）**
- Room DB に保存する際は epoch millis（INTEGER）へ変換
- Adapter層（`RawSampleRepository` 実装）で以下の変換を管理
  - ドメイン（μs） → DB保存（millis）: `timestampMicros / 1000`
  - DB取得（millis） → ドメイン（μs）: `timestampMillis * 1000`
- Raw サンプルの相対時刻 `t_rel_us` はマイクロ秒のまま保存（イベント期間は通常 0.5–2.0 s であり精度が必要）

**変換例**
```kotlin
// ドメイン → DB保存
val eventStartMillis = swingEvent.startedAtMicros / 1000
// DB取得 → ドメイン
val sampleTimestampMicros = rawSampleMillis * 1000
```

- 将来拡張: `SampleClockPort` 等でドリフト補正/補間に対応（v1は未実装）

---

## スイング検出エンジン（純粋ロジック）

要件の「スイング検出ロジック（加速度主体＋ジャイロ併用）」をドメイン内の純粋ロジックとして実装する。

- 入力: `SensorSample` ストリーム
- 前処理
  - ローパス（加速度 30–40 Hz 推奨）
  - 重力除去: センサー内姿勢の重力ベクトル ĝ を用いて `a_dyn = a_meas − ĝ·|g|`
  - スカラー化: `|a_dyn|`
  - スムージング: 15–25 ms 移動平均（中央値→平均）
- 主要指標
  - `A(t) = |a_dyn(t)|`
  - `W⊥(t) = sqrt(ωx² + ωy²)`（Z 自転除外、WT9011DCL 取り付け前提）
- 開始判定（Acceleration-driven）
  - `A(t)` が `start_threshold_A` を `T_rise` 以上継続
  - 直近 `refractory_period` 内の多重開始抑止
- 終了判定（Hybrid）
  - `A(t)` と `W⊥(t)` がそれぞれ `end_threshold_A` / `end_threshold_W` 未満を `T_fall` 継続
- 集計
  - `W⊥_max` と `v_tip = W⊥_max × R`（`R = d_sweet − d_hand`、補正係数は設定で管理）

参考となる（非依存な）関数イメージ:
```kotlin
private const val DEG2RAD: Float = (Math.PI / 180.0).toFloat()

fun tipSpeedNoAtt(gyroDegSamples: List<Vec3>, radiusMeters: Float, gain: Float = 1.0f): Float {
    var maxPerp = 0f
    for (wDeg in gyroDegSamples) {
        val wx = wDeg.x * DEG2RAD
        val wy = wDeg.y * DEG2RAD
        val wperp = kotlin.math.sqrt(wx*wx + wy*wy)
        if (wperp > maxPerp) maxPerp = wperp
    }
    return maxPerp * radiusMeters * gain
}
```

閾値・ウィンドウの初期値は要件の推奨（200 Hz想定）に従う。将来はキャリブレーションにより自動調整可能。

---

## UI（Compose）と状態管理（MVI風）

- 画面
  - `ConnectScreen`: スキャン/接続、権限誘導、状態表示
  - `MeasureScreen`: 計測開始/停止、リアルタイム指標、最新スイング速度
  - `HistoryScreen`: 記録リスト（日時/スピード/角度/ユーザー）
  - `SettingsScreen`: `L, d_hand, d_sweet, gain` 等の入力（スライダー/数値）

- ViewModel 契約
  - `StateFlow<UiState>`: 接続状態、計測状態、最新値、エラー、進行中フラグ
  - `