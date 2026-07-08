# Phase 31: Anthropic 請求書取得の Billing 主化

## 課題

Anthropic の請求書取得はメール（Gmail）主・claude.ai Billing フォールバックの構成だった。

1. Gmail はページ（アクセシビリティツリー）が巨大で、スナップショット1回あたりのトークン消費が大きい
2. メール経由を主とする限り、Gmail 前提の制約（他メールサービス非対応）が Anthropic にも掛かる

## 方針

主従を反転する：claude.ai Billing（レシピ A、Stripe ホスト型請求書）を主、Gmail メール添付をフォールバックにする。

- `claude.ai/settings/billing` と Stripe 請求書ページは DOM が小さく、スナップショットが Gmail より大幅に軽い
- ステップ数は両ルートで同程度（Billing を開く → 対象行の「表示」→ Stripe タブでダウンロード）
- PDF は両ルートとも Stripe 生成の同一 Invoice（メール添付はその写し）のため、証憑品質は変わらない

## 決定事項ログ

### 2026-07-08: Billing 主・メールフォールバックへの反転を決定

- ユーザー承認済み。理由：トークン消費の削減（Gmail の巨大スナップショット回避）と、Gmail 依存の緩和（Anthropic 分。ムームードメインはメール自体が適格請求書のため Gmail 必須のまま）
- 年月の導出基準がルートで異なる点（Billing = Stripe の Date of issue／メール = 本文の Paid 日付）は、
  月境界でズレが疑われる場合に添付 PDF の Date of issue へ合わせる注意書きを references に追加して吸収
