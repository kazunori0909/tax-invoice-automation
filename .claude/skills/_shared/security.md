# 共通セキュリティ注意事項・エラーハンドリング

## セキュリティ注意事項

- 認証情報・トークンをログやコミットメッセージに含めない
- 仕訳登録は必ずユーザー確認後に実行する
- `.env` は読み書きしない

## 共通エラーハンドリング

サービス固有のエラーは各スキルの「エラーハンドリング」セクションを参照。

| 状況 | 対応 |
|------|------|
| MCP 認証切れ | `mfc_ca_authorize` → `mfc_ca_exchange` を実施してリトライ |
| ブラウザ未ログイン | ユーザーにログインを依頼して停止 |
| PDF が存在しない | `/download <service>` でダウンロード後に再実行するよう案内 |
| 未仕訳明細が見つからない | ユーザーに報告して停止（カード連携・同期を確認するよう依頼） |

## MoneyForward MCP 有効化チェック

ダウンロード専用利用では OAuth 認証を発生させないため、`.claude/settings.local.json`
（Git 管理外）の `disabledMcpjsonServers` で MoneyForward MCP を無効化しておく運用を想定している。
MoneyForward MCP を使うスキル（`register`・`credit-payment`）の前提条件チェックで、
以下の手順により無効化を検知・解除する。

1. moneyforward の MCP ツール（`mcp__moneyforward__*`）が呼び出せるか確認する
   （見つからない・接続できない場合、無効化されている可能性がある）。
2. `.claude/settings.local.json` が存在すれば Read し、`disabledMcpjsonServers` 配列に
   `"moneyforward"` が含まれているか確認する。
   - ファイルが無い／キーが無い／配列に含まれない場合：無効化が原因ではないので、
     上記「共通エラーハンドリング」の MCP 認証切れ等の通常フローに委ねる。
3. 含まれる場合、その配列から `"moneyforward"` を除去して Edit する
   （配列が空になったらキーごと削除し、他のキーはそのまま残す）。
4. ユーザーに次のように伝えてスキルを停止する（同一セッション内では有効化できないため）：
   「MoneyForward MCP が無効化されていたため `.claude/settings.local.json` を編集して
   有効化しました。Claude Code を再起動（セッションを再開）してから、もう一度実行してください」

**共有 `.claude/settings.json` は変更しない**（Git 管理下のため。個人の利用スタイルに関する
設定は Git 管理外の `settings.local.json` にのみ持たせる）。
