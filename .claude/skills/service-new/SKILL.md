---
name: service-new
description: 新サービス追加スキル。サービスキー（英小文字 kebab-case）を引数にとり、services/<svc>.yml・download/references/<svc>.md の作成、local.example.yml/local.yml への追記、CLAUDE.md サービス表の更新を定型手順で行う。実機調査に Playwright MCP を使用。
---

# 新サービス追加

新しいサービスの請求書ダウンロード（＋仕訳登録）を追加する際の定型手順。
成果物は**設定ファイル群**であり、新しいスキルは作らない（単一 `/download` + `references/<svc>.md` 構成を維持する）。
どの情報をどこに置くかの設計原則は `docs/README.md` の「設定と手順の層構造」を参照。

## 引数

- **サービスキー**（必須）: 英小文字 kebab-case（例: `anthropic` / `google-ai`）。未指定の場合はユーザーに確認する。
  以降このキー `<svc>` を、設定ファイル名・references ファイル名・`invoices/<svc>/`・`local.yml` の `services:` キーの**すべてで同一に**使う。

## 原則（各 Step の手順より優先）

1. **保存先・ファイル名は規約が既定**：保存先は `invoices/<svc>/` 固定、ファイル名は
   月額 `YYYY-MM-<svc>-invoice.pdf`／年額 `YYYY-<svc>-invoice.pdf`。規約で表せない場合
   （複数ドメイン・複数サイト等で `{識別子}` が必要）のみ `services/<svc>.yml` に `filename_pattern` を定義し、
   references/<svc>.md・SKILL.md・その他ドキュメントに命名規則を重複記載しない
2. **Git 追跡ファイルに個人情報・実値を書かない**（SECURITY.md §9）。カード名・補助科目名・所有ドメイン・実金額・請求書番号・取引 ID・アカウント識別子を含む URL はすべて対象。実値は Git 管理外の `local.yml` に置き、追跡ファイル（`services/*.yml`・`references/*.md`・`local.example.yml`・CLAUDE.md 等）はプレースホルダー／`local.yml` 参照で表現する
3. **借方（勘定科目・補助科目）は config に持たない**：MF の自動仕訳ルールに委譲する。
   初回の `/register` 実行時にユーザー確認のうえ手動設定し、MF に自動仕訳ルールとして学習させる（register Step 5.2 参照）
4. 実機調査を伴うため、通常は `/phase-new` でフェーズを切ってから実施し、調査で判明した挙動・制約・ハマりどころは当該フェーズの KNOWLEDGE.md に記録する
5. ログイン画面に遷移した場合は**ユーザーにログインを依頼して停止**（認証情報は入力しない）

## Step 1: ヒアリング

実機調査の前に、ユーザーに以下を確認する：

| 項目 | 選択肢・影響先 |
|---|---|
| 請求書の入手経路 | 管理画面 / メール添付 / メール本文のみ → references の方式レシピ選択 |
| 請求頻度 | monthly / yearly → 原則 `billing_modes` を両モード定義（現ユーザーの契約は一例にすぎず、他ユーザーは逆サイクルでありうる）。サービス仕様上一方しか存在しない場合のみ単一（例: moomoo のドメイン契約は年契約のみ） |
| 複数対象の有無 | ドメイン・サイト等が複数 → `local.yml` に配列設定（moomoo / wix 型）＋ `filename_pattern` に `{識別子}` を含める |
| 支払い方法 | カード名（`local.yml` の `services.<svc>.card` に実値を設定） |

## Step 2: `services/<svc>.yml` の作成

`.claude/config/services/<svc>.yml` を以下のテンプレートで作成する。
既存の類例：`billing_modes` あり（標準）= `anthropic.yml` / `google-ai.yml` / `xserver.yml`、複数対象 = `moomoo.yml` / `wix.yml`（moomoo はサービス仕様上年契約のみのため `billing_modes` なし）。

```yaml
service_name: "<正式サービス名（画面表示用）>"
frequency: monthly            # monthly / yearly。切替可能なサービスは billing_modes を併設

# invoice: は原則書かない（保存先・ファイル名は規約。原則 1 参照）。
# 複数ドメイン・複数サイト等で {識別子} が必要な場合のみ定義する（moomoo.yml / wix.yml 参照）:
# invoice:
#   filename_pattern: "YYYY-MM-<svc>-{識別子}-invoice.pdf"

duplicate_check:
  method: by_trade_partner_code
  # 取引先で一意に特定できないサービス（同一取引先で複数ドメイン等）のみ by_memo_keyword + keyword を使う（moomoo.yml 参照）
  date_rule: <learned_anchor / from_pdf / yearly_from_pdf>
  # learned_anchor: 取引日が契約開始日で決まる月額サブスク向け。実際の請求日は /register が
  #   MF 連携データから学習して local.yml の cache.<svc>.billing_anchor に保存する
  # from_pdf / yearly_from_pdf: PDF に取引日が明記されるサービス向け

mf_search:
  # card（MF 連携カード名）は local.yml の services.<svc>.card を参照
  keyword: "<MF 連携明細の摘要に現れる文字列（実機で確認して記入）>"

memo_template: "<サービス名> {year}年{month}月請求分"
# <摘要の各変数を PDF のどこから読むかをコメントで書く>

trade_partner: "<請求書記載の正式な取引先名>"
```

- **借方・金額・税区分・初回設定は yml に書かない**：共通ルール（register/SKILL.md）と MF 自動仕訳ルールで決まる（原則 3 参照）。
  貸方も `services.<svc>.card` → `local.yml` の `credit_cards[].type` で決まるため yml に書かない

- `billing_modes` を使う場合：サイクル依存フィールド（`date_rule` / `memo_template`、例外時のみ `filename_pattern`）を `billing_modes.monthly:` / `billing_modes.yearly:` 配下に移す（`google-ai.yml` の構成をコピーする）。実データ未確認のモードは「★未確認」コメントを付けて雛形のみ定義する（`anthropic.yml` / `wix.yml` の yearly 参照）
- **契約者依存の固定日（「毎月14日」「月末」等）を yml に書かない**：`date_rule: learned_anchor` ＋ `cache.<svc>.billing_anchor` で表現する（register Step 0 参照）
- `frequency: yearly` の場合：請求月は `local.yml` の `services.<svc>.billing_month`（ユーザー設定）→ `cache.<svc>.billing_month`（`/register` が学習）の順で解決される（`/monthly` の対象月判定に使用）

## Step 3: 実機調査 → `download/references/<svc>.md` の作成

ユーザーがログイン済みであることを確認してから、Playwright MCP で請求書の取得導線を実際に確認する。

**方式レシピの選択基準**（レシピの実体は `download/SKILL.md` 参照）：

| 観察された挙動 | 使うレシピ |
|---|---|
| ダウンロードボタン/リンクで PDF ファイルが落ちる | A: download_event |
| 請求書が HTML ページでしか提供されない | B: page_pdf |
| 新規タブに PDF が直接開き download イベントが発生しない | C: fetch_base64 |
| メール添付・メール本文が証憑になる | Gmail 共通操作 ＋ A または B |

`.claude/skills/download/references/<svc>.md` を以下の構成で作成する：

```markdown
# <サービス名> 固有手順

**方式**：<レシピ名（フォールバックがあれば併記）>

## 手順

1. <起点 URL（アカウント識別子を含む場合はマスクする）>
2. <対象の特定方法（テーブル構造・検索クエリ・件名パターン等、実機で確認した値）>
3. **YYYY-MM の導出**：<どの表示・本文のどの値から導くか。曖昧さがあれば正とする基準を明記>
4. <ダウンロード操作（クリック対象のセレクタ・aria-label 等）> → リネーム移動

## 注意・落とし穴

- <実機調査で判明した罠。「次に読むモデルが同じ調査をやり直さない」粒度で書く>
- <機密扱いにすべき URL・ID があれば明記>

## 確認済み実例

- <方式>（YYYY-MM-DD 実行）：<何を実行してどう成功したか。実値（請求書番号・金額等）は形式のみ記載>
```

- セレクタ・URL・DOM 構造は**実機で確認した値だけ**を書く（推測で書かない）
- 「確認済み実例」には**実行日を必ず入れる**（DOM 変化時に鮮度を判断する材料になる）
- 年月・金額の導出元が複数あり得る場合（例: 一覧の表示日付と PDF 内の Date of issue が異なる）は、どちらを正とするかを明記する

## Step 4: `local.example.yml`（雛形）と `local.yml`(実値) への追記

1. **`local.example.yml`**（Git 管理）：`services:` に `<svc>:` キーと雛形を追記する。
   既存エントリの流儀（コメントアウトした任意キー＋**プレースホルダーのみ**）に合わせる。
   キーの説明はコメントで書かず、新しい種類のキーを導入した場合のみ
   `.claude/config/README.md`（セットアップガイド）の表へ追記する
2. **`local.yml`**（Git 管理外）：ユーザーの実値（`card` / `frequency` / `billing_month` / 配列設定等）を追記する。
   キーを追加した時点でそのサービスは「利用中」の宣言になる（`/monthly`・`全部` 指定の対象になる）

## Step 5: ドキュメント更新

1. **CLAUDE.md「サービスと保存先」表**：行を追加（設定ファイル名と保存先のみ。PII は書かない）
2. **`/download` と `/register` の SKILL.md**：frontmatter の description と「引数」のサービス名列挙に `<svc>` を追加
3. **docs/README.md**：運用スケジュール等に影響する場合のみ更新

## Step 6: 動作確認

1. `/download <svc>` を直近 1 件で実行し、`invoices/<svc>/` に `filename_pattern` どおりの名前で保存されることを確認する
2. 成功したら references/<svc>.md の「確認済み実例」に実行日付きで追記する
3. 仕訳登録まで確認する場合は `/register <svc>` を初回手動フローで実行する（借方はユーザー確認のうえ設定し、MF の自動仕訳ルールに学習させる）。yml に「★未確認」コメントがあれば実態を確認して外す

## Step 7: 完了チェックリスト

- [ ] `services/<svc>.yml`（未確定のフィールドには「★未確認」コメントを明記）
- [ ] `download/references/<svc>.md`（確認済み実例に実行日あり）
- [ ] `local.example.yml` 追記（プレースホルダーのみ）／`local.yml` 追記（実値）
- [ ] CLAUDE.md「サービスと保存先」表・`/download` `/register` の description 更新
- [ ] `/download <svc>` の動作確認済み
- [ ] **PII チェック**：`git diff` で今回変更した Git 追跡ファイルに実値（カード名・ドメイン・金額・ID 類）が混入していないか確認（SECURITY.md §9）
- [ ] 実機調査の気づきを進行中フェーズの KNOWLEDGE.md に記録
