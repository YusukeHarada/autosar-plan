# 診断通信（UDS / J1939）設計メモ

## 1. 診断通信の全体像

| プロトコル | 規格 | 主な用途 |
|-----------|------|---------|
| OBD-II | ISO 15031 / SAE J1979 | 排気ガス規制対応（乗用車） |
| UDS | ISO 14229 | 汎用ECU診断・書き込み・セキュリティ |
| J1939診断 | SAE J1939-73 | 商用車・建設機械の障害コード通知 |

建設機械では **UDS**（整備ツールからのオンデマンド診断）と **J1939診断**（DM1/DM2によるブロードキャスト通知）が共存する。遠隔監視（テレマティクス）はDM1のデータをゲートウェイ経由でクラウドへ送るケースが多い。

---

## 2. UDS（ISO 14229）主要サービス

| サービスID | 名称 | 用途 |
|-----------|------|------|
| 0x10 | DiagnosticSessionControl | セッション切替（Default / Extended / Programming） |
| 0x11 | ECUReset | ECU再起動 |
| 0x22 | ReadDataByIdentifier | センサ値・バージョン等の読出 |
| 0x2E | WriteDataByIdentifier | パラメータ書込 |
| 0x27 | SecurityAccess | シード/キーによるロック解除 |
| 0x14 | ClearDiagnosticInformation | DTC消去 |
| 0x19 | ReadDTCInformation | DTC読出（ステータス・スナップショット含む） |
| 0x85 | ControlDTCSetting | DTC記録の一時停止/再開 |
| 0x34/0x36/0x37 | RequestDownload / TransferData / RequestTransferExit | ファームウェア書込 |

**通信フロー（CAN上 / ISO 15765 TP）**

```
[診断ツール]  -- SingleFrame or FirstFrame -->  [ECU]
[ECU]         <-- FlowControl --                [診断ツール]
[ECU]         -- PositiveResponse (0x40+SID) -- [診断ツール]
              または NegativeResponse (0x7F)
```

ISO 15765（CATP）がペイロード8バイト超のメッセージをフレーム分割する。UDSはそのアプリケーション層として乗る。

---

## 3. AUTOSARのDcm・Demモジュール

**Dcm（Diagnostic Communication Manager）**
- UDSサービスの受信・解釈・ルーティングを担う
- セッション状態機械・セキュリティ状態機械を内部で管理
- 各サービスハンドラはアプリ層のコールバック（`Xxx_GetDataElement` 等）を呼び出す

**Dem（Diagnostic Event Manager）**
- アプリ・BSWからのイベント（`Dem_ReportErrorStatus`）を受け付けDTCに変換
- DTCのステータスビット（TestFailed / Confirmed / Pending 等）を管理
- **NvMとの連携**：Dem は NvM ブロックにDTCステータス・フリーズフレームを書き込む。シャットダウン時に`NvM_WriteAll`で永続化され、次回起動時に復元される

**DTCライフサイクル**

```
イベント発生 → TestFailed セット → Pending → Confirmed（閾値超過）
イベント消滅 → TestFailed クリア → Healing カウンタ → Passed
0x14サービス → Dem_ClearDTC → NvMブロック消去
```

---

## 4. J1939診断（DM1/DM2）との使い分け

| | DM1 | DM2 |
|--|-----|-----|
| 内容 | アクティブな障害コード（SPN+FMI） | 過去の障害コード |
| 送信方式 | 1秒周期ブロードキャスト | オンデマンド（要求応答） |
| 主な受信者 | 車載クラスタ・テレマティクスGW | サービスツール |

**UDSとJ1939診断の共存パターン**
- 建設機械メインコントローラがJ1939 CANバス上でDM1を送信しながら、UDS診断用に別CANバス（またはCANFD）を持つ構成が一般的
- サービスツールはUDS経由（ISO 15765 TP）でECUに接続し、詳細データ・書込・フラッシュを実施
- テレマティクスGWはJ1939 DM1を購読してクラウドに転送

---

## 5. 設計上の注意点

**セキュリティアクセス（0x27）**
- シードは起動ごとに乱数生成。固定シードは禁止
- キーアルゴリズムは量産ECUとサービスツールで同一ライブラリを使い乖離を防ぐ
- 失敗回数上限（デフォルト3回）を超えたら一定時間ロック（`securityAccessDelayTimer`）

**EOL（End of Line）テスト**
- 生産ラインでExtendedセッションに入り、0x2Eでシリアル番号・キャリブレーション値を書込
- 書込後に0x22で読み返し値の整合性を確認

**OTAとの連携**
- テレマティクスGWがOTAサーバーからパッケージを受信後、UDS 0x34〜0x37シーケンスでECUにフラッシュを転送するトリガーとして診断セッションを使用
- Programming Sessionへの遷移前に0x27でセキュリティ解除が必須

---

## 6. PoCとしてDcm+Demを選ぶ理由

- **合否判定が容易**：ISO 14229はサービスIDごとのリクエスト/レスポンスフォーマットが明確に規定されており、CANoeのDiag Basicスクリプトで自動テストスクリプトを記述しやすい
- **CANoeで自動化可能**：CAPL + Diagnostic Description（.cdd/.pdx）を使えば、UDSシーケンス全体を回帰テストとして実行できる
- **BSW間インタフェースの検証範囲が広い**：DcmはCom/PduR/CanTp、DemはNvM/FimなどAUTOSARスタックの主要モジュールと連携するため、統合検証のカバレッジが高い
- **建設機械固有要件との照合**：J1939 DM1との二重管理（DemがDTCを保持しJ1939変換レイヤが読み出す）もPoC段階で設計検証できる
