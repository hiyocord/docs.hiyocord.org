# @hiyocord/discord-interaction-client

Discord bot interactionの処理とレスポンス構築のための高レベルクライアントです。

## 概要

Discord interactionハンドラーの登録、解決、レスポンスビルダーなど、Discord botの開発に必要な機能を提供します。

## 主な機能

- Interactionハンドラーのレジストリパターン
- 型安全なレスポンスビルダー
- Fetch API互換のハンドラーインターフェース

## インストール

```bash
npm install @hiyocord/discord-interaction-client
```

## コアコンセプト

### 1. Interaction Types

Discord API v10で定義されているすべてのinteractionタイプをサポートします:

```ts
import {
  InteractionType,
  ApplicationCommandType
} from "@hiyocord/discord-interaction-client";

// InteractionType
// - Ping (1)
// - ApplicationCommand (2)
// - MessageComponent (3)
// - ApplicationCommandAutocomplete (4)
// - ModalSubmit (5)
```

### 2. Handler Registry

ハンドラーをinteractionタイプ別に登録します:

```ts
import {
  SimpleInteractionHandlerRegistry,
  InteractionType
} from "@hiyocord/discord-interaction-client";

const registry = new SimpleInteractionHandlerRegistry();

// Application Commandハンドラーの登録
registry.register(InteractionType.ApplicationCommand, {
  name: "ping",
  description: "Responds with pong",
  handle: async (interaction) => {
    return {
      type: 4,
      data: { content: "Pong!" }
    };
  }
});
```

### 3. Handler Resolver

登録されたハンドラーから適切なものを解決します:

```ts
import {
  SimpleInteractionHandlerResolver,
  DelegatingTypedInteractionHandlerResolver
} from "@hiyocord/discord-interaction-client";

const resolver = new SimpleInteractionHandlerResolver(registry);

// または、より高度なリゾルバー
const delegatingResolver = new DelegatingTypedInteractionHandlerResolver(registry);
```

### 4. Response Builders

流暢なインターフェースでinteractionレスポンスを構築します:

```ts
import {
  createBuilder,
  MessageFlags
} from "@hiyocord/discord-interaction-client";

// チャンネルメッセージレスポンス
const response = createBuilder(interaction)
  .reply()
  .content("Hello from Hiyocord!")
  .flags(MessageFlags.Ephemeral)  // エフェメラルメッセージ
  .build();

// Deferredレスポンス
const deferred = createBuilder(interaction)
  .deferReply()
  .flags(MessageFlags.Ephemeral)
  .build();

// メッセージ更新
const updated = createBuilder(interaction)
  .updateMessage()
  .content("Updated content")
  .build();

// モーダル表示
const modal = createBuilder(interaction)
  .modal()
  .customId("feedback_modal")
  .title("Feedback Form")
  .components([
    {
      type: 1,
      components: [
        {
          type: 4,
          custom_id: "feedback_text",
          label: "Your Feedback",
          style: 2,
          required: true
        }
      ]
    }
  ])
  .build();
```

## 完全な使用例

```ts
import {
  SimpleInteractionHandlerRegistry,
  SimpleInteractionHandlerResolver,
  fetchHandler,
  createBuilder,
  MessageFlags,
  ApplicationCommandHandler
} from "@hiyocord/discord-interaction-client";

// 1. レジストリの作成
const registry = new SimpleInteractionHandlerRegistry();

// 2. ハンドラーの定義と登録
const pingHandler: ApplicationCommandHandler = {
  name: "ping",
  description: "Responds with pong",
  handle: async (interaction) => {
    return createBuilder(interaction)
      .reply()
      .content("Pong!")
      .flags(MessageFlags.Ephemeral)
      .build();
  }
};

registry.register(InteractionType.ApplicationCommand, pingHandler);

// 3. リゾルバーの作成
const resolver = new SimpleInteractionHandlerResolver(registry);

// 4. Fetch APIハンドラーの作成
const handler = fetchHandler(resolver);

// 5. Cloudflare Workersでの使用
export default {
  async fetch(request: Request, env: Env): Promise<Response> {
    return handler.fetch(request, env);
  }
};
```

## Application Command Handler

スラッシュコマンドを処理します:

```ts
import type { ApplicationCommandHandler } from "@hiyocord/discord-interaction-client";

const commandHandler: ApplicationCommandHandler = {
  name: "greet",
  description: "Greets a user",
  options: [
    {
      name: "user",
      description: "User to greet",
      type: 6,  // USER type
      required: true
    }
  ],
  handle: async (interaction) => {
    const userId = interaction.data.options?.[0]?.value;
    return createBuilder(interaction)
      .reply()
      .content(`Hello <@${userId}>!`)
      .build();
  }
};
```

## Message Component Handler

ボタンやセレクトメニューを処理します:

```ts
import type { MessageComponentHandler } from "@hiyocord/discord-interaction-client";

const buttonHandler: MessageComponentHandler = {
  customId: "confirm_button",
  handle: async (interaction) => {
    return createBuilder(interaction)
      .updateMessage()
      .content("Confirmed!")
      .components([])  // ボタンを削除
      .build();
  }
};

registry.register(InteractionType.MessageComponent, buttonHandler);
```

## Modal Submit Handler

モーダル送信を処理します:

```ts
import type { ModalSubmitHandler } from "@hiyocord/discord-interaction-client";

const modalHandler: ModalSubmitHandler = {
  customId: "feedback_modal",
  handle: async (interaction) => {
    const feedback = interaction.data.components?.[0]?.components?.[0]?.value;
    return createBuilder(interaction)
      .reply()
      .content(`Thanks for your feedback: ${feedback}`)
      .flags(MessageFlags.Ephemeral)
      .build();
  }
};

registry.register(InteractionType.ModalSubmit, modalHandler);
```

## Autocomplete Handler

オートコンプリートを処理します:

```ts
import type { AutocompleteHandler } from "@hiyocord/discord-interaction-client";

const autocompleteHandler: AutocompleteHandler = {
  commandName: "search",
  handle: async (interaction) => {
    const focusedOption = interaction.data.options?.find(opt => opt.focused);
    const query = focusedOption?.value || "";

    // 候補を検索
    const choices = await searchDatabase(query);

    return {
      type: 8,  // APPLICATION_COMMAND_AUTOCOMPLETE_RESULT
      data: {
        choices: choices.map(c => ({
          name: c.name,
          value: c.value
        }))
      }
    };
  }
};

registry.register(InteractionType.ApplicationCommandAutocomplete, autocompleteHandler);
```

## レスポンスビルダー一覧

| ビルダー | 用途 | レスポンスタイプ |
|---------|------|-----------------|
| `reply()` | 新しいメッセージを返信 | CHANNEL_MESSAGE_WITH_SOURCE (4) |
| `deferReply()` | レスポンスを遅延（後で更新） | DEFERRED_CHANNEL_MESSAGE_WITH_SOURCE (5) |
| `updateMessage()` | 既存のメッセージを更新 | UPDATE_MESSAGE (7) |
| `deferUpdate()` | メッセージ更新を遅延 | DEFERRED_UPDATE_MESSAGE (6) |
| `modal()` | モーダルを表示 | MODAL (9) |
| `pong()` | Pingに応答 | PONG (1) |

## 高度な使用例: エンベッド付きメッセージ

```ts
const handler: ApplicationCommandHandler = {
  name: "info",
  description: "Shows bot information",
  handle: async (interaction) => {
    return createBuilder(interaction)
      .reply()
      .embeds([
        {
          title: "Hiyocord Bot",
          description: "Powered by Cloudflare Workers",
          color: 0x5865F2,
          fields: [
            {
              name: "Version",
              value: "1.0.0",
              inline: true
            },
            {
              name: "Uptime",
              value: "99.9%",
              inline: true
            }
          ],
          timestamp: new Date().toISOString(),
          footer: {
            text: "Hiyocord"
          }
        }
      ])
      .build();
  }
};
```

## パッケージ情報

- バージョン: v0.0.6
- 依存: `discord-api-types@^0.38.37`
- レジストリ: GitHub Packages
