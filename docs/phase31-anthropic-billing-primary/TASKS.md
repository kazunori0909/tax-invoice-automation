# Phase 31 タスク

- [x] `download/references/anthropic.md` を Billing 主・メールフォールバックの構成に書き換え
- [x] `README.md` の対象サービス表（Anthropic の取得元）と前提条件（Gmail 前提の対象サービス）を更新
- [x] `/download anthropic` の6月分で Billing 主フローを実機確認（`invoices/anthropic/2026-06-anthropic-invoice.pdf`）
- [x] 実機確認で判明した年月ズレ（Stripe "Date of issue" 不在／claude.ai テーブルと Stripe 支払い日の1日ズレ）を
      踏まえ、年月導出を claude.ai テーブルの日付列基準に修正

## 共通

- [x] 実機調査で判明した挙動・制約を KNOWLEDGE.md に記録
- [x] 個人情報チェック（更新した Git 追跡ファイルに実値が混入していないか）
