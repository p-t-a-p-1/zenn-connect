---
title: "Antigravity + Cloudflare Workersで爆速無料開発、 ~ パタパタ時計風画像生成サービス ~"
emoji: "💨"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["Antigravity", "Claude", "ReactRouter", "CloudflareWorkers", "Wasm"]
published: false
---

:::message
この記事は [Mavs Advent Calendar 2025](https://adventar.org/calendars/11595) の 16日目の記事です。🎅
:::

# はじめに

今回は、Antigravityを活用して、レトロな「パタパタ時計」風のメッセージボードアプリを爆速で開発した過程を共有します！！

実際のデプロイ周りは **Cloudflare Workers** を採用し、**Wasm (WebAssembly)** を用いた動的OGP生成なども行っています...！

# 作ったもの「Patapata Board」
パタパタ表示機（Split-Flap Display）の挙動をWeb上で再現したクリエイティブツールです。

[https://patapata-board.ptap1.workers.dev/](https://patapata-board.ptap1.workers.dev/)

:::message
**主な機能**

* **高精度なアニメーション**: CSS 3D Transformを駆使し、物理的なカードの回転をリアルに再現。
* **URL連動 & OGP生成**: 入力したメッセージがURLに保存され、SNSシェア時にその内容が「パタパタボードの画像」として自動生成されます。
* **デスクトップ時計**: 画面右下に現在時刻を表示するウィジェットを搭載。
:::


**簡易的なものです！！**


# 技術構成

Antigravity上で、以下のスタックを用いて開発しました。

| Category | Tech Stack | Note |
| --- | --- | --- |
| **Framework** | **React Router v7** | Framework Mode / SSR Enabled |
| **Platform** | **Cloudflare Workers** | エッジでのSSRとAPI処理 |
| **Language** | TypeScript |  |
| **Styling** | Tailwind CSS v4 |  |
| **OG Image** | **Satori + resvg (Wasm)** | `workers-og` を使用 |
| **IDE** | **Google Antigravity** | Agentによる自律開発 |
| **AI Model** | **Claude Opus 4.5** | Antigravity内で使用 |

## 技術的なこだわりポイント

### 1. Cloudflare Workers × React Router v7

デプロイ先には、コールドスタートがほぼゼロで、世界中のエッジで動作する **Cloudflare Workers** を選択しました。
React Router v7 は Cloudflare Workers アダプターを標準でサポートしており、`wrangler.json` の設定だけで簡単にSSR環境が構築できます。

```json:wrangler.json
{
  "name": "patapata-board",
  "main": "./workers/app.tsx",
  "compatibility_date": "2024-12-13",
  "compatibility_flags": [
    "nodejs_compat"
  ],
  // ...
}

```

### 2. Wasmによる動的OGP生成 (Edge Side)

本アプリの目玉機能である「シェア画像の動的生成」には、**WebAssembly (Wasm)** を活用しています。

通常、OGP画像の生成には `canvas` や `puppeteer` が使われますが、これらはWorkersのような軽量なエッジ環境では動作させるのが困難です。そこで今回は、以下の構成で実装しました。

* **Satori**: HTML/CSS (JSX) を SVG に変換するライブラリ。
* **resvg-wasm**: Rust製のSVGレンダリングエンジン `resvg` のWasm版。SVGをPNGに高速変換。
* **workers-og**: これらをCloudflare Workers上で簡単に扱えるようにするラッパー。

`package.json` にある通り、これらのWasm関連ライブラリを組み合わせることで、**サーバーレスかつ爆速なOGP生成** を実現しています。

```json:package.json
  "dependencies": {
    "@cf-wasm/resvg": "^0.3.3",
    "@resvg/resvg-wasm": "^2.6.2",
    "satori": "^0.18.3",
    "workers-og": "^0.0.27",
    // ...
  }

```

:::details 実装のイメージ（概念コード）
ユーザーがシェア用URLにアクセスすると、Workersがリクエストを受け取り、クエリパラメータのテキストを `Satori` でレイアウト解析、`resvg-wasm` でPNG化して返却します。これにより、どんなテキストでも瞬時にパタパタボード風の画像を生成できます。
:::

# 実際の開発フロー

実際にどのように開発を進めたかを紹介します。

## 1. Geminiと対話して要件を詰める

まず、Antigravityのチャット機能で作りたいアプリのイメージを伝えます。

```
あなたと対話しながら「パタパタ表示機」のWebアプリを作りたいです。
見た目は空港の案内板のようなレトロな感じで
```

対話形式で要件を詰めていきます。作りたいもののビジュアルがイメージできるのであればツールの"Canvas"機能で作るのが良いです。
納得のいくものができた段階で、以下の指示を行います。

```
これまでのやり取りを踏まえPRD（仕様書）を出力してください
```

これで包括的で詳細なPRD（仕様書）が生成されます。

## 2. Agentによる自律実装

ここがAntigravityの真骨頂です。生成されたPRDを元に、実装をAgentに任せます。

> **Instruction**: `docs/prd.md` に基づいて、React Router v7 と Cloudflare Workers のプロジェクトを初期化し、メインのボードコンポーネントを実装してください。

Agent Managerがタスクを分解し、「プロジェクト作成」「コンポーネント実装」「スタイリング」を次々と実行していきます。今回採用した **Claude Opus 4.5** はコーディング能力が非常に高く、複雑なCSS 3D Transformのアニメーションも一発で動作するレベルで出力してくれました。


### Agentによるコミット作成

Claude CodeやCursorにはスラッシュコマンド機能があると思いますが、
AntigravityにはWorkflowという機能で同等のことが可能です。


## 3. Browser Agentによる動作確認とキャプチャ

Antigravityには **Browser Agent** が搭載されており、AIが実際にChromeブラウザを操作して動作確認を行います。

> **Instruction**: アニメーションが正常か確認したいので、実際にブラウザで表示してキャプチャを撮ってください。

すると、Agentはバックグラウンドで：

1. `npm run dev` でローカルサーバーを起動
2. ブラウザでアクセス
3. **スクリーンショットを撮影してアーティファクトとして保存**

という手順を自動で行います。人間が手動でブラウザを開く必要すらなく、開発体験として非常に未来的でした。


# Antigravityの良さ

以下の良さは、他IDEでも設定すれば同じことができると思いますが、  
Antigravityでは特に設定なく無料ですぐ使えるのが良かったです。

## 無料で高機能（Plan + Task）

これくらいの規模の簡易開発であれば十分だと思いました。

## ブラウザサブエージェントが優秀

https://codelabs.developers.google.com/getting-started-google-antigravity?hl=ja#3

## Agentが画像生成


## Google Workspace用のプランも今後用意される（今はまだない）

AntigravityのPlanを眺めていると以下の記述がありました。

```
Your Plan: Free
Paid plans for Google Workspace accounts are coming soon. You can use a personal account with a Google AI plan to receive higher rate limits.

現在のプラン: 無料
Google Workspaceアカウント向けの有料プランは近日公開予定です。より高いレート制限を利用するには、Google AIプランをご契約の個人アカウントをご利用ください。
```

Google Workspaceアカウントと紐づくことができれば、会社標準の開発ツールになるかもしれません...！！

# まとめ
