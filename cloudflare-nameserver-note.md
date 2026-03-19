# Cloudflareネームサーバーに置き換える理由

## ネームサーバーの役割

ネームサーバーは「このドメインのDNS情報はどこにあるか」を指す設定。

- **変更前**: `example.com` のDNS → ドメイン管理会社が管理
- **変更後**: `example.com` のDNS → **Cloudflareが管理**

## なぜCloudflareに管理を移す必要があるのか

ITP対策で `example.com/sgtm/*` へのアクセスをCloudflare Workerで処理する必要がある。これには以下のCloudflare機能を使う：

- **Cloudflare Worker**（リバースプロキシ）
- **Workerルーティング**（特定パスだけWorkerに振り分け）

これらは**Cloudflareがそのドメインを管理している場合のみ**利用できる。ドメイン管理会社のネームサーバーのままでは、Cloudflareの機能を `example.com` に対して使えない。

## 比較

| | ドメイン管理会社 NS | Cloudflare NS |
|---|---|---|
| 通常のDNS管理 | できる | できる |
| Cloudflare Worker | **使えない** | 使える |
| CDN・SSL設定 | **使えない** | 使える |

## 補足

ネームサーバーを変えても、ドメイン管理会社でのドメイン契約・所有権はそのまま。変わるのは「DNS情報の問い合わせ先」だけ。

---

# Stape sgtm.example.com ドメイン設定

## sgtm.example.com は削除してはいけない

Cloudflare Worker経由のITP対策構成で、Workerの転送先として必要。

```
ブラウザ → example.com/sgtm/* → Cloudflare Worker → sgtm.example.com → Stape
```

## Stape Custom Loader設定

- **Domain**: `sgtm.example.com`（`example.com` は選択不可）
- **Same Origin Path**: `/sgtm`
- ブラウザからはメインドメイン（example.com）経由でアクセスされるためITP対策は有効
