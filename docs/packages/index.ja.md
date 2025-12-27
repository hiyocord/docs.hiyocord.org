# Hiyocord Packages

Hiyocord Packagesは、Hiyocordエコシステムで使用される共有ライブラリとAPIラッパーのコレクションです。OpenAPI仕様から自動生成された型安全なクライアントと、Discord interaction処理のための高レベルユーティリティを提供します。

## パッケージ一覧

### [@hiyocord/tsconfig](tsconfig.ja.md)

TypeScript設定ファイルの共有パッケージです。モノレポ全体で一貫したTypeScript設定を提供します。

### [@hiyocord/wrangler-configurer](wrangler-configurer.ja.md)

Wrangler設定ファイルを生成するためのコマンドラインユーティリティです。環境固有の設定をバージョン管理から分離します。

### [@hiyocord/rest-api-core](rest-api-core.ja.md)

OpenAPI仕様ベースのREST APIクライアント基盤パッケージです。型安全なクライアント生成をサポートします。

### [@hiyocord/discord-rest-api](discord-rest-api.ja.md)

Discord REST API v10の型安全なTypeScriptクライアントです。公式OpenAPI仕様から自動生成された型定義を使用します。

### [@hiyocord/github-rest-api](github-rest-api.ja.md)

GitHub REST APIの型安全なTypeScriptクライアントです。GitHub OpenAPI仕様から自動生成された型定義を使用します。

### [@hiyocord/discord-interaction-client](discord-interaction-client.ja.md)

Discord bot interactionの処理とレスポンス構築のための高レベルクライアントです。ハンドラーレジストリと型安全なレスポンスビルダーを提供します。

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

## モノレポ構造

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

## ビルドシステム

すべてのパッケージはViteでビルドされます:

- **入力**: `src/index.ts`
- **出力**: `dist/` (ESM形式)
- **型定義**: `vite-plugin-dts`で自動生成
- **パス解決**: `vite-tsconfig-paths`でエイリアスサポート

## CI/CD

GitHub Actionsワークフロー:

1. **Buildジョブ**: すべてのパッケージをビルド
2. **Lintジョブ**: Prettierでコードフォーマットをチェック
3. **Testジョブ**: （未実装）
4. **Publishジョブ**: Changesetsでバージョン管理とGitHub Packagesへの公開

## パッケージ公開

すべてのパッケージはGitHub Packages Registryに公開されます:

```bash
# .npmrc設定
@hiyocord:registry=https://npm.pkg.github.com
```

## ベストプラクティス

### 1. 型安全性の活用

OpenAPI生成型を活用して、コンパイル時にAPIエラーを検出します:

```ts
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

```ts
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

```ts
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

```ts
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
