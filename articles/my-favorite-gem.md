---
title: "【Gemini】マジで使うGem"
emoji: "🦀"
type: "idea" # tech: 技術記事 / idea: アイデア
topics: ["gemini", "業務効率化", "googleworkspace"]
published: false
---

:::message
この記事は [Mavs Advent Calendar 2025](https://adventar.org/calendars/11595) の 23日目の記事です。🎅
:::

# はじめに

🚧

弊社では、12月からGoogleWorkspaceを導入しております。経緯はこちらの記事にまとめております！
@[card](https://zenn.dev/mavericks/articles/google-workspace-migration-story)


## Gem...？？


# Gemたちの紹介


:::message alert
Gemとして紹介しますが、
ClaudeやChatGPTのカスタム指示としても使えるようになっています。
:::

## 記事解説

- 利用シーン：わからない技術記事を全体像を掴みたいとき

### カスタム指示

https://github.com/p-t-a-p-1/gem-prompts/blob/main/tech-explainer/instruction.md


### 出力例



## 画像表現

- 利用シーン：コードや仕様書を絵にして全体像を掴みたいとき

### カスタム指示




## Markdownの圧縮

- 利用シーン：長いMarkdownを散文的に短い説明にするとき


### カスタム指示


### 出力例


## Issue作成

- 利用シーン：

### カスタム指示

### 出力例




## ドメインストーリーテリング

### カスタム指示


### 出力例



## Webサイトアニメーションの言語化


### カスタム指示

https://github.com/p-t-a-p-1/gem-prompts/blob/main/animation-analyzer/instruction.md

### 出力例




# どうやってカスタムGemを作っているのか

私はカスタムGemを作るときもGeminiに聞いています。

以下は実際にWebサイトアニメーションの言語化をカスタムGemにするときのやり方です。

```
カスタムGemを作成したいです。カスタム指示を考えてください。
Gemにやってほしいこと
* ユーザーが指定したWebサイトのアニメーションを解析する
* リッチなアニメーションを言語化する
* ソースコード確認できる場合はどのような実装になっているのか簡易的に解説する
```

あとは出力内容をもとにGemを作成すればokです。

:::message
実はGoogle公式でもカスタムGemについてヒントを出してくれています。
@[card](https://support.google.com/gemini/answer/15235603?hl=ja)
:::


- あなたの役割と目標
- 具体的な作業指示
- 出力フォーマット
- 制約事項


# まとめ

🚧

良いGemをつくったらまた記事にします！！