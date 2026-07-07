# Xserver 固有手順

**方式**：レシピ A（ZIP で落ちてくるため解凍が必要）

## 手順

1. `https://secure.xserver.ne.jp/xapanel/xserver/payment/history/index` を開く
   （未ログイン時は `/xapanel/login/xserver/` に飛ぶ）
2. **検索期間を対象年の 1月〜12月に設定**：デフォルトは「今年1月〜今月」のため、
   終了月の select を `12` にして「検索する」をクリック。
   過去年は `select[name="start_year"]` / `select[name="end_year"]` も変更する
3. 対象行のチェックボックスを ON：`input[name="bill_id_list[]"]` はセル td が click を
   インターセプトするため、**行（`<tr>`）自体をクリック**する
4. 「請求書一括ダウンロード」ボタンをレシピ A でクリック
   → `.playwright-mcp/` に **ZIP 形式**で保存される（`seikyu<請求情報番号>-xserver<hash>.zip`、
   中に `seikyu<請求情報番号>-xserver.pdf`）。1 ファイルでも ZIP
5. 解凍 → リネーム移動（レシピ A の ZIP 手順）

## 年の導出

ファイル名の `YYYY` は**支払日の年**（例：2026/04/27 → `2026`）。
年額契約なら年内の請求は通常 1 件のみ（複数あればすべてダウンロード対象）。

## 注意・落とし穴

- 請求情報番号・サーバー ID は機密扱い
- 「お支払い/請求書発行」ページ（`/payment/index`）は**未払いの請求情報**のリスト。
  過去の支払いは「お支払い履歴／受領書発行」ページ（`/payment/history/index`）
- 受領書は請求書があれば証憑として不要なためダウンロードしない

## 確認済み実例（2026-05-24 初回実行）

- 対象年の 1〜12 月で検索し、年次更新の支払い 1 件がヒット（更新月は `local.yml` の `services.xserver.billing_month` 参照）
- ZIP を解凍して `invoices/xserver/YYYY-xserver-invoice.pdf` に保存
