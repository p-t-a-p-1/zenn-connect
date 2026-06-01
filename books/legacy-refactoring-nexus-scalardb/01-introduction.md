---
title: "はじめに"
---

この本を通して解説する、ScalarDBや分散トランザクション、マイクロサービス、AIを活用した開発などの関連情報は、Scalar Solution Blogでも発信されています。本書とあわせて参照いただくことで、個別トピックの理解も深めやすくなると思います！！

@[card](https://zenn.dev/p/scalar_sol_blog)

## はじめに

開発の現場において、システムが成長するにつれてソースコードは肥大化し、設計の崩れや密結合といった問題が深刻化していきます。    

保守性や拡張性の低下だけではなく、セキュリティや信頼性といった観点からも既存の課題を放置することはできませんが、目の前のタスクに追われ気づけば何年も放置してしまうなんてこともあると思います。

そこで本書では、以下のツールを組み合わせて、安全で高速にモダナイゼーションを実現する体験をお伝えします！

* Nexus Architect（AIエージェント）
  * レガシーシステムのリファクタリングや新規システム設計、DB移行やScalarDBアプリケーション開発を支援する、統合システムアーキテクチャに特化したAIエージェント
* ScalarDB（ミドルウェア）
  * 複数・異種のデータベースを移行することなく仮想的に統合し、データ一貫性を保ったトランザクションやリアルタイムな分析・AI活用を実現するユニバーサルHTAPエンジン
* Compound Engineering（AI開発ワークフロー）
  * 設計書を実装計画へ分解し、実装、レビュー、デバッグ、学習の蓄積を繰り返しながら、AIエージェントによる開発を継続的に改善していくための開発ワークフロー

この三つの技術を組み合わせることで、これまでの膨大な時間とコストを要したリファクタリングから脱却し、確かな事実に基づいた安全なモダナイゼーションを体験できると考えています！

![summary](/images/legacy-refactoring-nexus-scalardb/summary.png)
*本書の全体像*

## 本書の構成

本書は以下のステップに沿って、実際のレガシーコードの分析からリファクタリング、そしてアーキテクチャの自動検証までを体験できるように構成されています。

### 1. 導入、Nexus Architect、題材POSシステム

まず、本書の目的と登場する主要技術を整理します。続いて、Nexus Architectがどのような設計支援ツールなのかを確認し、検証対象として使うレガシーPOSモノリスの機能、データ構造、技術的負債を見ていきます。

- [はじめに](01-introduction)
- [アーキテクチャ設計支援ツールキット（Nexus Architect） とは？？](02-what-is-nexus-architect)
- [解析対象とするレガシーPOSシステムの機能と現状の構造](03-legacy-pos-features-and-structure)

### 2. Nexus Architectの導入と実行

Claude CodeにNexus Architectを導入し、調査、分析、MMI/DDD評価、統合評価、再設計、ScalarDB設計、設計レビュー、再レビューまでの流れを追体験します。ここでは、AIエージェントがどのようにレポートを生成し、レビュー指摘を設計へ戻していくのかを確認します。

- [Claude CodeにNexus Architectを導入する](04-setup-nexus-architect-in-claude)
- [Nexus Architectで調査と分析を実行する](05-run-investigation-and-analysis)
- [MMI・DDD評価と統合評価を実行する](06-run-evaluation-and-integration)
- [再設計とScalarDB設計を生成する](07-generate-redesign-and-scalardb-design)
- [設計レビューと再レビューで設計を固める](08-design-review-and-rereview)

### 3. 解析・ドメイン分析・評価レポートの読み解き

生成された現状把握、ドメイン分析、評価・採点のレポートを読み解きます。God Service、手書きSaga、DB跨ぎ更新、ユビキタス言語の揺れ、MMI/DDDスコアなどを手がかりに、どこから改善すべきかを整理します。

- [現状把握レポートを読み解く](09-understanding-current-state-reports)
- [ドメイン分析レポートを読み解く](10-understanding-domain-analysis-reports)
- [評価・採点レポートを読み解く](11-understanding-evaluation-reports)

### 4. 設計・ScalarDB・レビュー結果の読み解き

境界コンテキスト再設計、ターゲットアーキテクチャ、ScalarDBスキーマ、Outbox、Saga、CQRS Read Model、API Gateway、HA/DR設計を順に読み解きます。さらに、初回レビューのFAILからv3で本番適用可能な設計へ近づくまでの改善過程を確認します。

- [リファクタリング設計レポートの全体像を読み解く](12-understanding-design-report-overview)
- [境界コンテキスト再設計とターゲットアーキテクチャを読み解く](13-context-redesign-and-target-architecture)
- [ScalarDBスキーマ設計とOutboxパターンを読み解く](14-scalardb-schema-and-outbox)
- [トランザクション・Saga・Read Model設計を読み解く](15-transaction-saga-and-read-model)
- [API Gatewayと運用設計を読み解く](16-api-gateway-and-operations-design)
- [設計レビュー結果と学びを読み解く](17-design-review-results-and-lessons)

### 5. Compound Engineeringによる実装

レビュー完了となった設計書を入力に、Compound Engineeringを使って`pos-microservices`を実装していく流れを見ます。実装plan、13サービス構成、共通ライブラリ、ScalarDBスキーマ、CI/CD、Kubernetes、フロントエンド移行、実装中に蓄積された学習を整理します。

- [Compound Engineeringの基本と実装入力を理解する](18-compound-engineering-basics-and-inputs)
- [リファクタリング計画と実装構成を理解する](19-refactoring-plan-and-implemented-structure)
- [ScalarDB・インフラ・フロントエンド移行を理解する](20-scalardb-infra-and-frontend-migration)
- [実装で得た学習とアーキテクチャの変化を理解する](21-implementation-learning-and-architecture-delta)

### 6. 効果測定とまとめ

最後に、リファクタリング前後で構造がどのように変わったのかを、DDD準拠度、MMI、依存関係、運用性の観点から確認します。そのうえで、本書全体の取り組みを振り返り、実務で段階的にモダナイゼーションを進めるための考え方を整理します。

- [前後比較で見るリファクタリングの効果測定](22-measuring-refactoring-effects)
- [おわりに](23-conclusion)
