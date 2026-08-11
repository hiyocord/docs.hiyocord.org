# hiyocordのコントリビューター

このページは、Hiyocordプラットフォーム自体（`hiyocord-nexus`, `hiyocord-packages`など）の開発に貢献したい人向けの概要です。自分のDiscord bot（サービス）をHiyocordの上に構築したいだけの場合は、[hiyocord serviceの開発者](../service-developer/getting-started/index.ja.md)向けドキュメントを参照してください。

!!! note
    現時点では、各リポジトリに`CONTRIBUTING.md`のような詳細なガイドは整備されていません。ここでは各リポジトリの開発コマンドとリリースフローの要点のみをまとめています。詳細な手順や設計判断については、リポジトリ本体（README、CHANGELOG、Issue/PR）を参照してください。

## 対象リポジトリ

| リポジトリ | 役割 |
|---|---|
| [hiyocord-nexus](https://github.com/hiyocord/hiyocord-nexus) | 中央ルーティングハブ本体。`hiyocord-nexus`（Worker本体）、`hiyocord-nexus-core`（共有ライブラリ）、`hiyocord-nexus-types`（型定義）、`hiyocord-nexus-cli`（`gen-key`/`manifest`コマンド）、`hiyocord-nexus-web`（管理画面）のモノレポ |
| [hiyocord-packages](https://github.com/hiyocord/hiyocord-packages) | Service Worker側で使う共有ライブラリ群（`discord-rest-api`, `github-rest-api`, `discord-interaction-client`, `tsconfig`）のモノレポ |
| [wrangler-configurer](https://github.com/hiyocord/wrangler-configurer) | `wrangler.config.ts`から`wrangler.jsonc`を生成するCLIツール |
| [hiyocord-service-workers](https://github.com/hiyocord/hiyocord-service-workers) | Service Worker用のGitHubテンプレートリポジトリ |
| [discord-auth-service](https://github.com/hiyocord/discord-auth-service) | テンプレートを使った実サービスの実装例 |

## 開発環境構築

いずれのリポジトリも共通で、クローン後に`npm install`で依存関係をインストールします。

```bash
git clone https://github.com/hiyocord/<repo>.git
cd <repo>
npm install
```

### テスト・Lint

リポジトリごとに整備状況が異なります:

| リポジトリ | テスト | Lint |
|---|---|---|
| hiyocord-nexus | `npm run test`（Vitest。`hiyocord-nexus`と`hiyocord-nexus-core`の一部モジュールにテストあり） | ルートレベルのlint設定なし（`hiyocord-nexus-web`のみ`npm run lint`でESLint） |
| hiyocord-packages | `npm run test`（各パッケージは現状スタブで実質未実装） | `npm run lint`（Prettier）、`npm run eslint:check` |
| wrangler-configurer | 未実装（`test`スクリプトはstub） | 未実装（`lint`スクリプトはstub） |
| hiyocord-service-workers / discord-auth-service | 未実装（`npm test`はエラーで終了するプレースホルダー） | なし |

## リリースフロー

リポジトリによってリリースの仕組みが異なります:

- **hiyocord-nexus / hiyocord-packages**: [Changesets](https://github.com/changesets/changesets)を使用。変更をPRに含める際は`npx changeset`でchangesetファイルを追加します。`master`へのマージ後、CIが`npx changeset publish`でGitHub Packages（`npm.pkg.github.com`）に自動公開します
- **wrangler-configurer**: Changesetsは使わず、GitHub Actionsの`workflow_dispatch`（`npm version <patch|minor|major|premajor|prerelease>`を手動実行）でバージョンを上げ、タグpushをトリガーにGitHub Packagesとnpmjsの両方へ公開します
- **hiyocord-service-workers / discord-auth-service**: パッケージとしては公開されません。`master`へのpushで`wrangler deploy`によりCloudflare Workersへ直接デプロイされます

## ブランチ・レビュー運用

- すべてのリポジトリで既定ブランチは`master`です（`main`ではありません）
- PRテンプレート・Issueテンプレートは現時点でどのリポジトリにも存在しません
- [CODEOWNERS](https://github.com/hiyocord/hiyocord-packages/blob/master/.github/CODEOWNERS)は`hiyocord-packages`のみに存在し、`.github/workflows`と`*.gen.ts`（OpenAPI生成コード）が対象です
- 依存関係の更新はRenovate（[hiyocord/renovate](https://github.com/hiyocord/renovate)の共通設定を利用）とDependabotの両方で自動化されています

## 関連リソース

- [GitHub Organization](https://github.com/hiyocord)
- [Hiyocord Nexus アーキテクチャ](../service-developer/nexus/index.ja.md) - コントリビューターにとっても、Nexusの内部構造を理解する上で参考になります
- 各リポジトリの`CHANGELOG.md`（Changesetsにより自動生成）には、これまでの設計変更の経緯が記録されています
