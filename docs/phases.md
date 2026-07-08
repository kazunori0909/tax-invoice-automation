# 開発フェーズ記録（Phase 1–31）

現行アーキテクチャの主要な設計判断とその理由を、フェーズ順に凝縮した記録。
作業管理の詳細（タスク一覧・スコープ・検討過程）、操作手順そのもの（各 Skill に記載）、
および後続フェーズで上書き・撤回された決定は含まない。そのため一部のフェーズ番号は欠番になる。

> このファイルは「なぜ今の構成になっているか」の単一の履歴。
> 現行の運用方法・設定の置き場所は [README.md](README.md) と [CLAUDE.md](../CLAUDE.md) を参照。

---

## Phase 1: プロジェクト構成の方針決定

認証情報を保持しない設計として、**ユーザー手動ログイン + Playwright MCP 方式**を採用。
月1回程度の作業に完全自動化は過剰であり、認証情報の管理リスクや 2FA 対応の複雑さも避けられるため。
操作手順はサービスごとに Skill として分割し、ブラウザのログイン状態は `.playwright-profile/` に永続化する。

## Phase 2: Playwright MCP のセットアップ

MCP 設定はプロジェクトローカル（`.mcp.json`）に置き、Microsoft 公式 `@playwright/mcp` を使用。
ブラウザはシステム Chrome、**ヘッドレスはオフ**（月次作業時に人が確認しながら進めるため）。
実行時生成物 `.playwright-mcp/` は `.gitignore` 対象。

## Phase 3: 請求書ダウンロードの命名規則

月次サービスは `invoices/<service>/<YYYY-MM>-<service>-invoice.pdf`、年次サービスは
`invoices/<service>/<YYYY>-<service>-invoice.pdf` を既定とする。
ファイル名にサービス名を含める理由：MF 等にアップロードしてディレクトリ階層が失われても一意識別できるようにするため。
サービスごとの取得導線・操作手順は `.claude/skills/download/references/<svc>.md` に集約する。

## Phase 4: セキュリティ対応

`.env` への Read/Edit/Write を完全 deny（Bash 経由の読み取りも主要パターンを deny）。
`.playwright-profile/` と `invoices/` は機能上 Read が必要なため deny せず、git 管理外＋ポリシー文書
（`SECURITY.md`）で運用する。MCP の認証情報は `.mcp.json` に直書きせず `.env` から環境変数として注入する。

## Phase 5: マネーフォワード クラウド会計 MCP の接続

認証方式は **OAuth 2.0（対話的フロー）**。API トークンは使わず、認証情報は `.env` にも保存しない。
MCP サーバーは Beta 版を採用（Alpha 版は1時間ごとの手動再認証が必要なため）。
接続は `mcp-remote` 経由（stdio プロキシ）。URL を直接指定するとツールが検出されなかったため。

## Phase 6: 仕訳登録の方式（PDF起点 → データ連携起点）

請求書 PDF から新規仕訳を作る方式ではなく、**クレジットカードのデータ連携明細を起点**に
対象取引を発見し仕訳へ振替する方式を採用。請求書がドル建てでも実際の支払額（円）はカード明細に
依存するため、この方式なら二重入力にならず既存の手動運用とも整合する。
MoneyForward Cloud MCP は未仕訳明細の取得に未対応のため、明細の検索・振替は Playwright で
MF の Web UI を操作する（MCP＝マスタ取得・仕訳作成更新、Playwright＝未仕訳明細の検索・振替、
Claude Read＝PDF内容確認、という役割分担）。月ごとに対話するのではなく、バッチ処理計画を
まとめて1回提示して承認を得る運用とし、登録済みは自動スキップする。
免税事業者のため税区分は原則なし（`INVOICE_KIND_NOT_TARGET`）。サービスごとの仕訳ルール
（勘定科目・取引先・摘要等）は現在、各 `services/<svc>.yml` の `journal_rule:` に集約されている。

## Phase 7: 月次一括処理 Skill（`/monthly`）

年次サービスも `/monthly` のスコープに含め、月別の判定フローで対象を制御する。
個別 Skill（download・register）の手順を参照する委譲方式とし、手順の二重管理を避ける。
設計原則：エラー分離（1サービスの失敗が他を止めない）、冪等性（重複はスキップ）、
複数の中断・確認ポイントを設ける。

## Phase 8: クレジットカード引き落とし仕訳（`/credit-payment`）

カード課金時の仕訳（貸方＝未払金）に続く、引き落とし時の仕訳（借方＝未払金 → 貸方＝事業主借）を
検知・登録する Skill を独立させた。引き落とし日が月次作業と別タイミングのため `/monthly` には統合しない。
設定ファイル駆動（YAML）でカード追加に対応し、重複・金額不一致の検知はチェック範囲内の既存仕訳と照合する。

## Phase 9: Skill 共通化（`_shared/`）

複数 Skill から参照される内容のみ `_shared/` に切り出す（現在は `security.md` のみ）。
共有ファイルは SKILL.md 内に「読んで従うこと」と明記する形式とし、CLAUDE.md 専用の `@` 参照は使わない。
サービス固有の前提条件・手順本体は差分が大きいため各 Skill に残す。
MF MCP は証憑添付 API を持たないため、Playwright での PDF 添付手順は引き続き必須。

## Phase 10: カード設定管理の改善

口座・補助科目の ID は YAML に持たず、スキル実行時に名前 → ID を動的解決する（`getAccounts`/`getSubAccounts`）。
カード名・口座名などの個人情報は git 管理外の個人設定ファイルに分離する（後の Phase 12 で `local.yml` に統合）。
パラメータ名は `card`/`bank`（役割ベース）を採用。引き落とし仕訳と費用仕訳とで借方/貸方の役割が
入れ替わるため。

## Phase 11: register Skill の統合

サービスごとに分かれていた register Skill は、手順の構造が全サービス共通でパラメータのみ差分と
判明したため、`services/*.yml`（パラメータ）＋ 単一 `register/SKILL.md`（手順）に統合した。
download 側はメカニズムの差が大きいため、この時点では統合対象外とした（後に Phase 24 で別方式により統合）。
効果：新サービス追加は `services/<svc>.yml` の追加のみで完結する。

## Phase 12: サービス設定の「汎用テンプレ／個人設定」分離

`services/*.yml` は固定仕様（Git 管理）のみとし、個人設定（カード・取引先コード・ドメイン等）は
`local.yml`（Git 管理外）に統合する。`local.yml` の `services:` キーの有無を「利用中サービス」の
宣言として扱い、「全部実行」はこのキーを列挙して処理対象を決定する。

## Phase 13: 稼働ファイルからの個人情報マスキング

Git 追跡ファイル（CLAUDE.md・SKILL.md・設定テンプレ）の個人情報をプレースホルダー／`local.yml` 参照に
置換した。将来の公開に向けた前処理として実施し、Git 履歴は書き換えない方針とする。対象は稼働ファイルの
みで、完了済み過去フェーズの docs は対象外とした。

## Phase 14: 請求サイクル切替（billing_modes）

請求サイクル依存フィールド（ファイル名パターン・取引日ルール・摘要テンプレ等）を
`billing_modes.<monthly|yearly>` に集約し、`frequency` で切替える。サイクル変更は `local.yml` 側の
1行で完結する。既定の `frequency` は `monthly` に統一し、年額契約は個人設定側で `yearly` を指定する。
このフェーズでマネーフォワード クラウド（サブスク利用料）を新規サービスとして追加した。

## Phase 15: SSOT 化とメンテコスト削減

仕訳ルールを CLAUDE.md から各 `services/<svc>.yml` の `journal_rule:` に移設し、CLAUDE.md は目次化した。
単一 Skill からしか参照されない共有ファイルはインライン化して削除する（`_shared/` は複数 Skill が
参照するものに限定する方針）。PII 取扱いルールの本文は `SECURITY.md` に集約し、CLAUDE.md はリンクのみとする。

## Phase 16: 取引日は MF 連携データの実取引日を優先

取引日は `date_rule` の計算値よりも **MF 連携データ上の実取引日を優先**する。カード会社の締め処理により、
月末の請求が翌月付けで連携されることがあるため。計算値から一定以上ズレる場合は登録前にユーザーへ通知する。
一方で、PDF ファイル名の年月は請求書発行月基準のまま変更しない（取引日が月をまたいでずれても）。

## Phase 18: Wix サービス追加

支払い履歴の明細書表示から新タブで開く PDF を取得する導線を確立した。
`browser_run_code_unsafe` は Node.js グローバルが使えないサンドボックスのため、`browser_evaluate`
（ブラウザ側 fetch）で取得して base64 デコードする方式を採用。
税区分は課税仕入（インボイスに登録番号の記載があるため。免税事業者前提の他サービスと異なる）。
`trade_partner_code` はユーザーの手動設定をやめ、Skill が自動取得して `local.yml` にキャッシュする
方式に変更した。
また、このフェーズでフェーズ管理ルールを改訂：`docs/phases.md` への自己判断での追記を禁止し、
フェーズフォルダ（PLAN/TASKS/KNOWLEDGE）方式を導入した。

## Phase 19: Wix マルチサイト対応

単一サイト想定（`wix.site`）から複数サイト（`wix.sites[]`）に変更した（moomoo の `domains` 方式を踏襲）。
ファイル名にサイト識別子を含め、複数サイト運用時の衝突を防ぐ。`sites` 省略時は billing-history に
表示される全サイトを対象とする。

## Phase 21: local.yml の運用改善（cache 分離）

スキルが自動生成する値（`trade_partner_code` 等）とユーザー設定値を YAML 構造で分離するため、
`local.yml` に `cache:` セクションを新設した。「キャッシュだけ消せば安全に再生成される」運用を実現する。
Wix の `plan`（プラン名）は `local.yml` に持たせず、`/register` 実行時に請求書 PDF から都度読み取る
方式に統一した。

## Phase 22: Google AI サービスの一般化・年払い対応

`service_name` をプラン名（Pro 等）依存の表記から一般化し、契約プランの違いを吸収できるようにした。
`billing_modes` を導入して年払いにも対応。将来 Google Workspace を追加した際の名前空間衝突を避けるため、
サービスキーを `google-ai` に変更した。

## Phase 23: フェーズ管理のスキル化

フェーズ開始・圧縮の手作業（採番・雛形整備・README 更新漏れ）を解消するため、`/phase-new`・
`/phase-compress` の2スキルを追加した。`KNOWLEDGE.md` は雛形に含めない。空の雛形があると
「とりあえず埋める」圧力が生じるため、実機調査で気づきが出た時点でのみ作成する運用とする。

## Phase 24: download Skill の単一化

サービスごとに分かれていた download Skill（構造がほぼ同一でコピペ状態だった）を、単一の
`/download <svc|全部>` ＋ `references/<svc>.md`（サービス固有手順のみ）構成に統合した。
命名規則（`filename_pattern`）は `services/<svc>.yml` を唯一の情報源とし、手順・データの二重管理を解消した。

## Phase 26: 請求日アンカーの学習・キャッシュ化

契約者依存の請求日（何日締め・月末等）はサービス yml に固定記載せず、`date_rule: learned_anchor` を
導入して `local.yml` の `cache.<svc>.billing_anchor` に学習・キャッシュする方式に変更した。
初回は MF 登録済み仕訳 → 未仕訳カード明細 → 請求書 PDF の優先順で実際の請求日を観測し、以降は
期待日 ±3 日を許容してドリフト（プラン変更・再契約による請求日変化）を検知する。
全サービスを月払い/年払い両対応の `billing_modes` 構造に統一し、年払いの請求月も同様に
`services.<svc>.billing_month` → `cache.<svc>.billing_month` → 実行時確認の順で解決する。

## Phase 27: config 層構造の明文化

`config/services/*.yml` の配置（サービス軸でもスキル配下でもなく中立の config 層）を維持する方針を
確定した。download / register / monthly / service-new の4スキルが共有する宣言的データであり、
特定スキル配下へ移すと共有フィールドの重複かスキル境界を越えた参照が発生するため。
「config = 宣言的データ層、skills = 手順層」という設計原則を docs/README.md に明文化した。

## Phase 28: カード種別対応（個人用＝事業主借直接／事業用＝未払金）

連携カードの貸方（事業主借 or 未払金）は、MF 上でそのカードの連携口座がどちらの勘定科目配下に
登録されているかで決まる。`credit_cards[].type`（personal/business）を `services.<svc>.card` との
名前一致で解決し、register がその期待勘定科目（personal→事業主借／business→未払金）を照合する
方式に統一した。個人用カードは全て事業主借配下（直接記帳）へ寄せ、未払金経由の2段階記帳を廃止。
事業用カード（未払金型）のみ `/credit-payment` で引き落とし仕訳（未払金→普通預金）を扱う。

## Phase 30: Moomoo ドメイン設定の一本化

`services.moomoo.domains[]` の `name`（TLD除去名）と `fqdn` の重複フィールドを廃し、`fqdn` のみに
一本化した。ファイル名生成には `.` を `_` に置換した `{domain_fqdn_us}` を、摘要・重複チェックには
無変換の `{domain_fqdn}` を使う（区切り文字にハイフンを使うと既存ドメイン名自体のハイフンと
衝突するため、ドメイン名に通常出現しないアンダースコアを採用）。

## Phase 31: Anthropic 請求書取得の Billing 主化

Anthropic の請求書取得を、メール（Gmail）主から claude.ai Billing（Stripe ホスト型請求書）主・
メールフォールバックへ反転した。Gmail はページ全体のスナップショットが巨大でトークン消費が
大きいため。年月の導出は Stripe 側の日付ではなく claude.ai 請求書テーブルの日付列を基準にする
（Stripe に発行日相当の項目がなく、MF 連携カード明細の取引日と一致するのはテーブル側の日付のため）。
