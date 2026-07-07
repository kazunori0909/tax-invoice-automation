---
name: download
description: 請求書ダウンロード統合スキル。サービス名（anthropic / google-ai / moneyforward / moomoo / wix / xserver 等）または「全部」を引数にとり、references/<svc>.md の固有手順と共通レシピで請求書 PDF を invoices/<svc>/ に保存する。「全部」指定時は local.yml の services キーを利用中サービスとして列挙する。Playwright MCP を使用。
---

# 請求書ダウンロード

## 引数

- **サービス名**（必須）: 個別サービス名（`anthropic` / `google-ai` / `moneyforward` / `moomoo` / `wix` / `xserver` 等）または `全部` / `all`
- **期間**（任意）: 「2026年全部」「2026-05」「直近」など。未指定の場合はユーザーに確認する

`全部` / `all` 指定時は `/register` と同じ規則で、`.claude/config/local.yml` の `services:` 直下のキーを利用中サービスの一覧とみなす（キーの有無＝利用中の宣言）。`local.yml` に存在しないサービスは処理しない。

## 前提条件（全サービス共通）

- `.mcp.json` で `@playwright/mcp` が設定済み
- 対象サービス（または Gmail）にユーザーが手動ログイン済み（`.playwright-profile/` にセッション永続化）
- ログイン画面に遷移した場合は **ユーザーにログインを依頼して停止**（認証情報は入力しない）

## Step 0: 設定と固有手順の読み込み

対象サービス `<svc>` ごとに以下を Read する：

1. **保存先・ファイル名の解決**：保存先は `invoices/<svc>/`（規約・固定）。ファイル名は規約
   （月額 `YYYY-MM-<svc>-invoice.pdf`／年額 `YYYY-<svc>-invoice.pdf`）を既定とし、
   `services/<svc>.yml` に `filename_pattern` の定義がある場合のみそちらを使う（複数ドメイン・複数サイト等、
   規約で表せない例外）。`billing_modes` を持つサービスは実効 frequency（`local.yml` の
   `services.<svc>.frequency` 優先）でモードを解決する（register Step 0 のモード解決と同一）。
   **識別子（ドメイン・サイト名等）をファイル名に含める場合は `.`（ドット）を `_`（アンダースコア）に置換する**
   （例：`example.com` → `example_com`。ドメイン名自体のハイフンは変換しない）。摘要・重複チェック等
   ファイル名以外の用途では無変換の値を使う（moomoo の `{domain_fqdn_us}`＝ファイル名／`{domain_fqdn}`＝摘要、
   wix の `{name_us}`＝ファイル名／`{site}`＝摘要が実例）。新サービスで同様の複数対象の特例が必要な場合も
   この変換規則に合わせる
2. **`local.yml` の `services.<svc>`**：`frequency` / `domains`（moomoo）/ `sites`（wix）等の個人設定
3. **本スキルの `references/<svc>.md`**：取得導線（URL・ページ構造）・使用レシピ・年月の導出ルール・落とし穴

## 共通フロー

各サービスについて以下を実行する（references/<svc>.md が固有の差分を定義する）：

1. **対象の特定**：references の導線に従い、指定期間の請求書（メール・明細行）を特定する
2. **スキップ判定**：`invoices/<svc>/{filename_pattern に従う名前}` が既に存在する対象はスキップ
3. **ダウンロード**：references が指定する下記レシピ（A/B/C）で取得する
4. **リネーム移動**：`mv` で `filename_pattern` に従う名前へ移動する。年月・年の値は references の導出ルールに従う
5. **後片付け**：開いた追加タブを閉じる（`browser_tabs action=close`）。全件完了後に `browser_close`

複数件を一括ダウンロードする場合は 1〜4 を対象ごとに繰り返す。

## ダウンロード方式レシピ

### レシピ A: download_event（ダウンロードボタンをクリック）

```
mcp__playwright__browser_click  (target = ダウンロードボタン/リンクの ref)
```

Playwright MCP が download イベントを自動捕捉し、`.playwright-mcp/` に保存する。

```bash
mv .playwright-mcp/<保存されたファイル> invoices/<svc>/<filename_pattern に従う名前>
```

ZIP で落ちてくるサービスは解凍してから移動する：

```bash
cd .playwright-mcp && unzip -o <file>.zip && mv <中身>.pdf ../invoices/<svc>/<名前> && rm <file>.zip
```

### レシピ B: page_pdf（HTML ページを page.pdf() で PDF 化）

請求書が HTML でしか提供されない場合。対象タブに切り替えてから実行する：

```
mcp__playwright__browser_tabs  (action=select, index=1)
mcp__playwright__browser_run_code_unsafe
  code: async (page) => {
    await page.pdf({
      path: '<プロジェクトルートの絶対パス>/.playwright-mcp/<svc>.pdf',
      format: 'A4',              // またはページ幅に合わせた width/height 指定（references 参照）
      printBackground: true,
      margin: { top: '15mm', bottom: '15mm', left: '15mm', right: '15mm' },
    });
  }
```

- `page.pdf()` は headed Chrome でも動作する（@playwright/mcp 環境で確認済み）
- レイアウト（format か width/height 指定か・margin）は references 側の指定に従う
- 印刷ビューが `window.print()` を自動で呼ぶ場合（Gmail 等）は、開く**前に**抑制する（下記 Gmail 共通操作）

### レシピ C: fetch_base64（PDF タブから fetch で取得）

新規タブに PDF が直接開き、download イベントが発生しない場合。
`browser_run_code_unsafe` は Node.js グローバル（`require`・`Buffer`・`fs` 等）が使えないため、
ブラウザ側 fetch + Bash デコードで取得する：

```
mcp__playwright__browser_tabs  (action=select, index=1)
mcp__playwright__browser_evaluate
  code: |
    const res = await fetch(window.location.href, { credentials: 'include' });
    const bytes = new Uint8Array(await res.arrayBuffer());
    let binary = '';
    for (let i = 0; i < bytes.byteLength; i++) binary += String.fromCharCode(bytes[i]);
    return btoa(binary);
  filename: .playwright-mcp/<svc>-base64.txt
```

出力はダブルクォートで囲まれた JSON 文字列のため、除去してからデコードする（macOS は `base64 -D`）：

```bash
tr -d '"' < .playwright-mcp/<svc>-base64.txt > .playwright-mcp/<svc>-b64-clean.txt
base64 -D -i .playwright-mcp/<svc>-b64-clean.txt -o invoices/<svc>/<名前>.pdf
rm .playwright-mcp/<svc>-base64.txt .playwright-mcp/<svc>-b64-clean.txt
```

### Gmail 共通操作（メールが証憑になるサービス）

- 検索 URL：`https://mail.google.com/mail/u/0/#search/<URL エンコードした検索クエリ>`（例：`from%3Amail.anthropic.com`）
- メール一覧の行は `tr.zA`、開いたメールの本文は `.a3s`。行クリックは JS で行う：

```javascript
const rows = Array.from(document.querySelectorAll('tr.zA'));
rows.find(r => r.innerText.includes('<件名キーワード>')).click();
```

- 添付 PDF のダウンロードはレシピ A：`[aria-label="添付ファイル <ファイル名> をダウンロード"]` をクリック
- **メール本文自体を PDF 化する場合**（レシピ B）：「すべて印刷」ボタンが新規印刷タブを開くと同時に
  `window.print()` を呼び、Chrome の印刷ダイアログが立ち上がってしまう。クリック前に必ず抑制する：

```
mcp__playwright__browser_run_code_unsafe
  code: async (page) => {
    await page.context().addInitScript(() => { window.print = () => {}; });
  }
```

その後「すべて印刷」（`[role="button"]` の `aria-label="すべて印刷"`）をクリック → 印刷タブでレシピ B を実行。

- 印刷ビュー（`view=pt`）の URL 直接ナビゲートは `ik`（アカウントハッシュ）・`permthid` の導出が必要なため、印刷ボタンクリックの方が確実
- Gmail の DOM クラス名（`.a3s` 等）は短く変わりやすいが、現状は安定して使える
- 印刷ビュー URL にはアカウント識別子（`ik=<hash>`）が含まれるため機密扱い

## 注意事項（全サービス共通）

- セキュリティ・共通エラーは `.claude/skills/_shared/security.md` を参照
- 請求書番号・取引 ID・アカウント識別子を含む URL やファイル名も機密扱い（ログ・コミットに含めない）
- ダウンロード失敗時はユーザーに報告して停止（リトライしない）
