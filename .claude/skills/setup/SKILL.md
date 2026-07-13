---
name: setup
description: 初期セットアップ支援。local.yml 作成の案内、利用スタイル（ダウンロードのみ／仕訳登録も使う）の確認、ダウンロードのみの場合の MoneyForward MCP 無効化（書き込み前に必ずユーザー確認）を行う。
---

# 初期セットアップ

新規クローン直後、または利用スタイル（ダウンロードのみ⇔仕訳登録も使う）を変更したいときに実行する。
カード口座名・取引先コード等の個人情報はこのスキルでは生成・推測せず、ユーザー本人に `local.yml` へ
記入してもらう（`.claude/config/README.md` 参照）。

## 引数

なし。

## 手順

### Step 1: local.yml の確認

`.claude/config/local.yml` の存在を確認する。

- **存在する場合**：既存の設定を尊重し、そのまま Step 2 へ進む。
- **存在しない場合**：`.claude/config/local.example.yml` を `.claude/config/local.yml` に
  コピーしてよいかユーザーに確認する（雛形はプレースホルダーのみで個人情報を含まないためコピー自体は
  安全だが、書き込み前に確認する）。承認後にコピーし、`.claude/config/README.md` を参照して
  カード口座・取引先コード・所有ドメイン等の個人設定を記入するようユーザーに案内する
  （記入自体はユーザー本人が行う）。

### Step 2: 利用スタイルの確認

ユーザーに次のどちらかを確認する（`AskUserQuestion` 等で選択させる）：

- **ダウンロードのみ**：各サービスの請求書 PDF を `/download` で取得するだけ。仕訳登録は使わない。
- **仕訳登録も使う**：`/register`・`/monthly`・`/credit-payment` でマネーフォワードへの仕訳登録まで行う。

### Step 3: MoneyForward MCP の無効化（ダウンロードのみの場合のみ・要ユーザー確認）

Step 2 で「ダウンロードのみ」と回答された場合にのみ実施する。「仕訳登録も使う」の場合は
何もしない（MoneyForward MCP は通常どおり有効のままにする）。

1. `.claude/settings.local.json`（Git 管理外）を Read する（存在しない場合は新規作成する前提で進める）。
2. `disabledMcpjsonServers` に `"moneyforward"` を追加する変更内容を**ユーザーに提示し、
   書き込みの実行可否を確認する**（提案なしに無断で書き込まない）。
3. 承認が得られたら Edit/Write で反映する（既存の `disabledMcpjsonServers` があれば配列に追加、
   無ければキーごと新設）。
4. 次のように案内する：「MoneyForward MCP を無効化しました。Claude Code の再起動後から OAuth 認証を
   求められなくなります。後で仕訳登録も使いたくなった場合は `/register` 等を実行すると
   `.claude/skills/_shared/security.md`『MoneyForward MCP 有効化チェック』により自動的に
   有効化されます（そのときも再起動が必要です）」

### Step 4: 次のステップ案内

- Claude Code を起動（Step 3 で設定を変更した場合は再起動）する。`.mcp.json` に定義された
  `playwright`（Step 3 で無効化していなければ `moneyforward` も）の MCP サーバー使用を確認する
  プロンプトが出るので許可する。
- `/download` を実行すると Playwright がブラウザ（Chrome）を起動する。各サービス未ログインの
  場合はそのブラウザ上で手動ログインする（ログイン状態は `.playwright-profile/` に永続化され、
  次回以降は不要）。
- 仕訳登録も使う場合、`/register` 等の初回実行時に MoneyForward MCP の OAuth 認可のためブラウザが
  開くので、マネーフォワードにログインしてアプリ連携を許可する（以後は再認証不要）。

## エラーハンドリング

`.claude/skills/_shared/security.md` を参照。
