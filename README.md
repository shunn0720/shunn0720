# Claude Code 実装指示プロンプト集
# 設計書（demo-server-spec.md）と一緒にこのファイルの指示を1つずつ貼って使う

---

## ■ 事前準備（最初に1回だけ実行）

```
プロジェクトディレクトリ tsunagu-demo-bot/ を作成し、
以下のサブディレクトリも作ってください。
- cogs/
- utils/

空の __init__.py を cogs/ と utils/ に作成してください。
```

---

## ■ Phase 1: 基盤ファイル作成

以下の指示をClaude Codeに貼る：

```
設計書（demo-server-spec.md）に従って、以下の4ファイルを作成してください。
省略せず、全コードを出力してください。

【1. config.py】
- セクション4のチャンネル構成（カテゴリ名・チャンネル名・VC）を定数で定義
- セクション5のロール構成（名前・色コード）を定数で定義
- セクション10のEmbed色ルール（教育系0xffc6e2、福祉系0xc6c6ff、通知0xffffc6、成功0x4ECDC4、エラー0xFF6B6B）を定数で定義
- 環境変数のキー名を定数で定義
- サーバー名「つなぐラボ｜Discord×AI 体験」を定数で定義

【2. db.py】
- セクション9のdb.py接続方式に従って実装
- asyncpg.create_pool() でコネクションプール管理
- init_tables() でBot起動時にテーブル自動作成（セクション8のSQL）
- CREATE INDEX には IF NOT EXISTS を必ず付ける

【3. utils/embed_builder.py】
- config.pyのEmbed色定数を使って、用途別のEmbed生成関数を提供
- education_embed(), welfare_embed(), notify_embed(), success_embed(), error_embed()
- 各関数はtitle, description, fieldsを受け取りdiscord.Embedを返す
- フッターに「つなぐラボ｜Discord × AI で現場を支える」を共通設定

【4. main.py】
- セクション9のmain.py接続管理コードに従って実装
- discord.py v2.3.2+ のcommands.Botを使用
- setup_hook()でDB接続→テーブル作成→Cog読み込み→PersistentView登録→tree.sync
- close()でプール解放
- on_ready でログ出力
- ローカルは.env、Railwayは自動注入で条件分岐
- intents: message_content=True, members=True, guilds=True
- command_prefix は環境変数BOT_PREFIXから取得（デフォルト "!"）
```

---

## ■ Phase 2: サーバー構築（ワンコマンド）

```
設計書に従って、以下の2ファイルを作成してください。
省略せず、全コードを出力してください。
config.py, db.py, utils/embed_builder.py は Phase 1 で作成済みです。

【1. cogs/setup_server.py】
- セクション7-1の仕様に従って !setup コマンドを実装
- 処理順序: サーバー名変更→ロール作成→カテゴリ作成→チャンネル作成→権限設定→Embed配置→FAQ View登録→完了Embed
- セクション4-2の権限テーブルに厳密に従う
- 2回目以降は既存をスキップして不足分だけ追加（冪等性）
- 管理者権限チェック
- セクション6の全Embed仕様（#ようこそ、#できること一覧2枚、#よくある質問）を実装
- FAQ PersistentView（4ボタン、ephemeral応答）もここで定義・登録

【2. cogs/welcome.py】
- セクション7-4の仕様に従って実装
- on_member_join で「体験ユーザー」ロール自動付与
- #bot-log に参加通知Embed
- DM送信（失敗時はスキップ）
```

---

## ■ Phase 3: コア機能（自習サポート）

```
設計書に従って、以下の4ファイルを作成してください。
省略せず、全コードを出力してください。

【1. utils/ai_client.py】
- Gemini 3.1 Flash-Lite を使用（モデル名: gemini-3.1-flash-lite）
- google-genai SDK を使用
- generate_study_response(question, history) 関数を実装
- セクション7-2のシステムプロンプトをそのまま組み込む
- レスポンスからsubject（教科）を判定して返す（JSON形式で返させる）
- max_tokens: 300, temperature: 0.7

【2. utils/rate_limiter.py】
- PostgreSQLのrate_limitsテーブルを使ってユーザーごとの日次制限を管理
- check_and_increment(pool, user_id, guild_id, limit) → bool
- get_remaining(pool, user_id, guild_id, limit) → int

【3. cogs/study.py】
- セクション7-2の自習サポートBot仕様に従って実装
- #自習サポート チャンネルにメッセージが来たらAI応答
- PersistentViewボタン「📝 質問する」でモーダル入力も可
- 直近5往復の会話履歴をコンテキストに含める
- レート制限チェック（30回/日）
- 応答後に study_log cog の記録関数を呼ぶ

【4. cogs/study_log.py】
- セクション7-3の学習記録仕様に従って実装
- log_study(pool, user_id, user_name, question, subject, channel_id, guild_id)
- #学習記録チャンネルにミニEmbed自動投稿
- /study-summary スラッシュコマンドで日次・週次サマリ
```

---

## ■ Phase 4: サブ機能

```
設計書に従って、以下の3ファイルを作成してください。
省略せず、全コードを出力してください。

【1. cogs/faq.py】
- セクション7-5に従ってPersistentView実装
- ※setup_server.pyでFAQ Viewを定義済みの場合はここでは定義せず、
  setup_server.pyからインポートする形でもOK
- custom_id: "faq_view" / ボタン4つ / ephemeral応答

【2. cogs/inquiry.py】
- セクション7-6に従って実装
- #導入相談チャンネルでのメッセージ検知→#bot-logに通知

【3. cogs/dashboard.py】
- セクション7-7に従って実装
- /dashboard スラッシュコマンド（スタッフ以上）
- PostgreSQLから集計→Embed表示
```

---

## ■ Phase 5: デプロイ準備

```
設計書に従って、以下のファイルを作成してください。

【1. .env.example】 — セクション9のローカル開発用テンプレート
【2. .gitignore】 — セクション15のデプロイ設定
【3. requirements.txt】 — セクション16
【4. railway.json】 — セクション15のRailwayデプロイ設定
【5. Procfile】 — worker: python main.py
【6. README.md】 — プロジェクト概要、セットアップ手順、デプロイ手順、コマンド一覧を含む
```

---

## ■ トラブルシューティング

Claude Codeがファイルを作らない場合：
1. 「config.py を作成してください」と1ファイルずつ指示する
2. 設計書の該当セクション番号を明示する
3. 「省略せず全コードを出力してください」を毎回付ける
4. 長すぎる場合は「まず config.py と db.py だけ作って」と2ファイルに絞る
