# Wix プレミアムプラン固有手順

**方式**：レシピ C（新規タブに PDF が直接開き、download イベントが発生しない）

## Step 0 の補足：対象サイトの解決

`local.yml` の `services.wix.sites` を読む：

- **`sites` 指定あり**：リストのサイトのみ対象。`name` はサイト識別子（摘要・テーブル照合に使用。例：`example.com`）
- **`sites` 未指定**：請求履歴テーブルに表示される全サイトを対象とし、各行からサイト識別子を読み取る

**ファイル名用の正規化**：`name` のドットをアンダースコアに変換（例：`example.com` → `example_com`。識別子のファイル名変換規則は `download/SKILL.md` Step 0 参照）。摘要には元の値をそのまま使用する。プラン名は local.yml に設定不要（`/register` 実行時に請求書 PDF の明細列から読み取る）。

## 手順

1. `https://manage.wix.com/account/billing-history` を開く
2. snapshot でテーブル全行を確認し、対象年月 × 対象サイトの行を特定
3. 各対象行：「…」ボタン → ポップアップメニューの「明細書を表示」をクリック
   → 新規タブに PDF が開く（URL：`https://manage.wix.com/premium-invoice/api/v1/invoice/{invoice_id}`）
4. 新タブでレシピ C を実行 → リネーム移動 → PDF タブを閉じて次の行へ

## 年月の導出

`YYYY-MM` は請求書の**発行日**の年月（ページ内または URL から確認。例：2026年5月28日発行 → `2026-05`）

## 注意・落とし穴

- 請求書 ID（URL 内）は機密扱い

## 確認済み実例（2026-06-05 初回実行）

- 「…」→「明細書を表示」で新タブに PDF が開き、fetch → base64 → `tr -d '"'` → `base64 -D` でデコード成功

## 未確認事項（次回実機テスト時に確認）

- 複数サイト管理時に billing-history テーブルがどのような列構成になるか
- テーブルのサイト識別子が `name`（例：`example.com`）と一致するかどうか
- 確認でき次第 KNOWLEDGE.md に記録し、必要に応じて本ファイルを更新する
