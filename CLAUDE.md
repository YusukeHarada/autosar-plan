# CLAUDE.md

このリポジトリはAUTOSAR準拠プラットフォームソフトウェア移行、およびSDV化に向けた建設機械向け総合検討資料をまとめたものである。

---

## リポジトリの目的

建設機械向けプラットフォームソフトウェアを、現状の内製開発体制からAUTOSAR準拠・外部委託体制へ移行し、段階的にSDV化を進めるための検討資料を管理する。

## ドキュメント構成

| ファイル | 内容 | 対象読者 |
|---|---|---|
| `proposal.md` | 社内提案資料 | 非エンジニアの管理職にも伝わる平易な文体 |
| `sdv.md` | SDV概念・建設機械への考察・ロードマップ | 開発部門・経営層 |
| `roadmap.md` | フェーズ別ロードマップ・必要技術・人材 | 開発部門・管理職 |
| `vendor-selection.md` | 委託先選定基準・PoC評価 | 技術担当者向けチェックリスト |
| `requirements-spec.md` | 要求仕様の作り方 | AUTOSARの詳細知識がなくても使えるレベル |
| `communication-spec.md` | CAN・Ethernet・J1939通信仕様の書き方 | 技術担当者・委託先 |
| `toolchain.md` | AUTOSARコード生成ツールチェーン | 技術担当者 |
| `autosar-modules.md` | AUTOSARモジュール解説（SOME/IP・SoAD等） | 技術担当者 |
| `hypervisor.md` | SoC Hypervisorアーキテクチャ | 技術担当者・アーキテクト |
| `cloud-connectivity.md` | クラウド連携・OTA・IoT Core | 技術担当者・アーキテクト |
| `dev-process.md` | CI/CD・アジャイル・TDD/ATDD | 開発部門 |

## 背景・前提

- 対象：建設機械向けプラットフォームソフトウェア（BSW・RTE相当）
- RTOS系：油圧制御等の安全クリティカルな制御系（Classic AUTOSAR領域）
- Linux系：HMI・ゲートウェイ・人検知・クラウド通信（Linuxミドルウェア、Adaptive AUTOSARは将来検討）
- 機能安全（ISO 25119）およびサイバーセキュリティへの対応が必要
- 外部委託でClassic AUTOSAR準拠システムを構築し、内製コードを段階的に廃止する方針
- クラウド連携はLinuxゲートウェイ経由でAWS IoT Core / Azure IoT Hubに接続
- Hypervisor統合は2028年以降の次世代機で検討

## 用語統一ルール

- Classic AUTOSAR（× AUTOSAR Classic）
- Adaptive AUTOSAR（× AUTOSAR Adaptive Platform、AP）
- ArXML（× arxml、ARXML）
- J1939（× SAE J1939）
- SOME/IP（× SomeIP、someip）

## AIアシスタントへの注意事項

- **具体的な数値・製品名・バージョン番号は自社確認済み情報以外は「要確定」と記載する**
- `proposal.md` は非エンジニアの幹部向けのため専門用語には必ず補足を付ける
- `requirements-spec.md`・`communication-spec.md` のサンプル値は仮の値。実際のシステムへの具体化はユーザー自身が行う
- 各ドキュメントは独立して読める構成にする（委託先・幹部が単独で受け取る場合を想定）
- ステータスはDraft→In Review→Approvedで管理し、READMEの一覧表を更新する
