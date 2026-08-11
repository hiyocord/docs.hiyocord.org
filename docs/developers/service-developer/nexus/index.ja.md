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

Nexusは5つのパッケージで構成されるモノレポです:

#### 1. hiyocord-nexus
メインのCloudflare Workerアプリケーションです。

**主なエンドポイント:**

- `POST /interactions` - Discord interactionの受付とルーティング
- `GET /.well-known/nexus-public-key` - Nexusの公開鍵を配布
- `ALL /proxy/discord/api/v10/*` - Service Worker向けDiscord APIプロキシ
- `GET/POST/DELETE /api/manifests[...]` - マニフェストのCRUD
- `POST /api/manifests/:id/approve` / `POST /api/manifests/:id/reject` - マニフェストの承認・却下
- `GET /api/auth/discord/authorize` / `POST /api/auth/discord` / `GET /api/auth/me` / `POST /api/auth/logout` - 管理画面用のDiscord OAuth2認証

完全な仕様は[openapi.yaml](https://github.com/hiyocord/hiyocord-nexus/blob/master/openapi.yaml)を参照してください。

**主な機能:**

- Discord署名検証
- Interactionルーティング
- マニフェスト管理・承認ワークフロー
- コマンド自動登録
- Discord APIプロキシ（権限スコープ制御）

#### 2. hiyocord-nexus-core
再利用可能なコアライブラリです。NexusとService Workerの両方から利用されます。

**提供機能:**

- Ed25519によるリクエスト署名・検証（`ecdsa-p256`, `rsa-pss-2048`はアルゴリズムとして定義済みですが未実装で、内部的にはEd25519にフォールバックします）
- Nexus→Service Worker検証ミドルウェア（`nexusVerifyMiddleware`）
- Service Worker→Nexus用のDiscord APIプロキシ・Nexus APIクライアント
- マニフェスト生成（`createManifest`）

#### 3. hiyocord-nexus-types
型定義とスキーマを提供します。

**含まれる型:**

- Manifestスキーマ（v1.0.0）
- Permission型（`DISCORD_BOT` / `DISCORD_API_SCOPE`）
- OpenAPI生成型（Nexus API）

#### 4. hiyocord-nexus-cli {: #nexus-cli }
`gen-key`と`manifest`の2つのCLIコマンドを提供するパッケージです。`hiyocord-service-workers`テンプレートや`hiyocord-nexus`自身のデプロイパイプラインから利用されます。

- `gen-key` - Ed25519鍵ペアを生成
- `manifest` - ビルド済みの`HiyocordExport`を読み込み、Nexusのマニフェストスキーマに変換して`POST /api/manifests`に登録

#### 5. hiyocord-nexus-web {: #nexus-web }
Nexusの管理用Web UI（React + react-router-dom、Cloudflare Pagesとして稼働中）です。Discordアカウントでログインし、登録されたマニフェストの一覧・詳細確認・承認/却下を行えます。ルートは`Home`（ダッシュボード）、`Login`、`Callback`（OAuth2コールバック）、`Manifests`（一覧）、`ManifestDetail`（詳細・承認操作）です。

## マニフェストシステム

### マニフェストとは

マニフェストは、サービスワーカーが処理できるDiscord interactionを定義するJSON形式の設定ファイルです。

### マニフェストスキーマ v1.0.0 {: #manifest-schema }

```ts
interface Manifest {
  version: string;                    // "1.0.0"
  id: string;                         // サービスの一意識別子
  name: string;                       // サービス名
  base_url: string;                   // サービスのベースURL
  description: string;                // サービスの説明
  signature_algorithm: string;        // 署名アルゴリズム（例: "ed25519"）
  public_key: string;                 // Base64エンコードされた公開鍵
  application_commands: {             // アプリケーションコマンド定義
    global: Command[];                // グローバルコマンド
    guild: GuildCommand[];            // ギルド固有コマンド
  };
  message_component_ids: string[];    // 処理するボタン/メニューのcustom_id
  modal_submit_ids: string[];         // 処理するモーダルのcustom_id
  permissions?: Permission[];         // 必要なパーミッション
}
```

**重要なフィールド:**

- `message_component_ids`: ボタンやセレクトメニューなどのメッセージコンポーネントのcustom_idを配列で指定
- `modal_submit_ids`: モーダル送信のcustom_idを配列で指定
- `signature_algorithm`: Service Workerの署名アルゴリズム（実装済みは "ed25519" のみ）
- `public_key`: Service WorkerがNexusへリクエストする際の署名検証用公開鍵
- `permissions`: Service Workerに許可するDiscord APIアクセス範囲（後述の[Discord APIプロキシと権限スコープ](#discord-api-proxy)を参照）

### マニフェストの登録

マニフェストは通常、生のJSONを直接POSTするのではなく、[hiyocord-nexus-cli](#nexus-cli)の`manifest`コマンドで登録します。Service Worker側の`src/manifest.ts`で`HiyocordExport`を定義し、ビルド後に以下を実行します:

```bash
npx manifest \
  --entryPoint=./dist/index.js \
  --nexusUrl=https://nexus.hiyocord.org \
  --baseUrl=https://my-worker.workers.dev \
  --signatureAlgorithm=ed25519 \
  --publicKey=YOUR_SERVICE_PUBLIC_KEY
```

内部的には、`registry`（登録済みハンドラー）と`service`メタデータから[Manifestスキーマ](#manifest-schema)を組み立て、`POST /api/manifests`にリクエストします。詳細は[Getting Started ステップ5](../getting-started/index.ja.md#step5-manifest)を参照してください。

## ルーティングメカニズム

### Interactionタイプ別ルーティング

Nexusは、interactionのタイプに応じて適切なマニフェストを検索します。

#### 1. Application Commands (スラッシュコマンド)
```ts
// /test コマンドの場合
1. グローバルコマンドからマニフェストを検索
2. 見つからない場合、ギルド固有コマンドから検索
3. マッチしたサービスにリクエストを転送
```

#### 2. Message Components (ボタン・セレクトメニュー)

メッセージコンポーネントのインタラクションは、`custom_id`でルーティングされます。

```ts
// ボタンクリック: custom_id = "confirm_action"
// セレクトメニュー選択: custom_id = "role_select"

1. message_component_ids配列に該当のcustom_idを含むマニフェストを検索
2. マッチしたサービスにリクエストを転送
```

**例**: ボタンハンドラーの実装

```typescript
// Service Worker側
import { createManifest } from "@hiyocord/hiyocord-nexus-core";

const manifest = createManifest({
  id: "my-service",
  name: "My Service",
  baseUrl: "https://my-service.workers.dev",
  description: "Service with interactive components",
  signatureAlgorithm: "ed25519",
  publicKey: "YOUR_PUBLIC_KEY",
  commands: [],
  messageComponentIds: ["confirm_button", "cancel_button", "role_select"], // ここに登録
});
```

#### 3. Modal Submits

モーダル送信のインタラクションも、`custom_id`でルーティングされます。

```ts
// モーダル送信: custom_id = "feedback_modal"

1. modal_submit_ids配列に該当のcustom_idを含むマニフェストを検索
2. マッチしたサービスにリクエストを転送
```

**例**: モーダルハンドラーの実装

```typescript
// Service Worker側
const manifest = createManifest({
  id: "my-service",
  name: "My Service",
  baseUrl: "https://my-service.workers.dev",
  description: "Service with modals",
  signatureAlgorithm: "ed25519",
  publicKey: "YOUR_PUBLIC_KEY",
  commands: [],
  modalSubmitIds: ["feedback_modal", "settings_modal"], // ここに登録
});
```

### リクエスト転送

マニフェストが見つかると、Nexusは以下の処理を行います:

1. **リクエストの署名**: Ed25519公開鍵暗号でリクエストに署名
2. **ヘッダーの追加**:
   - `X-Hiyocord-Signature`: Ed25519署名
   - `X-Hiyocord-Timestamp`: タイムスタンプ（リプレイ攻撃防止）
   - `X-Hiyocord-Algorithm`: 署名アルゴリズム（例: "ed25519"）
3. **転送**: サービスワーカーの `/interactions` エンドポイントに転送

詳細は[認証システムドキュメント](./authentication.ja.md)を参照してください。

## セキュリティ

### Discord署名検証

Nexusは、Discord APIからのリクエストを検証します:

```ts
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

Nexus⇄Service Worker間のリクエストには、Ed25519公開鍵暗号による署名を付与します（HMACではありません）。ヘッダーの正規化（アルファベット順ソート、`CF-*`ヘッダーの除外）とタイムスタンプによるリプレイ攻撃防止（60秒以内）を組み合わせています。詳細な仕組み・鍵の設定方法は[認証システム](./authentication.ja.md)を参照してください。

## マニフェスト承認ワークフロー {: #manifest-approval-workflow }

Nexusに登録されたマニフェストは、Discordへのコマンド登録やinteractionの転送が行われる前に、管理者による承認を必要とします。

### ステータス

マニフェストは`pending`（承認待ち）・`approved`（承認済み）・`rejected`（却下）のいずれかのステータスを持ちます。

- 新規登録時、マニフェストは常に`pending`から始まります
- `approved`になったマニフェストのみ、interactionの転送先として選択され、Discordのアプリケーションコマンドとして同期されます
- `pending`/`rejected`のマニフェストへのinteractionはルーティングされません

### 承認・却下

管理者は[hiyocord-nexus-web](#nexus-web)にDiscordアカウントでログインし（`LOGIN_ALLOW_USER`で許可されたユーザーのみ）、`POST /api/manifests/:id/approve`または`POST /api/manifests/:id/reject`でマニフェストを承認・却下します。

### 権限変更時の再承認

すでに`approved`のマニフェストが再登録され、`permissions`フィールドの内容が変更されていた場合、Nexusは自動的にそのマニフェストを`pending`に差し戻します。これにより、Service Worker側のコード変更だけでDiscord APIへのアクセス範囲が無断で拡大することを防いでいます。

## Discord APIプロキシと権限スコープ {: #discord-api-proxy }

Service Workerは、Discord Bot Tokenを直接保持しません。Discord REST APIを呼び出す必要がある場合は、Nexusが提供する`ALL /proxy/discord/api/v10/*`エンドポイントを経由します。

### 呼び出しの流れ

1. Service Workerが`X-Hiyocord-Manifest-Id`ヘッダーと署名ヘッダーを付けてNexusにリクエスト
2. Nexusの`verifyServiceWorker`ミドルウェアが、マニフェストに登録された公開鍵でリクエストを検証
3. マニフェストの`permissions`をもとにアクセス可否を判定
4. 許可されていれば、Nexusが保持するDiscord Bot Tokenを使って実際のDiscord APIを呼び出し、結果を返す

### 権限（Permission）の種類

マニフェストの`permissions`フィールドには、次のいずれかを指定します:

```ts
type Permission =
  | { type: "DISCORD_BOT" }                              // Discord APIへのフルアクセス
  | { type: "DISCORD_API_SCOPE"; scopes: Record<string, string[]> }; // パス＋メソッド単位の許可リスト
```

`DISCORD_API_SCOPE`の例:

```json
{
  "type": "DISCORD_API_SCOPE",
  "scopes": {
    "/channels/:channel_id/messages": ["POST"],
    "/channels/:channel_id/messages/:message_id": ["GET", "PATCH", "DELETE"],
    "/guilds/:guild_id/members/:user_id": ["GET"]
  }
}
```

必要最小限のスコープのみを要求することを推奨します。`DISCORD_BOT`はフルアクセスとなるため、権限変更時の再承認（[マニフェスト承認ワークフロー](#manifest-approval-workflow)）の対象にもなりやすい点に注意してください。

## 環境変数

Nexusには以下の環境変数（Cloudflare Bindings）が必要です:

| 変数名 | 説明 |
|--------|------|
| `KV` | Cloudflare KVネームスペース（マニフェスト・承認状態の保存用） |
| `DISCORD_APPLICATION_ID` | Discord BotのApplication ID |
| `DISCORD_BOT_TOKEN` | Discord Bot Token |
| `DISCORD_CLIENT_SECRET` | Discord OAuth Client Secret（管理画面ログイン・Discord APIプロキシ用） |
| `DISCORD_PUBLIC_KEY` | Discord Public Key（Discordからのinteraction署名検証用） |
| `NEXUS_PRIVATE_KEY` | Service Workerへの署名付きリクエスト送信用のEd25519秘密鍵 |
| `NEXUS_PUBLIC_KEY` | Service Worker配布用のEd25519公開鍵 |
| `NEXUS_SIGNATURE_ALGORITHM` | 署名アルゴリズム（省略可、デフォルト`ed25519`） |
| `JWT_SECRET` | 管理画面セッション用JWTの署名シークレット |
| `LOGIN_ALLOW_USER` | 管理画面へのログインを許可するDiscordユーザーIDのカンマ区切りリスト |

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
wrangler secret put NEXUS_PRIVATE_KEY
wrangler secret put NEXUS_PUBLIC_KEY
wrangler secret put JWT_SECRET
wrangler secret put LOGIN_ALLOW_USER
```

`NEXUS_PRIVATE_KEY`/`NEXUS_PUBLIC_KEY`は`npx tsx generate-keypair.ts`で生成できます。詳細は[認証システム](./authentication.ja.md)を参照してください。

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

### POST /api/manifests

サービスマニフェストを登録します（通常は[hiyocord-nexus-cli](#nexus-cli)の`manifest`コマンド経由で呼び出されます）。登録直後のマニフェストは`pending`ステータスになり、[承認](#manifest-approval-workflow)されるまでDiscordコマンドの同期やinteractionの転送は行われません。

**リクエスト:**

```json
{
  "version": "1.0.0",
  "id": "service-id",
  "name": "Service Name",
  "base_url": "https://service.workers.dev",
  "description": "Service description",
  "signature_algorithm": "ed25519",
  "public_key": "BASE64_PUBLIC_KEY",
  "application_commands": {
    "global": [
      {
        "name": "command",
        "description": "Command description"
      }
    ]
  },
  "message_component_ids": [],
  "modal_submit_ids": [],
  "permissions": [{ "type": "DISCORD_BOT" }]
}
```

### POST /api/manifests/:id/approve・/reject

管理画面（hiyocord-nexus-web）から呼び出される、マニフェストの承認・却下エンドポイントです。承認されると、マニフェストのコマンドが実際にDiscord APIへ同期されます。

### GET /.well-known/nexus-public-key

Service WorkerがNexusの公開鍵を取得するためのエンドポイントです。

```json
{
  "algorithm": "ed25519",
  "public_key": "BASE64_PUBLIC_KEY"
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

### マニフェストを登録したのにコマンドが反映されない

- マニフェストが`pending`のまま承認されていない可能性があります。[承認ワークフロー](#manifest-approval-workflow)を確認し、hiyocord-nexus-webで承認してください

### 署名検証エラー

- Nexus→Service Workerの場合は`NEXUS_PRIVATE_KEY`（Nexus側）と`NEXUS_PUBLIC_KEY`（Service Worker側）が一致しているか確認
- Service Worker→Nexusの場合はマニフェストに登録した公開鍵と`HIYOCORD_PRIVATE_KEY`（Service Worker側）が一致しているか確認
- タイムスタンプが60秒以内であることを確認
- ヘッダーの正規化処理を確認
- 詳細は[認証システム](./authentication.ja.md)のトラブルシューティングを参照

## パッケージバージョン

- hiyocord-nexus-core: v0.8.4
- hiyocord-nexus-types: v0.5.3
- hiyocord-nexus-cli: v0.2.3

`hiyocord-nexus`本体（Workerアプリケーション）とWeb UIはprivateパッケージのため個別のバージョン番号はありません。最新バージョンは[GitHub Releases](https://github.com/hiyocord/hiyocord-nexus/releases)で確認できます。
