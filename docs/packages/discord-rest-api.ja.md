# @hiyocord/discord-rest-api

Discord REST API v10の型安全なTypeScriptクライアントです。

## 概要

公式のDiscord OpenAPI仕様から自動生成された型定義を使用し、Botトークン認証を自動で処理します。

## 主な機能

- Discord API v10の完全な型サポート
- 自動Botトークン注入
- OpenAPI仕様からの型生成

## インストール

```bash
npm install @hiyocord/discord-rest-api
```

## 基本的な使い方

```ts
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

## OpenAPI型の再生成

```bash
npm run openapi
```

このコマンドは、最新のDiscord OpenAPI仕様をダウンロードし、`discord-api-spec.gen.ts`を再生成します。

## TODO

- レート制限処理の実装

## パッケージ情報

- バージョン: v2.0.2
- 依存: `openapi-fetch@^0.15.0`
- レジストリ: GitHub Packages
- 生成ファイル: `discord-api-spec.gen.ts` (575 KB)
