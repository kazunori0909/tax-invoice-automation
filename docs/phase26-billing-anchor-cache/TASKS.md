# Phase 26 タスク

- [x] 方針（案B・キャッシュキー名・学習ソース優先順）のユーザー承認（全サービス両サイクル対応へ拡張して承認）
- [x] `register/SKILL.md`：`date_rule` 表へ `learned_anchor` を追加し、アンカーの解決・学習（書き戻し）・ドリフト検知手順、および yearly の `billing_month` 解決順（services → cache → 確認）を記述
- [x] `services/anthropic.yml`：`billing_modes`（monthly / yearly★未確認）を導入し `month_end` を `learned_anchor` へ置換
- [x] `services/google-ai.yml`：`fixed_day_14`・摘要テンプレ・`journal_rule` の固定日記述を `learned_anchor` ベースへ置換
- [x] `services/wix.yml`：`billing_modes`（monthly / yearly★未確認）を導入し `transaction_date: 毎月28日` を除去
- [x] `services/xserver.yml`：「4月下旬〜5月上旬」等の更新月依存記述・年額の実額目安を一般化
- [x] `services/moneyforward.yml`：年またぎの具体日付実例を一般化・`billing_month` の cache 参照を追記
- [x] `services/moomoo.yml`：サービス仕様上「年契約のみ」（月払い無効）である旨を明記（ユーザー確認済み。billing_modes 対象外）
- [x] `monthly/SKILL.md`：yearly の請求月解決を `services.<svc>.billing_month` → `cache.<svc>.billing_month` → ユーザー確認 の順へ更新
- [x] `local.example.yml`：`cache.<svc>.billing_anchor` / `billing_month` の説明・例（実値と無関係な例示値）を追記
- [x] 既存ユーザー環境の `local.yml` へ現行の実アンカー値を移行（実値は local.yml のみに記載）
- [x] `service-new/SKILL.md`：`date_rule` 選択肢（learned_anchor / from_pdf / yearly_from_pdf）・billing_modes 原則両モード定義・固定日禁止ルールを追記
- [x] `docs/README.md`：`cache:` セクションの説明（billing_anchor / billing_month）を更新
- [ ] 実機確認：次回 `/register`（月次）で learned_anchor の期待日計算・ドリフト検知が機能すること
- [ ] 実機確認：キャッシュ未設定状態からの学習（cache 行を削除して再実行）で書き戻しが機能すること
- [ ] anthropic yearly・wix yearly の「★未確認」モードは実契約が現れた時点で確定する（本フェーズでは雛形のみ）

## 共通

- [x] 個人情報チェック（更新した Git 追跡ファイルに実値が混入していないか）
- [ ] 実機調査で判明した挙動・制約を KNOWLEDGE.md に記録（本フェーズは設定変更のみのため、実機確認タスク実施時に必要に応じて作成）
