# NatuEveについて
## 読み方：なちゅいべ

## プロダクトビジョン
生態系を守り、未来へ繋ぐ

## なちゅいべ概要
メイン目的：

生物のイベントを一元管理するサイト。生物のイベントが集まる場所。
生物のイベントを推進することで、保全活動を支援する。
関心を上げ、イベント参加のハードルを下げる。
サブ目的：イベントの結果を収集し、分析に役立てる。
目的外： 外来種対策


# 想定デバイス

- Web

- モバイル
    - Android
    - iPhone
- タブレット
    - Android
    - iPad

# 対象リポジトリ

## 共通バックエンド
https://github.com/GokujyouKaisennDonnburi/NatuIve_API

## Webフロントエンド
https://github.com/GokujyouKaisennDonnburi/NatuIve_Web_Frontend

## モバイル(iOS,Android共通)
https://github.com/GokujyouKaisennDonnburi/NatuIve_Mobile


# 構成図（初期時点）
```mermaid
graph TB
    subgraph clients["クライアント層"]
        web["Webフロントエンド<br/>Next.js / TypeScript<br/>(Node + Bun)"]
        ios["iOSアプリ<br/>SwiftUI (UI個別実装)"]
        android["Androidアプリ<br/>Jetpack Compose (UI個別実装)"]
        kmp["KMP共通処理<br/>Ktor Client + モデル/バリデーション<br/>(ネットワーク層を共有)"]
    end

    subgraph shared["共通フロント資産"]
        authui["共通ログインUIパッケージ<br/>(Web向け・複数プロダクトで共有)"]
    end

    subgraph backend["バックエンド層 (自前実装)"]
        api["APIサーバー<br/>Go (Gin/Echo等)<br/>JWT検証・CRUD・検索"]
        db[("PostgreSQL<br/>イベント/ユーザー/レポート")]
    end

    subgraph auth["認証基盤 (マネージド)"]
        supabase["Supabase Auth<br/>ログイン・トークン発行<br/>(複数プロダクト共有・無料枠)"]
    end

    subgraph schema["型・スキーマ共有"]
        openapi["OpenAPI定義<br/>→ TS型 / Kotlin型 / Go型を生成"]
    end

    ios --> kmp
    android --> kmp
    web -.依存.-> authui

    web -->|HTTP/JSON + JWT| api
    kmp -->|HTTP/JSON + JWT| api

    web -->|ログイン| supabase
    kmp -->|ログイン| supabase
    authui -.呼び出し.-> supabase

    api -->|JWT検証 JWKS| supabase
    api --> db

    openapi -.型生成.-> web
    openapi -.型生成.-> kmp
    openapi -.型生成.-> api

    subgraph deploy_now["デプロイ：今年"]
        xserver["Xserver VPS<br/>API + PostgreSQL<br/>(Docker Compose)<br/>固定費 月約1,000円"]
    end

    subgraph deploy_future["デプロイ：1年後に移行予定"]
        cf["Cloudflare<br/>Pages (フロント配信)"]
        aws["AWS<br/>App Runner/ECS (API)<br/>RDS/Aurora (DB)"]
    end

    api -.今年.-> xserver
    api -.1年後.-> aws
    web -.1年後.-> cf

    classDef confirmed fill:#2d5016,stroke:#1a3009,color:#fff
    classDef managed fill:#1e4d5c,stroke:#0d2b35,color:#fff
    classDef future fill:#4a4a4a,stroke:#2a2a2a,color:#fff,stroke-dasharray: 5 5
    classDef datastore fill:#5c4a1e,stroke:#352b0d,color:#fff

    class web,ios,android,kmp,api,authui confirmed
    class supabase,openapi managed
    class db,xserver datastore
    class cf,aws future
```
