# @hiyocord/github-rest-api

GitHub REST APIの型安全なTypeScriptクライアントです。

## 概要

公式のGitHub OpenAPI仕様から自動生成された型定義を使用します。Discord REST APIクライアントと同様のパターンで実装されています。

## 主な機能

- GitHub REST APIの完全な型サポート
- 自動トークン認証
- ESMとCommonJSの両方をサポート

## インストール

```bash
npm install @hiyocord/github-rest-api
```

## 基本的な使い方

```ts
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

## OpenAPI型の再生成

```bash
npm run openapi
```

このコマンドは、最新のGitHub OpenAPI仕様をダウンロードし、`github-api-spec.gen.ts`を再生成します。

## TODO

- GitHub Apps認証サポート
- Personal Access Token (PAT)サポートの実装

## パッケージ情報

- バージョン: v1.0.2
- 依存: `openapi-fetch@^0.15.0`
- レジストリ: GitHub Packages
- 生成ファイル: `github-api-spec.gen.ts` (5.7 MB)
