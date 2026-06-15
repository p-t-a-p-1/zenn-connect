---
title: "MMI・DDD評価と統合評価を実行する"
---

:::message
MMI評価、DDD評価、統合評価を実行し、レガシーPOSの移行準備度と改善優先度を数値で確認します。
:::

## モジュラリティ成熟度（MMI）とDDD適合度の並列評価（evaluate-mmi & evaluate-ddd）

次に、モジュラリティ成熟度（MMI）とドメイン駆動設計（DDD）適合度の並列評価を実行します。

```text
⏺ evaluate-mmi と evaluate-ddd を並列実行します。
...
⏺ evaluate-mmi 完了。次に evaluate-ddd を実行します。
⏺ Skill(architect:evaluate-ddd)
  ⎿  Successfully loaded skill · jp.anthropic.claude-sonnet-4-6

⏺ 3つの DDD 評価エージェントを並列起動します。

⏺ 3 agents finished (ctrl+o to expand)
   ├ DDD strategic design layer evaluation · 3 tool uses · 10.2k tokens
   │ ⎿  Done
   ├ DDD tactical design layer evaluation · 2 tool uses · 10.6k tokens
   │ ⎿  Done
   └ DDD architecture layer evaluation · 1 tool use · 10.3k tokens
     ⎿  Done

⏺ 3軸スコアが揃いました。DDD スコアを計算してレポートを生成します。

  - Strategic: avg 1.3 × 0.30 = 0.39
  - Tactical: avg 1.3 × 0.45 = 0.59
  - Architecture: avg 1.0 × 0.25 = 0.25
  - DDD Score = (0.39 + 0.59 + 0.25) / 5 × 100 = 24.5%（Minimal DDD adoption）
```

#### レポート出力例

##### DDD戦略的設計評価

![ddd-strategic-evaluation-01](/images/legacy-refactoring-nexus-scalardb/ddd-strategic-evaluation-01.png)
*基準ごとのスコア、詳細内容*

![ddd-strategic-evaluation-02](/images/legacy-refactoring-nexus-scalardb/ddd-strategic-evaluation-02.png)
*改善提言（戦略的設計）*


評価が完了すると、AIエージェントは結果を要約して出力します。MMI平均は46%（Needs Improvement）、DDDスコアは24.5%（Minimal DDD Adoption）であり、両評価とも根本的な再設計が必要という結論で一致しました。

## 統合評価フェーズ（/architect:integrate-evaluations）

次に、MMIとDDDの個別評価を統合し、システム全体の4大横断的課題をあぶり出した上で、目標スコアを含むロードマップを自動策定します。

```text
⏺ integrate-evaluations フェーズが完了しました。

  ---
  統合評価完了サマリ
```

スコアサマリ

| 評価 | スコア |
| :--- | :--- |
| システム平均 MMI | 46%（Needs Improvement） |
| DDD スコア | 24.5%（Minimal DDD Adoption） |

```text
  横断的課題（両評価で一致）
  1. 手書き Saga の構造的欠陥 ─ CheckoutSaga/ReturnSaga が MMI・DDD 双方で最低評価
  2. God Service の崩壊 ─ OrderService(976行)＋PaymentService/ReturnService の欠落
  3. Anemic Domain Model ─ 値オブジェクト・集約ルート・ドメインロジックが皆無
  4. Architecture 層の欠落 ─ Application Service 層の完全欠如、DIP 未適用
```


#### レポート出力例

##### 総合評価レポート

![integrated-evaluation-01](/images/legacy-refactoring-nexus-scalardb/integrated-evaluation-01.png)
*総合評価、MMIとDDDの統合マトリクス*

![integrated-evaluation-02](/images/legacy-refactoring-nexus-scalardb/integrated-evaluation-02.png)
*数値サマリなど*

## 本章のまとめ

* MMI評価では、モジュールごとのマイクロサービス移行準備度を数値化しました。
* DDD評価では、境界コンテキスト、ユビキタス言語、集約、レイヤリングなどの設計成熟度を確認しました。
* 統合評価により、手書きSaga、God Service、貧弱なドメインモデル、アーキテクチャ層の欠落が横断的な改善課題として整理されました。

## 用語解説

### モジュラリティ成熟度（MMI）
Modularity Maturity Index（モジュール成熟度指標）の略で、既存モジュールがマイクロサービスとして独立しやすい状態にあるかを評価する指標です。本章では、各モジュールの独立性や責務分離の状態を数値化するために使っています。

### DDDスコア
ドメイン駆動設計の観点から、戦略設計、戦術設計、アーキテクチャ層の成熟度をまとめて数値化したものです。本章では24.5%という低い値になっており、境界コンテキストや集約、ドメインロジックの再設計が必要だと判断しています。

### 統合評価
MMIとDDDの結果を合わせ、複数の評価軸に共通して現れる構造的な課題を抽出する評価です。

### 改善優先度
リスクや影響範囲をもとに、どの課題から先に直すべきかを整理した順序です。大規模リファクタリングでは、作業の着手順を決めるために重要になります。
