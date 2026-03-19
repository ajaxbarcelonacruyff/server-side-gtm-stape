# GA4計測をサーバーサイドへ。見えていなかったデータを、取り戻す。

[English version](README.md)

Stape.io と Custom Loader を使ったサーバーサイドGTM（sGTM）の導入ガイドです。セットアップ手順だけでなく、HubSpot + Cloudflare 環境で直面した制約と、実環境テストに基づく推奨構成を記録しています。

---

## 目次

1. [構築するもの](#1-構築するもの)
2. [アーキテクチャ概要](#2-アーキテクチャ概要)
3. [なぜこの構成か？](#3-なぜこの構成か)
4. [前提条件](#4-前提条件)
5. [セットアップ手順](#5-セットアップ手順)
6. [疎通確認チェックリスト](#6-疎通確認チェックリスト)
7. [既知の制約](#7-既知の制約)
8. [関連ドキュメント](#8-関連ドキュメント)

---

## 1. 構築するもの

**対象読者:** HubSpot（またはその他SaaSホスティング）上でGA4を運用しており、広告ブロッカーやブラウザのプライバシー制限によるデータ欠損を疑っているチーム。

**得られるもの:**

- GTMスクリプトを `googletagmanager.com` ではなく自社ドメインから配信
- GA4イベントをサーバーコンテナ経由で送信し、クライアントサイドのブロックを回避
- 1st Party Cookieの寿命を延長し、Safari ITPの影響を軽減
- 実際のデータロスを評価するための計測ベースライン

サイトの移行やホスティングの変更は不要です。サイト側の変更はGTMコードの差し替えのみです。

---

## 2. アーキテクチャ概要

### 標準的なStape構成（サブドメイン）

```
ブラウザ
  │
  ├── www.example.com                → HubSpot（変更なし）
  └── sgtm.example.com              → Cloudflare DNS（CNAME）
        └── your-container.stape.io  → Stape sGTMサーバーコンテナ
              ├── /gtm.js           ← Custom Loader: 自社サブドメインからスクリプト配信
              └── /collect          ← GA4イベントをGoogleへ転送
```

**コンポーネント:**

| コンポーネント | 役割 |
|--------------|------|
| GTM Webコンテナ | ブラウザでタグを実行し、イベントをサーバーコンテナへルーティング |
| Stape.io | sGTMサーバーコンテナをホスト。Custom Loaderとカスタムドメインを管理 |
| Custom Loader | GTMスクリプトを `googletagmanager.com` ではなく自社サブドメインから配信 |
| sgtm.example.com | スクリプトとイベントの1st Partyエンドポイントとして機能するサブドメイン |
| GA4サーバータグ | サーバーサイドでイベントを受信し、Google Analyticsへ転送 |

### 検討した構成: パスベースの同一オリジン構成

```
ブラウザ
  └── example.com/sgtm/*  ──Cloudflare Worker──→  Stape sGTM
```

すべてのトラッキングリクエストが同一オリジンとなり、完全な1st Party Cookie扱いになります。HubSpot環境でこれが機能しなかった理由は[セクション7](#7-既知の制約)を参照してください。

---

## 3. なぜこの構成か？

### 課題

標準的なクライアントサイドGTMには、3つの問題が重なっています。

1. **広告ブロッカー** が `googletagmanager.com` や `google-analytics.com` へのリクエストを遮断し、イベントが無断で破棄される。
2. **Safari ITP** がJavaScriptで設定されたCookieの有効期限をトラッキング分類に応じて7日以下に制限し、リピーターのアトリビューションが途切れる。
3. **サードパーティスクリプトのブロック** — 明示的な広告ブロッカーがなくても、ブラウザがサードパーティスクリプトをデフォルトで制限する傾向が強まっている。

サーバーサイドGTMは計測ロジックをクライアントから分離します。スクリプトは自社ドメインから読み込み、イベントはサーバーで処理され、Cookieはサーバーサイドでより長い有効期限で設定できます。

### パスベース vs サブドメイン

| アプローチ | ITP保護 | 複雑さ | HubSpot対応 |
|----------|---------|-------|------------|
| サブドメイン (`sgtm.example.com`) | 部分的 | 低 | 可 |
| パスベース (`example.com/sgtm/*`) | 完全（同一オリジン） | 中 | **不可** * |

\* HubSpotのCloudflare for SaaSレイヤーがカスタムWorker Routeより優先されるため — [セクション7](#7-既知の制約)を参照。

パスベースのアプローチは理論上より強力です。Safariは `example.com/sgtm/` を `example.com` と同一オリジンとして扱うため、トラッカー分類が適用されません。ただし、メインドメイン上のリバースプロキシを経由するルーティングが必要です。

**HubSpotの制約:** HubSpotは内部でCloudflare for SaaSを利用しています。そのため、HubSpotのCloudflareレイヤーが自分のCloudflare Workerより先にリクエストを処理し、パスベースのWorkerルーティングはWorkerが発火せず404を返します。詳細な調査は [sgtm-worker-issue.md](sgtm-worker-issue.md) を参照してください。

### 推奨構成

サブドメイン構成（`sgtm.example.com`）で運用を開始してください。ITPのサブドメイン1st Party Cookieへの実影響は限定的です。Safariがトラッカーと判定した場合のみCookieが制限されます。まずデータ収集を開始し、GA4のデータで計測可能なアトリビューション損失が確認された場合のみ、パスベースルーティングを再検討してください。

---

## 4. 前提条件

| 要件 | 詳細 |
|-----|------|
| Google Tag Managerアカウント | Webコンテナがサイトにインストール済み |
| Stape.ioアカウント | 無料プランで開始可能 |
| カスタムドメイン | DNS管理権限があること（Cloudflareまたはレジストラ経由） |
| Cloudflareアカウント | 無料プラン。カスタムサブドメインのDNS管理に必要 |
| GA4プロパティ | 測定ID（形式: `G-XXXXXXXXXX`） |

> ドメインのDNSがまだCloudflareにない場合、レジストラでネームサーバーを変更する必要があります。これはドメインの所有権を移転するものではなく、DNS管理のみがCloudflareに移ります。詳細は [cloudflare-nameserver-note.md](cloudflare-nameserver-note.md) を参照。

---

## 5. セットアップ手順

詳細な手順は [sgtm-setup.md](sgtm-setup.md) を参照してください。以下は5ステップの要約です。

**ステップ1 — GTMサーバーコンテナの作成**
GTMで「新規コンテナ」を作成し、ターゲットプラットフォームに「サーバー」を選択。「タグ付けサーバーを手動で設定する」を選び、コンテナ設定コードをコピー。

**ステップ2 — Stape.ioでのサーバー構築**
コンテナ設定コードをStapeに貼り付けてサーバーコンテナを作成。カスタムドメイン（`sgtm.example.com`）を追加し、Power-upsから **Custom Loader** を有効化。必要に応じてSame Origin Pathを設定（例: `lib`）。Stapeが生成する新しいGTMインストールコードをコピー。

**ステップ3 — サイトの更新**
サイトの標準GTMスニペットをStapeのCustom Loaderスニペットに差し替え。GTM Webコンテナ内のGoogleタグに設定パラメータ `server_container_url` = `https://sgtm.example.com` を追加。

**ステップ4 — サーバーコンテナの設定**
サーバーコンテナ設定でカスタムドメインURLを登録。`measurement_id` のイベントデータ変数、All Eventsトリガー、GA4サーバータグを作成。

**ステップ5 — （オプション）パスベースルーティング用Cloudflare Worker**
HubSpot以外のサイトで完全な同一オリジン保護が必要な場合、`example.com/sgtm/*` をStapeコンテナにプロキシするCloudflare Workerを追加:

```javascript
export default {
  async fetch(request) {
    const url = new URL(request.url);
    url.hostname = "your-container.stape.io";
    return fetch(new Request(url, request));
  }
};
```

Workerルートを `example.com/sgtm/*` に設定し、StapeのSame Origin Pathを `/sgtm` に、GTMの `server_container_url` を `https://example.com/sgtm` に更新。

---

## 6. 疎通確認チェックリスト

各デプロイ手順の後に以下を確認し、エンドツーエンドで正常に動作していることを検証してください。

- [ ] **スクリプトソース**: DevTools > Networkで、`gtm.js` が `googletagmanager.com` ではなく `sgtm.example.com`（またはカスタムパス）から読み込まれていることを確認。
- [ ] **イベント送信**: `collect?` リクエストをフィルタリング。ステータスが `200` または `204` で、リクエスト先がカスタムドメイン/パスであること。
- [ ] **サーバープレビュー**: GTMサーバーコンテナのプレビューを開き、左側のイベントリストにイベントが表示され、「Tags Fired」にGA4タグが表示されていること。
- [ ] **Healthエンドポイント**: ブラウザで `https://sgtm.example.com/health` にアクセスし、`ok` と表示されること。
- [ ] **GA4 DebugView**: URLに `?gtm_debug=x` を付加し、GA4のDebugViewにリアルタイムでイベントが表示されることを確認。
- [ ] **Cookie確認**: DevTools > Application > Cookiesで、`_ga` および `_ga_XXXXXXXX` Cookieが存在し、自社ドメインに設定されていること。

---

## 7. 既知の制約

### パスベースルーティングはHubSpot環境で機能しない

`example.com/sgtm/*` をCloudflare Worker経由でStapeにルーティングし、完全な同一オリジンを実現することを目指しましたが、以下の理由で失敗しました。

- HubSpotは内部で **Cloudflare for SaaS** を利用している。
- HubSpotがホストするドメインでは、HubSpotのCloudflare for SaaSレイヤーが、同一Cloudflareネットワーク内のカスタムWorker Routeよりも優先される。
- レスポンスヘッダー `x-hs-cfworker-meta: {"resolver":"NotFoundResolver"}` が、HubSpotのエッジが404を返しWorkerが発火しなかったことを示している。

Cloudflare Workers / Route設定内での回避策は見つかりませんでした。これを機能させるには、HubSpot以外でサイトをホスティングする（例: Cloudflare Pages、Vercel、Netlify）必要があり、その場合ルーティングの完全な制御が可能になります。

DNS設定、Workerコード、テストしたルート、レスポンスヘッダー分析を含む完全な調査報告は [sgtm-worker-issue.md](sgtm-worker-issue.md) を参照してください。

### サブドメインのITPリスクは実在するが限定的

`sgtm.example.com` はサブドメインです。SafariのITPはトラッキング挙動を検出した場合、これをクロスサイトトラッカーとして分類する可能性があります。ただし実際には、同一登録ドメイン上の1st Partyサブドメインがこの判定を受けることは稀であり、影響はトラッキング同意のないSafariユーザーに限定されます。推奨アプローチは、サブドメイン構成で運用を開始し、GA4データを監視し、有意なアトリビューション損失が確認された場合にのみ対応を検討することです。

---

## 8. 関連ドキュメント

| ドキュメント | 内容 |
|-------------|------|
| [sgtm-setup.md](sgtm-setup.md) | 詳細セットアップガイド: GTMサーバーコンテナ、Stape Custom Loader、Cloudflare Workers（日本語） |
| [sgtm-worker-issue.md](sgtm-worker-issue.md) | 調査報告: HubSpotサイトでCloudflare Workerパスベースルーティングが機能しない理由（日本語） |
| [cloudflare-nameserver-note.md](cloudflare-nameserver-note.md) | Cloudflareネームサーバー委任が必要な理由とStapeドメイン設定の注意点（日本語） |
