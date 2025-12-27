# Getting Started

このガイドでは、Hiyocordを使用してDiscord botを作成し、デプロイするまでの手順を説明します。

## 前提条件

開始する前に、以下のものを準備してください:

- **Node.js 22以上** - [nodejs.org](https://nodejs.org/)からインストール
- **npm** - Node.jsに含まれています
- **Cloudflareアカウント** - [cloudflare.com](https://cloudflare.com/)で無料登録
- **Discord Botアカウント** - [Discord Developer Portal](https://discord.com/developers/applications)で作成
- **Git** - バージョン管理用

## ステップ1: Discord Botの作成

### 1.1 アプリケーションの作成

1. [Discord Developer Portal](https://discord.com/developers/applications)にアクセス
2. 「New Application」をクリック
3. Bot名を入力して「Create」

### 1.2 Botユーザーの作成

1. 左メニューから「Bot」を選択
2. 「Add Bot」をクリック
3. 「TOKEN」セクションの「Reset Token」をクリックしてトークンをコピー（後で使用）

### 1.3 必要な情報の取得

以下の情報をメモしてください:

- **Application ID**: General Information ページの「APPLICATION ID」
- **Public Key**: General Information ページの「PUBLIC KEY」
- **Bot Token**: Bot ページで取得したトークン

### 1.4 Bot権限の設定

1. 左メニューから「Bot」を選択
2. 「Privileged Gateway Intents」で必要なIntentsを有効化
3. 「Bot Permissions」で必要な権限を選択

### 1.5 Botをサーバーに追加

1. 左メニューから「OAuth2」→「URL Generator」を選択
2. 「SCOPES」で以下を選択:
   - `bot`
   - `applications.commands`
3. 「BOT PERMISSIONS」で必要な権限を選択
4. 生成されたURLをブラウザで開いてBotをサーバーに追加

## ステップ2: Cloudflareのセットアップ

### 2.1 Wrangler CLIのインストール

```bash
npm install -g wrangler
```

### 2.2 Cloudflareへのログイン

```bash
wrangler login
```

ブラウザが開き、Cloudflareアカウントへのログインを求められます。

## ステップ3: Hiyocord Nexusのセットアップ

### 3.1 Nexusリポジトリのクローン

```bash
git clone https://github.com/hiyocord/hiyocord-nexus.git
cd hiyocord-nexus
```

### 3.2 依存関係のインストール

```bash
npm install
```

### 3.3 KVネームスペースの作成

```bash
wrangler kv:namespace create "nexus"
```

出力されたIDをメモしてください。

### 3.4 環境変数の設定

```bash
# Wranglerシークレットの設定
wrangler secret put DISCORD_BOT_TOKEN
# Bot Tokenを入力

wrangler secret put DISCORD_PUBLIC_KEY
# Public Keyを入力

wrangler secret put DISCORD_CLIENT_SECRET
# Client Secretを入力（OAuth2ページから取得）

wrangler secret put HIYOCORD_SECRET
# 任意のランダムな文字列を生成して入力（例: openssl rand -hex 32）
```

`wrangler.toml`を編集して、Application IDとKVネームスペースIDを設定:

```toml
name = "hiyocord-nexus"
main = "packages/hiyocord-nexus/src/index.ts"

[vars]
DISCORD_APPLICATION_ID = "YOUR_APPLICATION_ID"

[[kv_namespaces]]
binding = "KV"
id = "YOUR_KV_NAMESPACE_ID"
```

### 3.5 Nexusのデプロイ

```bash
npm run build
npm run deploy
```

デプロイが完了すると、Workerのエンドポイント（例: `https://hiyocord-nexus.your-subdomain.workers.dev`）が表示されます。

### 3.6 Discord Developer Portalの設定

1. [Discord Developer Portal](https://discord.com/developers/applications)に戻る
2. 作成したアプリケーションを選択
3. 左メニューから「General Information」を選択
4. 「INTERACTIONS ENDPOINT URL」に以下を入力:
   ```
   https://hiyocord-nexus.your-subdomain.workers.dev/interactions
   ```
5. 「Save Changes」をクリック

Discordが検証を行い、成功すると設定が保存されます。

## ステップ4: Service Workerの作成

### 4.1 テンプレートから新しいリポジトリを作成

1. [hiyocord-service-workers](https://github.com/hiyocord/hiyocord-service-workers)にアクセス
2. 「Use this template」をクリック
3. リポジトリ名を入力（例: `my-discord-bot`）
4. 「Create repository」をクリック

### 4.2 リポジトリのクローン

```bash
git clone https://github.com/your-username/my-discord-bot.git
cd my-discord-bot
```

### 4.3 依存関係のインストール

```bash
npm install
```

### 4.4 環境変数の設定

```bash
wrangler secret put HIYOCORD_SECRET
# Nexusで設定したものと同じシークレットを入力
```

### 4.5 Worker名の変更

`wrangler.config.ts`を編集:

```ts
import type { WranglerConfigurerOptions } from "@hiyocord/wrangler-configurer";

export default {
  params: {
    name: "my-discord-bot",  // 変更
    main: "src/index.ts",
    compatibility_date: "2025-10-08",
    compatibility_flags: ["nodejs_compat"],
    observability: {
      enabled: true,
      head_sampling_rate: 1,
      logs: {
        enabled: true,
        invocation_logs: true
      }
    }
  }
} satisfies WranglerConfigurerOptions;
```

`wrangler.config.ts`は[@hiyocord/wrangler-configurer](../packages/index.ja.md#hiyocordwrangler-configurer)によって読み込まれ、`wrangler.jsonc`に変換されます。このツールを使用することで、環境固有の設定（KVネームスペースIDなど）をバージョン管理から分離できます。

設定を反映するには:

```bash
npx wrangler-configurer
```

### 4.6 最初のコマンドの作成

`src/handlers/ping.ts`を作成:

```ts
import {
  ApplicationCommandHandler,
  createBuilder,
  MessageFlags
} from "@hiyocord/discord-interaction-client";

export default {
  name: "ping",
  description: "Responds with pong",
  handle: async (interaction) => {
    return createBuilder(interaction)
      .reply()
      .content("Pong! 🏓")
      .flags(MessageFlags.Ephemeral)
      .build();
  }
} satisfies ApplicationCommandHandler;
```

### 4.7 ハンドラーの登録

`src/register.ts`を編集:

```ts
import {
  InteractionType,
  SimpleInteractionHandlerRegistry,
  SimpleInteractionHandlerResolver
} from "@hiyocord/discord-interaction-client";
import pingHandler from "./handlers/ping";

export const registry = new SimpleInteractionHandlerRegistry();

registry.register(InteractionType.ApplicationCommand, pingHandler);

export const resolver = new SimpleInteractionHandlerResolver(registry);
```

### 4.8 Service Workerのデプロイ

```bash
npm run build
npm run deploy
```

デプロイが完了すると、Workerのエンドポイントが表示されます（例: `https://my-discord-bot.your-subdomain.workers.dev`）。

## ステップ5: マニフェストの登録

### 5.1 登録スクリプトの作成

`scripts/register-manifest.ts`を作成:

```ts
const NEXUS_URL = "https://hiyocord-nexus.your-subdomain.workers.dev";
const WORKER_URL = "https://my-discord-bot.your-subdomain.workers.dev";

const manifest = {
  version: "1.0.0",
  id: "my-discord-bot",
  name: "My Discord Bot",
  base_url: WORKER_URL,
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

const response = await fetch(`${NEXUS_URL}/manifest`, {
  method: "POST",
  headers: {
    "Content-Type": "application/json"
  },
  body: JSON.stringify(manifest)
});

if (!response.ok) {
  console.error("Failed to register manifest:", await response.text());
  process.exit(1);
}

const result = await response.json();
console.log("Manifest registered successfully:", result);
```

### 5.2 マニフェストの登録

```bash
npx tsx scripts/register-manifest.ts
```

成功すると、Nexusがコマンドを自動的にDiscord APIに登録します。

## ステップ6: テスト

### 6.1 コマンドの確認

Discordサーバーで `/` を入力すると、登録したコマンドが表示されます。

### 6.2 コマンドの実行

`/ping` を実行すると、"Pong! 🏓" というエフェメラルメッセージが返ってきます。

## 次のステップ

おめでとうございます！Hiyocordを使った最初のDiscord botが動作しています。

### さらに学ぶ

- [Service Workers ガイド](../service-workers/index.ja.md) - より高度なコマンドとインタラクションの実装
- [Packages リファレンス](../packages/index.ja.md) - 利用可能なライブラリの詳細
- [Nexus アーキテクチャ](../nexus/index.ja.md) - Nexusの内部動作の理解

### 実装例

#### ユーザー情報の取得

```ts
// src/handlers/userinfo.ts
import {
  ApplicationCommandHandler,
  createBuilder
} from "@hiyocord/discord-interaction-client";

export default {
  name: "userinfo",
  description: "Get user information",
  options: [
    {
      name: "user",
      description: "User to get info about",
      type: 6,
      required: false
    }
  ],
  handle: async (interaction) => {
    const targetUser = interaction.data.options?.[0]?.value
      ? interaction.data.resolved?.users?.[interaction.data.options[0].value as string]
      : interaction.member?.user;

    if (!targetUser) {
      return createBuilder(interaction)
        .reply()
        .content("Could not find user")
        .build();
    }

    return createBuilder(interaction)
      .reply()
      .embeds([
        {
          title: "User Information",
          thumbnail: {
            url: `https://cdn.discordapp.com/avatars/${targetUser.id}/${targetUser.avatar}.png`
          },
          fields: [
            {
              name: "Username",
              value: targetUser.username,
              inline: true
            },
            {
              name: "ID",
              value: targetUser.id,
              inline: true
            },
            {
              name: "Bot",
              value: targetUser.bot ? "Yes" : "No",
              inline: true
            }
          ],
          color: 0x5865F2
        }
      ])
      .build();
  }
} satisfies ApplicationCommandHandler;
```

#### ボタン付きメッセージ

```ts
// src/handlers/vote.ts
import {
  ApplicationCommandHandler,
  createBuilder
} from "@hiyocord/discord-interaction-client";

export default {
  name: "vote",
  description: "Create a simple vote",
  options: [
    {
      name: "question",
      description: "Question to vote on",
      type: 3,
      required: true
    }
  ],
  handle: async (interaction) => {
    const question = interaction.data.options?.[0]?.value as string;

    return createBuilder(interaction)
      .reply()
      .content(`📊 **Vote:** ${question}`)
      .components([
        {
          type: 1,
          components: [
            {
              type: 2,
              style: 3,
              label: "👍 Yes",
              custom_id: "vote_yes"
            },
            {
              type: 2,
              style: 4,
              label: "👎 No",
              custom_id: "vote_no"
            }
          ]
        }
      ])
      .build();
  }
} satisfies ApplicationCommandHandler;
```

### トラブルシューティング

#### コマンドが表示されない

- マニフェストが正しく登録されているかNexusのログを確認
- Discord側でコマンドの同期に数分かかることがあります
- Botがサーバーに参加しているか確認

#### "Interaction failed"エラー

- Service WorkerがNexusからのリクエストに応答していない可能性があります
- `HIYOCORD_SECRET`が一致しているか確認
- Cloudflare Workersのログを確認: `wrangler tail`

#### 署名検証エラー

- NexusとService Workerの`HIYOCORD_SECRET`が一致しているか確認
- Discord Developer Portalの「PUBLIC KEY」が正しく設定されているか確認

### ヘルプとサポート

問題が解決しない場合:

- [GitHub Discussions](https://github.com/hiyocord/discussions) - コミュニティに質問
- [GitHub Issues](https://github.com/hiyocord/) - バグ報告や機能リクエスト
- リポジトリのREADMEとソースコードを確認

Happy botting! 🎉
