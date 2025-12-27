# Hiyocord Nexus

Hiyocord Nexusは、Hiyocordエコシステムの中核となるスケーラブルなDiscord integrationハブです。Cloudflare Workers上で動作し、Discord interactionを受け取り、登録されたサービスワーカーにルーティングする役割を担います。

## アーキテクチャ

### 概要

Nexusは、Discord botとバックエンドサービス間の中央ルーターとして機能します。マニフェストベースのルーティングシステムにより、複数のサービスを動的に管理できます。

```
Discord API
    ↓
[Nexus] ← マニフェスト登録
    ↓ (ルーティング)
    ├─→ Service Worker A
    ├─→ Service Worker B
    └─→ Service Worker C
```

### 主要コンポーネント

Nexusは3つのパッケージで構成されています:

#### 1. hiyocord-nexus
メインのCloudflare Workerアプリケーションです。

**エンドポイント:**
- `POST /interactions` - Discord interactionの受付とルーティング
- `POST /manifest` - サービスマニフェストの登録

**主な機能:**
- Discord署名検証
- Interactionルーティング
- マニフェスト管理
- コマンド自動登録

#### 2. hiyocord-nexus-core
再利用可能なコアライブラリです。

**提供機能:**
- リクエスト署名（HMAC-SHA256）
- 署名検証ミドルウェア
- Discord APIクライアントミドルウェア
- 認証タイプ定義

#### 3. hiyocord-nexus-types
型定義とスキーマを提供します。

**含まれる型:**
- Manifestスキーマ（v1.0.0）
- DiscordCommandビルダー
- OpenAPI生成型

#### 4. hiyocord-nexus-web
Nexus管理用のWeb UIを提供する予定のパッケージです（開発中）。

## マニフェストシステム

### マニフェストとは

マニフェストは、サービスワーカーが処理できるDiscord interactionを定義するJSON形式の設定ファイルです。

### マニフェストスキーマ v1.0.0

```typescript
interface Manifest {
  version: string;           // "1.0.0"
  id: string;                // サービスの一意識別子
  name: string;              // サービス名
  base_url: string;          // サービスのベースURL
  application_commands: {    // アプリケーションコマンド定義
    global?: Command[];      // グローバルコマンド
    guild?: {
      [guild_id: string]: Command[];  // ギルド固有コマンド
    };
  };
  message_components?: string[];  // 処理するボタン/メニューのcustom_id
  modal_submits?: string[];       // 処理するモーダルのcustom_id
}
```

### マニフェスト登録の例

```typescript
const manifest = {
  version: "1.0.0",
  id: "my-bot-service",
  name: "My Bot Service",
  base_url: "https://my-worker.workers.dev",
  application_commands: {
    global: [
      {
        name: "ping",
        description: "Responds with pong",
        type: 1
      }
    ]
  }
};

// マニフェストを登録
await fetch("https://nexus.hiyocord.org/manifest", {
  method: "POST",
  headers: { "Content-Type": "application/json" },
  body: JSON.stringify(manifest)
});
```

## ルーティングメカニズム

### Interactionタイプ別ルーティング

Nexusは、interactionのタイプに応じて適切なマニフェストを検索します。

#### 1. Application Commands (スラッシュコマンド)
```typescript
// /test コマンドの場合
1. グローバルコマンドからマニフェストを検索
2. 見つからない場合、ギルド固有コマンドから検索
3. マッチしたサービスにリクエストを転送
```

#### 2. Message Components (ボタン・メニュー)
```typescript
// custom_id: "confirm_action" の場合
1. message_components配列に "confirm_action" を含むマニフェストを検索
2. マッチしたサービスにリクエストを転送
```

#### 3. Modal Submits
```typescript
// custom_id: "feedback_modal" の場合
1. modal_submits配列に "feedback_modal" を含むマニフェストを検索
2. マッチしたサービスにリクエストを転送
```

### リクエスト転送

マニフェストが見つかると、Nexusは以下の処理を行います:

1. **リクエストの署名**: HMAC-SHA256でリクエストに署名
2. **ヘッダーの追加**:
   - `X-Hiyocord-Signature`: 署名
   - `X-Hiyocord-Timestamp`: タイムスタンプ
3. **転送**: サービスワーカーの `/interactions` エンドポイントに転送

## セキュリティ

### Discord署名検証

Nexusは、Discord APIからのリクエストを検証します:

```typescript
import { verifyKey } from "discord-interactions";

// Ed25519署名検証
const isValid = verifyKey(
  rawBody,
  signature,
  timestamp,
  publicKey
);
```

### Hiyocord署名システム

サービスワーカーへのリクエストには独自の署名を付与します:

**署名生成:**
1. ヘッダーの正規化（アルファベット順、CF-*ヘッダーを除外）
2. リクエストボディとヘッダーを結合
3. HMAC-SHA256で署名
4. Hex形式で出力

**署名検証:**
- タイムスタンプチェック（1分以内）
- 署名の暗号学的検証

```typescript
import { sign } from "@hiyocord/hiyocord-nexus-core";

const { headers, signature } = await sign({
  method: "POST",
  path: "/interactions",
  body: jsonBody,
  headers: originalHeaders
}, secret);
```

## 環境変数

Nexusには以下の環境変数（Cloudflare Bindings）が必要です:

| 変数名 | 説明 |
|--------|------|
| `KV` | Cloudflare KVネームスペース（マニフェスト保存用） |
| `DISCORD_APPLICATION_ID` | Discord BotのApplication ID |
| `DISCORD_BOT_TOKEN` | Discord Bot Token |
| `DISCORD_CLIENT_SECRET` | Discord OAuth Client Secret |
| `DISCORD_PUBLIC_KEY` | Discord Public Key（署名検証用） |
| `HIYOCORD_SECRET` | サービス間認証用の共有シークレット |

## デプロイ

### 前提条件

- Cloudflareアカウント
- Discord Botの作成とToken取得
- Wrangler CLIのインストール

### デプロイ手順

1. **環境変数の設定**:
```bash
# wrangler.tomlに追加
[vars]
DISCORD_APPLICATION_ID = "your-app-id"

# シークレットの設定
wrangler secret put DISCORD_BOT_TOKEN
wrangler secret put DISCORD_PUBLIC_KEY
wrangler secret put DISCORD_CLIENT_SECRET
wrangler secret put HIYOCORD_SECRET
```

2. **KVネームスペースの作成**:
```bash
wrangler kv:namespace create "nexus"
# 出力されたIDをwrangler.tomlに追加
```

3. **ビルドとデプロイ**:
```bash
npm run build
npm run deploy
```

### CI/CD

GitHub Actionsによる自動デプロイが設定されています:

- **Buildジョブ**: すべてのパッケージをビルド
- **Deployジョブ**: masterブランチへのpush時に自動デプロイ
- **Publishジョブ**: Changesetsによるバージョン管理とnpm公開

## API仕様

完全なAPI仕様は、リポジトリの[openapi.yaml](https://github.com/hiyocord/hiyocord-nexus/blob/master/openapi.yaml)で確認できます。

### POST /interactions

Discord interactionを処理します。

**リクエスト:**
```json
{
  "type": 2,
  "data": {
    "name": "test",
    "options": []
  },
  "guild_id": "123456789",
  "channel_id": "987654321",
  "member": { ... }
}
```

**レスポンス:**
```json
{
  "type": 4,
  "data": {
    "content": "Response from service worker"
  }
}
```

### POST /manifest

サービスマニフェストを登録します。

**リクエスト:**
```json
{
  "version": "1.0.0",
  "id": "service-id",
  "name": "Service Name",
  "base_url": "https://service.workers.dev",
  "application_commands": {
    "global": [
      {
        "name": "command",
        "description": "Command description"
      }
    ]
  }
}
```

**レスポンス:**
```json
{
  "success": true,
  "registered_commands": 1
}
```

## トラブルシューティング

### マニフェストが登録されない

- KVネームスペースが正しく設定されているか確認
- `base_url`が正しいか確認
- マニフェストスキーマがv1.0.0に準拠しているか確認

### Interactionがルーティングされない

- コマンド名が正確に一致しているか確認
- グローバル/ギルドコマンドの区別を確認
- Cloudflare Workersのログを確認

### 署名検証エラー

- `HIYOCORD_SECRET`がNexusとService Workerで一致しているか確認
- タイムスタンプが1分以内であることを確認
- ヘッダーの正規化処理を確認

## パッケージバージョン

- hiyocord-nexus: v0.0.2
- hiyocord-nexus-core: v0.0.2
- hiyocord-nexus-types: v0.0.2

最新バージョンは[GitHub Releases](https://github.com/hiyocord/hiyocord-nexus/releases)で確認できます。
