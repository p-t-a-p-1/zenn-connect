---
title: "【Nexus Architect本レビュー用】（第12回）リファクタリング設計レポートの全体像を読み解く"
emoji: "📘"
type: "tech"
topics: ["architecture","scalardb","refactoring"]
published: false
publication_name: "scalar_sol_blog"
---


:::message
設計レポート全体を俯瞰し、境界コンテキスト、ターゲットアーキテクチャ、ScalarDB設計、レビュー結果のつながりを把握します。
:::


ここまでのページでは、Nexus Architectを実行して生成されたレポートのうち、主に以下の流れを見てきました。

```
現状把握（00_summary/） → ドメイン分析（01_analysis） → 評価（02_evaluation）
```

ここからは、その後段にあたる設計・レビューのレポートを数ページに分けて深掘りします。

```
現状把握（00_summary/） → ドメイン分析（01_analysis） → 評価（02_evaluation） → 設計（03_design） → レビュー（review/）
```

この設計・レビュー編では、特に以下の2つにフォーカスします。

- `03_design/` に出力されたリファクタリング後の設計方針
- `review/` に出力された設計レビューと、その指摘を受けた改善の軌跡

この一連のレポートで重要なのは、単にAIがきれいな設計書を出してくれたという話ではありません。

むしろ重要なのは、**初回の設計がレビューで明確にFAILし、その指摘をもとに設計が段階的に強くなっていく過程**です。

レガシーシステムのリファクタリングでは、最初から完璧なターゲットアーキテクチャを描くことはほとんどできません。現状分析、ドメイン分割、技術制約、運用要件、コスト制約、そしてレビューからのフィードバックを何度も往復しながら、ようやく実装に移してよい設計に近づいていきます。

今回のレポート群には、その往復がかなり具体的に現れています。


## 設計・レビュー

### 各レポート内容の概要

設計フェーズでは、以下のようなMarkdown文書が生成されています。

```text
reports/
  ├── 03_design/
  │   ├── bounded-contexts-redesign.md  # 境界コンテキスト再設計
  │   ├── context-map.md                # 境界コンテキスト間の関係性
  │   ├── target-architecture.md        # 13サービス構成のターゲットアーキテクチャ
  │   ├── transformation-plan.md        # Phase0〜5の段階的移行ロードマップ
  │   ├── scalardb-schema.md            # Namespace・テーブル・キー・Outboxスキーマ
  │   ├── scalardb-transaction.md       # 分散トランザクション・Outbox・Saga境界
  │   ├── saga-design.md                # 外部副作用を扱うハイブリッドSaga
  │   ├── read-model-design.md          # Dashboard集計のCQRS Read Model
  │   ├── api-gateway-design.md         # API Gateway/BFF
  │   └── disaster-recovery.md          # HA/DR・縮退運用・RTO/RPO
  ├── review/
  │   ├── review-synthesis.md           # 初回レビュー（FAIL）
  │   ├── review-synthesis-v2.md        # P0解消後レビュー（CONDITIONAL_PASS）
  │   └── review-synthesis-v3.md        # P1解消後レビュー（4.275点）
  ├── 08_infrastructure/
  │   ├── infrastructure-architecture.md
  │   ├── deployment-guide.md
  │   ├── disaster-recovery-design.md
  │   ├── observability-design.md
  │   └── security-design.md
  └── 05_estimate/
      ├── cost-summary.md
      ├── infrastructure-detail.md
      └── scalardb-sizing.md
```

この中で中心になるのは、`03_design/` と `review/` です。

現状把握・ドメイン分析・評価レポートで見た通り、既存システムには以下のような問題がありました。

- `OrderService` が976行のGod Serviceになっている
- `CheckoutSaga` / `ReturnSaga` が複数DB更新と補償処理を1クラスで抱えている
- MySQLとPostgreSQLを跨ぐ一貫性が手書きSagaに依存している
- DAO/repositoryの命名が混在し、レイヤ境界が曖昧
- Dashboard集計がアプリケーション側のループや全件取得に寄っている
- 監査ログやイベント通知の欠落を構造的に防げていない

設計レポートでは、これらの課題に対して、いきなりマイクロサービス化するのではなく、**モノリス内モジュラ化 → ScalarDB導入 → DDD戦術パターン適用 → 周辺サービス抽出 → コアサービス抽出** という段階的な道筋を描いています。

## 本章のまとめ

* 設計レポート群は、境界コンテキスト、ターゲットアーキテクチャ、ScalarDB、API、運用、レビュー結果に分かれています。
* 中心になるのは、現状課題をどう分割し、どの順序で安全に移行するかを示す`03_design/`と`review/`です。
* 設計の狙いは、一気にマイクロサービス化することではなく、モノリス内で境界を作り、ScalarDBやDDDパターンを段階的に導入することです。

## 用語解説

### 設計レポート
分析結果をもとに、目指す構造、移行手順、リスク、レビュー結果を文書化した成果物です。実装前の判断材料になります。

### 分散トランザクション
複数のDBやサービスをまたいで、一貫した更新を実現するための仕組みです。本書ではScalarDBを使った設計が中心になります。

### レビュー結果
設計に対する指摘、判定、改善状況をまとめたものです。設計をv1からv2、v3へ改善するための入力になります。

### 段階的移行
一度に全体を作り直すのではなく、リスクの低い順に構造を変えていく進め方です。レガシーリファクタリングでは特に重要です。
