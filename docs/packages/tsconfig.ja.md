# @hiyocord/tsconfig

TypeScript設定ファイルの共有パッケージです。

## 用途

- モノレポ全体で一貫したTypeScript設定を提供
- 環境別の設定（Node.js、Workers、Test）

## 設定ファイル

- `node/tsconfig.json` - Node.js環境用
- `shared/tsconfig.json` - 共通設定
- `test/tsconfig.json` - テスト環境用
- `workers/tsconfig.json` - Cloudflare Workers用

## 使用例

```json
{
  "extends": "@hiyocord/tsconfig/node/tsconfig.json"
}
```
