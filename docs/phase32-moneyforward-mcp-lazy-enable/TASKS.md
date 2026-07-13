# Phase 32 タスク

- [x] `README.md` のセットアップ手順に、「ダウンロードのみ使う場合は任意で
      `.claude/settings.local.json` に `"disabledMcpjsonServers": ["moneyforward"]` を
      追記する（仕訳登録も使うなら不要）」という手順を追加
      （**`.claude/settings.json` は変更しない**）
- [x] `.claude/skills/_shared/security.md` に
      「MoneyForward MCP 有効化チェック」の共通手順を追加
      （ツール不在検知 → `settings.local.json` 編集 → 再起動を促して停止。
      ファイルが無い/キーが無い場合は「無効化が原因ではない」として通常のエラーハンドリングに委ねる）
- [x] `register/SKILL.md` の前提条件（Step 0）から上記共通手順を参照するよう追記
- [x] `credit-payment/SKILL.md` の前提条件（Step 0）から上記共通手順を参照するよう追記
- [x] `docs/README.md`（開発フェーズ記録テーブル）を更新。運用スケジュール・スキル一覧は
      MCP 内部実装の詳細のため変更不要と判断
- [x] 実機確認：無効化→再起動→ツール消失／解除→再起動→ツール復活のフルサイクルを実施し、
      期待どおりの挙動を確認（初回承認プロンプトは今回の環境では出なかった。未承認環境は未検証）
- [x] 実機調査で判明した挙動・制約を KNOWLEDGE.md に記録
- [x] `/setup` スキルを新規作成：`local.yml` 作成案内・利用スタイル確認・
      ダウンロードのみの場合の MoneyForward MCP 無効化を書き込み前確認つきで実施
- [x] `README.md` セットアップ手順を `/setup` 起点に書き換え（手動手順は fallback として残す）
- [x] `docs/README.md` に「セットアップスキル」セクションを追加し `/setup` を掲載
- [x] `CLAUDE.md` に「MCP 無効化は利用スタイル確認＋書き込み前確認が必須」のルールを追記、
      ディレクトリ表に `setup/` を追加

## 共通

- [x] 個人情報チェック（更新した Git 追跡ファイルに実値が混入していないか。diff を確認し実値混入なし）
