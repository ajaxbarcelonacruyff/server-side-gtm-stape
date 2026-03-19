# sGTM 同一オリジン構成の調査報告

## 概要

ITP対策として `example.com/sgtm/` パス経由でsGTMへルーティングする構成を検討したが、HubSpotとCloudflareの仕様上の制約により実現できないことが判明した。

---

## 1. 背景・目的

### 現状構成

```
ブラウザ → sgtm.example.com → Stape sGTM
```

サブドメイン経由でsGTMにリクエストを送信している。

### 課題

SafariのITP（Intelligent Tracking Prevention）は、サブドメインを別オリジンとして扱い、トラッカー判定された場合にCookieを制限する可能性がある。

### 目指した構成

```
ブラウザ → example.com/sgtm/* → Cloudflare Worker → Stape sGTM
```

メインドメインのパス経由でsGTMにアクセスすることで、完全な1st Party扱いとしITPの制限を回避する。

---

## 2. 実施した設定

### Cloudflare DNS

| タイプ | 名前 | コンテンツ | プロキシ |
|--------|------|-----------|---------|
| A | example.com | xxx.xxx.xxx.xxx | プロキシ済み（オレンジ雲） |
| A | example.com | xxx.xxx.xxx.xx | プロキシ済み（オレンジ雲） |
| CNAME | sgtm | jp.stape.io | - |

### Cloudflare Worker

```javascript
export default {
  async fetch(request) {
    const url = new URL(request.url);
    url.hostname = "xxxxxxxx.jp.stape.io";
    return fetch(new Request(url, request));
  }
};
```

### Workers Route

| ルート | Worker |
|--------|--------|
| `example.com/sgtm/*` | sgtm |

---

## 3. 発生した問題

`https://example.com/sgtm/healthy` にアクセスすると 404 が返る。

### 確認結果

| 確認項目 | 結果 |
|----------|------|
| `xxxxxxxx.jp.stape.io/healthy` | OK |
| `xxxxxxxx.jp.stape.io/sgtm/healthy` | OK |
| `example.com/sgtm/healthy` | 404 |
| Cloudflare経由の確認（CF-RAYヘッダー） | あり（Cloudflareは経由している） |
| Worker ログ | リクエストの記録なし（Workerが発火していない） |

### レスポンスヘッダー（抜粋）

```
HTTP/1.1 404 Not Found
Server: cloudflare
CF-RAY: xxxxxxxxxxxxxxxx-XXX
x-hs-cfworker-meta: {"resolver":"NotFoundResolver"}
X-Hs-Reason: 404 predicted at edge
x-hs-portal-id: XXXXXXXX
```

---

## 4. 原因

### HubSpot の Cloudflare for SaaS

HubSpotは **Cloudflare for SaaS**（SaaS事業者向けのCloudflare機能）を利用している。

通常、Cloudflareゾーンに設定したWorker Routeはリクエストを優先的に処理する。しかし、オリジンサーバー（HubSpot）がCloudflare for SaaSを経由している場合、**HubSpot側のCloudflare処理がユーザー側のWorker Routeより優先される**。

レスポンスヘッダーの `x-hs-cfworker-meta: {"resolver":"NotFoundResolver"}` が、HubSpot側のCloudflare Workerがリクエストを先に処理し404を返していることを示している。

### 試行した対策と結果

| 対策 | 結果 |
|------|------|
| DNS Aレコードをプロキシ済み（オレンジ雲）に変更 | 効果なし |
| Workerを静的レスポンスに差し替えてテスト | Workerが発火せず効果なし |
| Worker Routeを削除→再作成 | 効果なし |
| Page Rules / Redirect Rules / Transform Rulesの確認 | 競合するルールなし |

いずれの対策でもWorkerは発火せず、HubSpot側の処理が優先された。

---

## 5. 結論

`example.com` のオリジンがHubSpot（Cloudflare for SaaS利用）である限り、パスベース（`/sgtm/*`）のCloudflare Workerルーティングは機能しない。

---

## 6. 今後の方針

現行のサブドメイン構成（`sgtm.example.com`）で sGTM を運用する。

```
ブラウザ → sgtm.example.com → Stape sGTM
```

### ITP の実影響について

- サブドメインでも1st Party Cookieとして扱われるケースが多く、実影響は限定的
- ITPが問題になるのはSafariがトラッカー判定した場合のみ
- まずこの構成でsGTMを稼働させ、GA4のデータからITPの影響を判断する

### パスベース構成を実現する場合の代替案

HubSpotを経由しない構成（例: Cloudflare Pages、Vercel等でサイトをホスティング）に変更すれば、Cloudflare Workerのパスベースルーティングが利用可能になる。ただし、サイトの移行コストが大きいため、ITPの実影響が確認されてから検討する。
