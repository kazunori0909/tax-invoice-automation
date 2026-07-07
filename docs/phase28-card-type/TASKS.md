# Phase 28 タスク

※ Git 追跡対象。カード名・補助科目名等の実値は書かない（PII。実値は `local.yml`）。

## 設定（config）

- [x] `local.example.yml`：`credit_cards[]` の雛形を新スキーマへ更新（personal＝`owner_sub_account`／business＝未払金・普通預金・締め日）。`type` の凡例追記
- [x] `local.yml`（管理外）：個人用カード2件を新スキーマで記述（実値はここのみ）

## サービス仕様（git 管理）

- [x] `services/{anthropic,google-ai,moneyforward,xserver,moomoo,wix}.yml` の `journal_rule`：
      貸方直書き（`credit_account: 未払金`）を廃止し、貸方はカード依存（`services.card`→`credit_cards.type`）で解決（フラグは置かない）
- [x] type から自明な冗長フィールドを削除（business の `card_account`/`bank_account`、`credit_by_linked_card`）。skill 側で導出
- [x] business の `card_sub_account`/`bank_sub_account` も config 任意化（連携時は自動仕訳から取得。未連携手動登録時のみ指定）
- [x] `journal_rule` のスリム化：`tax`（個人依存）・重複系（`trade_partner`/`transaction_date`/`billing_cycle`/`evidence`/`yearly_note`/`domains_note`）・
      手順系（`amount_source`/`setup_note`→register/SKILL.md へ移動）・`status` を全サービスから削除。service-new の雛形も更新

## スキル

- [x] `register/SKILL.md`：Step 0 にカード種別・貸方の解決を追加。個人用の期待貸方＝事業主借/補助科目、
      MF がまだ未払金の場合の移行通知。未払金→事業主借の振替は作らない旨
- [x] `credit-payment/SKILL.md`：`type: business`（未払金型）専用へ役割を明記。個人用は対象外。貸方を普通預金へ

## ドキュメント

- [x] `CLAUDE.md`「仕訳ルール」：貸方はカード依存（personal＝事業主借直接／business＝未払金＋credit-payment）を追記
- [x] `docs/README.md`：`credit_cards.type` と挙動、credit-payment の役割変更、開発フェーズ記録に Phase 28 行

## 移行（ユーザー実施・確認）

- [x] カードA 連携口座の登録先を事業主借（カード別補助科目）へ変更（MF UI）
- [x] 既存の未払金残高を手動で事業主借へ修正（2026-07-06 MF 実データ確認：未払金残高 0）
- [x] `/register anthropic` で貸方＝事業主借/該当補助科目を確認（2026-07-01 付仕訳で確認済み）

## 共通

- [x] 個人情報チェック（更新した Git 追跡ファイルに実値が混入していないか）
