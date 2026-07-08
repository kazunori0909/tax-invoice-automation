# tax-invoice-automation

確定申告向けの仕訳作業を半自動化するプロジェクト。  
各種サービスの請求書をダウンロードし、マネーフォワード クラウド会計へ仕訳登録する。

## できること

- 各種サブスクリプション・サービスの請求書 PDF を自動ダウンロード（ログインのみ人が実施）
- マネーフォワード クラウド会計への仕訳登録・証憑 PDF 添付を自動化
- `/monthly` コマンド1つで月次・年次の定型作業を一括実行
- 完全自動化はせず、登録前の確認など要所で人の判断を挟む半自動運用

## 対象サービス

利用中サービスは `local.yml` の `services:` で宣言する（詳細は [docs/README.md](docs/README.md) 参照）。

| サービス | トップページ | 請求書の取得元 | ヘルプ |
|---|---|---|---|
| マネーフォワード クラウド | [biz.moneyforward.com](https://biz.moneyforward.com) | [利用明細](https://erp.moneyforward.com/office_usage_detail_statements) | [請求書の確認方法](https://biz.moneyforward.com/support/plan/guide/g010.html) |
| Anthropic (Claude) | [claude.ai](https://claude.ai) | メール添付 PDF（Gmail）。<br/>無い月は [Billing](https://claude.ai/settings/billing) からフォールバック | [Paid Plan Billing FAQs](https://support.claude.com/en/articles/8325618-paid-plan-billing-faqs) |
| Google AI | [one.google.com](https://one.google.com) | [Google Pay 取引履歴](https://pay.google.com/payments/home)（Google One 経由） | [Google での購入履歴を確認する](https://support.google.com/googleone/answer/11828789?hl=ja) |
| Xserver | [xserver.ne.jp](https://www.xserver.ne.jp) | [料金支払い履歴](https://secure.xserver.ne.jp/xapanel/xserver/payment/history/index) | [請求書・受領書・見積書](https://www.xserver.ne.jp/manual/man_order_pay_bill.php) |
| ムームードメイン | [muumuu-domain.com](https://muumuu-domain.com) | メール本文を PDF 化<br/>（PDF 発行なし・メールが適格請求書） | [適格請求書・領収書は発行してもらえますか](https://support.muumuu-domain.com/hc/ja/articles/360045800534) |
| Wix | [wix.com](https://www.wix.com) | [請求履歴](https://manage.wix.com/account/billing-history) | [Wix サービスの請求書](https://support.wix.com/ja/wix-%E3%82%B5%E3%83%BC%E3%83%93%E3%82%B9%E3%81%AE%E8%AB%8B%E6%B1%82%E6%9B%B8) |

## セットアップ

### 前提条件

- [Claude Code](https://docs.claude.com/claude-code) がインストール済みであること
- Node.js / npm（`npx` が使えること。`.mcp.json` の MCP サーバーは `npx` 経由で起動する）
- Google Chrome がインストール済みであること（Playwright MCP は `--browser=chrome` でローカルの Chrome を操作する）
- メールで届く請求書を扱うサービス（Anthropic・ムームードメイン等）は Gmail 前提（Gmail の画面構造に依存した操作のため、他メールサービスは非対応）
- **仕訳の自動登録（`/register`・`/monthly`・`/credit-payment`）を使う場合のみ**
   - マネーフォワード クラウド会計の契約・ログイン情報。  
     請求書ダウンロード（`/download`）だけの利用なら不要。  
     MCP サーバーの公式設定手順は[マネーフォワード クラウド会計のヘルプ](https://biz.moneyforward.com/support/account/guide/others/ot10.html#ttl02)を参照

### 手順

1. リポジトリをチェックアウトする
2. `.claude/config/local.example.yml` を `.claude/config/local.yml` にコピーし、
   カード口座・取引先コード・所有ドメインなど個人設定を記入する。
   書き方の詳細は [.claude/config/README.md](.claude/config/README.md) を参照
3. このディレクトリで Claude Code を起動する。`.mcp.json` に定義された `playwright` / `moneyforward` の
   MCP サーバー使用を確認するプロンプトが出るので許可する
4. `/download` や `/monthly` を初めて実行すると Playwright がブラウザ（Chrome）を起動する。
   各サービスに未ログインの場合はそのブラウザ上で手動ログインする（ログイン状態は `.playwright-profile/` に永続化され、次回以降は不要）
5. `/register` 等でマネーフォワード MCP に初めてアクセスすると、OAuth 認可のためブラウザが開く。
   マネーフォワードにログインし、アプリ連携を許可する（以後は再認証不要）

## 運用スケジュール

| タイミング | スキル | 内容 |
|---|---|---|
| 月末〜翌月初 | `/monthly` | 請求書ダウンロード → 仕訳登録（月次サービス毎月・年次サービスは対象月のみ） |
| カードの引き落とし日以降 | `/credit-payment` | クレジットカード引き落とし仕訳の確認・自動登録 |

> 引き落とし日はカードごとに `local.yml` の `credit_cards`（`closing`/`payment_day`）で決まる。土日祝の場合は翌営業日。

詳細は [docs/README.md](docs/README.md) を参照。

## アーキテクチャ

ユーザーがブラウザで手動ログイン → Claude Code が Playwright MCP でブラウザ操作して請求書をダウンロード → マネーフォワード MCP で仕訳登録（人による確認込み）。

サービス認証情報はプロジェクトに保存しない。

## ディレクトリ構成

```
tax-invoice-automation/
├── CLAUDE.md                   # 仕訳ルール・運用方針
├── docs/                       # フェーズ別の計画・タスク・運用スケジュール
├── .claude/skills/             # Skill（/monthly, /download, /register, /credit-payment 等）
├── .claude/config/             # サービス設定（services/*.yml）・個人設定（local.yml・git 管理外）
├── .mcp.json                   # MCP 設定
├── invoices/                   # 請求書 PDF（git 管理外）
└── .playwright-profile/        # Playwright ブラウザプロファイル（git 管理外）
```

## 参考リンク

- [Claude Code ドキュメント](https://docs.claude.com/claude-code)
- [Playwright MCP](https://github.com/microsoft/playwright-mcp) — ブラウザ操作・請求書ダウンロードに使用
- [マネーフォワード クラウド会計 MCP の設定手順（公式ヘルプ）](https://biz.moneyforward.com/support/account/guide/others/ot10.html#ttl02)
- [マネーフォワード クラウド 開発者向けドキュメント](https://developers.biz.moneyforward.com)
