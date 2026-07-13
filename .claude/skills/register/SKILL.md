---
name: register
description: 仕訳登録スキル。サービス名（anthropic / google-ai / moomoo / xserver 等）または「全部」を引数にとり、対応するサービス設定と共通テンプレートで処理する。「全部」指定時は local.yml の services キーを利用中サービスとして列挙する。MoneyForward MCP + Playwright MCP を使用。
---

# 仕訳登録

## 引数

- **サービス名**（必須）: 個別サービス名（`anthropic` / `google-ai` / `moomoo` / `xserver` 等）または `全部` / `all`
- **期間**（任意）: 「2026年全部」「2026-05」「直近」など
- 引数未指定の場合はユーザーに確認する

### `全部` / `all` が指定された場合

ハードコードした一覧ではなく、**`.claude/config/local.yml` の `services:` セクションのキー**を
利用中サービスの一覧とみなす（キーの有無＝利用中の宣言）。

1. `local.yml` を Read し、`services:` 直下のキー（例: `anthropic` / `google-ai` / ...）を列挙する。
2. 各サービスについて、対応する `services/<key>.yml` が存在することを確認する（無ければユーザーに報告）。
3. 列挙した各サービスに対して、以下の Step 0〜7 を順番に実行する。
4. 重複チェック（Step 1）で登録済みのものは自動スキップされるため、頻度（月次/年次）に関わらず安全に一括実行できる。

`local.yml` の `services:` に存在しないサービスは「未利用」として処理しない。

## 前提条件

- MoneyForward MCP が有効・認証済み（無効化されている場合は `.claude/skills/_shared/security.md`
  の「MoneyForward MCP 有効化チェック」を実施。未認証の場合は `mfc_ca_authorize` → `mfc_ca_exchange` を実施）
- Playwright MCP が設定済み、かつマネーフォワードにログイン済み
- 対象サービスの請求書 PDF が所定の `invoices/{service}/` に存在する

## 手順

### 0. サービス設定を読み込む

設定は2ファイルに分かれている。両方を Read してマージし、1つの `config` として扱う。

1. **汎用テンプレ（Git 管理）**：`.claude/config/services/{service}.yml`
   サービス仕様（検索キーワード・摘要テンプレ・取引先名・重複チェック等）。
2. **個人設定（Git 管理外）**：`.claude/config/local.yml` の `services.{service}`
   `card`（MF ホーム「未仕訳N件」リンクの表示名）、`domains`（moomoo のみ）、
   `frequency`（オプション。月額/年額契約の違いを上書きする場合に指定）。
3. **スキル自動生成キャッシュ（Git 管理外）**：`.claude/config/local.yml` の `cache.{service}`
   `trade_partner_code` など、ユーザーが設定するのではなくスキルが自動取得して書き戻す値。

対象サービス `<svc>` に対し、`services/<svc>.yml`（汎用テンプレ）と `local.yml` の `services.<svc>`（個人設定）・`cache.<svc>`（キャッシュ）を読み込む。
利用可能なサービスは `local.yml` の `services:` に列挙されているキー（`anthropic` / `google-ai` / `moomoo` / `xserver` / `moneyforward` 等）。

**マージ後の `config.*` の参照元：**

| 参照 | 取得元 |
|------|--------|
| `config.mf_search.card` | `local.yml` の `services.{service}.card` |
| `config.duplicate_check.trade_partner_code` / `config.trade_partner_code` | `local.yml` の `cache.{service}.trade_partner_code`（スキル自動生成キャッシュ） |
| `config.domains`（moomoo） | `local.yml` の `services.moomoo.domains` |
| 上記以外（`service_name`・`mf_search.keyword`・`trade_partner`・`memo_template` 等） | `services/{service}.yml` |

**借方（勘定科目・補助科目）は config に持たない**：MF の自動仕訳ルールに委譲する（Step 5 参照）。

`local.yml` に対象サービスのキーが存在しない場合は「未利用サービス」として扱い、ユーザーに確認して停止する。

### カード種別・貸方（credit）の解決

カード決済サービス（`services.{service}.card` が設定されている）の貸方（勘定科目・補助科目）は
サービス固定ではなく**データ連携カード**で決まる。以下で解決する：

1. `services.{service}.card`（MF 連携カード表示名）と**名前一致**する `local.yml` の
   `credit_cards[]` エントリを探す（`name` で照合）。見つからなければユーザーに通知して停止する
   （そのサービスがどのカードで決済されるか未登録）。
2. エントリの `type` で期待貸方の勘定科目を決める（`type` 省略時は `business` とみなす）：
   - `type: personal` → 貸方 = **事業主借**（連携が事業主借配下＝課金が事業主借で直接入る）
   - `type: business` → 貸方 = **未払金**（連携が未払金配下）
   - 補助科目名は config に持たない。連携カードの MF 上の登録内容がそのまま反映されるため、
     MF が実際に設定した値をそのまま採用する（Step 5 で照合するのは勘定科目のみ）。

### 請求サイクル（billing_modes）の解決

月額/年額の両方を持つサービス（`services/<svc>.yml` に `billing_modes` がある場合）は、
**実効 frequency** に応じてサイクル依存フィールドを解決する。

1. **実効 frequency** = `local.yml` の `services.{service}.frequency`（あれば優先）｜ `services/{service}.yml` の `frequency`
2. `billing_modes[実効frequency]` の各キーを以下に充てる：
   - `filename_pattern` → `config.invoice.filename_pattern`
   - `date_rule` → `config.duplicate_check.date_rule`
   - `memo_template` → `config.memo_template`
3. `billing_modes` が無いサービス（単一サイクル）は、これらをトップレベルのフラットな位置から読む。

**保存先・ファイル名の規約**：保存先は常に `invoices/{service}/`（yml には書かない）。
`filename_pattern` が yml に無い場合は規約
（月額 `YYYY-MM-{service}-invoice.pdf`／年額 `YYYY-{service}-invoice.pdf`）を用いる。
yml に定義があるのは規約で表せない例外（moomoo の `{domain_fqdn_us}`・wix の `{name_us}` 等）のみ。

以下の手順内の `{config.フィールド}` は、マージ後の値に置き換えて処理する。

### trade_partner_code の自動解決

`config.duplicate_check.method == by_trade_partner_code` の場合に実行する。

1. `local.yml` の `cache.{service}.trade_partner_code` を確認する。
2. **キャッシュ済みの場合（値が存在する）**：その値を `config.trade_partner_code` として使用し、このステップを終了する。
3. **未設定の場合**：
   - `mfc_ca_getTradePartners` を呼び出し、`config.trade_partner`（取引先名）に完全一致する取引先を探す。
   - 見つかった場合：その `code` を `local.yml` の `cache.{service}.trade_partner_code` に書き込み（キャッシュ。`cache:` キーが無ければ新設する）、`config.trade_partner_code` に設定する。
   - 見つからない場合：ユーザーに報告して停止する（取引先が MF に未登録の可能性あり。`mfc_ca_postTradePartners` で登録後に再実行）。

**注意：** `by_memo_keyword` サービス（anthropic・moomoo 等）では `trade_partner_code` は不要なため、このステップをスキップする。

### billing_anchor の自動解決（`date_rule: learned_anchor` の場合）

契約者依存の請求日（アンカー日）はサービス yml に書かず、`local.yml` の
`cache.{service}.billing_anchor` に保持する（trade_partner_code と同じ自動生成キャッシュ）。

1. `local.yml` の `cache.{service}.billing_anchor` を確認する。
2. **キャッシュ済みの場合**：その値（`1`〜`31` の数字、または末日契約を表す `last_day`）から
   対象月の期待取引日を計算する。`last_day`、および短い月に存在しない日（29〜31）は
   対象月の末日へクランプする。
3. **未設定の場合（初回）**：以下の優先順で実際の請求日を観測して確定する。
   1. **MF 登録済み仕訳**：`mfc_ca_getJournals` を直近 1 年程度の期間で呼び、`duplicate_check` の
      条件（キーワード／取引先コード）に合致する既存仕訳の取引日を観測する。
      複数件がすべて月末日なら `last_day`、そうでなければ最頻の day を採用する。
   2. **MF 未仕訳カード明細**：Step 4 で特定した実取引日の day を採用する（学習は Step 4 の時点で行う）。
   3. **請求書 PDF の発行日**（MF 未同期時のフォールバック）。
   - 確定した値を `local.yml` の `cache.{service}.billing_anchor` に書き込み
     （`cache:` キーが無ければ新設）、学習した値をユーザーに報告する。
   - 観測が月末日 1 件のみの場合は `last_day` か固定日か判別できないため、暫定で day 値を採用し、
     次回以降のドリフト検知で補正する。

**ドリフト検知**：Step 4 で採用した MF 実取引日が、キャッシュから計算した期待日から
±3 日を超えてズレた場合は、プラン変更・再契約で請求日が変わった可能性がある。
ユーザーに通知し、承認を得て `billing_anchor` を新しい値に更新する
（±3 日以内のズレはカード計上タイミングの揺れとして扱い、キャッシュは更新しない）。

### billing_month の解決（実効 frequency が `yearly` の場合）

年払いの請求月も契約時期依存のため、以下の順で解決する：

1. `local.yml` の `services.{service}.billing_month`（ユーザー明示設定。最優先）
2. `local.yml` の `cache.{service}.billing_month`（スキル学習値）
3. どちらも無い場合：重複チェック（Step 1）で既存の年額仕訳が見つかればその取引月を採用して
   `cache.{service}.billing_month` に書き込む。見つからなければユーザーに確認する。

年払いの取引日そのものは PDF から読む（`yearly_from_pdf`）。

---

## Step 1: 重複チェック（MCP）

`mfc_ca_getJournals` で対象期間の仕訳を取得し、登録済みかを確認する。

**`duplicate_check.method` に応じて検索条件を切り替える：**

- `by_memo_keyword`：摘要に `{config.duplicate_check.keyword}` を含む仕訳を検索
- `by_trade_partner_code`：取引先コード `{config.duplicate_check.trade_partner_code}` で検索

**`date_rule` に応じた取引日の判断：**

| date_rule | 取引日の決定方法 |
|-----------|----------------|
| `learned_anchor` | `cache.{service}.billing_anchor`（Step 0 で解決）から計算した対象月の期待日。`last_day`・短い月に存在しない日は末日へクランプ。**重複チェックの検索範囲は期待日 ±3 日**（月境界をまたぐ場合は前後の月へ拡張。カード計上が翌月 1 日になることがあるため） |
| `yearly_from_pdf` | PDF 発行日（年1件。請求月は Step 0 の billing_month 解決結果を参照） |
| `from_pdf` | PDF から確認した請求日・引落日 |

**`yearly_per_domain`（Moomoo）の場合：** `domains` リストの各ドメインについて個別に判定する。

---

## Step 2: 処理計画の表示

重複チェック結果をもとに、以下の形式で処理計画を表示してユーザーに確認を求める。

```
処理計画（{config.service_name}）:
  ⏭ YYYY-MM-DD: 登録済みのためスキップ（仕訳No.XX, ¥X,XXX）
  ✅ YYYY-MM-DD: 未登録 → 登録予定

未登録 N 件を登録してよいですか？
```

- 登録済みは自動スキップ（追加確認不要）
- 未登録 0 件の場合は「すべて登録済みです」と表示して終了
- ユーザーの承認後に Step 3 以降を実行

---

## Step 3: 請求書 PDF の確認

`invoices/{service}/` 配下の `{config.invoice.filename_pattern}`（規約または yml 定義。Step 0 参照）に従うファイルを Read する。

確認事項：
- 取引日（発行日・引落日）
- 請求金額（Step 4 で特定する MF 連携明細の円決済額との照合に使う）
- 摘要テンプレート（`{config.memo_template}`）に必要な変数（期間・日付等）を抽出する
  （各値の PDF 上の読み取り元は yml の `memo_template` 付近のコメントを参照）

---

## Step 4: 未仕訳明細の検索（Playwright）

### 4.1 マネーフォワード クラウド確定申告のホームを開く

```
mcp__playwright__browser_navigate
  url: https://accounting.moneyforward.com/
```

ログインしていない場合はユーザーにログインを依頼して停止。

### 4.2 対象カードの未仕訳明細ページへ移動

ホーム画面の「{config.mf_search.card} 未仕訳 N件」リンクをクリックして移動する。  
または `https://accounting.moneyforward.com/transaction_journals` に直接移動して絞り込む。

**注意：** URL の `account_id_hash` はセッションごとに変わる可能性があるため、
ホームから「未仕訳 N件」リンクをたどる方が確実。

### 4.3 対象取引を特定

スナップショットを取得し、以下の条件に合致する行を探す：
- 摘要に `{config.mf_search.keyword}` を含む
- 金額が請求書 PDF の金額（Step 3）と概ね一致する（外貨建ては為替で数％ズレうる。大きく異なる場合はユーザーに確認して停止）
- 取引日は `{config.duplicate_check.date_rule}` に従う日付を基準とするが、**±3 日以内のズレは許容する**

**明細が見つかった場合：MF に表示されている取引日を採用する**（`date_rule` の計算値より優先）。  
取引日が `date_rule` から 1 日以上ズレる場合は、その旨をユーザーに通知する。

`date_rule: learned_anchor` のサービスでは、ここで観測した実取引日を学習に使う：

- `cache.{service}.billing_anchor` が未設定なら、観測した day を書き込む（Step 0 の学習ソース 2）
- 期待日から ±3 日を超えるズレはドリフトとしてユーザーに通知し、承認後にキャッシュを更新する（Step 0 参照）

最初のスナップショットに表示されない場合は「続きを表示」ボタンをクリックして追加件数を読み込む。  
見つからない場合はユーザーに報告して停止（カード明細が未連携・未同期の可能性あり）。

---

## Step 5: 仕訳内容の入力と登録（Playwright）

### 5.1 「詳細」ボタンをクリックして展開

対象行の「詳細」ボタンをクリックして、展開フォームを開く。

```
mcp__playwright__browser_click  (target = 詳細ボタンの ref)
```

### 5.2 借方（勘定科目・補助科目）の確認 — MF 自動仕訳ルールに委譲

借方の値は config に持たない。**MF の自動仕訳ルールが設定した値をそのまま採用する**（変更しない）。

初回のみ（自動仕訳ルールが未学習で、借方が明らかに不適切な場合。例: 諸口・広告宣伝費等の誤自動判定）：
1. ユーザーに勘定科目・補助科目を確認する（過去の類似仕訳があれば候補として提示する）。
2. 確認した値を手動設定する：

```
mcp__playwright__browser_click  (target = 借方勘定科目ボタンの ref)
→ listbox から確認した勘定科目を選択（補助科目も同様）
```

3. 登録すると MF が自動仕訳ルールとして学習するため、次回以降このステップは確認のみになる旨をユーザーに案内する。

### 5.3 借方取引先の確認・設定（`trade_partner` が設定されている場合）

`{config.trade_partner}` が設定されており、かつ MF が自動設定していない場合に手動選択する。

```
mcp__playwright__browser_click  (target = 借方「未選択」ボタン（debit-trade-partner）の ref)
→ "{config.trade_partner}" を選択
```

MF マスタに未登録の場合は以下で登録後、ページをリロードして再試行する：

```
mfc_ca_postTradePartners
  name: "{config.trade_partner}"
  invoice_registration_number: "<請求書 PDF 記載の登録番号（T+13桁）>"  # PDF に記載がある場合
```

### 5.4 摘要を入力

`{config.memo_template}` に従い、Step 3 で抽出した変数を埋め込んで摘要を組み立てる。

```
mcp__playwright__browser_click  (target = 摘要テキストボックスの ref)
mcp__playwright__browser_press_key  key=Control+a
mcp__playwright__browser_type
  target = 摘要テキストボックスの ref
  text = "<組み立てた摘要文字列>"
```

**借方・貸方・取引先・金額について：**  
MF 自動仕訳ルールが設定済みであれば変更不要。確認のみ。

**全サービス共通ルール（config には書かない）：**
- **金額**：MF 連携カード明細の円決済額を正とする（請求書が外貨建て・税込表記でも明細額と照合する）
- **税区分**：MF 自動設定のまま変更しない（ユーザーの課税事業者/免税事業者の別と適格請求書の記載で決まるため、config には持たない）
- **初回設定**：借方勘定科目・補助科目（Step 5.2 でユーザーに確認）・取引先は初回のみ手動設定。次回以降は MF 自動仕訳ルールが補完するため確認のみ

**貸方の確認（Step 0「カード種別・貸方の解決」の期待値と照合）：**  
- MF が設定した貸方の**勘定科目**が、`type` から期待される科目（personal＝事業主借、business＝未払金）と一致するか確認する。補助科目名は照合しない（MF の実データをそのまま採用）。
- **個人用カード（personal）なのに MF が貸方を「未払金」で登録している場合**：そのカードの MF 上の連携口座が未払金配下のままになっている。「MF の連携口座設定でこのカードを事業主借配下に変更してください」とユーザーに通知する（変更後の課金から事業主借直接で入る）。**未払金→事業主借の振替仕訳は作らない**（直接記帳の方針のため）。

---

## Step 6: 請求書 PDF の添付（Playwright）

仕訳の詳細フォームが展開された状態（Step 5 の続き）で実行する。
MF クラウド会計 MCP は証憑添付 API を持たないため、この手順は Playwright 操作が必須。

PDF パス：
```
<プロジェクトルートの絶対パス>/invoices/{service}/{ファイル名}
```

### 6.1 「添付ファイル」ボタンをクリック

展開フォーム内の「添付ファイル」ボタンをクリックする（証憑添付パネルが開く）。

```
mcp__playwright__browser_click  (target = 「添付ファイル」ボタンの ref)
```

### 6.2 「ファイルを選択」→ PDF をアップロード

```
mcp__playwright__browser_click  (target = 「ファイルを選択」ボタンの ref)
# → ファイルチューザー（Modal state）が開く
mcp__playwright__browser_file_upload
  paths: ["<上記の PDF 絶対パス>"]
```

「アップロードが完了しました」が表示されたら成功。完了メッセージを必ず確認してから次へ。

### 6.3 添付パネルを閉じる

パネル右上の閉じるボタン（`ca-client-test-child-voucher-attachment-dialog-close-button`）をクリックする。

```
mcp__playwright__browser_click  (target = 閉じるボタンの ref)
```

---

## Step 7: 登録確認（MCP）

`mfc_ca_getJournals` で登録した仕訳を取得し、内容を確認する。

確認項目：
- 取引日・金額が正しいか
- 摘要が正しいか
- 取引先が正しいか
- `voucher_file_ids` が 1 件以上あるか（PDF 添付確認）

### 最終レポート

```
登録完了レポート（{config.service_name}）:
  ⏭ スキップ（登録済み）: N 件
  ✅ 新規登録: N 件
    - YYYY-MM-DD: ¥X,XXX（仕訳No.XX）

すべての処理が完了しました。
```

---

## セキュリティ・共通エラー

`.claude/skills/_shared/security.md` を参照。
