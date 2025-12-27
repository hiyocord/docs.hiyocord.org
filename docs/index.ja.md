# Hiyocord ドキュメント

Hiyocordは、Cloudflare Workersベースの分散マイクロサービス集合体であり、様々な外部サービスとDiscordを接続するためのプラットフォームです。

## 概要

Hiyocordプロジェクトは、Discord botの開発とデプロイを効率化するための複数のリポジトリで構成されています。各コンポーネントは独立して動作しながら、統合されたエコシステムを形成しています。

## アーキテクチャ

Hiyocordは3つの主要なコンポーネントで構成されています:

### 1. Hiyocord Nexus
Discord interactionの中央ハブとして機能します。Discordからのインタラクション（コマンド、ボタンクリック、モーダル送信など）を受け取り、登録されたマニフェストに基づいて適切なサービスワーカーにルーティングします。

**主な機能:**

- Discord interactionの検証とルーティング
- マニフェストベースのサービス登録
- Discord APIへの自動コマンド登録
- HMAC-SHA256ベースのサービス間認証

[詳細を見る →](nexus/index.ja.md)

### 2. Hiyocord Packages
共有ライブラリとAPIラッパーのコレクションです。Discord REST API、GitHub REST API、interaction処理などの共通機能を提供します。

**含まれるパッケージ:**

- `@hiyocord/discord-rest-api` - Discord API v10クライアント
- `@hiyocord/github-rest-api` - GitHub APIクライアント
- `@hiyocord/discord-interaction-client` - Interaction処理とレスポンスビルダー
- `@hiyocord/rest-api-core` - OpenAPIベースのREST APIクライアント基盤

[詳細を見る →](packages/index.ja.md)

### 3. Hiyocord Service Workers
Discord bot機能を実装するためのテンプレートリポジトリです。Cloudflare Workers上でDiscord interactionを処理し、Nexusと連携して動作します。

**特徴:**

- Honoフレームワークベースの軽量実装
- レジストリパターンによるハンドラー管理
- 自動デプロイパイプライン
- 包括的なオブザーバビリティ

[詳細を見る →](service-workers/index.ja.md)

## クイックスタート

### 前提条件

- Node.js 22以上
- npm
- Cloudflareアカウント
- Discord Bot Token

### 基本的なワークフロー

1. **Service Workerの作成**: `hiyocord-service-workers`をテンプレートとして新しいワーカーを作成
2. **コマンドハンドラーの実装**: Discord interactionに対応するハンドラーを作成
3. **マニフェストの登録**: Nexusにサービスのマニフェストを登録
4. **デプロイ**: Cloudflare Workersにデプロイ

詳細な手順については、[Getting Started Guide](getting-started/index.ja.md)を参照してください。

## 技術スタック

- **ランタイム**: Cloudflare Workers
- **言語**: TypeScript
- **フレームワーク**: Hono
- **ビルドツール**: Vite
- **API**: Discord API v10
- **ストレージ**: Cloudflare KV
- **CI/CD**: GitHub Actions

## プロジェクト構成

```
hiyocord/
├── hiyocord-nexus/          # 中央ルーティングハブ
├── hiyocord-packages/        # 共有ライブラリ
├── hiyocord-service-workers/ # サービスワーカーテンプレート
└── docs.hiyocord.org/        # このドキュメント
```

## リソース

- [GitHub Organization](https://github.com/hiyocord)
- [Discord API Documentation](https://discord.com/developers/docs)
- [Cloudflare Workers Documentation](https://developers.cloudflare.com/workers/)

## ライセンス

すべてのHiyocordプロジェクトはMITライセンスの下で公開されています。
