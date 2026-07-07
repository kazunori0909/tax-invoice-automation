# Phase 27: config 層構造の設計原則明文化と説明断片の統合

## 課題

1. **`config/services/*.yml` が「なぜ config にあるのか」がどこにも明文化されていない**。
   Phase 24 で `download/references/<svc>.md` がスキル配下に置かれたことで、
   「サービス関連ファイルはスキル配下（download / register）に寄せるべきでは」という
   疑問が自然に生じる状態になっている。実際には `services/<svc>.yml` は
   download / register / monthly / service-new の 4 スキルが読む共有データであり、
   特定スキル配下へ移すと共有フィールドの重複かスキル境界を越えた参照が必ず発生する。
2. **`services/*.yml` の役割・参照方法の説明が複数ファイルに断片的に分散している**
   （register/SKILL.md Step 0・service-new/SKILL.md・CLAUDE.md「サービスと保存先」・
   docs/README.md）。層構造の全体像（データ層と手順層の分離）を 1 箇所で説明する場所がない。

## 方針

配置は現状維持し、設計原則を明文化する（案A）。比較した案と採否：

- **案A（採用）**：`config/services/*.yml` の配置は現状維持。docs/README.md に
  「config = 宣言的データ層（Git 管理の汎用仕様／Git 管理外の個人値・キャッシュ）、
  skills = 手順層、download/references = 手順のサービス固有差分」という層構造と、
  `services/<svc>.yml` のフィールド×消費者マップを明文化する。
  他ファイルの断片的な説明は正とする場所へのリンクに寄せる
- **案B（不採用）**：サービス軸への統合（`config/services/<svc>/` に spec.yml + download.md）。
  1 サービスの情報が 1 箇所に集まるが、download スキルの参照資料がスキル外に出て
  自己完結性が下がり、移行コストに見合う効果がない
- **案C（不採用）**：`services/*.yml` を download / register 配下へ移設・分割。
  `filename_pattern`・`frequency`・`billing_modes` 等の共有フィールドの重複、
  または他スキルのディレクトリを参照する越境が発生し、Phase 11（register-* 統合）・
  Phase 24（命名規則の yml 一元化）・Phase 26（billing_modes 統一）の決定と矛盾する

### `services/<svc>.yml` のフィールド×消費者マップ（明文化する内容の元データ）

| フィールド | download | register | monthly | service-new |
|---|---|---|---|---|
| `invoice.dir` / `filename_pattern`（billing_modes 解決含む） | ✅ Step 0 | ✅ Step 3 | ✅ Step 2 | 作成 |
| `frequency` / `billing_modes` | ✅ | ✅ | ✅ Step 0 | 作成 |
| `duplicate_check` | – | ✅ Step 1 | ✅ Step 1 | 作成 |
| `mf_search` / `memo_*` / `trade_partner` / `journal_rule` | – | ✅ | （register 経由） | 作成 |

## 決定事項ログ

### 2026-07-04: 案A（配置現状維持＋明文化）を採用

- 「`services/*.yml` を download / register 配下へ移せないか」の壁打ちの結果、
  上記の案A/B/C 比較を経て案A を採用（ユーザー承認済み）
- 理由：`services/<svc>.yml` は 4 スキルが読む共有データであり、SSOT を中立な
  config 層に置く現構成が正しい。違和感の正体は配置ではなく「層構造が未明文化」であること
- パブリック公開方針とも整合：「skills はいじらない、config を書く」という
  利用者向けの境界を保つ
