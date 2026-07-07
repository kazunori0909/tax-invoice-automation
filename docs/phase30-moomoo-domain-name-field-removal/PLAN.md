# Phase 30: Moomoo ドメイン設定の `name` フィールド削除（fqdn 完全一本化）

## 課題

`local.yml` の `services.moomoo.domains[]` は `name` と `fqdn` の2フィールドを持つが、
両者は同じドメインの表現違いに過ぎない：

- `name`: TLD を除いたドメイン名。ファイル名の `{domain}` に使用（`services/moomoo.yml` の `filename_pattern`）
- `fqdn`: フルドメイン名。摘要・重複チェックの `{domain_fqdn}` に使用（`duplicate_check.keyword` / `memo_template`）

`name` は `fqdn` から機械的に導出できる値（TLD除去）であり、手動で2つを同期させる必要がある冗長な設計になっている。

## 方針

**`fqdn` に完全一本化し、ファイル名にもフルドメインをそのまま使う**（TLD 除去自体を廃止）。

```yaml
moomoo:
  domains:
    - fqdn: example.com
      billing_month: 2
```

- `filename_pattern` を `YYYY-moomoo-{domain_fqdn_us}-invoice.pdf` に変更
  （`{domain_fqdn_us}` = `fqdn` の `.` を `_` に置換した値。例: `example.com` → `example_com`。
  例: `2026-moomoo-example_com-invoice.pdf`。拡張子は `.pdf` のまま）
  - 置換理由：既存ドメイン名自体にハイフンを含むものがある（例: `my-example.com`）ため、
    「.」を「-」に変換するとドメイン本来の区切りとTLD区切りが区別できなくなる。
    「_」は通常ドメイン名に出現しないため、変換しても曖昧さが生じない
- 摘要・重複チェック（`duplicate_check.keyword` / `memo_template`）は引き続き `{domain_fqdn}`
  （変換なしの生 `fqdn`）を使う。ファイル名用の `{domain_fqdn_us}` とは別のプレースホルダーとして扱う
- 設定値は `fqdn` の1つのみで、導出は「ファイル名生成時に `.`→`_` 置換」という単純ルールのみ
- 既存の請求書 PDF（TLD なし形式）はリネームして移行する（`invoices/` は git 管理外・
  重複チェックは摘要ベースのため影響なし。MF アップロード済みの旧名はそのままでよい）

### 案の比較

| | 案A: TLD 除去で導出 | 案B: fqdn 完全一本化（採用） |
|---|---|---|
| 設定フィールド | `fqdn` のみ | `fqdn` のみ |
| ファイル名 | `…-example-invoice.pdf`（現状維持） | `…-example_com-invoice.pdf`（`.`→`_` 置換） |
| 変換ルール | 「末尾ドット1つを削る」を複数ファイルに記述 | 「ファイル名生成時のみ `.`→`_` 置換」の1ルール |
| プレースホルダー数 | `{domain}` と `{domain_fqdn}` の2つ | `{domain_fqdn_us}`（ファイル名用）と `{domain_fqdn}`（摘要用）の2つ（いずれも `fqdn` 由来） |
| 衝突リスク | `example.com` と `example.jp` が同名に潰れる | なし |
| 複合TLD（`.co.jp`） | 例外フィールド（`name_override`）を将来追加 | 問題自体が存在しない（`.co.jp` → `_co_jp` として一意） |
| 移行コスト | なし | 既存 PDF 2ファイルのリネーム |

## 決定事項ログ

### 2026-07-07: `name` フィールド廃止、`fqdn` に一本化

- ユーザーが `name`/`fqdn` 併記の冗長性を指摘。両者の関係（`name` = `fqdn` からTLD除去）を
  確認した結果、`name` は削除可能と判断。

### 2026-07-07: 設計見直し — TLD 除去自体を廃止し fqdn 完全一本化（案B）を採用

- 当初は案A（ファイル名は TLD なしを維持し fqdn から導出）としたが、見直しで以下の弱点を確認：
  1. 同一セカンドレベル・別TLD（`example.com`/`example.jp`）でファイル名が衝突し、
     「MF アップロードで階層が失われても一意識別」というファイル名規約の目的に反する
  2. フィールド1つの削減と引き換えに導出ルールと将来の例外フィールドが増え、複雑さが減らない
  3. `{domain}` と `{domain_fqdn}` の2プレースホルダーが残る
- 案B はこれらをすべて解消し、コストは既存 PDF 2ファイルのリネームのみのため採用（ユーザー承認済み）。

### 2026-07-07: ファイル名の TLD 区切り文字はアンダースコア（`_`）を採用

- 案Bのファイル名に `fqdn` をそのまま使うと `.` が残る点について、区切り文字の統一を検討。
- ハイフン（`-`）への変換は、既存ドメイン名自体にハイフンを含むもの（`my-example.com` 等）と
  衝突し、どこがTLD区切りか曖昧になるため不採用。
- アンダースコア（`_`）は通常ドメイン名に出現しないため、ハイフン区切りのドメイン名部分と
  明確に区別できる。ファイル名生成専用の `{domain_fqdn_us}`（`.`→`_` 置換）を新設し、
  摘要・重複チェック用の `{domain_fqdn}`（無変換の生 `fqdn`）とは別に扱うことで採用（ユーザー承認済み）。
