# J1939プロトコル技術詳解

建設機械向けJ1939プロトコルのPGN・SPN定義、DBCファイル作成、AUTOSARとの連携を技術担当者・委託先向けに解説する。仕様書の書き方は `communication-spec.md` を参照。本資料はJ1939プロトコル自体の技術解説を目的とする。

---

## 1. J1939概要

### OSI参照モデルでの位置付け

| OSI層 | 機能 | J1939対応仕様 |
|---|---|---|
| 物理層（L1） | 電気的特性・コネクタ | ISO 11898-2（CAN High Speed） |
| データリンク層（L2） | フレーム構造・アービトレーション | ISO 11898-1（CAN仕様）＋J1939-21 |
| ネットワーク層（L3） | アドレス管理・トランスポート | J1939-21（TP）・J1939-81（NAM） |
| アプリケーション層（L7） | PGN・SPN定義 | J1939-71（Vehicle Application Layer） |

J1939は**SAE（Society of Automotive Engineers）**が策定した商用車・農業機械・建設機械向けのCAN上位層プロトコルである。物理層はISO 11898-2（250 kbps標準、500 kbps・1 Mbps対応機器も存在）を使用する。

### 建設機械での使われ方

- **ISOBUS（ISO 11783）**：農業機械向けでJ1939をベースにトラクタ＋作業機の接続を定義。建設機械のアタッチメント連携にも転用される場合がある。
- **J1939-75（Application Layer for Earthmoving Machines）**：掘削機・ブルドーザ・クレーン等向けのPGN拡張定義。作業機圧力・バケット角度・積載量等の専用SPNが規定されている。
- エンジン・トランスミッション・ポンプ・各種コントローラ間の通信バスとして使用。1ネットワークに最大30ノードを接続可能（物理的制約による）。

---

## 2. フレーム構造

### 29bit拡張CANフレーム

J1939は必ず**29bitの拡張ID**（CAN 2.0B）を使用する。28bit目から0bit目の割り当ては以下の通り。

```
 28    26 25 24 23        16 15         8 7          0
┌────────┬──┬──┬───────────┬─────────────┬────────────┐
│Priority│ R│DP│ PDU Format │ PDU Specific│Source Addr │
│ 3bit   │1b│1b│   8bit     │    8bit     │   8bit     │
└────────┴──┴──┴───────────┴─────────────┴────────────┘
```

| フィールド | ビット数 | 説明 |
|---|---|---|
| Priority | 3bit | 送信優先度（0が最高、7が最低）。制御系は3〜5が多い |
| Reserved（R） | 1bit | 将来拡張用。通常0 |
| Data Page（DP） | 1bit | PGNページ拡張。0＝ページ0（標準）、1＝ページ1（拡張） |
| PDU Format（PF） | 8bit | PDUタイプ判定に使用（0x00〜0xEF：PDU1、0xF0〜0xFF：PDU2） |
| PDU Specific（PS） | 8bit | PDU1では宛先アドレス、PDU2ではグループ拡張 |
| Source Address（SA） | 8bit | 送信元アドレス（0x00〜0xFD：通常使用、0xFE：Anonymous、0xFF：Global） |

### PGNの計算方法

PGN = （DP × 65536）＋（PF × 256）＋（PDU2の場合はPS、PDU1の場合は0）

**PDU1（PF ≤ 0xEF）**：PSは宛先アドレスのため、PGNにはPS=0x00を代入して計算する。特定アドレス宛ての送信が可能。

**PDU2（PF ≥ 0xF0）**：PSはGroup Extension（GE）として扱い、PGNに含まれる。ブロードキャスト専用。

### 具体的なCANフレーム例

エンジン回転数（EEC1、PGN 0xF004 = 61444）の送信例：

```
29bit ID（16進）: 0CF00400
内訳：
  Priority    = 3  (0b011)
  R           = 0
  DP          = 0
  PDU Format  = 0xF0 (240) → PDU2
  PDU Specific= 0x04 (GE=4)
  Source Addr = 0x00 (ECM)

PGN = 0xF004 = 61444 (EEC1)
データフィールド（8バイト）:
  Byte 0: Engine Control Mode（SPN 899）
  Byte 1: Driver Demand Engine - Percent Torque（SPN 512）
  Byte 2: Actual Engine - Percent Torque（SPN 513）
  Byte 3-4: Engine Speed（SPN 190）LSB先（リトルエンディアン）
  Byte 5: Source Address of Controlling Device（SPN 1483）
  Byte 6: Engine Starter Mode（SPN 1675）
  Byte 7: Engine Demand Engine - Percent Torque（SPN 2432）
```

---

## 3. PGN（Parameter Group Number）

### 代表的なPGN一覧

| PGN（10進） | 略称 | 説明 | 送信周期（目安） |
|---|---|---|---|
| 61444 | EEC1 | Electronic Engine Controller 1（エンジン回転数・トルク） | 10ms |
| 61445 | EEC2 | Electronic Engine Controller 2（アクセル位置等） | 50ms |
| 65265 | CCVS | Cruise Control / Vehicle Speed（車速・クルーズ状態） | 100ms |
| 61442 | ETC1 | Electronic Transmission Controller 1（シフト位置等） | 10ms |
| 65257 | LFC | Fuel Consumption（燃料消費量） | 1000ms |
| 65262 | ET1 | Engine Temperature 1（冷却水温・燃料温度） | 1000ms |
| 65263 | EFL | Engine Fluid Level（油圧・油温） | 1000ms |
| 65110 | — | J1939-75：掘削機アーム角度等（要確定） | 要確定 |

### 建設機械固有PGNの例（J1939-75）

- 作業機シリンダ圧力（SPN 要確定）：バケットシリンダ・アームシリンダの圧力値
- 機体傾斜角：前後・左右の傾き（転倒防止制御に使用）
- 積載量：バケット内土砂のペイロード推定値

カスタムPGNを定義する場合は、PGN 0xFF00〜0xFFFF（Proprietary B）またはPDU1の相手先指定形式（Proprietary A）を使用する。

### データのバイト順・スケーリング

J1939のデータ表現ルール：
- **バイト順**：1バイト目がLSB（リトルエンディアン）が基本。ただしSPN定義書に従うこと。
- **スケーリング**：`実値 = 生データ × Resolution + Offset`
- **無効値**：0xFE = エラー表示、0xFF（1バイト）または 0xFFFF（2バイト）= 未実装

**エンジン回転数（SPN 190）変換例：**
```
Resolution  = 0.125 rpm/bit
Offset      = 0 rpm
Range       = 0 〜 8031.875 rpm
データ値 0x1900 (LSB=0x19=25, MSB=0x00=0) → 生データ = 25
実値 = 25 × 0.125 = 3.125 rpm  ← 実際の生データは要確定
```

---

## 4. SPN（Suspect Parameter Number）

### SPNとPGNの関係

1つのPGNは固定長8バイトのデータフィールドを持ち、その中に複数のSPNが詰め込まれる。SPNは個々のパラメータ定義を指す。

```
PGN 61444 (EEC1)
├── SPN 899  : Engine Control Mode          (Byte 0, bit 0-3)
├── SPN 512  : Driver Demand Engine Torque  (Byte 1, 全8bit)
├── SPN 513  : Actual Engine Torque         (Byte 2, 全8bit)
├── SPN 190  : Engine Speed                 (Byte 3-4, 全16bit)
└── ...
```

### SPNデータタイプ

| タイプ | 説明 | 例 |
|---|---|---|
| Measured | 測定値（数値）。ResolutionとOffsetで実値変換 | 回転数・温度・圧力 |
| Status | 状態値（列挙）。固定値が状態を表す | シフト位置・オン/オフ |
| Counter | カウンタ（累積値）。エラー回数・燃料消費量合計 | トリップ燃費 |
| Textual | テキスト表示用（通常8bit、ASCII） | 警告メッセージ等 |

---

## 5. DBCファイル

### DBCファイルの基本構造

DBC（Database CAN）はVector社が策定したCANネットワーク記述フォーマット。CANoe・CANdb++・CANalyzer等のツールで標準的に使用される。

```dbc
VERSION ""

NS_ :

BS_:

BU_: ECM TCM VCM

BO_ 2566844416 EEC1: 8 ECM
 SG_ EngineSpeed : 24|16@1+ (0.125,0) [0|8031.875] "rpm" VCM
 SG_ ActualEngineTorque : 16|8@1+ (1,-125) [-125|125] "%" VCM
 SG_ DriverDemandTorque : 8|8@1+ (1,-125) [-125|125] "%" VCM
 SG_ EngineControlMode : 0|4@1+ (1,0) [0|15] "" VCM

VAL_ 2566844416 EngineControlMode
 0 "Low Idle Governor"
 1 "Accelerator Pedal"
 3 "Remote Accelerator"
 ;
```

### J1939をDBCで表現する際の注意点

1. **29bit IDの表現**：DBCの`BO_`行のメッセージIDは10進表記。J1939の29bit拡張フレームを示すためIDに`0x80000000`を加算する慣例がある（ツール依存）。
   - 例：PGN 61444（0xF004）、SA=0x00、Priority=3 → CAN ID = `0x0CF00400` = `217056256`
   - DBCには `2566844416`（= `217056256 + 0x80000000`）と記載する場合が多い（要確定・ツール確認）

2. **SG_のビット定義**：`SG_ 名前 : 開始bit | ビット長 @ バイト順 符号 (resolution, offset) [min|max] "単位" 受信ノード`
   - `@1` = リトルエンディアン（Intel byte order）
   - `@0` = ビッグエンディアン（Motorola byte order）

3. **PGN・SPNのコメント記述**：DBCのコメント機能（`CM_`）でSPN番号を明示しておくと、後のトレーサビリティが向上する。

### ArXMLとの変換・連携

AUTOSARのネットワーク記述はArXMLで行うが、DBCとの相互変換ツールが各社から提供されている。

| ツール | 用途 |
|---|---|
| Vector CANdb++ | DBC作成・編集 |
| Vector DaVinci Developer | DBC→ArXML変換・COM設定 |
| ETAS ISOLAR-A | ArXML編集・J1939設定 |
| EB tresos Studio | ArXML編集・J1939 Stack設定 |

変換時の確認ポイント：
- SPN名・解像度・オフセットがArXML側に正しく引き継がれているか
- PGN→I-PDU のマッピングが正しいか
- 送信周期（CycleTime）が反映されているか

---

## 6. 診断との関係（DM1/DM2）

### DM1（Diagnostic Message 1）の構造

DM1はアクティブ（現在発生中）の故障コードを通知するPGN（PGN 65226、0xFECA）。

```
DM1フレームのデータフィールド（マルチパケットの場合あり）:
  Byte 0    : Lamp Status（MIL/RSL/AWL/PL点灯状態）
  Byte 1    : Lamp Status continued
  Byte 2-3  : SPN（SPNの下位19bit）＋FMI（5bit）＋CM(1bit)＋OC(7bit)
  Byte 4-5  : 2番目の故障コード（同じ構造）
  ...（最大21個まで 1パケットで、それ以降はTransport Protocolを使用）
```

### DM2（以前の故障コード）

- PGN 65227（0xFECB）
- 過去に発生し現在は解消された故障コードを保持
- 構造はDM1と同じ
- エンジン停止・バッテリ遮断後もフラッシュに保存される

### FMI（Failure Mode Identifier）

| FMI | 意味 |
|---|---|
| 0 | 高電圧・高データ値（Above Normal） |
| 1 | 低電圧・低データ値（Below Normal） |
| 2 | データが不安定・間欠的 |
| 3 | 電圧高（ショート to バッテリ） |
| 4 | 電圧低（ショート to グランド） |
| 5 | 電流低・開路 |
| 6 | 電流高・ショート to グランド |
| 7 | メカニカル故障 |
| 8 | 周期・波形異常 |
| 9 | 更新レート異常 |
| 10 | 異常な変化率 |
| 11 | 原因特定不可 |
| 12 | 不正デバイスまたはコンポーネント |
| 13 | キャリブレーション外 |
| 14 | 特殊指示 |
| 19 | ネットワーク通信エラー（受信タイムアウト等） |
| 31 | 条件あり（SAE J1939-73で定義） |

---

## 7. AUTOSARとの連携

### COM/PDUとPGN/SPNのマッピング

Classic AUTOSARのCOM（Communication）スタックは、I-PDU（Interaction Layer PDU）単位でデータを管理する。J1939の1PGN = 1I-PDUとして扱うのが基本。

```
J1939層                   AUTOSAR COM層
─────────────────         ─────────────────────────
PGN（例: EEC1）     →    I-PDU（例: EEC1_Tx）
  └── SPN 190      →      Signal（例: EngineSpeed）
  └── SPN 513      →      Signal（例: ActualEngineTorque）
```

設定項目（ArXML）：
- `J1939Tp`：J1939トランスポートプロトコルの設定
- `CanNm`：J1939ネットワーク管理との整合（アドレスクレーム等）
- `ComConfig`：I-PDUの送信周期・タイムアウト監視設定

### J1939Tp（Transport Protocol）

8バイトを超えるデータ（DM1の多故障コード等）はJ1939 TPを使用して分割送信する。

| フレームタイプ | PGN | 説明 |
|---|---|---|
| TP.CM (Connection Management) | 60416（0xEC00） | 接続開始・終了の制御 |
| TP.DT (Data Transfer) | 60160（0xEB00） | 実データの分割送信 |

AUTOSARのJ1939Tpモジュールがこの分割・再構成を自動処理するため、上位のCOMスタックからはシームレスに扱える。

---

## 8. 委託先との仕様協議ポイント

### DBCファイルの提供・管理方法

- 自社が保有するDBCファイルをバージョン管理（Git）し、委託先に提供する
- DBCに不明なSPNが含まれる場合、J1939-71・J1939-75の原典を委託先が参照できるよう確認する（有償規格のため購入要否を確認）
- カスタムPGN・SPNは別ファイル（例: `custom_pgn.dbc`）で管理し、標準PGN定義と混在させない

### カスタムPGNの定義方法と命名規則

建設機械固有の信号はProprietaryPGNを使用する。

| 範囲 | 名称 | 用途 |
|---|---|---|
| PGN 0xFF00〜0xFFFF | Proprietary B (PPB) | ブロードキャスト。会社固有の制御データ |
| PGN 0xEF00（PDU1） | Proprietary A (PPA) | ピアツーピア。特定ECU向け制御コマンド |

命名規則（例）：
```
SPN命名: [会社略称]_[機能]_[パラメータ名]
例: ABC_HYD_BucketPressure（ABC社・油圧系・バケット圧力）

PGN命名: [会社略称]_[サブシステム]_[グループ名]
例: ABC_HYD_WorkingEquipment1
```

### 協議時の確認事項チェックリスト

- [ ] 標準J1939-71・J1939-75のPGN/SPNで対応できない信号の洗い出し
- [ ] カスタムPGNのID割り当て（社内で一元管理する担当者を決める）
- [ ] DBCファイルの更新フロー（誰が変更し、誰がレビューするか）
- [ ] ArXML生成ツール（DaVinci・EB tresos等）でのDBC取り込み手順確認（要確定）
- [ ] TP使用が必要なメッセージの特定（8バイト超えのDM1等）
- [ ] 送信周期・タイムアウト値の定義（制御系は厳密な周期管理が必要）

---

## 参考規格・資料

| 規格 | 内容 |
|---|---|
| SAE J1939-21 | データリンク層 |
| SAE J1939-71 | Vehicle Application Layer（標準PGN/SPN定義） |
| SAE J1939-73 | Application Layer - Diagnostics（DM1/DM2/FMI定義） |
| SAE J1939-75 | Application Layer for Earthmoving Machines |
| SAE J1939-81 | Network Management（アドレスクレーム） |
| ISO 11898-1/2 | CAN物理層・データリンク層 |
| ISO 11783 | ISOBUS（農業機械・建設機械アタッチメント） |

> **注意**：本資料中の具体的な数値（PGN番号・SPN番号・ビット位置・スケール値等）は参考値である。実装時は上記SAE規格原典および委託先ECUメーカの仕様書で必ず確認すること（要確定箇所は個別に明示している）。
