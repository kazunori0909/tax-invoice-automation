---
name: monthly
description: 月次一括処理。local.yml の services を対象に、請求書ダウンロード → MF 未仕訳確認 → 仕訳登録を一連で実行する。月次サービスは毎回、年次サービスは請求月（billing_month）のみ実行。
---

# 月次一括処理 Skill

## 前提条件

- Playwright MCP が設定済み
- MoneyForward MCP が認証済み
- ユーザーが対象ブラウザ（claude.ai・Google Pay・MF クラウド）にログイン済み

## 引数

ユーザーから対象年月（例：「2026年5月」「先月」「直近」）を指定してもらう。  
未指定の場合は **当月（今日の日付基準）** を対象とする。

---

## Step 0: 処理スコープの決定

### 0.1 利用中サービスの列挙（local.yml 由来）

**処理対象サービスはハードコードしない。** `.claude/config/local.yml` を Read し、
`services:` 直下のキーを「利用中サービスの一覧」とみなす（キーの有無＝利用中の宣言）。
`local.yml` に存在しないサービスは、`services/<svc>.yml` テンプレが存在しても処理しない。

各サービスの頻度は以下の優先順位で決定する（`local.yml` による上書きを優先）：

1. `local.yml` の `services.<svc>.frequency`（設定されている場合）
2. `services/<svc>.yml` の `frequency`（デフォルト値）

これにより、同じサービスでも月額契約・年額契約を `local.yml` 側で切り替えられる。
値は `monthly` / `yearly` / `yearly_per_domain` のいずれか。

### 0.2 対象年月による絞り込み

対象年月を確定し、列挙した各サービスを頻度に応じてスコープに含めるか判断する。

| frequency | スコープ判定 |
|-----------|------------|
| `monthly` | 毎月対象 |
| `yearly_per_domain` | `local.yml` の `services.<svc>.domains` を読み、対象月 = 各ドメインの `billing_month` に一致するドメインのみ対象（例: Moomoo。サービス仕様上、年契約のみ） |
| `yearly` | 請求月に一致する月のみ対象（翌月は未登録の場合のリカバリとして追加）。請求月は `local.yml` の `services.<svc>.billing_month` → `cache.<svc>.billing_month`（register が学習した値）の順で解決し、どちらも未設定の場合はユーザーに確認 |

ダウンロード・登録は `/download <svc>` / `/register <svc>` を引数付きで実行する。

処理スコープをユーザーに提示し、続行を確認する：

```
処理スコープ（2026年5月）:
  📥 ダウンロード対象：Anthropic, Google
  📋 登録対象（確認後）：Anthropic, Google

年次サービス（Xserver・Moomoo）は今月対象外のためスキップします。

続行してよいですか？
```

---

## Step 1: 重複チェック（スコープ内の全サービス）

ダウンロード前に MCP で MF 仕訳を確認し、各サービスの登録状況を把握する。  
これにより「ダウンロード済み・登録済み」のサービスを事前にスキップできる。

Step 0 で列挙したスコープ内サービスをループし、各サービスについて
`mfc_ca_getJournals` で対象期間（対象月の前後 1 ヶ月程度）を取得して登録済みかを判定する。

判定方法は `services/<svc>.yml` の `duplicate_check` に従う（register Skill Step 1 と同一ロジック）：

- `method: by_memo_keyword` → 摘要に `duplicate_check.keyword` を含む仕訳を検索
- `method: by_trade_partner_code` → `local.yml` の `cache.<svc>.trade_partner_code` で検索
- `frequency: yearly_per_domain` → `local.yml` の `services.<svc>.domains` の各ドメインについて個別判定
  （摘要に `duplicate_check.keyword`（`{domain_fqdn}` を各ドメインの `fqdn` に置換）を含むか）

取引先コードは `local.yml` の `cache.<svc>`、ドメイン名は `local.yml` の `services.<svc>` から取得する。

---

## Step 2: ダウンロード処理

Step 0 でスコープに含めた各サービスについて、順番にダウンロードする
（依存関係はないが、ブラウザ操作のため直列実行）。サービス名はハードコードせず、
Step 0 で列挙した一覧をループする。

各サービスごとに以下を行う：

1. 対象 PDF のパスを組み立てる。保存先は `invoices/<svc>/`（規約・固定）、ファイル名は規約
   （月額 `YYYY-MM-<svc>-invoice.pdf`／年額 `YYYY-<svc>-invoice.pdf`）。
   - `services/<svc>.yml` に `filename_pattern` の定義がある場合（規約で表せない例外）はそちらを使う。
     `billing_modes` を持つサービスでは実効 frequency（`local.yml` 優先）でモードを解決する
     （register Step 0 のモード解決と同一）。
   - `frequency: yearly_per_domain`（例: Moomoo）の場合は、`local.yml` の `services.<svc>.domains`
     の各ドメインについて `{domain_fqdn_us}`（`fqdn` の `.` を `_` に置換した値）を差し替えて繰り返す。
2. 対象 PDF が既に存在すればスキップ。
3. 存在しなければ `/download` スキル（`<svc>` を引数に指定）の手順に従いダウンロードする
   （共通レシピは `download/SKILL.md`、サービス固有手順は `download/references/<svc>.md`）。

**ダウンロード完了後、取得済みファイルを列挙してユーザーに確認を求める：**

```
ダウンロード完了:
  ✅ invoices/anthropic/2026-05-anthropic-invoice.pdf（新規）
  ✅ invoices/google-ai/2026-05-google-ai-invoice.pdf（新規）
  ⏭ invoices/google-ai/2026-04-google-ai-invoice.pdf（既存、スキップ）

次のステップ（仕訳登録）に進んでよいですか？
```

---

## Step 3: 統合処理計画の表示

Step 1 の重複チェック結果と Step 2 のダウンロード結果をもとに、  
登録が必要なサービスと件数を統合表示してユーザーに最終確認を求める。

```
月次処理計画（2026年5月）:

  ■ Anthropic (Claude)
    ✅ 2026-05-31: 未登録 → 登録予定

  ■ Google AI
    ✅ 2026-05-14: 未登録 → 登録予定

  ■ Xserver
    ⏭ 今月対象外（4月〜5月の年次処理は完了済み）

  ■ ムームードメイン
    ⏭ 今月対象外（2月・4月の年次処理は完了済み）

合計 2 件を登録します。よろしいですか？
```

未登録件数が 0 件の場合は「すべて登録済みです」と表示して終了。

---

## Step 4: 仕訳登録処理

ユーザーの承認後、Step 0 でスコープに含めた各サービスを順番に登録する
（サービス名はハードコードせず、列挙した一覧をループ。ブラウザ操作のため直列実行）。

各サービスについて、**register Skill**（`<svc>` を引数に指定）の Step 3〜7 に従い実行する。  
（Step 1〜2 の重複チェック・確認は本 Skill の Step 1・3 で実施済みのためスキップ可）  
各サービスの詳細手順（未仕訳明細の検索・摘要入力・PDF添付）は register Skill と
`services/<svc>.yml` + `local.yml` のマージ設定に従う。

---

## Step 5: 最終レポート

全サービスの処理完了後、結果を集計して表示する。

```
月次処理完了レポート（2026年5月）:
────────────────────────────────────
  Anthropic (Claude)
    ✅ 2026-05-31: ¥3,XXX（仕訳No.XX）証憑1件添付

  Google AI
    ✅ 2026-05-14: ¥X,XXX（仕訳No.XX）証憑1件添付

  Xserver
    ⏭ 今月対象外

  ムームードメイン
    ⏭ 今月対象外
────────────────────────────────────
  新規登録: 2 件
  スキップ（登録済み）: 0 件
  スキップ（対象外）: 2 件

すべての月次処理が完了しました。
```

---

## 年次サービスの判断フロー

```
frequency = yearly_per_domain のサービス（例: Moomoo）
  → local.yml の services.<svc>.domains を読み、
    billing_month が対象月に一致するドメインをスコープに追加
frequency = yearly のサービス
  → 請求月を解決（services.<svc>.billing_month → cache.<svc>.billing_month → ユーザーに確認）
  → 対象月 = 請求月: スコープに追加
  → 対象月 = 請求月の翌月: 未登録の場合のみスコープに追加（リカバリ用）
上記いずれにも該当しない年次サービス
  → スキップ（スキップ理由をレポートに記載）
```

---

## エラーハンドリング

共通エラー（MCP 認証切れ・ブラウザ未ログイン等）は `.claude/skills/_shared/security.md` を参照。
本スキル固有：

| 状況 | 対応 |
|------|------|
| ダウンロード失敗（1サービス） | そのサービスをスキップして次に進む。最終レポートに失敗理由を記載 |
| MF 未仕訳明細が見つからない | ユーザーに報告し、カード連携・同期状況を確認するよう依頼。そのサービスをスキップ |
| PDF が破損または空 | ユーザーに報告して手動再ダウンロードを依頼 |
| 金額が想定外 | 自動登録を停止し、ユーザーに確認してから手動で判断 |

**基本方針：** 1サービスの失敗が他サービスの処理を止めない。  
失敗したサービスは最終レポートに記載し、`/register <service>` で個別に再実行できる旨を案内する。

---

## セキュリティ注意事項

`.claude/skills/_shared/security.md` を参照。
各サービス固有の注意事項は `.claude/config/services/{service}.yml`（汎用仕様）と `.claude/config/local.yml` の `services.{service}`（個人設定）、および `register/SKILL.md` を参照。
