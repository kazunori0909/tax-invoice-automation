# Phase 32: MoneyForward MCP の遅延有効化

## 課題

`.mcp.json` に `playwright` と `moneyforward` の2サーバーを定義しているため、
`/download` のみを使う（請求書ダウンロードだけで仕訳登録はしない）利用シーンでも
MoneyForward MCP の OAuth 認証（`mfc_ca_authorize` → `mfc_ca_exchange`）が発生してしまう。
請求書ダウンロードだけなら MoneyForward MCP は不要なため、初期セットアップ時点では
無効化しておき、`/register`・`/credit-payment` を実行するときにだけ有効化したい。

## 方針

### MCP サーバーの有効/無効切り替えの仕組み（claude-code-guide agent による確認済み）

- `.mcp.json` 自体にはサーバー単位の `disabled` フィールドは存在しない（接続設定のみ）
- `claude mcp` CLI にも `enable`/`disable` サブコマンドは存在しない
- 有効/無効は **`.claude/settings.json`（または `settings.local.json`）の
  `disabledMcpjsonServers` / `enabledMcpjsonServers`（string 配列）でのみ制御可能**
  （インストール済み Claude Code 拡張の `claude-code-settings.schema.json` で実在を確認済み）
- 両方に同名サーバーが登録されている場合は **denylist（`disabledMcpjsonServers`）が優先**
- **`.mcp.json` 同様、settings のこれらのキーも Claude Code のセッション開始時にのみ読み込まれる**。
  実行中のセッション内で書き換えても即時反映されず、**セッション再起動が必須**

### 採用案（2026-07-13 修正：`settings.json` は一切変更しない方式に変更）

`disabledMcpjsonServers` は **denylist が全スコープで union（合算）** される
（claude-code-guide agent が公式ドキュメント `mcp.md` の記述で確認：
「A `disabledMcpjsonServers` entry in any settings file still rejects the server」）。
そのため、コミット対象の `.claude/settings.json` に無効化を書いてしまうと、
Git 管理外の `settings.local.json` 側だけを書き換えても上書き（再有効化）できない。

→ **無効化・有効化のどちらも `.claude/settings.local.json`（Git 管理外・`.gitignore` 済み）
だけで完結させる**。コミット対象ファイルには一切触れないため、Git の diff は発生しない。

1. **既定の無効化は「セットアップ手順」としてユーザー自身の `settings.local.json` に書く**。
   `local.yml`（個人設定）と同じ考え方：個人の利用スタイル（ダウンロードのみ／仕訳登録も使う）に
   関わる設定は Git 管理外ファイルに置く。
   - `README.md` のセットアップ手順に、「ダウンロードのみ使う場合は
     `.claude/settings.local.json` に `"disabledMcpjsonServers": ["moneyforward"]` を追記する
     （仕訳登録も使うなら不要・省略可）」という任意ステップを追加する。
   - `playwright` は常時必要（ダウンロードに必須）なので対象外。
2. **`/register`・`/credit-payment` の前提条件チェック（Step 0）に「MCP 有効化チェック」を追加**。
   - moneyforward の MCP ツールが見当たらない（ToolSearch で不在）場合、無効化されていると判断。
   - `.claude/settings.local.json` を Read し、`disabledMcpjsonServers` から `"moneyforward"` を
     除去して Edit（配列が空になった場合はキー自体を削除。ファイル自体が存在しない場合は
     何もせず「MCP は無効化されていない＝別の原因」として扱う）。
   - ユーザーに「MoneyForward MCP を有効化しました。Claude Code の再起動（セッション再開）後に
     もう一度実行してください」と伝えてスキルを停止する（同一セッション内では継続不可のため）。
3. 手順は `_shared/`（`.claude/skills/_shared/security.md` 等）に共通スニペットとして
   1箇所にまとめ、`register/SKILL.md` と `credit-payment/SKILL.md` の前提条件から参照する
   （CLAUDE.md の「手順は共通化」方針・phase 9 の `_shared/` パターンに合わせる）。
4. **無効化に戻す自動処理は設けない**（都度手動でよい。頻度が低いため過剰設計を避ける）。
   ユーザーがダウンロード専用モードに戻したい場合は、手動で `settings.local.json` の
   `disabledMcpjsonServers` に `"moneyforward"` を書き戻す。

### 未確定事項（実装時に実機確認）

- 初回（そのユーザー・マシンで一度も MoneyForward MCP を承認したことがない場合）、
  `disabledMcpjsonServers` から除去して再起動しても、`.mcp.json` 定義サーバーの
  初回承認プロンプト（インタラクティブな「Enable this project's MCP server?」）が
  別途出るかどうかは未検証。実機確認で判明した挙動は KNOWLEDGE.md に記録する。
- `enabledMcpjsonServers` / `disabledMcpjsonServers` のスコープ横断マージについて、
  公式ドキュメントの明示的な記述は限定的（agent 調査でも断定しきれなかった）。
  実装時に実機で「`settings.local.json` のみの編集で確実に有効化できるか」を再検証する。

## 決定事項ログ

### 2026-07-13: MoneyForward MCP を既定で無効化し、register/credit-payment 実行時に自動有効化する方式を採用

- ユーザー要望：初期セットアップでは MoneyForward MCP（OAuth 認証が発生する）を無効にしておき、
  ダウンロードのみの利用では認証を求められないようにしたい。仕訳登録系スキルを使うときだけ
  有効化したい。
- 調査の結果、`.mcp.json` へのコメントアウト相当の機能はなく、`.claude/settings.json` の
  `disabledMcpjsonServers` で制御するのが唯一の正式な方法と判明。これを採用する。
- セッション開始時にしか読み込まれない制約があるため、「スキルが設定を書き換えて
  ユーザーに再起動を促し、いったん停止する」という半自動フローにした
  （完全自動化はできない旨、CLAUDE.md の運用方針とも整合）。

### 2026-07-13: 書き換え対象を `.claude/settings.json`（共有）から `settings.local.json`（個人・Git 管理外）に変更

- ユーザー指摘：`.claude/settings.json` は Git 管理下にあり、有効化のたびにスキルが書き換えると
  コミット対象ファイルに diff が出続けてしまう。
- 再調査の結果、`disabledMcpjsonServers` は denylist としてスコープ横断で union（合算）される
  ため、共有ファイル側に無効化を書いた場合、個人ファイル側だけでは上書き（再有効化）できない
  ことが判明。
- そのため方針を変更し、**無効化の既定値・有効化の書き換えの両方を `settings.local.json`
  （既に `.gitignore` 済み）だけで完結させる**ことにした。共有 `settings.json` は一切変更しない。
  初期の無効化はセットアップ手順（README.md）でユーザー自身に案内する（`local.yml` と同じ
  「個人の利用スタイルは Git 管理外ファイルに書く」という既存パターンに合わせる）。

### 2026-07-13: `/setup` スキルを新設し、初期セットアップ時の MCP 無効化を「利用スタイル確認＋書き込み前確認」つきの対話フローにする

- ユーザー要望：初期セットアップの案内時に、ダウンロードのみか仕訳登録も使うかを必ずユーザーに確認し、
  ダウンロードのみと回答された場合に限り、`disabledMcpjsonServers` の書き込みを実行する前に
  改めてユーザーに確認してから書き込みたい（無断で設定ファイルに書き込まない）。
- これは README.md の手動手順の説明だけでは担保できない（対話的な確認・分岐が必要）ため、
  `.claude/skills/setup/SKILL.md` を新設し、Step 2（利用スタイル確認）→ Step 3（ダウンロードのみの
  場合のみ、書き込み前確認つきで `settings.local.json` を編集）という手順にした。
- `register`/`credit-payment` 実行時の自動有効化（無効化の解除）は、ユーザーがそのスキルを
  明示的に実行した時点で暗黙の同意があるとみなし、従来どおり確認なしで書き換える
  （初期セットアップ時の無効化判断とは非対称。無効化＝ユーザーの利用スタイル選択なので
  慎重に、有効化＝ユーザーが今まさに使おうとしている機能の前提解消なので即時、という整理）。
- CLAUDE.md の「セキュリティ・運用上の注意」に本ルールを明文化した。
