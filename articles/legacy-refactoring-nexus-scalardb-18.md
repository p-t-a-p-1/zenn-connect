---
title: "【Nexus Architect本レビュー用】【連載】（第18回）Compound Engineeringの基本と実装入力を理解する"
emoji: "📘"
type: "tech"
topics: ["architecture","scalardb","refactoring"]
published: false
publication_name: "scalar_sol_blog"
---


:::message
レビュー済み設計を実装へつなぐために、Compound Engineeringの考え方、plan分割、入力資料の使い方を整理します。
:::


設計レビュー結果のページでは、Nexus Architectが生成した設計書とレビュー結果を読み解きました。

特に重要だったのは、初回設計がレビューでFAILとなり、その後のレビュー指摘を取り込むことで、最終的に実装へ進められる水準まで設計が強化された点です。

このページでは、そのレビュー完了となった設計書をもとに、Compound Engineeringをどのように使い始めたのかを整理します。

実装には、Claude Codeのプラグインである[Compound Engineering](https://github.com/EveryInc/compound-engineering-plugin/tree/main/plugins/compound-engineering)を利用しました。

Compound Engineeringは、単にコードを生成するためのツールというより、設計、計画、実装、レビュー、デバッグ、学習の蓄積を一連の開発ワークフローとして扱うためのプラグインです。READMEでも、`/ce-plan`による計画作成、`/ce-work`による実装、`/ce-code-review`によるレビュー、`/ce-compound`による知見の蓄積といった流れが主要な入口として整理されています。

今回のように、すでに詳細な設計書があり、そこから多数のサービス、共通ライブラリ、CI/CD、インフラ、フロントエンドまで広げていく作業では、この計画を小さく切り、実装し、失敗を記録し、次の実装に再利用する流れが非常に相性よく機能しました。


## Compound Engineeringとは

### 基本的な考え方

Compound Engineeringは、AIエージェントを単発のコード生成器として使うのではなく、ソフトウェア開発の一連の流れに組み込むためのプラグインです。

公式READMEでは、AI-powered development tools that get smarter with every useという説明がされています。つまり、1回ごとの作業をその場限りで終わらせず、計画、レビュー、デバッグ、解決した問題の記録を残し、次の作業を前回より進めやすくすることを狙っています。

通常のAIコーディングでは、以下のような問題が起きがちです。

- 毎回同じ前提説明をしなければならない
- 設計判断や過去の失敗が次の作業に引き継がれない
- 大きな作業を一気に依頼して、途中でスコープが膨らむ
- 実装はできても、レビューやデバッグの観点が弱い
- 修正した問題が知識として蓄積されない

Compound Engineeringは、この問題に対して、スキルとエージェントを組み合わせた開発ワークフローを提供します。

スキルは、ユーザーが直接呼び出す作業入口です。たとえば、戦略を整理する、要件を詰める、実装計画を作る、コードレビューする、実装する、デバッグする、解決済みの問題を記録する、といった単位です。

エージェントは、その裏側で動く専門家ロールです。アーキテクチャ、セキュリティ、信頼性、性能、テスト、データ整合性、保守性など、観点ごとのレビューや調査を担当します。

この構成により、Compound Engineeringでは、開発作業を以下のような循環として扱います。

```text
方針を決める
  ↓
要件を具体化する
  ↓
実装計画に分解する
  ↓
実装する
  ↓
レビューする
  ↓
デバッグする
  ↓
学習を記録する
  ↓
次の作業で再利用する
```

今回のPOSマイクロサービス化では、この循環のうち、特に計画、実装、レビュー、デバッグ、学習蓄積の部分を強く使っています。


### 主要なワークフロー

Compound Engineeringの中核になるワークフローは、以下の流れです。

| スキル | 役割 |
|---|---|
| `/ce-plan` | 作業を実装可能なplanに分解する |
| `/ce-work` | planに沿って実装する |
| `/ce-compound` | 解決した問題や学びを記録し、次の作業で再利用する |

今回のレガシーリファクタリング実装では、この`/ce-plan`→`/ce-work`→`/ce-compound`の流れを基本にしました。

`/ce-plan`では、Nexus Architectの設計書を直接実装するのではなく、サービス単位、機能単位、リスク単位のplanに分けます。

`/ce-work`では、そのplanに沿って、共通ライブラリ、サービス、CI、インフラ、フロントエンドを順番に実装します。

`/ce-compound`では、解決した問題を`docs/solutions/`に残し、後続の作業で再利用できるようにします。


### 今回の実装での使い方

今回のPOSマイクロサービス化では、Compound Engineeringを以下のように使いました。

まず、レビュー完了となったNexus Architectの設計書を、実装エージェントが参照し続けられる形に整理しました。具体的には、`CLAUDE.md`に実装原則と禁止事項をまとめ、`docs/design-references.md`に設計書の索引を置いています。

次に、`/ce-plan`相当の流れで、実装作業を`docs/plans/`に分割しました。最初に共通ライブラリを作り、その後にCatalog、Inventory、Order、Payment、Loyaltyなどのサービスを作り、さらにDocker Compose、CI/CD、Kubernetes、フロントエンド、耐障害性改善へ進めています。

実装では、`/ce-work`相当の流れで、各planを順番にコードへ落としました。ここでは、Hexagonal Architecture、ScalarDBアダプタ、Outbox、Saga、JWT/RBAC、Dockerfile、GitLab CIなどが実装されています。

さらに、実装中に発生した問題は、`/ce-debug`と`/ce-compound`相当の流れで原因を追い、`docs/solutions/`に記録しました。

この結果、単に1回のAI実装で終わるのではなく、以下のように知識が残る構造になりました。

```text
設計書
  ↓
CLAUDE.md / design-references.md
  ↓
docs/plans/
  ↓
実装コード
  ↓
docs/solutions/
  ↓
次のplanと実装
```

この形にできたことで、後続のサービス実装やバグ修正で、前に解いた問題を再利用しやすくなっています。


## 実装対象

### 入力として使った設計書

実装対象の作業ディレクトリは、以下です。

```text
refactoring-outputs/
  └── pos-microservices/
```

このディレクトリの`README.md`には、今回の実装対象が以下のように定義されています。

```text
legacy-pos-monolith（Spring Boot 2.7 / Java 11）を
13マイクロサービスへ段階的に移行するプロジェクト。
```

実装エージェント向けの指示は`CLAUDE.md`にまとめられており、設計書は読み取り専用、レガシーコードは変更禁止、実装は`docs/design-references.md`の索引から設計書を参照して進める、というルールになっています。

この時点で、実装の前提はかなり明確でした。

- レガシーコードは直接変更しない
- Nexus Architectの設計書を正とする
- 13サービス構成を段階的に作る
- Java 21 / Spring Boot 3.3.xに更新する
- ScalarDB Clusterを分散トランザクション基盤にする
- 各サービスはHexagonal Architectureで実装する
- 副作用はTransactional OutboxとSagaで扱う
- 共通部品は`shared/`に切り出す
- CI/CD、Docker、Kubernetes、GitLabマルチレポ構成まで含める

つまり、ここで行ったのは既存モノリスの一部を少しきれいにする作業ではありません。

レビュー済みのターゲットアーキテクチャに従って、マイクロサービス化後の実行可能なコードベースを新しく組み立てていく作業です。


### 最初に作られた実装ガイド

Compound Engineeringを使った実装では、まずプロジェクト全体の前提をエージェントが読み続けられる形に落としました。

その中心が`CLAUDE.md`です。

```text
pos-microservices/
├── shared/
│   ├── common/       # 共通ドメイン型
│   ├── outbox/       # Transactional Outbox共通実装
│   ├── saga/         # ハイブリッドSaga基底クラス
│   └── security/     # JWT/RBAC Spring Securityスターター
├── services/
│   ├── api-gateway/
│   ├── identity-service/
│   ├── catalog-service/
│   ├── inventory-service/
│   ├── cart-service/
│   ├── order-service/
│   ├── checkout-orchestrator/
│   ├── payment-service/
│   ├── loyalty-service/
│   ├── receipt-service/
│   ├── return-service/
│   ├── audit-service/
│   └── dashboard-service/
├── infrastructure/
├── db/
└── docs/
```

このファイルでは、単にディレクトリ構成だけでなく、実装上の禁止事項も明文化されています。

- 設計書を編集しない
- レガシーコードを変更しない
- ScalarDBトランザクション内で外部APIを呼ばない
- 業務ロジックをコントローラに書かない
- `System.out.println`を使わない
- SQL文字列結合をしない

これは地味ですが、AIエージェントに大きな実装を任せるうえでは非常に重要です。

なぜなら、AIにとって次に何を作るかだけでなく、何をしてはいけないかも、コード品質を左右するからです。

## 本章のまとめ

* Compound Engineeringでは、レビュー済みの設計書を入力にして、大きな実装を小さなplanへ分解して進めます。
* `docs/plans/`や`docs/solutions/`を使うことで、作業単位、完了条件、実装中に得た知識を蓄積できます。
* AIエージェントに実装を任せるほど、作るものだけでなく、変更してはいけないものや守るべき制約を明文化することが重要です。

## 用語解説

### Compound Engineering
設計、計画、実装、学習結果を積み上げながら、大きな開発を段階的に進める考え方です。本書ではClaude Codeの実装プラグインとして扱っています。

### plan
実装前に作成する作業計画です。目的、対象ファイル、完了条件、スコープ外を明確にし、AIエージェントの作業を制御します。

### `docs/solutions/`
実装中に得た解決策や失敗からの学びを残す場所です。後続の実装で同じ問題を繰り返さないための知識ベースになります。

### 禁止事項
実装時にしてはいけない操作や設計上の制約です。レガシーコードを変更しない、ScalarDB Tx内で外部APIを呼ばない、といったルールが該当します。
