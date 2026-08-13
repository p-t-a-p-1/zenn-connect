---
title: "AI駆動開発とNexus Architect"
---

AI駆動開発は、コード生成だけを速くする方法ではありません。

企画、設計、作業分解、実装、レビュー、運用まで、各工程の判断を次の工程へ渡す進め方です。

RADARでは、Claude Code上で動くNexus Architectを使い、ProductパイプラインとArchitectパイプラインをつなぎました。

AIエージェントは候補の作成、調査、文書化、コード生成を支援し、人間は開発範囲、業務ルール、外部サービス、レビュー結果、マージを判断します。

## AIに任せる仕事と人が決めること

![AIに任せる仕事と人が決めることを補助する白黒線画](/images/nexus-product-new-development/section/section-05-ai-human-boundary.png)


AIエージェントへ任せやすいのは、決めた方針に沿った反復作業です。

既存コードの調査、設計書の下書き、Issueへの分解、テストの作成、レビュー観点の整理は、その代表例です。

一方で、次の判断は人が確認します。

- 最初に解く業務課題
- 自社利用とSaaS化の順序
- 開発範囲とやらないこと
- 実運用へ進む条件
- 認証や権限など、失敗時の影響が大きい設計
- PRの承認とマージ

AIが候補を増やすほど、判断の境界を先に決めることが重要になります。RADARでは、環境変更、外部サービス登録、スコープ変更、レビューを打ち切る判断を人の操作として残しました。

## Nexus Architectの役割

![Nexus Architectの役割を補助する白黒線画](/images/nexus-product-new-development/section/section-05-nexus-role.png)


@[card](https://github.com/wfukatsu/nexus-architect)

Nexus Architectは、Claude Code向けのアーキテクチャ設計・開発支援プラグインです。RADARで使った流れは、次の2つのパイプラインに分かれます。

| パイプライン | 主な対象 |
| :--- | :--- |
| `product:*` | Vision、成功指標、仮説、ペルソナ、UI、機能、MVP |
| `architect:*` | 要求、コード調査、DDD評価、境界、データ、API、運用設計 |

Productパイプラインで **何を、誰のために作るか** を整理し、MVPで価値を確かめます。
その学びをArchitectパイプラインへ渡し、 **どう本番運用できる構造にするか** を設計します。

本書で扱うコマンドは、Claude CodeやCodexに標準で含まれるコマンドではありません。RADAR開発時に利用したプラグイン側のワークフローです。特定バージョンのコマンドリファレンスではなく、成果物をどうつなげたかを中心に説明します。

## 判断を成果物へ固定する

AIとの会話だけで開発を進めると、過去の判断が後から見つからなくなります。RADARでは、Vision、仮説、機能、要求、API、境界コンテキストへIDを付け、設計履歴を`docs/design-history/`に保存しました。

```text
企画の判断
  ↓
設計レポート
  ↓
MVPと実運用の学び
  ↓
本番設計
  ↓
Issueと受入基準
  ↓
実装・レビュー・運用
```

判断がファイルとIDに残っていれば、AIエージェントを別の工程で呼び出しても前提を引き継げます。
逆に、会話の中だけにある判断は、同じ質問を繰り返す原因になります。

![企画、設計、Issue、実装、レビュー、マージがつながるAI駆動開発の流れ](/images/nexus-architect-ai-devops-loop/ai-driven-development-flow.png)
*企画から設計、Issue、実装、レビュー、マージまでをつなぐ流れ*

Productで決めたことをArchitectへ渡し、Architectの設計をIssueと実装へ渡す。この受け渡しを本書全体で追います。
