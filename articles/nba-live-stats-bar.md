---
title: "MacのメニューバーにNBAの試合経過を表示する (Python + SwiftBar)"
emoji: "🏀"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["python", "uv", "swiftbar", "nba", "mac"]
published: false
---

:::message
この記事は [Mavs Advent Calendar 2025](https://adventar.org/calendars/11595) の 9日目の記事です。🎅
:::

# はじめに

Macのメニューバーに自分の好きなものを表示したい！

私はNBAの試合経過をリアルタイムで確認したいと考えていました。

ブラウザを常時開いておくのは作業の妨げになる、でもチラチラ確認したい...と思ったので作りました！
SwiftBarを活用することで、簡単にスクリプトの実行結果をメニューバーに柔軟に表示することができました。

今回は作成したものを実例として、
**Pythonで書いたスクリプトを、Macのメニューバーアプリのように手軽に扱う方法**を紹介します。

# 作ったもの

Macのメニューバーに、現在行われているNBAの試合スコアをリアルタイムに表示するプラグインです。

@[card](https://github.com/p-t-a-p-1/nba-live-stats-bar)


![メニューバーに進行中の試合のスコアと経過が表示されます](/images/nba-live-stats-bar/demo1.png)


**メニューバーに `🏀 LAL 110-108 GSW (Q4 2:00)` のように表示されます**


---

![クリックするとドロップダウンで以下の情報が表示されます](/images/nba-live-stats-bar/demo2.png)


クリックするとドロップダウンで以下の情報が表示されます。
- 当日の全試合のスコアとステータス
- 進行中の試合の直近のPlay-by-Play（得点経過など）
- 試合開始前の予定時刻

このように、単なるテキスト表示だけでなく、ドロップダウンを用いた詳細情報の表示や、
色・フォントの制御も可能です。

# 技術構成

## SwiftBar

SwiftBarは、シェルスクリプトやプログラムの標準出力をメニューバーに表示するmacOSアプリケーションです。

@[card](https://github.com/swiftbar/SwiftBar)

基本的な使い方は大きく分けて3ステップです。
1. インストール
2. プラグインの作成
3. プラグインフォルダの設定、配置

それぞれの手順を軽く説明します。

### 1. インストール
Homebrewでインストールできます。

```bash
brew install --cask swiftbar
```

### 2. プラグインの作成

後述するスクリプトを作成し、それをSwiftBarのプラグインとして配置します。

スクリプト名は以下のルールに従う必要があります。

```
{名前}.{時間}.{拡張子}
```

例： **nba-live-stats.1m.py**
この場合、1分ごとにスクリプトが実行されます。

詳しい設定値は以下を参照してください

@[card](https://github.com/swiftbar/SwiftBar?tab=readme-ov-file#plugin-naming)


### 3. プラグインフォルダの設定、配置

Swiftbarの初回起動時に、プラグインフォルダの設定を求めます。
基本的には、選択したフォルダ内の全てをプラグインファイルとしてインポートしようとするためフォルダ選択には注意してください。
隠しフォルダは無視されます、また `.swiftbarignore` で設定したファイルは除外されるので必要に応じて設定してください。


## Pythonスクリプトの解説

今回は `nba_api` という外部ライブラリを活用しております。

@[card](https://github.com/swar/nba_api)





## まとめ

