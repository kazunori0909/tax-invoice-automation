# Phase 29 タスク

- [x] A: 借方（debit_account / debit_sub_account）の扱いを決定（案A: MF 自動仕訳へ委譲＋初回ユーザー確認 を採用）
- [x] A の決定に沿って services/*.yml・local.example.yml・register SKILL.md（Step 0 マージ表・Step 5.2）を更新
- [x] B-1: `services.moomoo.domains[].payment_method` を local.yml / local.example.yml から削除
- [x] B-2: local.example.yml の `debit_account` 上書きコメント行を削除（効かない設定の除去）
- [x] C-3: `amount_hint` の全廃と register SKILL.md の参照箇所更新（PDF 金額との照合に置換）
- [x] C-4: `account_override`（xserver）と register 旧 Step 5.2 の削除
- [x] D-5: `invoice.dir` の規約化（`invoices/<svc>` 固定）と download / register / monthly / service-new の更新
- [x] D-6: `filename_pattern` のデフォルト規約化（moomoo / wix のみ例外定義を残す）
- [x] E-7: `memo_source` を廃止し memo_template 直下のコメントへ統合
- [x] E-8: 各 yml の重複コピペコメントを削減
- [x] E-9: anthropic の duplicate_check を by_trade_partner_code へ統一（MF MCP で実機確認済み・cache に取引先コード記入済み）
- [x] E-10: `trade_partner_invoice_number` を削除（初回登録時は PDF から読み取る）
- [x] docs/README.md「フィールド×消費者マップ」と規約の明文化
- [x] local.example.yml の説明を変更後の姿に更新
- [x] セットアップガイド `.claude/config/README.md` を新設し、local.example.yml を最小雛形化（docs/README.md・CLAUDE.md・service-new のポインタ更新含む）
- [x] F: `credit_cards[].owner_sub_account` を削除（貸方の補助科目は MF 連携口座設定の実データをそのまま採用。register の照合は勘定科目のみに簡略化）
- [ ] 次回 /monthly（または /register）実行時に、規約解決・借方委譲・anthropic のコード検索が想定どおり動くことを確認する

## 共通

- [x] 個人情報チェック（更新した Git 追跡ファイルに実値が混入していないか）
