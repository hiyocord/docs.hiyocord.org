# @hiyocord/wrangler-configurer

Wrangler設定ファイルを生成するためのコマンドラインユーティリティです。

## 概要

`wrangler.config.ts`を読み込み、環境変数とマージして`wrangler.jsonc`を出力します。Service BindingのIDなどの環境固有の設定をバージョン管理から分離することで、利用者の環境に手を加えずデプロイできるようにします。

## 主な機能

- TypeScriptで型安全な設定ファイルの記述
- 環境変数による設定のオーバーライド
- Wrangler設定の自動生成

## インストール

```bash
npm install -D @hiyocord/wrangler-configurer
```

## 使い方

プロジェクトルートに`wrangler.config.ts`を配置します:

```ts
// wrangler.config.ts
import type { WranglerConfigurerOptions } from '@hiyocord/wrangler-configurer'

export default {
  params: {
    name: "my-worker",
    main: "./src/index.ts",
    compatibility_date: "2025-10-08",
    compatibility_flags: ["nodejs_compat"],
    observability: {
      enabled: true,
      logs: {
        enabled: true,
        head_sampling_rate: 1,
        persist: true,
        invocation_logs: true
      }
    }
  }
} satisfies WranglerConfigurerOptions
```

`wrangler-configurer`コマンドで`wrangler.jsonc`を生成します:

```bash
npx wrangler-configurer
```

## 環境変数によるオーバーライド

環境変数`VITE_WRANGLER_ARGS`にJSONを指定することで、設定を動的に変更できます:

```bash
export VITE_WRANGLER_ARGS='{"kv_namespaces":[{"binding":"KV","id":"your-kv-id"}]}'
npx wrangler-configurer
```

## 使用例（Hiyocord Nexus）

```ts
// wrangler.config.ts
import type { WranglerConfigurerOptions } from '@hiyocord/wrangler-configurer'

export default {
  params: {
    name: "hiyocord-nexus",
    main: "./src/index.ts",
    compatibility_date: "2025-10-08",
    compatibility_flags: ["nodejs_compat"],
    preview_urls: true,
    workers_dev: false,
    observability: {
      enabled: true
    },
    upload_source_maps: true,
    kv_namespaces: [{
      binding: "KV",
      id: process.env["HIYOCORD_NEXUS_KV_ID"]
    }]
  }
} satisfies WranglerConfigurerOptions
```

環境変数でKVネームスペースIDを指定:

```bash
export HIYOCORD_NEXUS_KV_ID="your-kv-namespace-id"
npx wrangler-configurer
wrangler deploy
```

## 利点

- **バージョン管理からの分離**: Service BindingのIDなどの環境固有の設定をGitにコミットする必要がない
- **型安全性**: TypeScriptで設定を記述できるため、コンパイル時にエラーを検出
- **再利用性**: テンプレートリポジトリとして配布し、利用者が自分の環境にそのままデプロイ可能

## パッケージ情報

- バージョン: v1.0.1
- ピア依存: `wrangler@^4.43.0`
- レジストリ: npm
- リポジトリ: [github.com/hiyocord/wrangler-configurer](https://github.com/hiyocord/wrangler-configurer)
