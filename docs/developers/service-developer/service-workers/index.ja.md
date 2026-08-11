# Hiyocord Service Workers

Hiyocord Service Workersは、Discord bot機能を実装するためのテンプレートリポジトリです。Cloudflare Workers上でDiscord interactionを処理し、Hiyocord Nexusと連携して動作します。

## 概要

Service Workersは、実際のDiscord botロジックを実装する場所です。Nexusから転送されたinteractionを受け取り、処理し、レスポンスを返します。

### アーキテクチャ

```
Discord → Nexus → Service Worker
                      ↓
                  [Handler Registry]
                      ↓
                  Command Handlers
```

## 主要な特徴

### 1. テンプレートベース

このリポジトリは**GitHubテンプレートリポジトリ**として設計されています。新しいbotサービスを作成する際は、このリポジトリをテンプレートとして使用します。

### 2. Honoフレームワーク

軽量で高速なWebフレームワーク[Hono](https://hono.dev/)を使用しています。

```ts
import { Hono } from "hono";
import { nexusVerifyMiddleware } from "@hiyocord/hiyocord-nexus-core";
import { fetchHandler } from "@hiyocord/discord-interaction-client";
import { resolver } from "./register";

const app = new Hono();

app.use("/interactions", nexusVerifyMiddleware);
app.mount("/interactions", fetchHandler(resolver).fetch);
```

### 3. レジストリパターン

コマンドハンドラーをレジストリに登録することで、拡張性の高い設計を実現しています。

### 4. 自動デプロイ

GitHub Actionsによる自動ビルドとデプロイが設定されています。

### 5. オブザーバビリティ

Cloudflare Workersの観測機能が有効化されており、ログとトレースを取得できます。

## プロジェクト構造

```
hiyocord-service-workers/
├── packages/
│   └── workers/
│       ├── src/
│       │   ├── index.ts          # アプリケーションエントリポイント
│       │   ├── register.ts       # ハンドラーレジストリ
│       │   ├── manifest.ts       # マニフェスト定義（HiyocordExport）
│       │   ├── test.ts           # サンプルコマンド（グローバル）
│       │   └── test-guild.ts     # サンプルコマンド（ギルド限定）
│       ├── vite.config.ts        # Viteビルド設定
│       ├── wrangler.config.ts    # Wrangler設定
│       └── package.json
├── .github/workflows/
│   └── ci.yaml                   # CI/CDパイプライン（build → deploy → 鍵設定 → manifest登録）
└── package.json                  # ワークスペース設定
```

## クイックスタート

### 1. テンプレートからリポジトリを作成

GitHubで「Use this template」ボタンをクリックして、新しいリポジトリを作成します。

### 2. ローカルにクローン

```bash
git clone https://github.com/your-username/your-bot-service.git
cd your-bot-service
```

### 3. 依存関係のインストール

```bash
npm install
```

### 4. 環境変数の設定

NexusとService Worker間の認証にはEd25519鍵ペアを使用します。

```bash
# Nexusの公開鍵を取得して設定
curl https://your-nexus.workers.dev/.well-known/nexus-public-key \
  | jq -jr ".public_key" | wrangler secret put NEXUS_PUBLIC_KEY

# 自身の鍵ペアを生成
npx gen-key --format=json > keys.json
jq -jr ".public_key" keys.json | wrangler secret put HIYOCORD_PUBLIC_KEY
jq -jr ".private_key" keys.json | wrangler secret put HIYOCORD_PRIVATE_KEY
jq -jr ".algorithm" keys.json | wrangler secret put HIYOCORD_KEY_ALGORITHM
```

`gen-key`は`@hiyocord/hiyocord-nexus-cli`が提供するコマンドで、テンプレートに依存として含まれています。鍵の設定方法の詳細は[Nexus 認証システム](../nexus/authentication.ja.md)を参照してください。

### 5. ビルドとデプロイ

```bash
npm run build
npm run deploy
```

## コマンドハンドラーの実装

### 基本的なコマンド

`src/handlers/`ディレクトリに新しいハンドラーを作成します:

```ts
// src/handlers/hello.ts
import {
  ApplicationCommandHandler,
  createBuilder,
  MessageFlags
} from "@hiyocord/discord-interaction-client";

export default {
  name: "hello",
  description: "Says hello",
  handle: async (interaction) => {
    return createBuilder(interaction)
      .reply()
      .content("Hello from Hiyocord! 👋")
      .flags(MessageFlags.Ephemeral)
      .build();
  }
} satisfies ApplicationCommandHandler;
```

### レジストリに登録

`src/register.ts`でハンドラーを登録します:

```ts
import {
  InteractionType,
  SimpleInteractionHandlerRegistry,
  SimpleInteractionHandlerResolver
} from "@hiyocord/discord-interaction-client";
import helloHandler from "./handlers/hello";

export const registry = new SimpleInteractionHandlerRegistry();

registry.register(InteractionType.ApplicationCommand, helloHandler);

export const resolver = new SimpleInteractionHandlerResolver(registry);
```

### オプション付きコマンド

コマンドオプションを定義してユーザー入力を受け取ります:

```ts
// src/handlers/greet.ts
import {
  ApplicationCommandHandler,
  createBuilder
} from "@hiyocord/discord-interaction-client";

export default {
  name: "greet",
  description: "Greets a user",
  options: [
    {
      name: "user",
      description: "User to greet",
      type: 6,  // USER type
      required: true
    },
    {
      name: "message",
      description: "Custom message",
      type: 3,  // STRING type
      required: false
    }
  ],
  handle: async (interaction) => {
    const userId = interaction.data.options?.[0]?.value as string;
    const customMessage = interaction.data.options?.[1]?.value as string;

    const message = customMessage
      ? `${customMessage}, <@${userId}>!`
      : `Hello, <@${userId}>!`;

    return createBuilder(interaction)
      .reply()
      .content(message)
      .build();
  }
} satisfies ApplicationCommandHandler;
```

### サブコマンド

サブコマンドを使用して、関連するコマンドをグループ化します:

```ts
// src/handlers/admin.ts
import {
  ApplicationCommandHandler,
  createBuilder,
  MessageFlags
} from "@hiyocord/discord-interaction-client";

export default {
  name: "admin",
  description: "Admin commands",
  options: [
    {
      name: "ban",
      description: "Ban a user",
      type: 1,  // SUB_COMMAND
      options: [
        {
          name: "user",
          description: "User to ban",
          type: 6,
          required: true
        }
      ]
    },
    {
      name: "kick",
      description: "Kick a user",
      type: 1,  // SUB_COMMAND
      options: [
        {
          name: "user",
          description: "User to kick",
          type: 6,
          required: true
        }
      ]
    }
  ],
  handle: async (interaction) => {
    const subcommand = interaction.data.options?.[0]?.name;
    const userId = interaction.data.options?.[0]?.options?.[0]?.value;

    let content = "";

    switch (subcommand) {
      case "ban":
        // Ban処理
        content = `Banned user <@${userId}>`;
        break;
      case "kick":
        // Kick処理
        content = `Kicked user <@${userId}>`;
        break;
    }

    return createBuilder(interaction)
      .reply()
      .content(content)
      .flags(MessageFlags.Ephemeral)
      .build();
  }
} satisfies ApplicationCommandHandler;
```

## インタラクティブコンポーネント

### ボタン

ボタンを含むメッセージを送信します:

```ts
// src/handlers/confirm.ts
import {
  ApplicationCommandHandler,
  createBuilder
} from "@hiyocord/discord-interaction-client";

export default {
  name: "confirm",
  description: "Shows a confirmation dialog",
  handle: async (interaction) => {
    return createBuilder(interaction)
      .reply()
      .content("Are you sure?")
      .components([
        {
          type: 1,  // ACTION_ROW
          components: [
            {
              type: 2,  // BUTTON
              style: 3,  // SUCCESS (green)
              label: "Confirm",
              custom_id: "confirm_yes"
            },
            {
              type: 2,
              style: 4,  // DANGER (red)
              label: "Cancel",
              custom_id: "confirm_no"
            }
          ]
        }
      ])
      .build();
  }
} satisfies ApplicationCommandHandler;
```

### ボタンハンドラー

ボタンクリックを処理します:

```ts
// src/handlers/confirm-buttons.ts
import {
  MessageComponentHandler,
  createBuilder,
  InteractionType
} from "@hiyocord/discord-interaction-client";

const yesHandler: MessageComponentHandler = {
  customId: "confirm_yes",
  handle: async (interaction) => {
    return createBuilder(interaction)
      .updateMessage()
      .content("✅ Confirmed!")
      .components([])  // ボタンを削除
      .build();
  }
};

const noHandler: MessageComponentHandler = {
  customId: "confirm_no",
  handle: async (interaction) => {
    return createBuilder(interaction)
      .updateMessage()
      .content("❌ Cancelled")
      .components([])
      .build();
  }
};

// レジストリに登録
registry.register(InteractionType.MessageComponent, yesHandler);
registry.register(InteractionType.MessageComponent, noHandler);
```

### セレクトメニュー

セレクトメニューを使用して選択肢を提供します:

```ts
// src/handlers/choose.ts
import {
  ApplicationCommandHandler,
  createBuilder
} from "@hiyocord/discord-interaction-client";

export default {
  name: "choose",
  description: "Choose an option",
  handle: async (interaction) => {
    return createBuilder(interaction)
      .reply()
      .content("Select your favorite:")
      .components([
        {
          type: 1,  // ACTION_ROW
          components: [
            {
              type: 3,  // STRING_SELECT
              custom_id: "favorite_select",
              placeholder: "Choose one...",
              options: [
                {
                  label: "Option 1",
                  value: "opt1",
                  description: "First option",
                  emoji: { name: "1️⃣" }
                },
                {
                  label: "Option 2",
                  value: "opt2",
                  description: "Second option",
                  emoji: { name: "2️⃣" }
                },
                {
                  label: "Option 3",
                  value: "opt3",
                  description: "Third option",
                  emoji: { name: "3️⃣" }
                }
              ]
            }
          ]
        }
      ])
      .build();
  }
} satisfies ApplicationCommandHandler;
```

### セレクトメニューハンドラー

```ts
// src/handlers/choose-handler.ts
import {
  MessageComponentHandler,
  createBuilder,
  InteractionType
} from "@hiyocord/discord-interaction-client";

const selectHandler: MessageComponentHandler = {
  customId: "favorite_select",
  handle: async (interaction) => {
    const selected = interaction.data.values?.[0];

    return createBuilder(interaction)
      .updateMessage()
      .content(`You selected: ${selected}`)
      .components([])
      .build();
  }
};

registry.register(InteractionType.MessageComponent, selectHandler);
```

### モーダル

モーダルフォームを表示します:

```ts
// src/handlers/feedback.ts
import {
  ApplicationCommandHandler,
  createBuilder
} from "@hiyocord/discord-interaction-client";

export default {
  name: "feedback",
  description: "Submit feedback",
  handle: async (interaction) => {
    return createBuilder(interaction)
      .modal()
      .customId("feedback_modal")
      .title("Feedback Form")
      .components([
        {
          type: 1,  // ACTION_ROW
          components: [
            {
              type: 4,  // TEXT_INPUT
              custom_id: "feedback_title",
              label: "Title",
              style: 1,  // SHORT
              required: true,
              max_length: 100
            }
          ]
        },
        {
          type: 1,
          components: [
            {
              type: 4,
              custom_id: "feedback_body",
              label: "Details",
              style: 2,  // PARAGRAPH
              required: true,
              max_length: 1000,
              placeholder: "Please provide details..."
            }
          ]
        }
      ])
      .build();
  }
} satisfies ApplicationCommandHandler;
```

### モーダルハンドラー

モーダル送信を処理します:

```ts
// src/handlers/feedback-handler.ts
import {
  ModalSubmitHandler,
  createBuilder,
  InteractionType,
  MessageFlags
} from "@hiyocord/discord-interaction-client";

const modalHandler: ModalSubmitHandler = {
  customId: "feedback_modal",
  handle: async (interaction) => {
    const title = interaction.data.components?.[0]?.components?.[0]?.value;
    const body = interaction.data.components?.[1]?.components?.[0]?.value;

    // フィードバックを保存（例: KVストア、外部API等）
    await saveFeedback({ title, body, userId: interaction.member?.user?.id });

    return createBuilder(interaction)
      .reply()
      .content("Thank you for your feedback!")
      .flags(MessageFlags.Ephemeral)
      .build();
  }
};

registry.register(InteractionType.ModalSubmit, modalHandler);
```

## 外部APIとの連携

### Cloudflare KVの使用

データを永続化するためにCloudflare KVを使用します:

```ts
// wrangler.config.ts に追加
export default {
  // ...
  kv_namespaces: [
    { binding: "MY_KV", id: "your-kv-id" }
  ]
};
```

```ts
// src/index.ts で型を定義
type Bindings = {
  NEXUS_PUBLIC_KEY: string;
  HIYOCORD_PRIVATE_KEY: string;
  HIYOCORD_KEY_ALGORITHM: string;
  MY_KV: KVNamespace;
};

const app = new Hono<{Bindings: Bindings}>();
```

```ts
// src/handlers/save.ts
import {
  ApplicationCommandHandler,
  createBuilder,
  MessageFlags
} from "@hiyocord/discord-interaction-client";

export default {
  name: "save",
  description: "Save data",
  options: [
    {
      name: "key",
      description: "Key",
      type: 3,
      required: true
    },
    {
      name: "value",
      description: "Value",
      type: 3,
      required: true
    }
  ],
  handle: async (interaction, env) => {
    const key = interaction.data.options?.[0]?.value as string;
    const value = interaction.data.options?.[1]?.value as string;

    // KVに保存
    await env.MY_KV.put(key, value);

    return createBuilder(interaction)
      .reply()
      .content(`Saved: ${key} = ${value}`)
      .flags(MessageFlags.Ephemeral)
      .build();
  }
} satisfies ApplicationCommandHandler;
```

### 外部APIの呼び出し

```ts
// src/handlers/weather.ts
import {
  ApplicationCommandHandler,
  createBuilder
} from "@hiyocord/discord-interaction-client";

export default {
  name: "weather",
  description: "Get weather information",
  options: [
    {
      name: "city",
      description: "City name",
      type: 3,
      required: true
    }
  ],
  handle: async (interaction) => {
    const city = interaction.data.options?.[0]?.value as string;

    // 外部API呼び出し
    const response = await fetch(
      `https://api.weather.com/v1/weather?city=${city}`
    );
    const weather = await response.json();

    return createBuilder(interaction)
      .reply()
      .embeds([
        {
          title: `Weather in ${city}`,
          description: weather.description,
          fields: [
            {
              name: "Temperature",
              value: `${weather.temp}°C`,
              inline: true
            },
            {
              name: "Humidity",
              value: `${weather.humidity}%`,
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

## Nexusへのマニフェスト登録

Service Workerをデプロイした後、Nexusにマニフェストを登録する必要があります。マニフェストは`src/manifest.ts`で`HiyocordExport`として定義し、[hiyocord-nexus-cli](../nexus/index.ja.md#nexus-cli)の`manifest`コマンドで登録します。

### マニフェスト定義

```ts
// src/manifest.ts
import type { HiyocordExport } from "@hiyocord/hiyocord-nexus-cli";
import { registry } from "./register";

export default {
  registry,
  service: {
    baseUrl: "https://my-bot.workers.dev",
    id: "my-bot-service",
    name: "My Bot Service",
    description: "My Discord bot service",
    permissions: [{ type: "DISCORD_BOT" }],
    messageComponentIds: ["confirm_yes", "confirm_no", "favorite_select"],
    modalSubmitIds: ["feedback_modal"]
  },
  signing: {
    algorithm: "ed25519",
    publicKey: "YOUR_HIYOCORD_PUBLIC_KEY"
  }
} satisfies HiyocordExport;
```

`registry`から自動的にアプリケーションコマンドが抽出されます。`messageComponentIds`/`modalSubmitIds`はハンドラー登録済みのボタン・セレクトメニュー・モーダルの`custom_id`を列挙してください（Nexus側のマニフェストAPIでは`message_component_ids`/`modal_submit_ids`というsnake_caseのフィールド名に変換されます。詳細は[Nexusのマニフェストスキーマ](../nexus/index.ja.md#manifest-schema)を参照）。

### 登録の実行

```bash
npm run build

npx manifest \
  --entryPoint=./dist/index.js \
  --nexusUrl=https://nexus.hiyocord.org \
  --baseUrl=https://my-bot.workers.dev \
  --signatureAlgorithm=ed25519 \
  --publicKey=YOUR_HIYOCORD_PUBLIC_KEY
```

登録直後のマニフェストは`pending`ステータスです。Nexusの管理画面（hiyocord-nexus-web）で承認するまで、コマンドはDiscordに反映されません。詳細は[マニフェスト承認ワークフロー](../nexus/index.ja.md#manifest-approval-workflow)を参照してください。

## デプロイメント

### ローカルでのテスト

```bash
# 開発サーバー起動
npm run dev

# またはWrangler dev
wrangler dev
```

### プロダクションデプロイ

```bash
# ビルド
npm run build

# デプロイ
npm run deploy
```

### 自動デプロイ

masterブランチにpushすると、GitHub Actionsが自動的にデプロイを実行します:

```yaml
# .github/workflows/ci.yaml
- name: Deploy
  if: github.ref == 'refs/heads/master'
  run: npm run deploy
  env:
    CLOUDFLARE_API_TOKEN: ${{ secrets.CLOUDFLARE_API_TOKEN }}
```

## 設定

### Wrangler設定

`wrangler.config.ts`でWorkerの設定を行います:

```ts
import { defineConfig } from "@hiyocord/wrangler-configurer";

export default defineConfig({
  name: "my-bot-service",
  main: "src/index.ts",
  compatibility_date: "2025-10-08",
  compatibility_flags: ["nodejs_compat"],
  observability: {
    enabled: true,
    head_sampling_rate: 1,  // すべてのリクエストをサンプリング
    logs: {
      enabled: true,
      invocation_logs: true
    }
  }
});
```

### Vite設定

`vite.config.ts`でビルド設定をカスタマイズします:

```ts
import { defineConfig } from "vite";
import { cloudflareWorkersVitePlugin } from "@cloudflare/vite-plugin-cloudflare-workers";

export default defineConfig({
  plugins: [cloudflareWorkersVitePlugin()],
  build: {
    lib: {
      entry: "src/index.ts",
      formats: ["es"]
    },
    sourcemap: true
  }
});
```

## ベストプラクティス

### 1. ハンドラーの分離

各コマンドを個別のファイルに分離して、メンテナンス性を向上させます。

### 2. 型安全性

TypeScriptの型を活用して、コンパイル時にエラーを検出します。

### 3. エラーハンドリング

すべてのハンドラーで適切なエラーハンドリングを実装します:

```ts
handle: async (interaction) => {
  try {
    // 処理
  } catch (error) {
    console.error("Error:", error);
    return createBuilder(interaction)
      .reply()
      .content("An error occurred. Please try again later.")
      .flags(MessageFlags.Ephemeral)
      .build();
  }
}
```

### 4. エフェメラルメッセージの活用

エラーメッセージやプライベートな情報はエフェメラルフラグを使用します。

### 5. レスポンス時間

Discordは3秒以内のレスポンスを期待します。長時間かかる処理は遅延レスポンスを使用します:

```ts
// 即座に遅延レスポンスを返す
const deferred = createBuilder(interaction)
  .deferReply()
  .build();

// 後で更新
// （Webhookを使用してメッセージを送信）
```

## トラブルシューティング

### コマンドが表示されない

- マニフェストが`pending`のまま承認されていない可能性があります。Nexusの管理画面で承認してください
- Nexusにマニフェストが正しく登録されているか確認
- Discord Developer Portalでbotのスコープに`applications.commands`が含まれているか確認

### Interactionタイムアウト

- 3秒以内にレスポンスを返していない可能性があります
- 長時間かかる処理は`deferReply()`を使用

### 署名検証エラー

- Service Worker側の`NEXUS_PUBLIC_KEY`がNexusの`NEXUS_PUBLIC_KEY`（公開鍵）と一致しているか確認
- `nexusVerifyMiddleware`が`/interactions`に正しく適用されているか確認
- 詳細は[Nexus 認証システム](../nexus/authentication.ja.md)のトラブルシューティングを参照

### デプロイエラー

- Wrangler CLIが最新バージョンか確認: `npm install -D wrangler@latest`
- Cloudflare APIトークンが正しく設定されているか確認

## リソース

- [Hono Documentation](https://hono.dev/)
- [Cloudflare Workers Documentation](https://developers.cloudflare.com/workers/)
- [Discord API Documentation](https://discord.com/developers/docs)
- [Hiyocord Packages](../packages/index.ja.md)
- [Hiyocord Nexus](../nexus/index.ja.md)
