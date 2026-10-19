# デプロイ構成（全体）

なちゅいべ全体の本番デプロイ構成をまとめる（2026-10 時点）。
各リポジトリの設定値・月額試算・未決事項は、それぞれの `docs/Deployment.md` を参照する。

| 対象 | デプロイ先 | 詳細 |
|---|---|---|
| バックエンド（API） | Google Cloud Run（東京）＋Cloud Scheduler | [NatuIve_API: docs/Deployment.md](https://github.com/GokujyouKaisennDonnburi/NatuIve_API/blob/main/docs/Deployment.md)・ADR-0032 |
| DB | Supabase の PostgreSQL（東京） | 同上 |
| 認証 | Supabase Auth（既存のプロジェクト） | 同上 |
| Web フロントエンド | Cloudflare Workers（Paid）＋OpenNext | [NatuIve_Web_Frontend: docs/Deployment.md](https://github.com/GokujyouKaisennDonnburi/NatuIve_Web_Frontend/blob/main/docs/Deployment.md)・ADR-0001 |

## 構成図

```mermaid
flowchart TB
    subgraph clients["利用者"]
        browser["Web ブラウザ"]
    end

    subgraph cloudflare["Cloudflare（Workers Paid はフロントと中継で共有）"]
        dns["DNS<br/>natuportal.org"]
        workers["Workers: Web フロント<br/>Next.js（OpenNext）<br/>SSR＋静的ファイル配信"]
        relay["Workers: natueve-api-relay<br/>API の中継"]
        r2[("R2<br/>画像・PDF")]
    end

    subgraph gcp["Google Cloud"]
        run["Cloud Run（asia-northeast1）<br/>API（Go・Gin）"]
        scheduler["Cloud Scheduler<br/>1 分ごと"]
        secrets["Secret Manager"]
        registry["Artifact Registry"]
    end

    subgraph supabase["Supabase"]
        auth["Supabase Auth<br/>既存のプロジェクト<br/>ログイン・JWT 発行"]
        db[("PostgreSQL<br/>ap-northeast-1・東京<br/>17 を予定（確認中）")]
    end

    resend["Resend<br/>メール送信"]
    github["GitHub<br/>ソースコード"]

    browser -->|"ページの取得"| workers
    clients -->|"API 呼び出し（JWT 付き）"| relay
    clients -->|"ログイン"| auth
    browser -->|"アップロード（署名付き URL へ PUT）"| r2
    browser -->|"画像の表示（公開 URL）"| r2
    workers -.->|"SSR からの API 呼び出し（OGP・予定）"| relay
    relay -->|"秘密ヘッダーと利用者 IP を付けて転送（run.app の URL）"| run

    scheduler -->|"outbox の処理（実装予定）"| run
    run -->|"SQL"| db
    run -->|"JWT の検証（JWKS）"| auth
    run -->|"署名付き URL の発行・ファイルの確認や移動"| r2
    run -->|"通知メール"| resend
    secrets -.->|"秘密情報の注入（予定）"| run
    registry -.->|"コンテナイメージ（予定）"| run

    dns -.->|"サブドメインの割り当て（予定）"| workers
    dns -->|"natueve-api.natuportal.org"| relay

    github -.->|"自動ビルド・デプロイ（Workers Builds・予定）"| workers
    github -.->|"自動ビルド・デプロイ・マイグレーション（方式は未決）"| registry

    classDef confirmed fill:#2d5016,stroke:#1a3009,color:#fff
    classDef existing fill:#1e4d5c,stroke:#0d2b35,color:#fff
    classDef planned fill:#4a4a4a,stroke:#2a2a2a,color:#fff,stroke-dasharray: 5 5
    classDef datastore fill:#5c4a1e,stroke:#352b0d,color:#fff

    class workers,relay,run,scheduler confirmed
    class auth,resend,dns,browser,github existing
    class secrets,registry planned
    class db,r2 datastore
```

### 凡例

| 表記 | 意味 |
|---|---|
| 緑のノード | 今回決定した本番の実行基盤 |
| 青のノード | 既存のサービス・利用者 |
| 灰色・破線のノード | 使う予定のサービス（未導入） |
| 茶色のノード | データの保存先（新規・既存を問わない） |
| 実線の矢印 | 方式が決まっている通信（実装予定のものを含む） |
| 破線の矢印 | 未導入のサービスとの通信、または方式が未決の通信 |

## 主な流れ

- **ページの表示**: ブラウザは Cloudflare Workers からページを受け取る。Next.js が SSR で返し、JS・CSS などの静的ファイルはそのまま配信される。
- **データの取得・投稿**: ブラウザは、Cloud Run の API を JWT 付きで直接呼ぶ。API は Supabase の PostgreSQL を読み書きする。
- **ログイン**: ブラウザは Supabase Auth でログインし、JWT を受け取る。API は Supabase Auth の JWKS で JWT を検証する。
- **画像・PDF**: ブラウザは API から署名付き URL を受け取り（`POST /api/v1/uploads/presign`）、その URL へ直接 PUT してアップロードする。
  表示は R2 の公開 URL から直接行う。API は R2 上のファイルの確認・コピー・削除も行う。
- **通知メール**: API は送信予約（outbox）を DB に書く。Cloud Scheduler が 1 分ごとに API を呼んで送信処理を起動し（実装予定）、Resend で送る。


