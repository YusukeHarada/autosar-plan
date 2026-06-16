# 検証環境構築ガイド — Classic AUTOSAR 建設機械プラットフォーム

## 1. 全体像：Vモデルと検証レベル

```
要件定義 ─────────────────────────────── 車両テスト
  │                                          │
  アーキ設計 ───────────────────── HIL検証
    │                                    │
    コンポーネント設計 ─────── SIL検証
      │                              │
      モデル設計 ────── MIL検証
               ↓
          実装・統合
```

| レベル | 実施場所 | 主な検証対象 | 担当 |
|--------|----------|--------------|------|
| MIL | PC（Simulink） | 制御アルゴリズムの動作 | 委託先（自社レビュー） |
| SIL | PC（VEOS等） | AUTOSARソフト全体 | 委託先（自社受け入れ） |
| HIL | HILラック | ECU＋実配線 | **自社主体** |
| 車両テスト | 実機 | 統合・機能安全確認 | **自社主体** |

自社1〜2名は MIL/SIL の成果物レビューと、HIL以降の受け入れ検証に集中する。

---

## 2. MIL（Model-in-the-Loop）

委託先が Simulink でモデルを構築し、自社はレビューと受け入れ基準の合意を担当する。

**建設機械固有の検証ポイント**

- 油圧モデル：ポンプ流量・圧力特性のリファレンスモデルとの誤差 ±2% 以内
- 旋回・走行制御：急操作時のショック抑制（加速度閾値を要件として事前定義）
- アイドルストップ：油温・水温条件との組み合わせ

**成果物（委託先から受領）**

- Simulink モデル一式（.slx）
- テスト結果レポート（Simulink Test またはMAP）
- カバレッジレポート（MC/DC：ISO 25119 の要求）

---

## 3. SIL（Software-in-the-Loop）

PC 上で AUTOSAR 生成コードを実行し、ECU なしでソフトウェア全体を検証する。

**主要ツール選択肢**

| ツール | 特徴 | 費用感 |
|--------|------|--------|
| dSPACE VEOS | AUTOSAR対応、Simulink連携◎ | 高（ライセンス制） |
| ETAS ISOLAR-EVE | ISOLAR-A/B と統合 | 中〜高 |
| QEMU + Autosar OS | オープン系、カスタマイズ必要 | ライセンス費ゼロ |

**CI パイプラインへの組み込み（GitLab CI 例）**

```yaml
sil-test:
  stage: test
  script:
    - veos-run --config sil_config.xml --testcases tests/
    - python scripts/parse_results.py --format junit
  artifacts:
    reports:
      junit: results/sil_report.xml
```

委託先の CI に自社担当者がオブザーバーアクセスできる権限を契約時に明記する。

---

## 4. HIL（Hardware-in-the-Loop）

実 ECU と HIL シミュレータを接続し、実時間で統合テストを行う。

**最小構成（フェーズ1）**

```
PC (CANoe) ─── CAN/LIN バス ─── ECU（評価ボード）
                                    │
                         アナログI/O（油圧センサ模擬）
```

**フル構成（フェーズ2）**

```
dSPACE SCALEXIO ─── ECU ─── 実ハーネス（短縮版）
       │
   モデル（油圧・エンジン・操作器）
       │
  CANoe（バス監視・テスト自動化）
```

**機能安全検証証跡の生成**

ISO 25119 Part 4 では「テスト仕様 → 実施記録 → 結果」の3点セットが必要。

| 証跡ドキュメント | 生成元 | 管理ツール |
|------------------|--------|------------|
| テスト仕様 | 要件から手動作成 | Polarion / Jama |
| 実施ログ | CANoe Trace / dSPACE自動生成 | バイナリ+PDF保存 |
| 合否判定レポート | スクリプト自動集計 | Polarion へ自動インポート |

---

## 5. テスト自動化戦略

**Robot Framework ＋ CANoe Python API（ATDD）**

```
Polarion（テスト仕様）
    ↓ エクスポート
Robot Framework（テストスクリプト）
    ↓ CANoe COM API
CANoe（バス刺激・測定）
    ↓ 結果
Polarion（自動インポート）
```

**MISRA-C チェックの自動化**

委託先のビルドパイプラインに組み込み、違反ゼロを Pull Request マージ条件とする。

```bash
# PC-lint Plus / PRQA の例
pclint +ffn -u misra2012.lnt src/**/*.c > misra_report.txt
```

自社は週次でレポートをレビューし、偏差申請（Deviation）の承認を行う。

---

## 6. 現実的な構築順序

### 今すぐ始められる最小構成（予算：〜50万円）

1. **CANoe ライセンス取得**（Vector Informatik — 評価版30日 → 購入）
2. **DBC ファイル整備**：委託先から CAN マトリクスを受領し CANoe にインポート
3. **ECU評価ボード**：委託先から提供される評価ボードを CANoe に接続
4. **手動テスト手順書**を Excel で作成し、CANoe Trace で証跡を取得

この段階で ISO 25119 の最低限の証跡ループが回せる。

### フェーズ2（6〜12か月後、予算：〜500万円）

- dSPACE MicroLabBox または SCALEXIO を導入し、I/O 模擬を自動化
- Robot Framework による自動テストスクリプトを整備
- Polarion でテスト管理と要件トレーサビリティを一元化

### フェーズ3（量産前、予算：〜2000万円）

- フルスペック HIL ラック（実ハーネス・アクチュエータ模擬込み）
- SIL→HIL の回帰テストを CI/CD として常時実行
- 機能安全監査に対応した証跡パッケージを自動生成
