# Hiyocord Packages

Hiyocord Packagesは、Hiyocordエコシステムで使用される共有ライブラリとAPIラッパーのコレクションです。OpenAPI仕様から自動生成された型安全なクライアントと、Discord interaction処理のための高レベルユーティリティを提供します。

## パッケージ一覧

### @hiyocord/tsconfig

TypeScript設定ファイルの共有パッケージです。

**用途:**

- モノレポ全体で一貫したTypeScript設定を提供
- 環境別の設定（Node.js、Workers、Test）

**設定ファイル:**

- `node/tsconfig.json` - Node.js環境用
- `shared/tsconfig.json` - 共通設定
- `test/tsconfig.json` - テスト環境用
- `workers/tsconfig.json` - Cloudflare Workers用

**使用例:**

```json
{
  "extends": "@hiyocord/tsconfig/node/tsconfig.json"
}
```

---

### @hiyocord/rest-api-core

OpenAPI仕様ベースのREST APIクライアント基盤パッケージです。

**概要:**

`openapi-fetch`をラップし、再利用可能なREST APIクライアントファクトリを提供します。

**主な機能:**

- OpenAPI仕様からの型安全なクライアント生成
- 拡張可能な「ショートカット」システム
- HTTPメソッドの小文字ショートカット（get, post, put, delete, patch等）

**基本的な使い方:**

```typescript
import { createClient } from "@hiyocord/rest-api-core";
import type { paths } from "./api-spec.gen";

const client = createClient<paths>({
  baseUrl: "https://api.example.com"
});

// 型安全なAPIコール
const { data, error } = await client.GET("/users/{id}", {
  params: { path: { id: "123" } }
});
```

**カスタムショートカットの追加:**

```typescript
const shortcuts = {
  async getUser(id: string) {
    return this.GET("/users/{id}", {
      params: { path: { id } }
    });
  }
};

const client = createClient<paths>({
  baseUrl: "https://api.example.com",
  shortcuts
});

// ショートカットを使用
const { data } = await client.getUser("123");
```

**パッケージ情報:**

- バージョン: v2.0.0
- 依存: `openapi-fetch@^0.15.0`
- レジストリ: GitHub Packages

---

### @hiyocord/discord-rest-api

Discord REST API v10の型安全なTypeScriptクライアントです。

**概要:**

公式のDiscord OpenAPI仕様から自動生成された型定義を使用し、Botトークン認証を自動で処理します。

**主な機能:**

- Discord API v10の完全な型サポート
- 自動Botトークン注入
- OpenAPI仕様からの型生成

**インストール:**

```bash
npm install @hiyocord/discord-rest-api
```

**基本的な使い方:**

```typescript
import { getClient } from "@hiyocord/discord-rest-api";

const discord = getClient("YOUR_BOT_TOKEN");

// ギルド情報の取得
const { data: guild } = await discord.GET("/guilds/{guild.id}", {
  params: { path: { "guild.id": "123456789" } }
});

// メッセージの送信
const { data: message } = await discord.POST(
  "/channels/{channel.id}/messages",
  {
    params: { path: { "channel.id": "987654321" } },
    body: {
      content: "Hello from Hiyocord!"
    }
  }
);

// アプリケーションコマンドの登録
const { data: command } = await discord.POST(
  "/applications/{application.id}/commands",
  {
    params: { path: { "application.id": "your-app-id" } },
    body: {
      name: "ping",
      description: "Responds with pong",
      type: 1
    }
  }
);
```

**OpenAPI型の再生成:**

```bash
npm run openapi
```

このコマンドは、最新のDiscord OpenAPI仕様をダウンロードし、`discord-api-spec.gen.ts`を再生成します。

**TODO:**

- レート制限処理の実装

**パッケージ情報:**

- バージョン: v2.0.2
- 依存: `openapi-fetch@^0.15.0`
- レジストリ: GitHub Packages
- 生成ファイル: `discord-api-spec.gen.ts` (575 KB)

---

### @hiyocord/github-rest-api

GitHub REST APIの型安全なTypeScriptクライアントです。

**概要:**

公式のGitHub OpenAPI仕様から自動生成された型定義を使用します。Discord REST APIクライアントと同様のパターンで実装されています。

**主な機能:**

- GitHub REST APIの完全な型サポート
- 自動トークン認証
- ESMとCommonJSの両方をサポート

**インストール:**

```bash
npm install @hiyocord/github-rest-api
```

**基本的な使い方:**

```typescript
import { getClient } from "@hiyocord/github-rest-api";

const github = getClient("YOUR_GITHUB_TOKEN");

// リポジトリ情報の取得
const { data: repo } = await github.GET("/repos/{owner}/{repo}", {
  params: {
    path: {
      owner: "hiyocord",
      repo: "hiyocord-nexus"
    }
  }
});

// Issueの作成
const { data: issue } = await github.POST("/repos/{owner}/{repo}/issues", {
  params: {
    path: {
      owner: "hiyocord",
      repo: "hiyocord-nexus"
    }
  },
  body: {
    title: "New feature request",
    body: "Description of the feature"
  }
});
```

**OpenAPI型の再生成:**

```bash
npm run openapi
```

このコマンドは、最新のGitHub OpenAPI仕様をダウンロードし、`github-api-spec.gen.ts`を再生成します。

**TODO:**

- GitHub Apps認証サポート
- Personal Access Token (PAT)サポートの実装

**パッケージ情報:**

- バージョン: v1.0.2
- 依存: `openapi-fetch@^0.15.0`
- レジストリ: GitHub Packages
- 生成ファイル: `github-api-spec.gen.ts` (5.7 MB)

---

### @hiyocord/discord-interaction-client

Discord bot interactionの処理とレスポンス構築のための高レベルクライアントです。

**概要:**

Discord interactionハンドラーの登録、解決、レスポンスビルダーなど、Discord botの開発に必要な機能を提供します。

**主な機能:**

- Interactionハンドラーのレジストリパターン
- 型安全なレスポンスビルダー
- Fetch API互換のハンドラーインターフェース

**インストール:**

```bash
npm install @hiyocord/discord-interaction-client
```

#### コアコンセプト

##### 1. Interaction Types

Discord API v10で定義されているすべてのinteractionタイプをサポートします:

```typescript
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

##### 2. Handler Registry

ハンドラーをinteractionタイプ別に登録します:

```typescript
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

##### 3. Handler Resolver

登録されたハンドラーから適切なものを解決します:

```typescript
import {
  SimpleInteractionHandlerResolver,
  DelegatingTypedInteractionHandlerResolver
} from "@hiyocord/discord-interaction-client";

const resolver = new SimpleInteractionHandlerResolver(registry);

// または、より高度なリゾルバー
const delegatingResolver = new DelegatingTypedInteractionHandlerResolver(registry);
```

##### 4. Response Builders

流暢なインターフェースでinteractionレスポンスを構築します:

```typescript
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

#### 完全な使用例

```typescript
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

#### Application Command Handler

スラッシュコマンドを処理します:

```typescript
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

#### Message Component Handler

ボタンやセレクトメニューを処理します:

```typescript
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

#### Modal Submit Handler

モーダル送信を処理します:

```typescript
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

#### Autocomplete Handler

オートコンプリートを処理します:

```typescript
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

#### レスポンスビルダー一覧

| ビルダー | 用途 | レスポンスタイプ |
|---------|------|-----------------|
| `reply()` | 新しいメッセージを返信 | CHANNEL_MESSAGE_WITH_SOURCE (4) |
| `deferReply()` | レスポンスを遅延（後で更新） | DEFERRED_CHANNEL_MESSAGE_WITH_SOURCE (5) |
| `updateMessage()` | 既存のメッセージを更新 | UPDATE_MESSAGE (7) |
| `deferUpdate()` | メッセージ更新を遅延 | DEFERRED_UPDATE_MESSAGE (6) |
| `modal()` | モーダルを表示 | MODAL (9) |
| `pong()` | Pingに応答 | PONG (1) |

#### 高度な使用例: エンベッド付きメッセージ

```typescript
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

**パッケージ情報:**

- バージョン: v0.0.6
- 依存: `discord-api-types@^0.38.37`
- レジストリ: GitHub Packages

---

## 開発環境構築

### 前提条件

- Node.js 22以上
- npm

### セットアップ

```bash
# リポジトリのクローン
git clone https://github.com/hiyocord/hiyocord-packages.git
cd hiyocord-packages

# 依存関係のインストール
npm install

# ビルド
npm run build

# フォーマットチェック
npm run lint

# 自動フォーマット
npm run fixlint
```

### モノレポ構造

```
hiyocord-packages/
├── packages/
│   ├── tsconfig/              # 共有TypeScript設定
│   ├── rest-api-core/          # REST APIクライアント基盤
│   ├── discord-rest-api/       # Discord APIクライアント
│   ├── github-rest-api/        # GitHub APIクライアント
│   └── discord-interaction-client/  # Interaction処理
├── package.json               # ワークスペース設定
└── .github/workflows/
    └── build_and_release.yml  # CI/CDパイプライン
```

### ビルドシステム

すべてのパッケージはViteでビルドされます:

- **入力**: `src/index.ts`
- **出力**: `dist/` (ESM形式)
- **型定義**: `vite-plugin-dts`で自動生成
- **パス解決**: `vite-tsconfig-paths`でエイリアスサポート

### CI/CD

GitHub Actionsワークフロー:

1. **Buildジョブ**: すべてのパッケージをビルド
2. **Lintジョブ**: Prettierでコードフォーマットをチェック
3. **Testジョブ**: （未実装）
4. **Publishジョブ**: Changesetsでバージョン管理とGitHub Packagesへの公開

### パッケージ公開

すべてのパッケージはGitHub Packages Registryに公開されます:

```bash
# .npmrc設定
@hiyocord:registry=https://npm.pkg.github.com
```

### OpenAPI型の更新

Discord/GitHub APIの型を更新するには:

```bash
# discord-rest-apiパッケージ
cd packages/discord-rest-api
npm run openapi

# github-rest-apiパッケージ
cd packages/github-rest-api
npm run openapi
```

## ベストプラクティス

### 1. 型安全性の活用

OpenAPI生成型を活用して、コンパイル時にAPIエラーを検出します:

```typescript
// 型エラー: 存在しないパスパラメータ
await discord.GET("/guilds/{guild.id}", {
  params: { path: { invalid_param: "123" } }  // ❌ エラー
});

// 正しい使い方
await discord.GET("/guilds/{guild.id}", {
  params: { path: { "guild.id": "123" } }  // ✅ OK
});
```

### 2. ハンドラーの分離

各コマンドハンドラーを個別のファイルに分離します:

```typescript
// handlers/ping.ts
export default {
  name: "ping",
  description: "Responds with pong",
  handle: async (c) => { /* ... */ }
} satisfies ApplicationCommandHandler;

// register.ts
import pingHandler from "./handlers/ping";
registry.register(InteractionType.ApplicationCommand, pingHandler);
```

### 3. エラーハンドリング

適切なエラーハンドリングを実装します:

```typescript
const handler: ApplicationCommandHandler = {
  name: "data",
  description: "Fetches data",
  handle: async (interaction) => {
    try {
      const data = await fetchData();
      return createBuilder(interaction)
        .reply()
        .content(`Data: ${data}`)
        .build();
    } catch (error) {
      return createBuilder(interaction)
        .reply()
        .content("An error occurred while fetching data.")
        .flags(MessageFlags.Ephemeral)
        .build();
    }
  }
};
```

### 4. レスポンスビルダーの活用

ビルダーパターンで可読性の高いコードを書きます:

```typescript
// ❌ 生のオブジェクト
return {
  type: 4,
  data: {
    content: "Hello",
    flags: 64
  }
};

// ✅ ビルダー使用
return createBuilder(interaction)
  .reply()
  .content("Hello")
  .flags(MessageFlags.Ephemeral)
  .build();
```

## トラブルシューティング

### GitHub Packagesからのインストールエラー

`.npmrc`ファイルに正しいレジストリ設定があることを確認してください:

```
@hiyocord:registry=https://npm.pkg.github.com
//npm.pkg.github.com/:_authToken=${GITHUB_TOKEN}
```

### OpenAPI型生成エラー

最新の`openapi-typescript`を使用していることを確認してください:

```bash
npm install -D openapi-typescript@latest
```

### ビルドエラー

依存関係を再インストールしてみてください:

```bash
npm run clean
rm -rf node_modules
npm install
npm run build
```

## リソース

- [openapi-fetch Documentation](https://openapi-ts.pages.dev/openapi-fetch/)
- [Discord API Types](https://discord-api-types.dev/)
- [Vite Plugin DTS](https://github.com/qmhc/vite-plugin-dts)
