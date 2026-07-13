# Phase 32 KNOWLEDGE

## `disabledMcpjsonServers` の実機挙動（2026-07-13 実機確認）

- `.claude/settings.local.json` に `"disabledMcpjsonServers": ["moneyforward"]` を書いて
  Claude Code を再起動すると、`mcp__moneyforward__*` のツールは一覧から完全に消える
  （`.mcp.json` にサーバー定義自体は残っていても、ツールとしては見えなくなる）。
  一方 `playwright` は対象外にしているため通常どおり利用できる。
- 上記の状態から `disabledMcpjsonServers` の配列（またはキー自体）を削除して再起動すると、
  `mcp__moneyforward__*` のツールは問題なく復活する（19個確認）。
  **初回承認プロンプト（「Enable this project's MCP server?」等）は表示されなかった**
  （このマシン・ユーザーでは過去に一度 MoneyForward MCP を承認済みだったため。
  一度も承認したことがない環境での挙動は未検証）。
- 上記より、**単一ファイル（`settings.local.json`）内での追記・削除だけで無効化↔有効化を
  確実に往復できる**ことを確認済み。共有 `.claude/settings.json` は無関係・変更不要。
- 設定の反映はドキュメント記載どおり**セッション開始時のみ**で、再起動なしでは反映されない
  （同一セッション内でファイルを書き換えても `ToolSearch` の結果は変わらないことを確認）。
