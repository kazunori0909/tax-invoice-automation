# Phase 26: 請求日アンカーの学習・キャッシュ化

## 課題

1. **契約者依存の請求日が Git 管理の汎用テンプレに固定記載されている。**
   サブスク型サービス（クレカ決済）の請求日は「ユーザーがいつ契約したか」で決まるが、
   現行の `services/*.yml` には特定の日を前提とした記述がある：
   - `google-ai.yml`：`date_rule: fixed_day_14`、摘要テンプレの「{month}月14日〜{next_month}月13日」、`journal_rule` の「毎月14日」
   - `anthropic.yml`：`date_rule: month_end`、`journal_rule` の「毎月末日」
   - `wix.yml`：`journal_rule.transaction_date: 毎月28日`
   - `xserver.yml`：年額の「4月下旬〜5月上旬」等、特定更新月を前提としたコメント
2. **月払い／年払いの選択もユーザー次第**だが、`billing_modes` を持つのは一部サービスのみ
   （anthropic・wix は月額前提のフラット構成）。どのサービスも他ユーザーは逆のサイクルで
   契約している可能性がある。
3. 汎用テンプレ（他ユーザーがそのまま使える想定）としての再利用性が下がる。
   個人設定・スキル自動生成値は `local.yml` へ寄せる方針（Phase 12・21）とも不整合。
4. ユーザーに「自分の請求日は何日か」を調べて手動設定させるのは避けたい。
   初回ダウンロード・register 時に MF クレカ連携データ等から観測できるはずである。

## 方針

### 案A: `local.yml` の `services.<svc>.billing_day` としてユーザーが手動設定

- 実装は最小だが、ユーザーが自分でカード明細を調べて設定する手間が発生する。
- 「スキルが自動取得できる値はユーザーに設定させない」という Phase 21 の `cache:` 分離方針に反する。→ 不採用

### 案B: 請求日アンカーをスキルが学習し `local.yml` の `cache:` に書き戻す（採用）

`trade_partner_code` の自動解決（register Step 0）と同じパターンを請求日に適用する。

- **新 `date_rule: learned_anchor` を導入**。`services/*.yml` からは固定日の記述を除去し、
  実際の請求日（アンカー日）は `local.yml` の `cache.<svc>.billing_anchor` に保持する。
  - 値は day-of-month（例: `14`）または特殊値 `last_day`（末日契約）。短い月は末日へクランプ。
  - 対象は「取引日が契約開始日で決まり、PDF の日付とカード計上日がズレうる」月額サービス
    （anthropic・google-ai の monthly モード）。PDF に取引日が明記されるサービスは
    従来どおり `from_pdf` / `yearly_from_pdf` を使う（この 2 つは契約者依存の固定日を含まないため変更不要）。
- **初回（キャッシュ未設定）の学習ソース（優先順）**：
  1. MF 登録済み仕訳：`mfc_ca_getJournals` をキーワード／取引先コードで期間を広めに検索し、既存仕訳の取引日から day を観測（ブラウザ不要で最も安い。複数件あれば全件末日一致で `last_day` と判定できる）
  2. MF 未仕訳カード明細：register Step 4 で特定した実取引日（通常フローで必ず通るため実質無料）
  3. 請求書 PDF の発行日（download 直後などで MF 未同期の場合のフォールバック）
  - 観測できたら `cache.<svc>.billing_anchor` に書き戻す（`cache:` キーが無ければ新設。trade_partner_code と同じ）。
- **2 回目以降**：キャッシュ値から対象月の期待取引日を計算し、重複チェックの検索範囲（Step 1）・
  未仕訳明細の候補絞り込み（Step 4 の ±3 日許容）・摘要の期間計算に使う。
  「MF 連携データの実取引日を優先」（Phase 16）はそのまま維持する。
  従来 `month_end` 固有だった「重複チェック範囲を翌月 3 日まで拡張」は
  「期待日 ±3 日（月境界をまたいで拡張）」として全 learned_anchor に一般化する。
- **ドリフト検知**：観測した取引日がキャッシュから ±3 日を超えてズレた場合はユーザーに通知し、
  承認のうえキャッシュを更新する（プラン変更・再契約で請求日が変わるケースへの追従）。
- **摘要テンプレの一般化**：google-ai 月額の「{month}月14日〜{next_month}月13日」は
  `{period_start}〜{period_end}`（アンカー日〜翌月アンカー日前日として機械的に計算）へ置き換える。

### 全サービスの月払い／年払い両対応（billing_modes の全面採用）

現状のユーザー契約（例: xserver=年額、anthropic=月額）は一つのパターンにすぎず、
他ユーザーは逆のサイクルで契約しうる。よって：

- **全サービスの `services/*.yml` を `billing_modes`（monthly / yearly）構造に統一**する
  （anthropic・wix にも導入。既定 `frequency` は現行値を維持し、`local.yml` の
  `services.<svc>.frequency` で上書きする既存の仕組みはそのまま）。
- 実データで未確認のモード（anthropic yearly・wix yearly 等）は Phase 18/19 の慣例に従い
  「★未確認」コメントを付けて雛形のみ定義する。
- **年払いの請求月（`billing_month`）も同様に学習対象**とする。
  解決順：`local.yml` の `services.<svc>.billing_month`（ユーザー明示設定）
  → `cache.<svc>.billing_month`（register が既存仕訳・登録実績から学習）→ 実行時にユーザーへ確認。
  年払いの取引日そのものは従来どおり PDF から読む（`yearly_from_pdf`）。
- **例外：moomoo（`yearly_per_domain`）**。ドメイン契約は商品特性上、月払いが存在しないため
  monthly モードは定義しない。更新月はドメインごとに `local.yml` の `domains[].billing_month` で
  既に管理済み（＝契約者依存値は最初から Git 管理外）であり、現行構成を維持する。

## 決定事項ログ

### 2026-07-04: フェーズ起票

- 「Google AI 含め月契約・年払いのクレカ支払日はユーザー次第であり、14日・月末等の固定記載を避けたい。
  初期ダウンロード時や register 時に MF クレカ連携データから観測してキャッシュする案でどうか」というユーザー提起により起票。

### 2026-07-04: 案B採用＋全サービス両サイクル対応へ拡張

- 案B（learned_anchor ＋ `cache.<svc>.billing_anchor`）を採用。
- ユーザー指示により適用範囲を拡張：「現状他サービスは年契約だが、それは自分のパターンにすぎない。
  全サービスで月払い・年払い両方がありえるため、それを考慮して共通化する」。
  → 全サービス `billing_modes` 構造へ統一し、年払いの `billing_month` も
  `services.<svc>.billing_month` → `cache.<svc>.billing_month` → 実行時確認、の順で解決する設計とした。
- 未確認モード（anthropic yearly・wix yearly）は「★未確認」コメント付きの雛形として定義
  （実契約が現れた時点で実データ確認のうえ確定する。Phase 18/19 の慣例に準拠）。

### 2026-07-04: 稼働ファイルのコメントは現行仕様のみ（ユーザー指示）

- 将来パブリック公開する前提で、Git 追跡の稼働ファイル（yml・SKILL.md 等）には
  旧仕様との比較コメント（「以前は◯◯だった」「旧◯◯相当」等）を書かない方針とした。
  経緯・履歴はフェーズログ（`PLAN.md`／`docs/phases.md`）にのみ残す。
- 既存の該当コメント（mf_search キーワードの旧表記併記・「従来どおり」等）を除去し、
  CLAUDE.md の更新ルールに項目 7 として明文化した。

### 2026-07-04: moomoo は年契約のみ（ユーザー確認）

- ムームードメインはサービス仕様上、年契約しか存在しない（月払いは無効）とユーザーが確認。
  `billing_modes` は定義せず、現行の `yearly_per_domain` 構成（更新月は `local.yml` の
  `domains[].billing_month` で管理）を維持する。yml にもその旨をコメントで明記した。
