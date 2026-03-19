# サーバーサイドGTM（sGTM）導入ガイド：Stape + GA4（Custom Loader対応版）

このドキュメントでは、Stape.io を利用してサーバーサイドGTMを構築し、Custom Loaderを使用して計測精度を最大化するための設定手順をまとめています。

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
   - **Same Origin Path** (任意): `metrics` や `lib` など、サイトの一部に見えるパスを入力（高度な設定用）。
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
通常、GTMのスクリプトは `googletagmanager.com/gtm.js` から読み込まれますが、これを**自分のサブドメイン（例: sgtm.example.com/xxxx.js）から読み込ませる機能**です。

### 主なメリット
* **広告ブロック（AdBlock）回避**: 通信先が自社ドメインになるため、広告ブロックツールによってGTM自体が遮断されるリスクを大幅に軽減します。
* **ITP対策（Cookie寿命の延長）**: スクリプトの配信元とデータの送信先が同一ドメイン（1st Party）になることで、SafariなどのブラウザによるCookie制限を受けにくくなります。
* **検知回避**: GTM特有のファイル名やパスを隠蔽できるため、より確実にデータをサーバーへ届けられます。

---

## ITP対策：同一ドメイン構成（HubSpot + Cloudflare + Stape）

Custom Loaderはサブドメイン（例: `sgtm.example.com`）から配信するが、SafariのITPはサブドメインも別オリジンとして扱うため、トラッカー判定された場合にCookieが制限される可能性がある。メインドメインのパス（例: `example.com/sgtm/`）経由でStapeへルーティングすることで、完全な1st Party扱いになりITPの制限を受けなくなる。

HubSpotはサーバー直接操作ができないため、**Cloudflare Workers** を使ったリバースプロキシで実現する。

### 構成イメージ

```
ブラウザ
  ↓
Cloudflare（DNS + Worker）
  ├── /sgtm/* → Stape sGTM（Worker でプロキシ）
  └── それ以外     → HubSpot（既存の動作はそのまま）
```

### 前提・コスト

| 項目 | 内容 |
|------|------|
| Cloudflareプラン | **無料**で可 |
| HubSpotの変更 | GTMコードの差し替えのみ |
| お名前.comの変更 | ネームサーバーの書き換えのみ（ドメイン移管は不要） |

### 手順

#### ① Cloudflare登録・DNS移行

1. [cloudflare.com](https://cloudflare.com) で無料アカウントを作成。
2. 「Add a Site」でドメインを入力 → 既存DNSレコードが**自動インポート**される。
3. インポート結果を確認し、HubSpot用のCNAMEレコード（`*.hs-sites.com` 向け）が含まれているかチェック。メール（MXレコード）の漏れにも注意。
4. お名前.com Naviにログイン →「ドメイン設定」→「ネームサーバーの設定」→「その他のネームサーバーを使う」→ Cloudflareから指定された2つのネームサーバーを入力。

#### ② SSL設定（重要）

- Cloudflare管理画面 → SSL/TLS → **「Full」** を選択。
- 「Flexible」にすると無限リダイレクトが発生してサイトが壊れるため必ず確認。

#### ③ Cloudflare Worker作成

1. Cloudflare管理画面 →「Workers & Pages」→「Create Worker」。
2. 以下のコードを貼り付ける（`your-container.stape.io` は実際のStapeコンテナURLに差し替え）：

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

- 「Workers & Pages」→ 該当Worker →「Settings」→「Triggers」→「Add Custom Domain」または「Add Route」。
- ルートに `example.com/sgtm/*` を指定。

#### ⑤ Stape・GTM側の変更

- Stape Custom Loader設定の **「Same Origin Path」** に `/sgtm` を入力（手順2の設定を更新）。
- Webコンテナの「Googleタグ」→ `server_container_url` の値を `https://example.com/sgtm` に変更。

---

## 疎通確認チェックリスト
- [ ] **ブラウザコンソール**: `gtm.js` の読み込み元が自社サブドメインになっているか。
- [ ] **Networkタブ**: `collect?` リクエストの Status が `200` または `204` で、送信先がカスタムドメインか。
- [ ] **サーバープレビュー**: 左側リストにイベントが届き、「Tags Fired」にGA4タグがあるか。
- [ ] **Healthチェック**: `https://[カスタムドメイン]/health` にアクセスして `ok` と表示されるか。