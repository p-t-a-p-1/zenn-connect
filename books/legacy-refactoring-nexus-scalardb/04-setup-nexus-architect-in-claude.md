---
title: "Claude CodeにNexus Architectを導入する"
---

:::message
Nexus ArchitectをClaude Codeに導入し、/architect系・/scalardb系コマンドを使える状態にする手順を確認します。
:::

前章までは、本書全体の流れと、あえて技術的負債を組み込んだ題材のPOSシステムを確認しました。
ここからは、実際にClaude CodeへNexus Architectを導入し、レガシーシステムの調査・分析からターゲット構成の再設計、初期レビューまでを実行する準備を進めます。

## プラグインのインストールと初期設定

nexus-architect は、Claude Code および Codex で動作するシステムアーキテクチャ設計支援のためのプラグインです。静的解析の実行プログラムを単体で起動するのではなく、Claude Code の対話的なコンテキストにおいて、AIエージェントのスキルとして呼び出して利用します。

### インストール手順

ターミナルにおいて、以下のコマンドを実行し、プラグインを Claude Code の環境に追加します。

```bash
# プラグインマーケットプレイスを追加する
claude plugin marketplace add ./nexus-architect

# アーキテクト向けおよび ScalarDB 向けの両プラグインをインストールする
claude plugin install architect@nexus-architect --scope user
claude plugin install scalardb@nexus-architect --scope user
```

実行すると、以下のようにマーケットプレイスおよびプラグインが正常にインストールされた旨が表示されます。

```text
✔ Successfully added marketplace: nexus-architect (declared in user settings)
✔ Successfully installed plugin: architect@nexus-architect (scope: user)
✔ Successfully installed plugin: scalardb@nexus-architect (scope: user)
```

これにより、Claude Code のセッション内で `/architect:コマンド名` や `/scalardb:コマンド名` という形式で、アーキテクチャ設計および検証のための様々なスキルが実行可能になります。

## 本章のまとめ

* Nexus Architectは、Claude CodeやCodexのセッション内で呼び出す設計支援プラグインとして利用します。
* `architect`と`scalardb`の2種類のプラグインを導入することで、現状分析からScalarDB設計までのコマンドを使えるようになります。
* 次章以降では、導入したコマンドを使ってレガシーPOSの調査・分析・評価・設計を実行していきます。

## 用語解説

### プラグイン
Claude CodeやCodexに機能を追加する拡張単位です。Nexus Architectでは、アーキテクチャ設計やScalarDB設計のためのコマンド群をプラグインとして追加します。

### マーケットプレイス
プラグインの配布元を登録する仕組みです。ここではローカルにある`nexus-architect`をマーケットプレイスとして追加しています。

### スコープ
プラグインをどの範囲で有効にするかを表す設定です。`--scope user`は、ユーザー環境全体で使えるようにする指定です。

### `/architect`・`/scalardb`コマンド
Nexus Architectを呼び出すためのコマンド名前空間です。調査や評価は`/architect`、ScalarDB関連の設計支援は`/scalardb`として実行します。
