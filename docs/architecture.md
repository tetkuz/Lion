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
  - `settings/` DataStore/Preferences 実装
  - `repository/` Port を満たす具象リポジトリ
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
    val startedAtMicros: Long,
    val endedAtMicros: Long,
    val wPerpMaxRadPerSec: Float,
    val tipSpeedMetersPerSec: Float,
    val impactAngleRad: Float?,
    val sampleRateHz: Float?
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
```

- 不変条件
  - `SwingEvent.startedAtMicros <= endedAtMicros`
  - `0 <= SwingRawSample.tRelMicros <= (ended - started) * 1000`
  - `BatProfile.radiusMeters > 0`、`gain > 0`

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
    fun loadSwingSettings(): kotlinx.coroutines.flow.Flow<SwingSettings>
    suspend fun updateSwingSettings(value: SwingSettings)
}

interface BatProfileRepository {
    suspend fun getActiveProfile(userId: Long): BatProfile
    suspend fun save(profile: BatProfile): Long
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
- 役割
  - スキャン/接続/再接続の状態管理（明示的な状態遷移とタイムアウト）
  - GATT 購読（200 Hz 目安、実測は可変）→ バイト列デコード → `SensorSample`
  - 単位変換: 角速度 deg/s → rad/s、加速度 g → m/s²（必要時）
  - バックプレッシャ: `buffer(capacity = Channel.CONFLATED)` で最新値優先
  - 例外と切断: 再試行ポリシ、ユーザー誘導（権限/設定）
  - `SensorStreamPort.startStreaming()/stopStreaming()` を提供し明示的に制御

- 権限/設定
  - Bluetooth 有効化、位置情報権限（OSバージョンごとの要件を UI で誘導）
  - フォアグラウンド版では画面表示中のみ購読（ライフサイクルに追従）

---

## 時刻・単位ポリシー

- タイムスタンプは `timestampMicros: Long`（μs）。サンプルレートは可変を許容
- 角速度は rad/s、加速度は m/s² へ正規化（Adapter層で変換）
- 持続条件（`tRise`/`tFall`）は時間ベースで判定（ms→μs換算）
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
  - `onEvent(UiEvent)`: ユーザー意図（接続/開始/停止/設定保存 等）
  - `UiEffect`: トースト/ナビゲーションなどの一回性イベント

- ライフサイクル
  - v1 はフォアグラウンド限定（画面 ON 中のみ購読）。バックグラウンドは将来の ForegroundService で対応

---

## データ永続化（Room）

- エンティティ（最小）
  - `swing_sessions`:
    - `id`（PK）、`user_id`、`bat_profile_id`、`started_at`、`ended_at?`、`note?`
  - `swing_events`:
    - `id`（PK）、`session_id`、`started_at`、`ended_at`、`w_perp_max`、`tip_speed`、`impact_angle?`、`created_at`
  - `swing_raw_samples`:
    - `id`（PK）、`event_id`、`t_rel_us`、`ax`、`ay`、`az`、`gx`、`gy`、`gz`
  - 参照: 詳細は `docs/er_model.md`

- Repository 実装
  - `SessionRepository` / `SwingEventRepository` / `RawSampleRepository` を Room で実装
  - インデックス例: セッション（`user_id, started_at`）、イベント（`session_id, started_at`）、Raw（`event_id, t_rel_us` UNIQUE）

---

## 依存性注入（Hilt）

- セットアップ
  - `@HiltAndroidApp` を Application に付与
  - `@AndroidEntryPoint` を Activity/Fragment に付与
  - `@HiltViewModel` + `@Inject` コンストラクタで ViewModel へ依存注入

- Modules
  - `BleModule`: `SensorStreamPort`/`ConnectionStatePort` 実装の提供
  - `RepositoryModule`: `SessionRepository` / `SwingEventRepository` / `RawSampleRepository` / `SettingsRepository` 実装の提供
  - `UseCaseModule`: 各 UseCase のファクトリ
  - `CoroutineModule`: `DispatcherProvider`

---

## エラー・権限・状態遷移

- 権限ガード: Bluetooth/位置情報/通知（将来）
- エラー分類: 一過性（再試行）/恒久（設定誘導）/ユーザー起因（権限拒否）
- 状態遷移を UI に可視化（接続中/購読中/切断/再接続待ち）

---

## テスト戦略

- ユニットテスト（純粋ドメイン）
  - スイング検出の状態機械/閾値/持続時間ロジックを例外なく網羅
  - Property-based Testing（例: 乱数系列でも不変条件を満たす）
  - ゴールデンテスト: 録画センサ列（CSV）→ 期待 `SwingEvent` と照合

- コントラクトテスト
  - Port に対して共通仕様を定義し、Fake 実装と DB 実装が同一振る舞いを満たすことを検証

- UIテスト
  - Compose の状態遷移テスト（Fake Port 注入）
  - 重要な副作用（保存、トースト）の発火を検証

---

## 将来拡張（設計の余地）

- ForegroundService 対応（通知チャンネル、停止/再開、バッテリー配慮）
- WorkManager 連携（再起動時の自動復旧）
- 他 IMU 対応: Adapter 追加で差し替え可能に（Port 準拠）
- クラウド同期/エクスポート、共有（プライバシと匿名化）

---

## 受け入れ基準

- 本書に以下が含まれている
  - 全体アーキテクチャとパッケージ構成
  - 主要ドメインモデル/ポート（Session/Event/Raw＋μs時刻）
  - WT9011DCL BLE 抽象化とストリーム設計（start/stop、単位変換、可変Hz）
  - スイング検出エンジン（閾値・時間窓・不変条件）の要点
  - UI/MVI 風状態管理、Room 永続化、Hilt DI
  - テスト戦略（ユニット/プロパティ/ゴールデン/契約）
  - v1 の制約（フォアグラウンド限定）と将来拡張の方針

- `docs/requirements_specification.md` の主要項目と整合している


