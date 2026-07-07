# Anthropic (Claude) 固有手順

**方式**：Gmail 共通操作 + レシピ A（メール添付優先。メールが無い月は claude.ai フォールバック）

## メールの仕様

| 項目 | 値 |
|------|-----|
| 送信元 | `invoice+statements@mail.anthropic.com` |
| 件名 | `Your receipt from Anthropic, PBC #XXXX-XXXX-XXXX` |
| 添付1 | `Invoice-XXXXXXXX-XXXX.pdf`（欲しいもの） |
| 添付2 | `Receipt-XXXXXXXX-XXXX.pdf`（不要） |

## 手順（メール経由）

1. Gmail 検索：`from:mail.anthropic.com`
2. 対象月の "Your receipt from Anthropic" メールを開く
3. **YYYY-MM の導出**：本文の `Paid MMMM DD, YYYY` から（例：Paid May 31, 2026 → `2026-05`）

```javascript
const body = document.querySelector('.a3s')?.innerText || '';
const match = body.match(/Paid\s+(\w+)\s+\d+,\s+(\d{4})/);
```

4. Invoice PDF の添付をレシピ A でダウンロード → リネーム移動
5. 対象メールが見つからない月 → claude.ai フォールバックへ

## フォールバック（claude.ai 経由）

1. `https://claude.ai/settings/billing` を開く
   - 注意：`console.anthropic.com/settings/billing` は API 課金用で**別物**
2. 請求書テーブル（日付 / 合計 / ステータス / アクション）の対象行の「表示」リンクをクリック
   → 新規タブで Stripe ホスト型請求書（`https://invoice.stripe.com/i/acct_.../live_...`）が開く
3. **YYYY-MM の導出**：Stripe 請求書の **"Date of issue"** 基準（claude.ai の表示日付とは異なる点に注意）
4. 「請求書をダウンロード」ボタンをレシピ A でダウンロード → リネーム移動
5. Stripe タブを閉じて元のタブに戻る

## 注意・落とし穴

- Stripe ページの URL（`acct_xxx` を含む）は機密扱い

## 確認済み実例

- メール経由（2026-06-01 実行）：検索 `from:mail.anthropic.com` でヒット。添付の aria-label クリックで `.playwright-mcp/` に自動保存
- claude.ai 経由（2026-05-24 初回実行）：Stripe 請求書番号は `XXXXXXXX-NNNN` 形式。"Date of issue" でファイル名の年月を決定
