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

TODO：家の時計の写真を貼る

ふと↑のようなものをWebで表現したいと思い、Antigravityを活用して開発してみました。

実際のデプロイ周りは **Cloudflare Workers** を採用し、**Wasm (WebAssembly) を用いた動的OGP生成**なども行っています...！

## 話すこと

- 作ったものの紹介
- Antigravityでの爆速開発
- Cloudflare Workersでのデプロイ

# パタパタ時計風画像生成サービスについて

@[card](https://patapata-board.ptap1.workers.dev/)

## 主な機能

* **パタパタアニメーション**
  * CSS 3D Transformでのリアルな回転を再現しました！
* **動的OGP生成**
  * 入力したメッセージがURLに保存され、SNSシェア時にその内容が「パタパタボードの画像」として自動生成されます！
* **画像出力**
  * 入力したメッセージを画像として出力することができます！


gif動画を生成するようなものも作りたかったですが一旦保留にしてます...！


# 技術構成

Antigravity上で、以下のスタックを用いて開発しました。

| カテゴリ | 技術名 | 説明 |
| --- | --- | --- |
| **フレームワーク** | [**React Router v7**](https://reactrouter.com/) | Framework Mode / SSR Enabled |
| **プラットフォーム** | [**Cloudflare Workers**](https://www.cloudflare.com/ja-jp/developer-platform/products/workers/) | エッジでのSSRとAPI処理 |
| **言語** | TypeScript |  |
| **スタイリング** | Tailwind CSS v4 |  |
| **OG Image** | [**Satori**](https://github.com/vercel/satori) + resvg (Wasm)** | `workers-og` を使用 |
| **IDE** | [**Google Antigravity**](https://antigravity.google/) | Agentによる自律開発 |
| **AI Model** | **Claude Opus 4.5** & **Gemini 3.0 Pro** | Antigravity内で使用しているモデル |

## 技術的なこだわりポイント

### 1. Cloudflare Workers × React Router v7

デプロイ先には、**Cloudflare Workers** を選択しました。
また、React Router v7 は Cloudflare Workers アダプターを標準でサポートしており、
`wrangler.json` の設定だけで簡単にSSR環境が構築できます。

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

### 2. 動的OGP生成



当初はCloudflare Workers上でwasmで動的OGPを生成する方法を検証していたのですが
実装はできたのですが、リッチな表現をするとCPU制限がかかるため、一旦断念しました。

また、今回はCloudflare Pagesで
https://re-engines.com/2025/01/27/react-router-v7-ogp/


本アプリの目玉機能である「シェア画像の動的生成」には、**WebAssembly (Wasm)** を活用しています。


# 実際の開発フロー

実際にどのように開発を進めたかを紹介します！！


## 1. Geminiと対話して要件を詰める

まず、Antigravityのチャット機能で作りたいアプリのイメージを伝えます。

```
あなたと対話しながら「パタパタ表示機」のWebアプリを作りたいです。
見た目は空港の案内板のようなレトロな感じで
```

対話形式で要件を詰めていきます。
作りたいもののビジュアルがイメージできるのであればツールの"Canvas"機能で作るのが良いです。
納得のいくものができた段階で、以下の指示を行います。

```
これまでのやり取りを踏まえPRD（仕様書）を出力してください
```

これで包括的で詳細なPRD（仕様書）が生成されます。


実際に生成したPRDは以下です。

:::details Patapata Generator PRD
# Product Requirements Document (PRD): Patapata Board (RR7 Edition)

## 1. プロダクト概要

  * **プロダクト名**: Patapata Board (Single Row Generator)
  * **概要**: パタパタ表示機（Split-Flap Display）の挙動をWeb上で高精度にシミュレートするクリエイティブツール。
  * **コアバリュー**:
    1.  **動画素材作成**: 入力したメッセージのアニメーションを動画として書き出し、SNSで利用できる。
    2.  **動的シェア体験**: URLをSNSに貼るだけで、入力内容が反映された「ボードの静止画」がOGPとして自動生成され、クリック率を高める。
    3.  **デスクトップアクセサリ**: 画面右下に常時表示される「パタパタ時計」により、開いておきたくなる実用性を兼ね備える。

## 2. ターゲットユーザー

  * **SNSクリエイター**: X (Twitter) や Instagram のストーリーズ/投稿用に、レトロテックな演出素材を求める層。
  * **エンジニア・Webデザイナー**: 最新の Web技術（React Router v7, Edge Functions）のショーケースとして興味を持つ層。
  * **デスクワーカー**: 作業用BGMの代わりに、視覚的に心地よい「時計」としてブラウザを開いておきたい層。

## 3. 技術スタック (Tech Stack)

  * **Framework**: **React Router v7** (Framework Mode / SSR Enabled)
  * **Language**: TypeScript
  * **Styling**: Tailwind CSS
  * **Image Generation**: `@vercel/og` (Satori Engine)
  * **Deployment**: Vercel (Edge Functions)
  * **Icons**: Lucide React

## 4. 機能要件 (Functional Requirements)

### 4.1. メインボード (Main Display)

  * **表示仕様**: 1行 × 22文字（22ビット）。
  * **アニメーション**: CSS 3D Transform による物理的なカード回転動作。
  * **状態同期 (Two-way Binding)**:
      * URLクエリ (`?text=HELLO`) とボードの表示内容を同期。
      * ページロード時にURLパラメータを読み取り、アニメーションを自動再生する。

### 4.2. 時計ウィジェット (Clock Widget)

  * **配置**: 画面右下 (Bottom Right) に固定表示。
  * **表示内容**: 現在のローカル日時。
      * フォーマット: `MM-DD HH:MM` (例: `12-12 16:30`)
      * 合計: 11文字分を使用（余白含む）。
  * **動作仕様**:
      * 1分ごとに表示を更新し、変化した桁のみがパタパタと回転する。
      * **Client-side Only**: ユーザーのローカルタイムを表示するため、サーバーサイドレンダリング（SSR）は行わず、マウント後に表示する（Hydration Mismatch回避）。
  * **レスポンシブ**: モバイル画面（幅が狭い場合）では視認性を考慮し、非表示または下部中央へ配置変更する。

### 4.3. 動画書き出し (Video Export)

  * **機能**: ブラウザ標準の `getDisplayMedia` API を使用して画面収録を行う。
  * **自動化**: 「Export」ボタン押下時に、UI（入力欄や**時計ウィジェット**）を一時的に非表示にし、メインボードのみを録画する。

### 4.4. 動的OGP生成 (Server-Side / Resource Route)

  * **機能**: シェアされたURLのクエリパラメータに基づいて、ボードの静止画をサーバーサイドで動的生成する。
  * **仕様**:
      * 背景色: 黒 (\#111)。
      * レイアウト: 1行×22文字のグリッド。
      * テキスト: URLパラメータの `text` を反映。空のセルはブランク画像で埋める。

## 5. ルーティング & URL設計

React Router v7 のルーティングシステムを使用。

| URL パス | ファイル (例) | 役割 |
| :--- | :--- | :--- |
| `/` | `app/routes/_index.tsx` | メイン画面。ボード、入力UI、時計ウィジェットを表示。 |
| `/resource/og` | `app/routes/resource.og.tsx` | **Resource Route**。OGP画像を生成して返すAPIエンドポイント。 |

## 6. ロードマップ

1.  **Phase 1: コア機能実装**
      * React Router v7 プロジェクト作成。
      * メインのパタパタボードとアニメーション実装。
      * URL同期機能の実装。
2.  **Phase 2: サーバーサイド機能**
      * `@vercel/og` の導入。
      * Resource Route (`resource.og.tsx`) の実装とデザイン調整。
      * `meta` タグの動的出力設定。
3.  **Phase 3: ウィジェット & エクスポート**
      * `ClockWidget` の実装と配置。
      * 画面録画機能 (`getDisplayMedia`) の実装。
      * 録画時に時計やUIを隠す「Clean Mode」制御の実装。
:::

## 2. Agentによる自律実装

ここから実際にAntigravityを活用して開発します。

重要なポイントとして、**開発のための初期構築（インストールなど）**は人力で行った方が良いです。
リミット達するのを防ぐため、AIに任せなくても良い、かつハードル低めな初期構築作業は人間が行うのが望ましいです...！



> **Instruction**: `docs/prd.md` に基づいて、React Router v7 と Cloudflare Workers のプロジェクトを初期化し、メインのボードコンポーネントを実装してください。

Agent Managerがタスクを分解し、「プロジェクト作成」「コンポーネント実装」「スタイリング」を次々と実行していきます。今回採用した **Claude Opus 4.5** はコーディング能力が非常に高く、複雑なCSS 3D Transformのアニメーションも一発で動作するレベルで出力してくれました。


### Agentによるコミット作成

Claude CodeやCursorにはスラッシュコマンド機能があると思いますが、
AntigravityにはWorkflowという機能で同等のことが可能です。


:::details Workflowの例：こみっとくん
```
# Gitコミット作成エージェント

## 役割と目的
あなたは熟練した開発者であり、明確で正確なGitコミットを作成する責任があります。あなたの目標は、現在の変更点を分析し、作業内容を効果的に要約した単一のコミットを作成することです。

## 手順1：コンテキスト（状況）の把握
まず、以下のコマンドを実行して、現在のワークスペースの状態を把握してください。**この情報収集を行わずに次の手順に進まないでください。**

1.  **ステータスの確認**: `git status` を実行し、ステージングされている変更とされていない変更を確認する。
2.  **差分の分析**: `git diff HEAD` を実行し、具体的なコードの変更内容を詳細に調査する。
3.  **ブランチの確認**: `git branch --show-current` を実行し、現在作業しているブランチを確認する。
4.  **履歴の確認**: `git log --oneline -10` を実行し、直近のコミットの文脈やメッセージのスタイルを理解する。

## 手順2：実行
収集したコンテキストに基づいて、以下の行動をとってください。

1.  ベストプラクティス（簡潔な要約、命令形の使用など）に従った日本語のコミットメッセージを作成する。
2.  `git commit` コマンドを実行してコミットを作成する。

## 許可されているツール
- git add
- git status
- git commit
```
:::


## 3. Browser Agentによる動作確認とキャプチャ

Antigravityには **Browser Agent** が搭載されており、AIが実際にChromeブラウザを操作して動作確認を行います。

> **Instruction**: アニメーションが正常か確認したいので、実際にブラウザで表示してキャプチャを撮ってください。

すると、Agentはバックグラウンドで：

1. `npm run dev` でローカルサーバーを起動
2. ブラウザでアクセス
3. **スクリーンショットを撮影してアーティファクトとして保存**

という手順を自動で行います。人間が手動でブラウザを開く必要すらなく、開発体験として非常に未来的でした。

## 脆弱性の簡易チェック

この記事内で紹介されているプロンプトを参考に、簡易脆弱性診断を行いました。
https://zenn.dev/junpei_katayama/articles/self-vulnerability-check

レポート出力


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
