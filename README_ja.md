# サーバーサイドGTM（sGTM）導入ガイド：Stape.io 対応版

Stape.io を利用してサーバーサイドGTMを構築し、Custom Loaderで計測精度を最大化するための手順をまとめています。

## 関連ドキュメント

| ドキュメント | 内容 |
|-------------|------|
| [sgtm-setup.md](sgtm-setup.md) | HubSpot + Cloudflare Workers 構成を含む詳細セットアップガイド |
| [sgtm-worker-issue.md](sgtm-worker-issue.md) | 調査報告：HubSpot（Cloudflare for SaaS）環境でWorkerルーティングが機能しない問題 |
| [cloudflare-nameserver-note.md](cloudflare-nameserver-note.md) | Cloudflareネームサーバーが必要な理由とStapeドメイン設定の注意点 |

---

## 目次

1. [サーバーコンテナの準備（GTM）](#1-サーバーコンテナの準備gtm)
2. [インフラ構築とCustom Loader設定（Stape.io）](#2-インフラ構築とcustom-loader設定stapeio)
3. [Webサイト側の実装変更](#3-webサイト側の実装変更)
4. [サーバーコンテナ側の設定](#4-サーバーコンテナ側の設定)
5. [Custom Loaderとは？](#custom-loaderとは)
6. [ITP対策：同一ドメイン構成](#itp対策同一ドメイン構成)
7. [疎通確認チェックリスト](#疎通確認チェックリスト)

---

## 1. サーバーコンテナの準備（GTM）

1. **コンテナ作成**: GTMで「新規コンテナ」を作成し、ターゲットプラットフォームで **「サーバー」** を選択。
2. **手動設定**: 「タグ付けサーバーを手動で設定する」を選択し、表示された **「コンテナ設定コード」** をコピー。

## 2. インフラ構築とCustom Loader設定（Stape.io）

1. **コンテナ登録**: Stapeにログインし、コピーしたコードを貼り付けてコンテナを作成。
2. **カスタムドメイン設定**:
   - `Add Custom Domain` でサブドメイン（例: `sgtm.example.com`）を登録。
   - DNSレコード（A/CNAME）をドメイン管理サービスに登録。
3. **Custom Loaderの有効化**:
   - Stapeの「Power-ups」タブから **「Custom Loader」** を選択して有効化。
   - **Same Origin Path**（任意）: `metrics` や `lib` など、サイトの一部に見えるパスを入力（高度な設定用）。
   - 生成された **新しいインストール用コード** をコピーしておく。

## 3. Webサイト側の実装変更

1. **GTMコードの差し替え**: サイトに埋め込んでいる標準のGTMコードを、手順2でStapeから取得した **Custom Loader用のコード** に差し替える。
2. **Googleタグの修正**:
   - Webコンテナ内の「Googleタグ」を開く。
   - 設定パラメータに `server_container_url` を追加。
   - 値に **StapeのカスタムドメインURL**（`https://sgtm.example.com`）を入力。

## 4. サーバーコンテナ側の設定

1. **コンテナURLの登録**: 「管理」＞「コンテナ設定」で、カスタムドメインURLを登録。
2. **動的変数の作成**: 「変数」＞「イベントデータ」でパスを `measurement_id` とし、変数を作成。
3. **トリガー作成**: 「トリガー」＞「カスタム」で「すべてのイベント」を選択した `All Events` を作成。
4. **GA4サーバータグ作成**: タグの種類「GA4」を選択し、測定IDに作成した変数を、トリガーに `All Events` を設定。

---

## Custom Loaderとは？

通常、GTMのスクリプトは `googletagmanager.com/gtm.js` から読み込まれますが、これを**自分のサブドメイン（例: `sgtm.example.com/xxxx.js`）から読み込ませる機能**です。

### 主なメリット

- **広告ブロック（AdBlock）回避**: 通信先が自社ドメインになるため、広告ブロックツールによってGTM自体が遮断されるリスクを大幅に軽減します。
- **ITP対策（Cookie寿命の延長）**: スクリプトの配信元とデータの送信先が同一の1st Partyドメインになることで、SafariなどのブラウザによるCookie制限を受けにくくなります。
- **検知回避**: GTM特有のファイル名やパスを隠蔽できるため、より確実にデータをサーバーへ届けられます。

---

## ITP対策：同一ドメイン構成

### サブドメインだけでは不十分な理由

標準的なStape構成では、sGTMサーバーはサブドメイン（例: `sgtm.example.com`）上に置かれます。SafariのITPはサブドメインも別オリジンとして扱うため、そのサブドメインがトラッカーとして判定された場合、Cookie制限が適用される可能性があります。

```
ブラウザ → www.example.com      (メインサイト)
         → sgtm.example.com     (Stape sGTM) ← 別オリジン扱い
```

### 真の1st Party化：メインドメインのパス経由でルーティング

sGTMへのトラフィックを**メインドメインのパス**経由でルーティングすることで、ブラウザはすべてのリクエストを同一オリジンとみなし、完全な1st Party扱いのCookieが発行されます。

```
ブラウザ → www.example.com/analytics/collect  ──proxy──→  Stape sGTM
```

### 実装方法の比較

| 方法 | 難易度 | 条件 |
|------|--------|------|
| **Cloudflare Workers**（推奨） | ★☆☆ | ドメインがCloudflare上にある |
| **Nginx/Apache リバースプロキシ** | ★★★ | サーバーを直接管理できる |
| **その他CDNのリバースプロキシ** | ★★☆ | CloudFront・Vercel・Netlifyなど |

> HubSpotなどのSaaSホスティングでサーバーを直接操作できない場合、Cloudflare Workersが現実的な唯一の選択肢です。

### 前提・コスト

| 項目 | 内容 |
|------|------|
| Cloudflareプラン | **無料**で可 |
| サイト側の変更 | GTMコードの差し替えのみ |
| DNSレジストラの変更 | ネームサーバーの書き換えのみ（ドメイン移管は不要） |

### 手順：Cloudflare Workersを使った設定

#### ① Cloudflare登録・DNS移行

1. [cloudflare.com](https://cloudflare.com) で無料アカウントを作成。
2. 「Add a Site」でドメインを入力 → 既存DNSレコードが**自動インポート**される。
3. インポート結果を確認し、ホスティング先へのCNAMEレコードおよびメール用MXレコードの漏れがないかチェック。
4. ドメイン管理会社（お名前.comなど）のネームサーバー設定画面で、Cloudflareから指定された2つのネームサーバーに変更する。

#### ② SSL設定（重要）

- Cloudflare管理画面 → SSL/TLS → **「Full」** を選択。
- 「Flexible」にすると無限リダイレクトが発生してサイトが壊れるため必ず確認。

#### ③ Cloudflare Worker作成

1. 「Workers & Pages」→「Create Worker」。
2. デフォルトのコードを以下に差し替える（`your-container.stape.io` は実際のStapeコンテナURLに変更）：

```javascript
export default {
  async fetch(request) {
    const url = new URL(request.url);
    url.hostname = "your-container.stape.io";
    return fetch(new Request(url, request));
  }
};
```

3. Workerを保存・デプロイ。

#### ④ Workerのルーティング設定

- 該当Worker →「Settings」→「Triggers」→「Add Route」。
- ルートに `www.example.com/analytics/*` を指定。

#### ⑤ Stape・GTM側の変更

- Stape Custom Loader設定の **「Same Origin Path」** に `analytics` を入力。
- WebコンテナのGoogleタグ内 `server_container_url` を `https://www.example.com/analytics` に更新。

対応プラットフォーム：**Cloudflare**、Nginx、Apache、AWS CloudFront、Azure Front Door、Vercel、Netlifyなど。
詳細は [Stape Same Originドキュメント](https://stape.io/helpdesk/documentation/guidelines/how-to-use-same-origin) を参照。

---

## 疎通確認チェックリスト

- [ ] **ブラウザコンソール**: `gtm.js` の読み込み元がカスタムドメイン/パス（`googletagmanager.com` ではない）になっているか。
- [ ] **Networkタブ**: `collect?` リクエストのStatusが `200` または `204` で、送信先がカスタムドメイン/パスか。
- [ ] **サーバープレビュー**: 左側リストにイベントが届き、「Tags Fired」にGA4タグがあるか。
- [ ] **Healthチェック**: `https://[カスタムドメイン]/health` にアクセスして `ok` と表示されるか。
