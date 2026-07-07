---
name: phase-new
description: 新規開発フェーズの追加。テーマを引数にとり、次のフェーズ番号を採番して docs/phase<N>-<slug>/ に PLAN.md・TASKS.md の雛形を作成し、docs/README.md の開発フェーズ記録テーブルへ行を追加する。
---

# 新規フェーズ追加

## 引数

- **テーマ**（必須）: このフェーズで取り組む課題・作業内容（例:「Wix のマルチサイト対応」）。
  未指定の場合はユーザーに確認する。会話の文脈で課題・方針が既に固まっていれば、それを雛形に反映する。

## 手順

### Step 1: 採番

以下の**両方**を確認し、最大番号 + 1 を新フェーズ番号 `N` とする。
（圧縮済みフェーズはフォルダが削除されているため、フォルダだけを見ると番号が巻き戻る）

1. `docs/phase*/` のフォルダ名に含まれる番号
2. `docs/README.md` の「開発フェーズ記録」テーブルの番号列

### Step 2: slug 決定

フォルダ名は `docs/phase<N>-<slug>/`。slug は**英語 kebab-case 2〜4語**で内容を要約する。
既存例: `register-date-fix` / `wix-multi-site` / `config-simplification` / `google-ai-billing`

### Step 3: PLAN.md 作成

`docs/phase<N>-<slug>/PLAN.md` を以下の構成で作成する。
会話で既に判明している内容は埋め、未定の箇所は見出しとプレースホルダーだけ残す。

```markdown
# Phase <N>: <タイトル>

## 課題

（現状の何が問題か。複数あれば番号付きで）

## 方針

（どう解決するか。案の比較をしたなら案A/案Bと採否理由も書く）

## 決定事項ログ

### YYYY-MM-DD: <決定の見出し>

- （ユーザーが決めた／承認した内容と、その理由）
```

書き分け基準（CLAUDE.md「PLAN.md と KNOWLEDGE.md の書き分け」）:
PLAN.md には**設計上の選択とその理由**を書く。ツール・外部サービスの挙動の発見は KNOWLEDGE.md（Step 5 参照）。

### Step 4: TASKS.md 作成

`docs/phase<N>-<slug>/TASKS.md` をチェックボックス形式で作成する。
判明しているタスクを列挙し、CLAUDE.md「Skill や仕訳ルールを変更・追加した際の更新ルール」に
該当する項目（SKILL.md 更新・docs/README.md 更新等）があればタスクに含める。

```markdown
# Phase <N> タスク

- [ ] <タスク>

## 共通

- [ ] 個人情報チェック（更新した Git 追跡ファイルに実値が混入していないか）
```

### Step 5: KNOWLEDGE.md は雛形として作らない

CLAUDE.md の作成条件どおり、**実機操作・外部サービス調査で気づきが出た時点で作成する**
（空の雛形を先に置かない）。実機調査を伴う見込みのフェーズでは、TASKS.md の「共通」に
`- [ ] 実機調査で判明した挙動・制約を KNOWLEDGE.md に記録` を入れておく。

### Step 6: docs/README.md のテーブル更新

「開発フェーズ記録」テーブルの末尾に行を追加する:

```markdown
| <N> | <フェーズ名>（進行中） | <要点1行> | [phase<N>/](phase<N>-<slug>/) |
```

フェーズ完了時は「（進行中）」を「✅」に変える（docs/README.md「進捗管理ルール」参照）。

### Step 7: 個人情報チェック

作成・更新した Git 追跡ファイル（PLAN.md・TASKS.md・docs/README.md）に個人情報を
書いていないか確認する。実値が必要な場合は `local.yml` 参照／プレースホルダーで表現する
（SECURITY.md §9「個人情報を Git 追跡ファイルに書かないルール」）。
