# Phase 25: 新サービス追加手順のスキル化

## 課題

1. Phase 24 の download スキル単一化により、新サービスの追加は「`services/<svc>.yml` 作成 + `download/references/<svc>.md` 作成 + `local.example.yml` 追記」の定型作業になったが、この手順自体はどこにも明文化されておらず、既存ファイルの模倣と CLAUDE.md・SECURITY.md の暗黙ルールの読み取りに依存している
2. Fable 以外のモデルで作業した場合、暗黙ルール（`filename_pattern` の一元管理、PII のプレースホルダー化、`journal_rule` の初回確定フロー、`billing_modes` の要否判断、references の記述粒度）を見落として成果物の品質がばらつく懸念がある

## 方針

- 新サービス追加の手順・成果物テンプレート・完了チェックリストを **`/service-new <サービスキー>` スキル**として明文化する（`/phase-new` と同じ「対象-new」の命名規則）
- テンプレートは既存 6 サービスの実ファイルから抽出し、SKILL.md に埋め込む（別ファイルに分割するほどの分量ではない）
- スキルに含める手順:
  1. 事前ヒアリング（請求書の入手経路・請求頻度・複数対象の有無・勘定科目）
  2. `services/<svc>.yml` の作成（`journal_rule` は `status: 未確定` で開始）
  3. 実機調査 → `download/references/<svc>.md` の作成（方式レシピ A/B/C の選択基準・確認済み実例の記録）
  4. `local.example.yml`（プレースホルダー）・`local.yml`（実値）への追記
  5. ドキュメント更新（CLAUDE.md「サービスと保存先」表・`/download` `/register` の description のサービス列挙）
  6. `/download <svc>` での動作確認
  7. PII チェック（SECURITY.md §9）

## 決定事項ログ

### 2026-07-03: 新サービス追加手順をスキル化する

- ユーザー指示。Phase 24 の単一化で追加手順が定型化されたため、スキルとしてルールを明文化し、Fable 以外のモデルで作業した場合でも精度が出るようにする
- スキル名は `/service-new`（`/phase-new` と同じ命名規則）
- Phase 24 決定事項「download スキルを量産するジェネレータスキルは作らない」とは矛盾しない：本スキルはスキルを生成するのではなく、設定ファイル（yml + references）の作成手順を定型化するもの
