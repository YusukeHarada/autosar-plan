# AUTOSAR開発における自動化戦略

建設機械メーカーが外部委託でAUTOSAR準拠プラットフォームを構築し、自社担当者1〜2名で要求定義・受け入れ検証を担う前提での自動化方針。

---

## 1. 自動化できるもの・できないものの整理

| 領域 | 自動化可否 | 判断主体 |
|------|-----------|---------|
| ArXMLからのコード生成 | 可 | ツール |
| ビルド・静的解析 | 可 | ツール |
| SIL/通信テスト実行 | 可 | ツール |
| 要求IDとテストのトレース | 可（半自動） | ツール＋人 |
| 要求の妥当性確認 | 不可 | 自社担当者 |
| ArXML設計判断（ポート・インタフェース） | 不可 | 担当者＋委託先 |
| 安全目標の解釈・ASILアサイン | 不可 | 担当者（機能安全担当者） |

**原則:** ツールは「正しく実装されているか」を確認するが、「何が正しいか」は人が決める。

---

## 2. コード生成の自動化

### ArXML → RTE・BSWコンフィグコード

```yaml
# GitLab CI例
generate:
  stage: codegen
  script:
    - python tools/dbc2arxml.py input/network.dbc -o arxml/Com.arxml
    - vendor_tool/arxml2rte.sh arxml/ -o generated/
  artifacts:
    paths: [generated/]
```

- **DBC → ArXML変換:** `cantools` や社内スクリプトで自動化。CAN信号定義の変更がArXMLに即反映される
- **RTE生成:** Vector DaVinci / EB tresos のCLIを使いCI上で実行
- **ビルド:** CMake + Ninja構成、GitLab CIでステージ分割（generate → build → test）

---

## 3. テスト自動化

### SILテスト（CIパイプライン組み込み）

```yaml
sil_test:
  stage: test
  script:
    - cmake --build build/ --target sil_test
    - ./build/sil_test --gtest_output=xml:results/sil.xml
  artifacts:
    reports:
      junit: results/sil.xml
```

### CANoe CLI（通信テスト）

```bash
# CANoe CLIで非対話実行
CANoe64.exe /cfg test.cfg /run /test TestModule1 /quit /exitcode
```

通信テストのパス・フェイルをCIのexit codeで判定し、GitLabのパイプラインに統合する。

### Robot Frameworkによる受け入れテスト

```robot
*** Test Cases ***
ブレーキ指令の応答確認
    Send CAN Message    0x200    speed=0 brake=1
    Verify Signal       0x300    brake_active    1    timeout=100ms
```

要求IDをテスト名に含めることで、後述のトレーサビリティ自動収集と連携できる。

### MISRA-C静的解析

- **Polyspace / Helix QAC:** CLIでCI実行し、違反レポートをアーティファクトとして保存
- 委託先に「CI通過を納品条件」として契約仕様に明記するのが効果的

---

## 4. 要求管理・トレーサビリティの自動化

```
要求ID（Polarion / GitLab Issue）
  ↓ 紐付け
ArXML要素（SoftwareComponent / Port）
  ↓ 紐付け
テストケース（Robot Framework / gtest）
  ↓ 自動収集
機能安全証跡レポート（PDF / HTML）
```

- **Polarion ALM:** 要求とテスト結果をAPI連携で自動紐付け
- **GitLab Issuesのみの場合:** ラベル・カスタムフィールドで要求IDを管理し、CI結果をIssueにコメント投稿するスクリプトで代替可能
- **証跡レポート:** CIの成果物（テスト結果XML・静的解析レポート）をまとめてPDFに変換するスクリプトを用意する

---

## 5. ArXML差分レビューの自動化

委託先から受け取ったArXMLを自動検証するスクリプト例：

```python
# arxml_check.py
import xml.etree.ElementTree as ET

def check_required_ports(arxml_path, requirements_csv):
    tree = ET.parse(arxml_path)
    # 要求仕様に定義されたポートが全て存在するか確認
    ...
```

- **差分確認:** `git diff` でArXMLの変更箇所を可視化。構造的なdiffにはXML diffツール（`xmldiff`等）を活用
- **整合チェック:** 要求仕様（CSV/Excel）から期待するインタフェース一覧を生成し、ArXMLと突き合わせる

---

## 6. 建設機械向け自動化優先順位

### 今すぐ始める（コストゼロ〜低コスト）

1. GitLab CI設定ファイル（`.gitlab-ci.yml`）の作成 → ビルド自動化
2. `git diff` によるArXML変更の可視化
3. MISRA違反レポートのCI自動収集（既存ツールのCLI活用）
4. Robot Frameworkによる受け入れテストの雛形作成

### 中期的に投資すべき（効果大）

1. **DBC → ArXML変換スクリプト:** 仕様変更のたびに手作業が発生するため、早期に自動化するとROIが高い
2. **トレーサビリティ自動生成:** 担当者1〜2名では手動管理が限界になるため、Polarion連携またはスクリプトによる証跡収集を優先
3. **SILテストのCI統合:** 委託先の実装品質を継続的に可視化できるため、受け入れ工数を大幅に削減できる
