# Phase 30 タスク

## 設定ファイル

- [x] `.claude/config/local.yml` の `services.moomoo.domains[]` から `name` を削除（`fqdn` のみ残す）
- [x] `.claude/config/local.example.yml` のサンプルを同様に更新
- [x] `.claude/config/README.md` の moomoo 用設定サンプル（`domains` の書き方）を更新
- [x] `.claude/config/services/moomoo.yml` の `filename_pattern` を
      `YYYY-moomoo-{domain_fqdn_us}-invoice.pdf` に変更し、`{domain}` の説明コメントを削除

## Skill・ドキュメント

- [x] `.claude/skills/download/references/moomoo.md` の「ファイル名の `{domain}`」節を、
      `{domain_fqdn_us}`（`fqdn` の `.` を `_` に置換）の説明に置き換え
- [x] `.claude/skills/register/SKILL.md` の `{domain}` 言及箇所（例外パターン説明）を
      `{domain_fqdn_us}` に更新
- [x] `.claude/skills/monthly/SKILL.md` の `{domain}`（ファイル名生成箇所）を `{domain_fqdn_us}` に、
      `{domain_fqdn}`（摘要・重複チェック箇所）は無変換のまま維持
- [x] docs/README.md の「設定と手順の層構造」フィールド×消費者マップに
      moomoo 関連の記載があれば更新（該当なし。フィールド名に依存しない一般記述のため変更不要）

## 移行

- [x] `invoices/moomoo/` の既存 PDF 2ファイルを新ファイル名（フルドメイン形式）にリネーム

## 共通

- [x] 個人情報チェック（更新した Git 追跡ファイルに実値が混入していないか）
