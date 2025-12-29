# 認証システム

Hiyocord Nexusは2つの認証レイヤーを使用してセキュリティを確保しています。

## Discord からの認証

NexusはDiscordからのリクエストをEd25519署名で検証します。

```typescript
import { verifyKey } from "discord-interactions";

const isValid = verifyKey(
  rawBody,
  signature,
  timestamp,
  publicKey
);
```

## Nexus ⇄ Service Worker間の認証

### 概要

Nexus と Service Worker 間の認証には **Ed25519公開鍵暗号**を使用します。これにより、各Service Workerが独立した鍵ペアを持ち、1つのサービスが侵害されても他のサービスに影響を与えません。

### アーキテクチャ

```
┌──────────────┐                 ┌──────────────┐
│    Nexus     │                 │   Service    │
│              │                 │    Worker    │
│ Private Key  │ ──────────────> │ Public Key   │
│              │   署名リクエスト  │              │
│ Public Key   │ <────────────── │ Private Key  │
│              │   署名リクエスト  │              │
└──────────────┘                 └──────────────┘
```

### 認証フロー

#### 1. Nexus → Service Worker (Interaction転送)

Nexusはinteractionをサービスワーカーに転送する際、自身の秘密鍵で署名します:

```typescript
// Nexus側 (interaction-transfer.ts)
const algorithm = ctx.getNexusSignatureAlgorithm(); // "ed25519"
const privateKey = ctx.getNexusPrivateKey();

const signedHeaders = await signRequest(
  algorithm,
  privateKey,
  headers,
  body
);

// 署名されたヘッダー:
// X-Hiyocord-Signature: base64-encoded-signature
// X-Hiyocord-Timestamp: 1234567890
// X-Hiyocord-Algorithm: ed25519
```

Service Workerは、Nexusの公開鍵を使って署名を検証します:

```typescript
// Service Worker側
import { nexusVerifyMiddleware } from "@hiyocord/hiyocord-nexus-core";

app.post("/interactions", nexusVerifyMiddleware, async (c) => {
  // リクエストが検証済み
  const interaction = await c.req.json();
  return c.json(response);
});
```

#### 2. Service Worker → Nexus (Discord API Proxy)

Service WorkerがNexus経由でDiscord APIを呼び出す際は、Service Workerの秘密鍵で署名します:

```typescript
// Service Worker側
import { signServiceWorkerRequest } from "@hiyocord/hiyocord-nexus-core";

const signedHeaders = await signServiceWorkerRequest(
  algorithm,
  privateKey,
  manifestId,
  headers,
  body
);

// 署名されたヘッダー:
// X-Hiyocord-Signature: base64-encoded-signature
// X-Hiyocord-Timestamp: 1234567890
// X-Hiyocord-Algorithm: ed25519
// X-Hiyocord-Manifest-Id: your-service-id
```

Nexusは、マニフェストに登録されたService Workerの公開鍵で署名を検証します:

```typescript
// Nexus側 (service-worker-verify middleware)
app.all("/proxy/discord/api/v10/*", verifyServiceWorker, async (c) => {
  const manifestId = c.var.manifestId;
  // リクエストが検証済み
  return c.json(await DiscordApiProxyService(ctx, c.req.raw, manifestId));
});
```

### 署名アルゴリズム

現在サポートされているアルゴリズム:

- **ed25519** (推奨・実装済み)
- ecdsa-p256 (将来実装予定)
- rsa-pss-2048 (将来実装予定)

### 鍵の生成

#### Nexus鍵ペアの生成

```bash
# リポジトリに含まれるスクリプトを使用
cd hiyocord-nexus
npx tsx scripts/generate-keypair.ts
```

出力例:

```
=== Ed25519 Key Pair Generated ===

Public Key (配布用):
Vx1234567890abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ==

Private Key (秘密):
Ax9876543210zyxwvutsrqponmlkjihgfedcbaZYXWVUTSRQPONMLKJIHGFEDCBA==
```

#### Service Worker鍵ペアの生成

Service Worker側でも同じスクリプトを使用できます:

```bash
npx tsx scripts/generate-keypair.ts
```

### 環境変数の設定

#### Nexus側

```bash
# Nexusの秘密鍵と公開鍵を設定
wrangler secret put NEXUS_PRIVATE_KEY
wrangler secret put NEXUS_PUBLIC_KEY

# オプション: 署名アルゴリズムを指定 (デフォルト: ed25519)
wrangler secret put NEXUS_SIGNATURE_ALGORITHM
```

または`wrangler.toml`:

```toml
[vars]
NEXUS_SIGNATURE_ALGORITHM = "ed25519"  # optional

# Secretsはwrangler secretコマンドで設定
# NEXUS_PRIVATE_KEY
# NEXUS_PUBLIC_KEY
```

#### Service Worker側

```bash
# Service Workerの秘密鍵を設定
wrangler secret put SERVICE_WORKER_PRIVATE_KEY

# Service Workerの公開鍵はマニフェストに含める
# Nexusの公開鍵は/.well-known/nexus-public-keyエンドポイントから取得
```

### Nexus公開鍵の取得

Service WorkerはNexusの公開鍵を以下のエンドポイントから取得できます:

```bash
curl https://your-nexus.workers.dev/.well-known/nexus-public-key
```

レスポンス:

```json
{
  "algorithm": "ed25519",
  "public_key": "Vx1234567890abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ=="
}
```

### マニフェストへの公開鍵の追加

Service Workerの公開鍵は、Nexusに登録するマニフェストに含める必要があります:

```typescript
import { createManifest } from "@hiyocord/hiyocord-nexus-core";

const manifest = createManifest({
  id: "my-service",
  name: "My Service",
  baseUrl: "https://my-service.workers.dev",
  description: "My Discord bot service",
  signatureAlgorithm: "ed25519",  // 署名アルゴリズム
  publicKey: "SERVICE_WORKER_PUBLIC_KEY_HERE",  // Service Workerの公開鍵
  commands: [],
});
```

### セキュリティの特徴

#### 1. 非対称暗号化

- 秘密鍵は決して共有されない
- 各サービスが独立した鍵ペアを持つ
- 1つのサービスの侵害が他に波及しない

#### 2. リプレイ攻撃の防止

タイムスタンプによるリプレイ攻撃の防止:

```typescript
// 60秒以内のリクエストのみ受け付ける
const now = Date.now();
const requestTime = parseInt(timestamp, 10);
if (Math.abs(now - requestTime) > 60_000) {
  return false;
}
```

#### 3. ヘッダーの正規化

署名時にヘッダーを正規化することで、改ざんを防止:

```typescript
// - 小文字化
// - アルファベット順ソート
// - CF-*ヘッダーを除外
// - 署名関連ヘッダーを除外
const canonicalHeaders = canonicalizeHeaders(headers);
```

### トラブルシューティング

#### 署名検証エラー

**症状**: `Invalid signature` エラー

**原因**:

- 秘密鍵と公開鍵が一致していない
- タイムスタンプが60秒以上古い
- ヘッダーまたはボディが改ざんされている

**解決方法**:

1. 鍵ペアの確認:

```bash
# 秘密鍵から公開鍵を導出して確認
# (Node.jsスクリプトで検証)
```

2. タイムスタンプの確認:

```typescript
console.log("Current time:", Date.now());
console.log("Request time:", timestamp);
```

3. ネットワーク遅延の確認:

```bash
# サーバー間の時刻同期を確認
```

#### マニフェスト公開鍵エラー

**症状**: `Service worker public key not configured`

**原因**:

- マニフェストに`public_key`フィールドがない
- マニフェストに`signature_algorithm`フィールドがない

**解決方法**:

マニフェストを再作成して登録:

```typescript
const manifest = createManifest({
  // ... 他のフィールド
  signatureAlgorithm: "ed25519",
  publicKey: "YOUR_PUBLIC_KEY_HERE",
});

await fetch("https://nexus.hiyocord.org/manifest", {
  method: "POST",
  headers: { "Content-Type": "application/json" },
  body: JSON.stringify(manifest),
});
```

#### 環境変数エラー

**症状**: `NEXUS_PRIVATE_KEY is not configured`

**原因**:

- 環境変数が設定されていない

**解決方法**:

```bash
wrangler secret put NEXUS_PRIVATE_KEY
# 秘密鍵を入力
```

### Migration from HMAC

以前のHMAC-SHA256認証からの移行:

#### 変更点

1. **環境変数**:
   - 削除: `HIYOCORD_SECRET`
   - 追加: `NEXUS_PRIVATE_KEY`, `NEXUS_PUBLIC_KEY`

2. **マニフェスト**:
   - 追加フィールド: `signature_algorithm`, `public_key`

3. **Service Worker側のコード**:

Before (HMAC):

```typescript
import { nexusVerifyMiddleware } from "@hiyocord/hiyocord-nexus-core";

app.post("/interactions", nexusVerifyMiddleware, handler);
```

After (Ed25519) - 変更なし:

```typescript
import { nexusVerifyMiddleware } from "@hiyocord/hiyocord-nexus-core";

// 内部実装が変わったが、APIは同じ
app.post("/interactions", nexusVerifyMiddleware, handler);
```

4. **Discord API Proxy呼び出し**:

Before (HMAC):

```typescript
// ヘッダーなし
await fetch("https://nexus/proxy/discord/api/v10/...", {
  method: "POST",
  body: data,
});
```

After (Ed25519):

```typescript
import { signDiscordApiProxyRequest } from "@hiyocord/hiyocord-nexus-core";

const signedHeaders = await signDiscordApiProxyRequest(
  algorithm,
  privateKey,
  manifestId,
  headers,
  body
);

await fetch("https://nexus/proxy/discord/api/v10/...", {
  method: "POST",
  headers: signedHeaders,
  body: data,
});
```

## 参考資料

- [Ed25519署名アルゴリズム](https://ed25519.cr.yp.to/)
- [Web Crypto API](https://developer.mozilla.org/en-US/docs/Web/API/Web_Crypto_API)
- [RFC 8032: Edwards-Curve Digital Signature Algorithm (EdDSA)](https://datatracker.ietf.org/doc/html/rfc8032)
