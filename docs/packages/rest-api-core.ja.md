# @hiyocord/rest-api-core

OpenAPI仕様ベースのREST APIクライアント基盤パッケージです。

## 概要

`openapi-fetch`をラップし、再利用可能なREST APIクライアントファクトリを提供します。

## 主な機能

- OpenAPI仕様からの型安全なクライアント生成
- 拡張可能な「ショートカット」システム
- HTTPメソッドの小文字ショートカット（get, post, put, delete, patch等）

## 基本的な使い方

```ts
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

## カスタムショートカットの追加

```ts
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

## パッケージ情報

- バージョン: v2.0.0
- 依存: `openapi-fetch@^0.15.0`
- レジストリ: GitHub Packages
