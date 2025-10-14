# 画面遷移・機能定義（Compose／スマホ縦向き中心）

本ドキュメントは、`docs/architecture.md` と `docs/requirements_specification.md` に整合し、ナビゲーションと各画面の機能・状態・イベント・ガードを定義する。

---

## ナビゲーション構成

- StartDestination: `Connect`
- Graph: `Connect` → `Measure`
- AppBar メニュー: `History`, `Settings`
- 戻る挙動:
  - `Measure` → `Connect`（切断確認あり）
  - `History`・`Settings` は `popBackStack()`

### 画面遷移（Mermaid）

```mermaid
stateDiagram-v2
  [*] --> Connect

  state "AppBarメニュー" as Menu

  Connect --> Measure: 接続成功
  Connect --> Menu: ⋮（オーバーフロー）
  Measure --> Connect: 切断/エラー
  Measure --> Measure: 計測開始/停止
  Measure --> Menu: ⋮（オーバーフロー）

  Menu --> History: 履歴を開く
  Menu --> Settings: 設定を開く

  History --> EventDetail: イベント選択
  EventDetail --> History: 戻る(popBackStack)

  History --> [*]: 戻る(popBackStack)
  Settings --> [*]: 戻る(popBackStack)
```

### NavHost（Compose）概要（抜粋）

```kotlin
NavHost(navController, startDestination = "connect") {
  composable("connect") { ConnectScreen(...) }
  composable("measure") { MeasureScreen(...) }
  composable("history") { HistoryScreen(...) }
  composable("settings") { SettingsScreen(...) }
  composable("event_detail/{eventId}") { backStackEntry ->
    val eventId = backStackEntry.arguments?.getString("eventId")!!.toLong()
    EventDetailScreen(eventId = eventId, ...)
  }
}
```

---

## 画面別の機能・状態・イベント

### ConnectScreen

- 表示
  - ステータス: `isScanning`, `isConnected`, 権限状態（Bluetooth/位置情報）
  - デバイスリスト（RSSI/名称/接続可否）
- 操作（UiEvent）
  - `StartScan`, `StopScan`, `RequestPermission`, `Connect(device)`, `Disconnect`
- 状態（UiState 一例）
  - `devices: List<DeviceItem>`、`isScanning: Boolean`、`permissionState: PermissionState`、`error: String?`
- 遷移
  - 接続成功→ `Measure`
- ガード/エラー
  - 権限未許可→シート/ダイアログで誘導
  - 接続失敗/タイムアウト→トースト＋再試行

### MeasureScreen

- 表示
  - リアルタイム指標: `A(t)`、`W⊥(t)` の要約（数値/ゲージ）
  - 最新スイング: `tip_speed`（km/h表示）、`w_perp_max`
  - セッション情報: 開始時刻・カウント
  - ボタン: 計測開始/停止、セッション終了
- 操作（UiEvent）
  - `StartMeasuring`, `StopMeasuring`, `EndSession`, `AcknowledgeSaved`
- 状態（UiState 一例）
  - `swinging: Boolean`, `wPerpMax: Float`, `tipSpeed: Float`, `sessionInfo: SessionInfo?`, `saving: Boolean`
- ガード/エラー
  - 切断/エラー時: 確認ダイアログ→ `Connect` へ復帰（未保存イベントがあれば保存処理）
  - ドロップ可否: 検出側は `buffer(CONFLATED)` 可、Raw保存側はドロップ禁止

### HistoryScreen

- 表示
  - リスト: 日付、速度、角度、ユーザー、セッション/イベント単位
  - フィルタ/並び替え（最新順/速度順）
- フィルター
  - ユーザー（`user_id`）
  - セッション（`session_id`）
  - 期間（from/to）
  - バットプロファイル（`bat_profile_id`）
  - 速度レンジ（`tip_speed` min/max）
- 操作（UiEvent）
  - `SetUserFilter(id)`, `SetSessionFilter(id)`, `SetDateRange(from,to)`, `SetBatProfile(id)`, `SetSpeedRange(min,max)`, `ChangeSort(order)`, `OpenEvent(eventId)`
- 状態（UiState 一例）
  - `items: List<HistoryItem>`, `filters: HistoryFilter`, `isEmpty: Boolean`, `loading: Boolean`
- DAO検索（参考）
  - `SELECT e.* FROM swing_events e JOIN swing_sessions s ON e.session_id = s.id WHERE s.user_id IN (?) AND s.bat_profile_id IN (?) AND e.started_at BETWEEN ? AND ? AND e.tip_speed BETWEEN ? AND ? ORDER BY e.started_at DESC LIMIT ? OFFSET ?`
- インデックス
  - 既存 `idx_events_session_started` を活用。期間検索が多い場合は `started_at` 単独Indexも検討
  - セッション/ユーザー/プロファイル連動は `JOIN` で対応

### SettingsScreen

- 表示/操作
  - バットプロファイル: `name`, `length_m`, `d_hand_m`, `d_sweet_m`, `gain`
  - ユーザー切替/作成、プロファイルの追加/編集/削除
  - 表示単位: 入力は cm、保存は m。速度は km/h 表示（内部 m/s）
- 操作（UiEvent）
  - `SaveBatProfile(profile)`, `SwitchUser(userId)`, `CreateProfile`, `DeleteProfile(id)`
- 状態（UiState 一例）
  - `profiles: List<BatProfile>`, `currentUser: User`, `editing: BatProfile?`, `saving: Boolean`

### EventDetailScreen

- 目的
  - 1スイング内で「バットがどのように加速していったか」を可視化
- 入力
  - `eventId`
- 表示
  - 時系列グラフ: `W⊥(t)`, `A(t)`, `tip_speed(t) = W⊥(t)×R×gain`
  - メタ情報: 最高速度/ピーク時刻/サンプル数/`sample_rate_hz`
- データ
  - `swing_raw_samples`（`event_id`、`t_rel_us`）を取得
  - `W⊥(t) = sqrt(ωx² + ωy²)`（内部は rad/s 推奨。deg/sなら変換）
  - `A(t)` は重力除去・スムージング（初期は軽い移動平均でも可）
- パフォーマンス
  - グラフ描画用にデシメート（例: 1000Hz→200Hz）
  - `t_rel_us` を秒へ変換して x 軸に使用
- 操作（UiEvent）
  - `Load(eventId)`, `SetSmoothing(windowMs)`, `ToggleSeries(Wperp|A|TipSpeed)`
- 技術選択
  - MPAndroidChart を採用（ピンチ/パン/ダブルタップズーム対応）
    - 依存関係（例）
      ```kotlin
      dependencies {
        implementation("com.github.PhilJay:MPAndroidChart:3.1.0")
      }
      ```
    - Compose ラップ例（抜粋）
      ```kotlin
      AndroidView(factory = { context ->
        LineChart(context).apply {
          setScaleEnabled(true)
          setPinchZoom(true)
          isDoubleTapToZoomEnabled = true
          description.isEnabled = false
          legend.isEnabled = true
          xAxis.position = XAxis.XAxisPosition.BOTTOM
          axisRight.isEnabled = false
          setTouchEnabled(true)
          isDragEnabled = true
        }
      }, update = { chart ->
        val wEntries = wPerp.map { Entry(it.tSec, it.value) }
        val aEntries = aNorm.map { Entry(it.tSec, it.value) }
        val vEntries = tipSpeed.map { Entry(it.tSec, it.value) }

        val dataSets = listOf(
          LineDataSet(wEntries, "W⊥(rad/s)").apply { color = Color.RED; setDrawCircles(false); lineWidth = 1.5f },
          LineDataSet(aEntries, "A(m/s²)").apply { color = Color.BLUE; setDrawCircles(false); lineWidth = 1.5f },
          LineDataSet(vEntries, "v_tip(m/s)").apply { color = Color.GREEN; setDrawCircles(false); lineWidth = 1.5f }
        )
        chart.data = LineData(dataSets)
        chart.invalidate()
      })
      ```
    - 実装メモ
      - 表示はデシメート（例: 1000Hz → 200Hz）して点数を抑制（系列あたり ≤ 400点目安）
      - x軸は `t_rel_us / 1e6` を秒表示に整形（`ValueFormatter`）
      - ピーク値の `Highlight`、凡例色分け、左右軸の使い分けは必要に応じて設定

---

## 権限/エラー/空状態

- 権限
  - `Connect` で Bluetooth/位置情報の不足を検出し、要求→結果ハンドリング（許可/永久拒否）
- エラー
  - 接続失敗/切断/タイムアウト: トースト＋再試行。計測中切断→確認後セッション終了
- 空状態
  - `History` にプレースホルダ（「まだスイングが記録されていません」）＋計測開始導線
  - `Settings` にプロファイル未作成時のガイダンス

---

## UI方針（縦向き）

- アプリバー＋オーバーフローメニュー（`History`/`Settings`）
- `Measure`: 上＝リアルタイム数値、中＝ゲージ、下＝開始/停止ボタン
- スクロール時も重要指標は常時表示（SmallTopAppBar + pinned）

---

## 受け入れ基準

- 本書に以下を含む
  - ナビグラフ（Mermaid）と Compose `NavHost` 概要
  - 各画面の表示/操作/状態/遷移/ガード
  - 権限/エラー/空状態の扱い
  - 縦向き UI 指針
- `docs/architecture.md` と `docs/requirements_specification.md` に整合


