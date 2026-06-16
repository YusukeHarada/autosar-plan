# 建設機械向け AUTOSAR 移行検討リポジトリ

**対象読者：** 開発部門内関係者  
**機密レベル：** 社外秘

建設機械向けプラットフォームソフトウェアのAUTOSAR準拠・外部委託移行、およびSDV化に向けた総合検討資料。

---

## 背景

建設機械の制御系・通信系ソフトウェアを内製で開発してきたが、以下の課題がある。

- 機能安全（ISO 25119）・セキュリティ規格への対応が必要だが、現状の内製体制では対応不可能
- 仕様書・設計書・テストが存在しない状態で4〜5年の開発が蓄積されている
- AUTOSARを自社で習得・導入するリソースがない
- SDV化（自律施工・フリート管理・OTA）の流れが建設機械にも波及しつつある

これらを踏まえ、**Classic AUTOSAR（RTOS系）＋ Linuxミドルウェア（Linux系）のハイブリッド構成を外部委託で構築**し、段階的にSDV化を進める方針を検討している。

---

## ドキュメント一覧

### 戦略・提案

| ファイル | 内容 | 対象読者 | ステータス |
|---|---|---|---|
| [proposal.md](proposal.md) | 社内提案資料（なぜ変えるべきか） | 管理職・幹部 | Draft |
| [sdv.md](sdv.md) | SDVの概念と建設機械への考察 | 開発部門・経営層 | Draft |
| [roadmap.md](roadmap.md) | フェーズ別ロードマップ・必要技術・人材 | 開発部門・管理職 | Draft |

### 外部委託・選定

| ファイル | 内容 | 対象読者 | ステータス |
|---|---|---|---|
| [vendor-selection.md](vendor-selection.md) | 委託先選定基準・PoC評価 | 技術担当者 | Draft |

### 技術仕様

| ファイル | 内容 | 対象読者 | ステータス |
|---|---|---|---|
| [requirements-spec.md](requirements-spec.md) | AUTOSARを前提とした要求仕様の作り方 | 技術担当者 | Draft |
| [communication-spec.md](communication-spec.md) | CAN・Ethernet・J1939通信仕様の書き方 | 技術担当者・委託先 | Draft |
| [toolchain.md](toolchain.md) | AUTOSARコード生成ツールチェーン | 技術担当者 | Draft |
| [autosar-modules.md](autosar-modules.md) | AUTOSARモジュール解説（SOME/IP・SoAD等） | 技術担当者 | Draft |
| [diagnostics.md](diagnostics.md) | UDS・診断通信（Dcm・Dem・J1939 DM1/DM2） | 技術担当者 | Draft |

### アーキテクチャ

| ファイル | 内容 | 対象読者 | ステータス |
|---|---|---|---|
| [architecture.md](architecture.md) | 将来アーキテクチャ・セキュリティ・ゲートウェイ設計 | 技術担当者・アーキテクト | Draft |
| [hypervisor.md](hypervisor.md) | SoC Hypervisorによる統合アーキテクチャ | 技術担当者・アーキテクト | Draft |
| [cloud-connectivity.md](cloud-connectivity.md) | クラウド連携・OTA・IoT Core | 技術担当者・アーキテクト | Draft |
| [digital-twin.md](digital-twin.md) | デジタルツインの概念・建設機械への適用 | 開発部門・経営層 | Draft |

### 開発プロセス・品質

| ファイル | 内容 | 対象読者 | ステータス |
|---|---|---|---|
| [dev-process.md](dev-process.md) | CI/CD・アジャイル・TDD/ATDD・段階的整備 | 開発部門 | Draft |
| [verification-environment.md](verification-environment.md) | MIL/SIL/HIL検証環境・テスト自動化 | 技術担当者 | Draft |
| [automation.md](automation.md) | コード生成・テスト・トレーサビリティの自動化 | 技術担当者 | Draft |
| [shift-left.md](shift-left.md) | シフトレフトの考え方・具体的手法・AI活用 | 開発部門 | Draft |

---

## 移行方針の概要

```
現状                          移行後（〜5年）              将来（5〜10年）
─────────────────────────────────────────────────────────────────────────
内製BSW・RTE（仕様書なし）  →  Classic AUTOSAR（外部委託）  →  安定稼働・機能安全認証
内製Linuxミドルウェア       →  Linuxミドルウェア（外部委託） →  Adaptive AUTOSAR検討
独自通信定義                →  DBC + ArXML 標準化           →  SOME/IP フル活用
クラウド未連携              →  Linux GW経由 IoT Core連携    →  OTA・予知保全・自律化
テストなし・CIなし          →  SIL + GitLab CI 構築         →  HIL・デジタルツイン連携
```

---

## フェーズ別アクションプラン

| フェーズ | 期間 | 主な取り組み |
|---|---|---|
| **Phase 1：土台整備** | 今〜2年 | 機能安全証跡・CI/CD・AUTOSAR PoC（Dcm+Dem）・フリート管理初版・SIL環境構築 |
| **Phase 2：移行完了** | 2〜5年 | Classic AUTOSAR本格稼働・クラウド連携・OTA（Linux系）・ISO 25119認証・HIL整備 |
| **Phase 3：SDV化** | 5〜10年 | 半自律化・Hypervisor統合検討・デジタルツイン・Adaptive AUTOSAR評価 |

---

## 関連規格

| 規格 | 内容 |
|---|---|
| ISO 25119 | 農業・建設機械の機能安全 |
| ISO 13849 | 機械安全（PLa〜PLe） |
| ISO/SAE 21434 | 自動車サイバーセキュリティ |
| ISO 14229（UDS） | 統合診断サービス |
| AUTOSAR Classic Platform | RTOS系ECU向けソフトウェア標準 |
| AUTOSAR Adaptive Platform | Linux系ECU向けソフトウェア標準（将来検討） |
| J1939 / J1939-22 | 建設・農業機械向けCAN上位プロトコル |
| SOME/IP | AUTOSAR標準のEthernetサービス通信プロトコル |
