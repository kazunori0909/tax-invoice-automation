# tax-invoice-automation ドキュメント

## はじめに（初回セットアップ）

本プロジェクトを使い始める前に、個人設定ファイル `local.yml` を作成する必要があります。
作り方・各キーの意味は [.claude/config/README.md](../.claude/config/README.md)（セットアップガイド）を参照してください。

> **`services:` に記載したサービスのみが処理対象になります。**
> 新サービスを追加・停止する場合は `services:` のキーを追加・削除するだけでスキルに反映されます。

---

## 運用スケジュール

### 月次作業（毎月）

| タイミング | スキル | 内容 |
|---|---|---|
| 月末〜翌月初（請求書が出たら） | `/monthly` | 月次サービスの請求書ダウンロード → MF 仕訳登録 → 証憑 PDF 添付 |
| カード引き落とし後 | `/credit-payment` | 未払金型（`type: business`）カードの引き落とし仕訳の確認（`local.yml` の `credit_cards:` が対象。個人用カードは事業主借直接のため対象外） |

> 引き落とし日はカードごとに `local.yml` の `credit_cards`（`closing`/`payment_day`）で決まる。土日祝の場合は翌営業日。

### 年次作業

| タイミング | スキル | 内容 |
|---|---|---|
| 各サービスの請求月 | `/monthly` | 年次サービスの請求書ダウンロード → MF 仕訳登録 → 証憑 PDF 添付（`local.yml` の `billing_month` で自動判定） |

---

## スキル一覧

各スキルの主な処理は「請求書 PDF のダウンロード」「MF 未仕訳明細の特定・仕訳登録」「証憑 PDF の添付」です。

### メインスキル

| スキル | いつ使う | 説明 |
|---|---|---|
| `/monthly` | 月末〜翌月初 | `local.yml` の `services:` を読み、当月対象サービスの請求書ダウンロード → 仕訳登録を一括実行 |
| `/credit-payment` | カード引き落とし後 | 未払金型（`type: business`）カードの引き落とし仕訳を確認（`local.yml` の `credit_cards:` が対象。個人用カード＝事業主借直接は対象外） |

### 個別スキル（再実行・単体テスト用）

| スキル | 説明 |
|---|---|
| `/download [service\|全部]` | 請求書 PDF のダウンロード。共通フロー・方式レシピは `download/SKILL.md`、サービス固有手順は `download/references/<svc>.md` に分離。`全部` 指定で `local.yml` の `services:` の全サービスを処理 |
| `/register [service\|全部]` | MF 仕訳登録・証憑 PDF 添付。`全部` 指定で `local.yml` の `services:` の全サービスを処理 |

> サービスの請求サイクル（月額/年額）は `local.yml` の `services.<svc>.frequency` で切替（`services/*.yml` の `billing_modes` を持つサービスのみ）。

### 開発運用スキル（フェーズ管理）

| スキル | 説明 |
|---|---|
| `/phase-new [テーマ]` | 次のフェーズ番号を採番し `docs/phase<N>-<slug>/` に PLAN.md・TASKS.md の雛形を作成、下記フェーズ記録テーブルへ行を追加 |
| `/phase-compress [番号...]` | 完了済みフェーズを [phases.md](phases.md) に統合してフォルダを削除（**ユーザーが圧縮を指示したときのみ実行**） |
| `/service-new [サービスキー]` | 新サービスの請求書ダウンロード追加。`services/<svc>.yml`・`download/references/<svc>.md` の作成、`local.yml` 追記、ドキュメント更新を定型手順で実施 |

---

## 設定と手順の層構造

本プロジェクトのファイルは**データ層（config）と手順層（skills）**に分離している。
サービスに関する情報を追加・変更するときは、この表で置き場所を決める（本セクションが層構造の説明の正）。

| 層 | パス | Git | 内容 |
|---|---|---|---|
| データ層：汎用仕様 | `.claude/config/services/<svc>.yml` | 管理 | サービスの宣言的仕様（頻度・検索キーワード・摘要テンプレ・取引先・重複チェック・命名規則の例外）。誰の契約でも変わらない値のみ |
| データ層：個人値・キャッシュ | `.claude/config/local.yml` | 管理外 | `services:`（カード名・ドメイン等の個人設定）・`cache:`（スキルが学習・自動生成する値）・`credit_cards:` |
| 手順層：共通手順 | `.claude/skills/<skill>/SKILL.md` | 管理 | 全サービス共通の手順。サービス差分はデータ層のパラメータで表現する |
| 手順層：サービス固有差分 | `.claude/skills/download/references/<svc>.md` | 管理 | ダウンロードの取得導線（URL・セレクタ・方式レシピ選択）。消費者が `/download` のみのためスキル配下に置く |

### 設計原則

- **`services/<svc>.yml` は複数スキルが読む共有データの SSOT**。下記マップのとおり 4 スキルが参照するため、特定スキルの配下ではなく中立な `config/` に置く。スキル配下へ移すと、共有フィールドの重複か、他スキルのディレクトリを参照する越境が発生する
- **手順はスキルへ、データは config へ**。サービス固有の情報でも、「手順」（導線・クリック対象・落とし穴）なら `download/references/<svc>.md`、「データ」（値・パターン・ルール）なら `services/<svc>.yml`（汎用）または `local.yml`（個人値）
- **利用者がカスタマイズするのは config のみ**。skills は共通ロジックであり、サービスの追加・変更で編集するのは原則データ層（＋download の references）だけで済むようにする

### 保存先・ファイル名・借方の規約（yml に書かない値）

- **保存先**：`invoices/<サービスキー>/` 固定
- **ファイル名**：月額 `YYYY-MM-<svc>-invoice.pdf`／年額 `YYYY-<svc>-invoice.pdf`。
  複数ドメイン・複数サイト等で規約で表せないサービスのみ yml の `filename_pattern` で例外を定義（moomoo / wix）
- **借方（勘定科目・補助科目）**：MF の自動仕訳ルールに委譲。初回の `/register` のみユーザー確認のうえ手動設定し、MF に学習させる

### `services/<svc>.yml` のフィールド×消費者マップ

| フィールド | `/download` | `/register` | `/monthly` | `/service-new` |
|---|---|---|---|---|
| `invoice.filename_pattern`（規約の例外時のみ。`billing_modes` 解決含む） | ✅ Step 0 | ✅ Step 3 | ✅ Step 2（スキップ判定） | 作成 |
| `frequency` / `billing_modes` | ✅ Step 0 | ✅ Step 0 | ✅ Step 0（スコープ判定） | 作成 |
| `duplicate_check` | – | ✅ Step 1 | ✅ Step 1 | 作成 |
| `service_name` / `mf_search` / `memo_template` / `trade_partner` | – | ✅ Step 2〜7 | （register 経由） | 作成 |

---

## 開発フェーズ記録

過去の開発フェーズの**重要な変更点・仕様決定**は各フェーズのフォルダ（`docs/phase*/`）に記録。
Phase 1–24 は [phases.md](phases.md) に統合済み（「なぜ今の構成になっているか」の単一履歴）。

| # | フェーズ | 要点 | 記録 |
|---|---------|------|------|
| 1 | プロジェクト構成の整理 | 案B（手動ログイン + Playwright MCP）への切り替え | [phases.md](phases.md) |
| 2 | Playwright MCP セットアップ | `.mcp.json`・プロファイル永続化・ヘッドレスオフ | [phases.md](phases.md) |
| 3 | 請求書ダウンロード Skill | サービス別の取得導線・命名規則の確定 | [phases.md](phases.md) |
| 4 | セキュリティ対応 | `.env` deny・`SECURITY.md` 策定 | [phases.md](phases.md) |
| 5 | MF クラウド会計 MCP 接続 | OAuth・ベータ版 URL・`mcp-remote` 経由 | [phases.md](phases.md) |
| 6 | 仕訳登録の半自動化 | PDF 起点 → データ連携明細起点へ転換／仕訳ルール確定 | [phases.md](phases.md) |
| 7 | 月次一括処理 `/monthly` | 委譲方式・エラー分離・冪等性 | [phases.md](phases.md) |
| 8 | カード引き落とし `/credit-payment` | 未払金→事業主借の検知・登録（YAML 駆動） | [phases.md](phases.md) |
| 9 | Skill 共通化 | `_shared/`（pdf-attach・security）抽出 | [phases.md](phases.md) |
| 10 | カード設定管理改善 | ID 動的解決・`credit-cards.yml` gitignore 化 | [phases.md](phases.md) |
| 11 | register-* 統合 | `services/*.yml` + 単一 `/register` | [phases.md](phases.md) |
| 12 | サービス設定の分離 | 固定情報→Git管理、個人設定→`local.yml` 統合 | [phases.md](phases.md) |
| 13 | 個人情報マスキング | 稼働ファイルの PII を除去（public 化の前処理） | [phases.md](phases.md) |
| 14 | billing_modes ＋ MF 追加 | 請求サイクル切替・MF クラウドをサービス追加 | [phases.md](phases.md) |
| 15 | リファクタリング | 仕訳ルールを yml へ移設・CLAUDE.md スリム化・PII ルール集約 | [phases.md](phases.md) |
| 16 | register 取引日修正 ✅ | MF 連携データの実取引日を date_rule より優先採用・重複チェック範囲拡張 | [phases.md](phases.md) |
| 17 | debit_sub_account の local.yml 移管 ✅ | 補助科目名を config から local.yml へ移管（後に MF 自動仕訳への全面委譲へ置き換え。詳細は phases.md に非掲載） | [phases.md](phases.md) |
| 18 | Wix サービス追加 | 支払い履歴からの PDF 取得導線を確立・`services/wix.yml` を整備。仕訳登録の初回確定は未実施（未確定事項として記録） | [phases.md](phases.md) |
| 19 | Wix マルチサイト対応 | `wix.site` → `wix.sites[]` へ変更しファイル名にサイト名を追加。複数サイト時の表示は未確認（未確定事項として記録） | [phases.md](phases.md) |
| 20 | local.yml 可読性改善 ✅ | STEP 1/2 構成・`[DL]`/`[REG]` タグでダウンロード専用/仕訳登録用を明示（後の簡略化で解消。詳細は phases.md に非掲載） | [phases.md](phases.md) |
| 21 | local.yml 簡略化 ✅ | Wix の `plan` を請求書 PDF からの読み取りに一本化・`trade_partner_code` 等を `cache:` セクションへ分離 | [phases.md](phases.md) |
| 22 | Google AI 一般化＋年払い対応 ✅ | プラン名（Pro 等）依存の記述を除去・`billing_modes` 導入・サービスキーを `google` → `google-ai` にリネーム | [phases.md](phases.md) |
| 23 | フェーズ管理のスキル化 ✅ | `/phase-new`（フェーズ追加の雛形作成）・`/phase-compress`（完了フェーズの phases.md への圧縮）を追加 | [phases.md](phases.md) |
| 24 | download スキル単一化 ✅ | `download-*` 6本を単一 `/download` + `references/<svc>.md` に統合・命名規則を `services/*.yml` へ一元化・PII 除去 | [phases.md](phases.md) |
| 25 | 新サービス追加のスキル化（進行中） | `/service-new`：yml・references 作成〜動作確認の手順と PII 等のルールを明文化（他モデルでの精度向上） | [phase25/](phase25-service-new-skill/) |
| 26 | 請求日アンカーの学習・キャッシュ化（進行中） | 契約者依存の請求日（14日・月末等）を `services/*.yml` から除去し、MF 連携データから観測して `local.yml` の `cache:` に学習 | [phase26/](phase26-billing-anchor-cache/) |
| 27 | config 層構造の明文化 ✅ | `config/services/*.yml` の配置は現状維持と決定（案A）。データ層/手順層の設計原則とフィールド×消費者マップを明文化し、分散した説明を統合 | [phase27/](phase27-config-layering-docs/) |
| 28 | カード種別対応（個人用＝事業主借直接） | 個人用カードの課金を `通信費/事業主借/<カード補助科目>` の直接記帳に統一（未払金型カードを事業主借直接へ寄せる）。`credit_cards.type` 導入・`journal_rule` の貸方をカード依存（`services.card`→`credit_cards.type`）に・credit-payment を未払金型（事業用）専用に | [phase28/](phase28-card-type/) |
| 29 | config スリム化（進行中） | 全フィールドを消費者と突き合わせ、デッド・情報ゼロ・規約導出可能なフィールドを削減。借方の勘定科目・補助科目の MF 自動仕訳委譲を検討 | [phase29/](phase29-config-slimming/) |
| 30 | Moomoo ドメイン設定の name フィールド削除 ✅ | `domains[].name` を削除し `fqdn` に完全一本化。ファイル名は `.`→`_` 置換した `{domain_fqdn_us}`、摘要は無変換の `{domain_fqdn}` を使用 | [phase30/](phase30-moomoo-domain-name-field-removal/) |

---

## 進捗管理ルール

- **セッション内**：Claude Code の TodoWrite で揮発的に追跡
- **セッション横断**：各フェーズの `TASKS.md` にチェックボックスで永続化
- セッション開始時：該当 `TASKS.md` を読んで未完了タスクを TodoWrite に展開
- タスク完了時：TodoWrite と `TASKS.md` の両方を更新
- フェーズ完了時：本ファイルのステータス表を更新

## アーキテクチャ方針（案B）

- **認証情報を保持しない**：ユーザーがブラウザで手動ログイン
- **Playwright MCP** で Claude Code がブラウザ操作（永続的なユーザーデータディレクトリでログイン状態を維持）
- **ダウンロードは単一の `/download` スキル**＋サービス固有手順の `references/<svc>.md`（`/download anthropic` 等で個別実行可能）
- 月1回の半自動化が目的（完全自動化は過剰）
