---
title: "Antigravity活用した0からの開発 ~ パタパタ時計風テキスト生成サービスを添えて ~"
emoji: "💨"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["antigravity", "reactrouter", "cloudflarepages", "個人開発"]
published: true
publication_name: "mavericks"
---

:::message
この記事は [Mavs Advent Calendar 2025](https://adventar.org/calendars/11595) の 16日目の記事です。🎅
:::

# はじめに

今回は、Antigravityを活用して、レトロな「パタパタ時計」風のメッセージボードアプリを爆速で開発した過程を共有します！！

![家にあるパタパタ時計](/images/patapata-board/20251216-083231.jpg)
*家にあるパタパタ時計*

おじいちゃん・おばあちゃんの家によくあるやつです。近所の雑貨屋で購入しました。

ふと↑のようなものをWebで表現したいと思い、Antigravityを活用して開発してみました。


**頭にあるイメージを言語化、コードに落とし込み、実際に公開するまでの一連の流れをなるべくわかりやすく解説してみました。**


## 話すこと

- 作ったものの紹介（3割）
- Antigravityでの爆速開発（7割）

# パタパタ時計風テキスト生成サービス

以下のサイトから実際に生成される様子を確認できます！！

@[card](https://patapata-board.pages.dev/)

![生成例](/images/patapata-board/demo.png)
*生成例*

日本語、英語、絵文字、数字、顔文字、ヒエログリフには対応してます。


gif動画を生成するようなものも作りたかったですが一旦保留にしてます...！


# 技術構成

Antigravity上で、以下のスタックを用いて開発しました。

| カテゴリ | 技術名 | 説明 |
| --- | --- | --- |
| **フレームワーク** | [**React Router v7**](https://reactrouter.com/) | Framework Mode / SSR Enabled |
| **プラットフォーム** | [**Cloudflare Pages**](https://pages.cloudflare.com/) | FunctionsによるAPI処理とホスティング |
| **言語** | TypeScript |  |
| **スタイリング** | Tailwind CSS v4 |  |
| **OG Image** | [**@cloudflare/pages-plugin-vercel-og**](https://developers.cloudflare.com/pages/functions/plugins/vercel-og/) | Pages FunctionsのMiddlewareで使用 |
| **IDE** | [**Google Antigravity**](https://antigravity.google/) | Agentによる自律開発 |
| **AI Model** | **Claude Opus 4.5** & **Gemini 3.0 Pro** | Antigravity内で使用しているモデル |

## 技術的なこだわりポイント

### 1. Cloudflare Pages × React Router v7

デプロイ先には、**Cloudflare Pages** を選択しました。
React Router v7 は Cloudflare Workers/Pages アダプターを標準でサポートしており、
`wrangler.json` の設定だけで簡単にデプロイ環境が構築できます。

### 2. Middlewareによる動的OGP生成

動的OGP生成には、[**Cloudflare Pages Functions Middleware**](https://developers.cloudflare.com/pages/functions/middleware/) を活用しています。

`functions/_middleware.tsx` にて `/og-image.png` へのリクエストをインターセプトし、
その場でOGP画像を生成して返しています。また、通常のHTMLリクエストに対しては自動的に `<meta>` タグを注入する処理も行っています。

ライブラリには `@cloudflare/pages-plugin-vercel-og` を採用しました。
これは Vercel OGを Cloudflare Pages 上で簡単に使えるようにしたプラグインです。

以下の方の記事を参考にしました...！！
@[card](https://re-engines.com/2025/01/27/react-router-v7-ogp/)


![出力例](/images/patapata-board/og-demo-image.png)
*出力例*

# 実際の開発フロー

実際にどのように開発を進めたかを紹介します！！
その中で、Antigravityを最大限活用する方法を紹介します。

## 1. AIエージェントと対話して要件を詰める

まず、Antigravityのチャット機能で作りたいアプリのイメージを伝えます。
大事なことは対話するということです。
こちら主導で内容精査してもらうのもありですが、あまり深くまでイメージできていなかったり質問されてその時考えたいといった場合は対話することを意識すると良いです！

```
あなたと対話しながら「パタパタ表示機」のWebアプリを作りたいです。
見た目は空港の案内板のようなレトロな感じで
```

このような導入から、対話形式で要件を詰めていきます。私はGemini 3.0 Proを用いました。
納得のいくものができた段階で、以下の指示を行います。


```
これまでのやり取りを踏まえPRD（仕様書）を出力してください
```

これで包括的で詳細なPRD（仕様書）が生成されます。

:::message
作りたいもののビジュアルがイメージできるのであれば、Geminiのツール"Canvas"機能でモック作成 → PRD出力の方が良いです！
:::


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
| `/resource/og` | `functions/_middleware.tsx` | **Middleware**。`/og-image.png` を生成し、HTMLレスポンスにmetaタグを注入する。 |

## 6. ロードマップ

1.  **Phase 1: コア機能実装**
      * React Router v7 プロジェクト作成。
      * メインのパタパタボードとアニメーション実装。
      * URL同期機能の実装。
2.  **Phase 2: サーバーサイド機能**
      * `@cloudflare/pages-plugin-vercel-og` の導入。
      * Middleware (`functions/_middleware.tsx`) の実装。
      * OGP画像の動的生成ロジックの実装。
3.  **Phase 3: ウィジェット & エクスポート**
      * `ClockWidget` の実装と配置。
      * 画面録画機能 (`getDisplayMedia`) の実装。
      * 録画時に時計やUIを隠す「Clean Mode」制御の実装。
:::

:::message
PRD内ではDeploymentに `Vercel (Edge Functions)` が提案されていますが、本記事の実装では **Cloudflare Pages** を採用しました。AIが生成した初期案をベースにしつつ、実際の開発過程で技術選定を柔軟に変更しています。
:::

## 2. Agentによる自律実装

ここから実際にAntigravityを活用して開発します。

重要なポイントとして、**開発のための初期構築（インストールなど）**は人力で行った方が良いです。
今回無料プランということもあり、利用制限があります。
リミットに達するのを防ぐため、AIに任せなくても良い、かつハードル低めな初期構築作業なので、人間が行うのが望ましいです。

初期構築したプロジェクトにdocsフォルダを作成し、先ほど出力したprd.mdを配置します。
その後、実際にAntigravityのAIエージェントを活用し以下のようなやり取りで開発を始めます。

初回はClaude Opusを利用しました。
```
@prd.mdを参照して、実装を開始してください
```

Antigravityは実装プラン（Implementation Plan）を生成してくれるので、内容をレビューして実装を開始します。

## 2.5 *~ 脱線 ~* Antigravityの設定について

今回は最小限の設定にとどめています。

### Antigravity全体の設定

@[card](https://dev.classmethod.jp/articles/google-antigravity-five-tips/)

この方の記事の **Customizations でのグローバル指示** を参考にさせていただきました。不便な点は全くなく良い感じの出力になっています。

### Agentによるコミット作成

Claude CodeやCursorにはスラッシュコマンド機能があると思いますが、
AntigravityにはWorkflowという機能で同等のことが可能です。

![コミット用Workflow](/images/patapata-board/commit.png)
*コミット用Workflow設定例*

実際の開発に合わせて内容は改変してください
```
# Gitコミット作成エージェント

## 役割と目的
あなたは熟練した開発者であり、明確で正確なGitコミットを作成する責任があります。あなたの目標は、現在の変更点を分析し、作業内容を効果的に要約した単一のコミットを作成することです。

## 手順1：コンテキスト（状況）の把握
まず、以下のコマンドを実行して、現在のワークスペースの状態を把握してください。この情報収集を行わずに次の手順に進まないでください。

1. ステータスの確認: git status を実行し、ステージングされている変更とされていない変更を確認する。
2. 差分の分析: git diff HEAD を実行し、具体的なコードの変更内容を詳細に調査する。
3. ブランチの確認: git branch --show-current を実行し、現在作業しているブランチを確認する。
4. 履歴の確認: git log --oneline -10 を実行し、直近のコミットの文脈やメッセージのスタイルを理解する。

## 手順2：実行
収集したコンテキストに基づいて、以下の行動をとってください。

1. ベストプラクティス（簡潔な要約、命令形の使用など）に従ったコミットメッセージを作成する。
2. git commit コマンドを実行してコミットを作成する。

## 許可されているツール
- git add
- git status
- git commit
```


![コミット指示例](/images/patapata-board/commit-input.png)
*コミット指示例*

使い方は `/commit` で指示します。あとは上記の指示に沿って動いてくれます。

![コミット出力例](/images/patapata-board/commit-output.png)
*コミット出力例*

個人開発レベルだと十分です！

### MCP

MCPはContext7のみです。
複雑な実装であったり、一部不安なところなどは `use context7` でAIエージェントが正確な情報を元に実装するようになります。（ざっくり）

:::message
Context7 は、プログラミングで使う様々なライブラリやフレームワークの最新の使い方や情報を、AI が簡単に調べられるようにするサービスです。
例えば React や Next.js などの技術について質問すると、最新の公式ドキュメントから正確な情報を取得して回答してくれます。
これにより、古くなった情報ではなく、今現在使える正しい方法でプログラミングの手助けができるようになります。
:::

実際のインストールは以下を参考にしてみてください！
@[card](https://github.com/upstash/context7)


## 3. テスト - Browser Agentによる動作確認とキャプチャ

Antigravityには **Browser Agent** が搭載されており、AIが実際にChromeブラウザを操作して動作確認を行います。

@[card](https://codelabs.developers.google.com/getting-started-google-antigravity?hl=ja#3)

個人的にはここがAntigravityの一番便利だった部分でした。

![ブラウザサブエージェントによるブラウザテスト](/images/patapata-board/browser.png)
*ブラウザサブエージェントによるブラウザテスト*

上記画像のように、基本的にAIエージェントがブラウザを操作して動作確認を行います。
実際にスクショ撮って動作確認してくれたり、エビデンスを残してユーザーに確認依頼投げてもらう体験はAntigravityが今のところ一番良い気がします。

## 4. 脆弱性の簡易チェック

この記事内で紹介されているプロンプトを参考に、簡易脆弱性診断を行いました。
https://zenn.dev/junpei_katayama/articles/self-vulnerability-check

こちらも不便な点は全くなく良い感じの出力になっていたためありがたく参考にさせていただきました...！

## 5. デプロイ

デプロイについても詰まるところは特になく、サクサク進んでくれました。
Cloudflare側の設定についてもAntigravityに質問して進めることで効率的にデプロイ作業が可能です。

プロンプト参考例
```
このアプリケーションをCloudflare Pagesにデプロイするにあたり、必要な手順を教えてください。
なお、アプリケーション側で設定するべきものが不足している場合は、対応してください。
```


# まとめ

今回は、Antigravityを活用し、パタパタ時計風のテキスト生成サービスを爆速で開発してみました。

パタパタ時計を見て作りたいものを決め、実際デプロイするまでは2時間くらいでした。
実際にかかった時間の内訳は以下の通りです。

```
- PRD作成：10分
- Antigravity設定・実開発：1時間
- デプロイ設定：30分
```

今回紹介したAntigravityでの設定は、他IDEでも同じことができると思いますが、  
Antigravityでは特に設定なく無料ですぐ使えるのが良かったです。

## おまけ Google Workspace用のプランも今後用意される? （2025/12/16時点ではまだない）

AntigravityのPlanを眺めていると以下の記述がありました。

```
Your Plan: Free
Paid plans for Google Workspace accounts are coming soon. You can use a personal account with a Google AI plan to receive higher rate limits.

現在のプラン: 無料
Google Workspaceアカウント向けの有料プランは近日公開予定です。より高いレート制限を利用するには、Google AIプランをご契約の個人アカウントをご利用ください。
```

Google Workspaceアカウントと紐づくことができれば、会社標準の開発ツールになるかもしれません...！！
