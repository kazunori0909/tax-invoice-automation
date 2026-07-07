# Phase 29: config スリム化（MF 自動仕訳への委譲と不要フィールドの削減）

## 課題

`.claude/config/` 配下（`services/*.yml`・`local.yml`・`local.example.yml`）に、
実際にはスキルが消費していない・情報量がない・規約で導出できるフィールドが蓄積している。
特に借方の勘定科目・補助科目は MF クラウドの自動仕訳ルールで補完できるため、
config 側で保持する意味があるか（初回案内に置き換えられないか）を見直す。

全フィールドを消費側（register / download / monthly / credit-payment / service-new）と
突き合わせた洗い出し結果は以下のとおり。

### A. ユーザー提起：借方（debit_account / debit_sub_account）の MF 自動仕訳への委譲

- 現状の register Step 5 は既に「MF 自動仕訳ルールが設定済みなら変更不要・確認のみ」で、
  config の値は (1) 初回の手動設定値 (2) 毎回の照合用期待値 としてのみ機能している。
- `journal_rule.debit_account` は全 6 サービスで「通信費」と同一値であり、
  サービス固有データとしての情報量がない。
- 案の比較（未決定）：
  - **案A（ユーザー提案）**：config から削除し、初回実行時に「MF の自動仕訳ルール設定」を
    案内する手順に置き換える。以後スキルは借方をノーチェック（MF に全委譲）。
  - **案B**：現状維持（期待値として保持し毎回照合。ドリフト検知が残る）。
  - **案C**：billing_anchor / trade_partner_code と同じパターンで、既存仕訳から学習して
    `cache.<svc>` に保持する。ユーザー設定は不要になり、照合（ドリフト検知）は残る。

### B. 未消費（デッド）フィールド — 即削除候補

1. `services.moomoo.domains[].payment_method`（local.yml / local.example.yml）：
   「[REG] 連携明細の照合に使用」と注記されているが、**どのスキルにも参照箇所がない**。
   `services.moomoo.card` と役割が重複（ドメインごとに決済カードが異なるケースが
   実際に発生したら、その時に per-domain の `card` 上書きとして再設計すればよい）。
2. `services.<svc>.debit_account` の local 上書き（local.example.yml のコメント行 [REG] 省略可）：
   register SKILL.md のマージ表に取得元として定義されておらず、**書いても効かない**。
   （A の結論次第でフィールドごと消えるが、独立した不整合として記録）

### C. 実質情報ゼロのフィールド — 削除候補

3. `amount_hint`（billing_modes 配下含む）：anthropic の実レンジ以外はすべて
   「プラン料金（カード明細額と照合）」の同語反復で情報がない。
   共通ルール「金額は MF 連携カード明細の円決済額を正とする」が register に既にあるため、
   フィールドごと廃止するか、実情報がある場合のみの任意フィールドに降格する。
4. `account_override`（xserver のみ）：「初回のみ必要・次回以降は自動仕訳ルールが更新済み」
   という一時ワークアラウンドで、`journal_rule.debit_account: 通信費` と重複。
   register Step 5.2 ごと削除し、「MF 自動仕訳が期待値と異なれば直す」一般ルールに吸収できる
   （A の結論に依存）。

### D. 規約で導出可能なフィールド — 削減候補（convention over configuration）

5. `invoice.dir`：全サービスで `invoices/<サービスキー>` の機械的導出。規約化すれば削除可。
6. `filename_pattern`：moomoo（{domain}）・wix（{name_normalized}）以外は
   `YYYY(-MM)-<svc>-invoice.pdf` の機械的パターン。デフォルト規約＋例外のみ宣言にできる。
   ただし月額/年額で YYYY-MM / YYYY が切り替わるため、規約は実効 frequency 連動で定義する。

### E. 重複・整理候補

7. `memo_source`：memo_template の変数の読み取り元説明で、テンプレート付近のコメントと
   重複気味。billing_modes 内コメントまたは memo_template への統合を検討。
8. コピペコメントの重複：journal_rule の貸方解決の説明（2 行）が全 6 yml に同文で存在。
   billing_modes の解決説明（3〜4 行）も 5 yml に重複。正は register SKILL.md／
   docs/README.md の消費者マップなので、各 yml はポインタ 1 行に削る。
9. `duplicate_check.method: by_memo_keyword`（anthropic）：trade_partner が設定済みなのに
   摘要キーワード方式。`by_trade_partner_code` に統一できれば method 分岐を減らせる
   （moomoo はドメイン識別に摘要が必要なため memo 方式を残す）。要実機確認。
10. `trade_partner_invoice_number`（wix・moomoo）：新規取引先登録（初回のみ）でしか
    使わず、請求書 PDF からも読める。削除候補（公開情報のため残害はなく優先度低）。

### 維持するフィールド（洗い出しの結論として現状維持）

`service_name`・`mf_search.keyword`（カード明細上の表記＝実機調査の成果）・
`trade_partner`・`duplicate_check`（method/keyword）・`date_rule`・`memo_template`・
`frequency`/`billing_modes` は、いずれも消費者が明確で情報量があるため維持する。

## 方針

案A（借方を MF 自動仕訳ルールへ委譲）＋ B〜E の削減候補をすべて実施する。

## 決定事項ログ

### 2026-07-06: フェーズ起票

- config 全フィールドを消費側と突き合わせて上記 A〜E を洗い出した。

### 2026-07-06: 案A採用＋全削減候補の実施（ユーザー承認）

- **A: 案Aを採用**。`journal_rule`（debit_account / debit_sub_account）を config から全廃し、
  借方は MF 自動仕訳ルールに委譲。初回の `/register` のみユーザー確認のうえ手動設定し、
  MF に学習させる（register Step 5.2 に手順化）。理由：全サービス同一値で情報量がなく、
  register は元々「自動仕訳ルール設定済みなら確認のみ」だった。月1回の目視確認運用のため
  期待値照合（案B/C）の保険は不要と判断。
- **B-1**: `services.moomoo.domains[].payment_method` を削除（未消費・`card` と重複）。
- **B-2**: local.example.yml の `debit_account` 上書きコメント行を削除（register が読まない設定）。
- **C-3**: `amount_hint` を全廃。金額照合は共通ルール「MF 連携明細の円決済額を正とし、
  請求書 PDF の金額と照合（外貨は為替ズレ許容）」に一本化。
- **C-4**: `account_override`（xserver）と register Step 5.2（旧）を削除。
  Step 5.2（新・借方委譲）の「借方が明らかに不適切な場合はユーザー確認」に吸収。
- **D-5**: `invoice.dir` を規約化（`invoices/<サービスキー>/` 固定）。yml から削除。
- **D-6**: `filename_pattern` を規約化（月額 `YYYY-MM-<svc>-invoice.pdf`／年額 `YYYY-<svc>-invoice.pdf`）。
  規約で表せない moomoo（`{domain}`）・wix（`{name_normalized}`）のみ yml に例外を残す。
  規約の定義場所は download / register / monthly の各 SKILL（解決手順の一部）＋ docs/README.md。
- **E-7**: `memo_source` フィールドを廃止し、読み取り元の説明は各 yml の `memo_template` 直下の
  コメントに統合（billing_modes ではモードごとに記載）。
- **E-8**: 全 yml に同文コピペされていた貸方解決・billing_modes 解決の説明コメントを削除
  （journal_rule 廃止に伴い貸方コメントは消滅。解決手順の正は register Step 0）。
- **E-9**: anthropic の `duplicate_check` を `by_trade_partner_code` に統一。
  **実機確認済み（2026-07-06）**：既存の Claude Pro 仕訳 3 件（4〜6月分）すべてに
  取引先「Anthropic, PBC」のコードが付与されており、コード検索で過去仕訳を発見できる。
  取得したコードは `local.yml` の `cache.anthropic.trade_partner_code` に記入済み。
  moomoo は取引先（GMOペパボ）が全ドメイン共通のため `by_memo_keyword` を維持。
- **E-10**: `trade_partner_invoice_number` を削除。初回の取引先登録時は請求書 PDF 記載の
  登録番号を読み取って `mfc_ca_postTradePartners` に渡す（register Step 5.3 に明記）。

### 2026-07-06: local.example.yml の説明を Markdown ガイドへ分離（ユーザー提案）

- YAML コメントで使い方を説明する方式（Phase 20 の STEP 1/2 構成・`[DL]`/`[REG]` タグ）をやめ、
  **セットアップガイド `.claude/config/README.md`** に説明を集約。`local.example.yml` は
  「形を見れば書ける」最小限の雛形（コメントアウトした任意キー＋プレースホルダー）に縮小した。
- 理由：非エンジニアには YAML の大量コメントより Markdown ページの方が読みやすい。
  GitHub でフォルダを開くと自動表示される README.md を採用し、`local.example.yml` と並列に配置。
- 追随更新：docs/README.md「はじめに」をガイドへのポインタに縮小、CLAUDE.md ディレクトリ表に
  ガイドを追記、service-new Step 4 を「キー説明はコメントで書かず、新しい種類のキーは
  ガイドの表へ追記する」方式に変更。

### 2026-07-06: F. `credit_cards[].owner_sub_account` を削除（ユーザー指摘）

- ガイド作成中に発覚：貸方の解決は「MF 上でカードの連携口座がどの勘定科目（事業主借／未払金）
  配下に登録されているか」で決まり、それは連携口座設定という MF 側の状態そのものである
  （A の借方委譲と同じ構造の話）。`owner_sub_account`（事業主借の補助科目名）の消費箇所を
  確認したところ、register Step 5「貸方の確認」で MF の実際値と照合する**期待値としてのみ**
  使われており、書き込み・検索には一切使われていなかった。ユーザーから「設定してしまえば
  そうなるのだから不要では」と指摘があり、削除を決定。
- `credit_cards[].card_sub_account` / `bank_sub_account`（business 型）は残す：
  credit-payment スキルが「引き落とし口座が MF 未連携で手動登録する場合」の実データとして
  使う別の消費経路があり（自動連携時は「自動仕訳から取得」で不要＝既に任意項目）、
  owner_sub_account のような純粋な重複ではないため対象外。
- 変更箇所：local.yml・local.example.yml・README.md から `owner_sub_account` を削除。
  register/SKILL.md の「カード種別・貸方の解決」と「貸方の確認」を、勘定科目（事業主借／未払金）
  のみを照合する形に簡略化（補助科目名は MF の実データをそのまま採用し、比較しない）。
