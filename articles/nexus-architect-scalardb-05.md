---
title: "【連載】（第5回）Nexus Architect × ScalarDB：Compound Engineeringで実装へつなげる"
emoji: "🧭"
type: "tech"
topics: ["scalardb","refactoring","マイクロサービス","kubernetes","claudecode"]
published: false
publication_name: "scalar_sol_blog"
---
## Compound Engineeringの基本と実装入力を理解する

:::message
レビュー済み設計を実装へつなぐために、Compound Engineeringの考え方、plan分割、入力資料の使い方を整理します。
:::


設計レビュー結果のページでは、Nexus Architectが生成した設計書とレビュー結果を読み解きました。

特に重要だったのは、初回設計がレビューでFAILとなり、その後のレビュー指摘を取り込むことで、最終的に実装へ進められる水準まで設計が強化された点です。

このページでは、そのレビュー完了となった設計書をもとに、Compound Engineeringをどのように使い始めたのかを整理します。

実装には、Claude Codeのプラグインである[Compound Engineering](https://github.com/EveryInc/compound-engineering-plugin/tree/main/plugins/compound-engineering)を利用しました。

Compound Engineeringは、単にコードを生成するためのツールというより、設計、計画、実装、レビュー、デバッグ、学習の蓄積を一連の開発ワークフローとして扱うためのプラグインです。READMEでも、`/ce-plan`による計画作成、`/ce-work`による実装、`/ce-code-review`によるレビュー、`/ce-compound`による知見の蓄積といった流れが主要な入口として整理されています。

今回のように、すでに詳細な設計書があり、そこから多数のサービス、共通ライブラリ、CI/CD、インフラ、フロントエンドまで広げていく作業では、この計画を小さく切り、実装し、失敗を記録し、次の実装に再利用する流れが非常に相性よく機能しました。


### Compound Engineeringとは

#### 基本的な考え方

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


#### 主要なワークフロー

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


#### 今回の実装での使い方

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


### 実装対象

#### 入力として使った設計書

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


#### 最初に作られた実装ガイド

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

---

## リファクタリング計画と実装構成を理解する

:::message
Compound Engineeringで作成した実装planと、その結果としてできた13サービス・フロントエンド・共通ライブラリの構成を確認します。
:::

### 実装の進め方

#### planを小さく分ける

Compound Engineeringでは、大きな作業をいきなり実装するのではなく、まず`docs/plans/`に計画を残してから実装を進めています。

実際の出力ディレクトリには、以下のようなplanが残っています。

```text
docs/plans/
  ├── 2026-05-14-001-feat-phase1-shared-modules-plan.md
  ├── 2026-05-14-002-feat-phase2-outbox-saga-identity-ci-plan.md
  ├── 2026-05-15-001-feat-catalog-service-implementation-plan.md
  ├── 2026-05-15-002-feat-inventory-service-implementation-plan.md
  ├── 2026-05-15-003-feat-order-service-implementation-plan.md
  ├── 2026-05-15-004-feat-payment-service-implementation-plan.md
  ├── 2026-05-15-005-feat-loyalty-service-implementation-plan.md
  ├── 2026-05-15-006-feat-receipt-service-implementation-plan.md
  ├── 2026-05-15-007-feat-cart-service-implementation-plan.md
  ├── 2026-05-19-008-feat-identity-service-implementation-plan.md
  ├── 2026-05-19-009-feat-docker-compose-all-services-plan.md
  ├── 2026-05-19-011-feat-github-actions-cicd-plan.md
  ├── 2026-05-19-014-feat-kubernetes-manifests-plan.md
  ├── 2026-05-20-015-feat-pos-frontend-spa-phase1-plan.md
  ├── 2026-05-21-016-feat-pos-frontend-admin-screens-phase2-plan.md
  └── 2026-05-26-008-feat-inventory-occ-retry-plan.md
```

このplan群を見ると、実装が一枚岩ではなく、かなり細かい単位に分割されていたことが分かります。

最初に`shared/common`と`shared/security`を作り、次に`shared/outbox`、`shared/saga`、ScalarDBスキーマ、Identity Serviceの骨格を作る。その後、Catalog、Inventory、Order、Payment、Loyaltyといった業務サービスを順番に独立させ、さらにDocker Compose、CI/CD、Kubernetes、フロントエンド、可観測性、リトライ、ロールバック自動化へ進む。

Nexus Architectの設計・レビューで固めた流れを、実装作業単位に分解すると、以下のようになります。

```text
設計レビュー完了
  ↓
shared/common・security
  ↓
shared/outbox・saga・ScalarDB schema
  ↓
中核サービスの実装
  ↓
周辺サービスの実装
  ↓
API Gateway・フロントエンド
  ↓
CI/CD・Docker・Kubernetes
  ↓
可観測性・耐障害性・運用改善
```

この順序が重要です。

たとえば、Order ServiceでOutboxを使うには、先に`shared/outbox`が必要です。Checkout OrchestratorやReturn Serviceのタイムアウト補償を扱うには、先に`shared/saga`が必要です。各サービスでJWT/RBACを統一するには、先に`shared/security`が必要です。

AIに任せる場合でも、人間が作る場合でも、依存関係を無視して実装すると、後半で大きな手戻りが起きます。今回のplan群は、その依存関係を実装順に落としている点がよくできています。


#### planの中身はかなり具体的

planは単なるTODOリストではありません。

たとえばCatalog Serviceのplanでは、以下のような要件が定義されています。

- `Product`集約をScalarDBの`catalog.products`に永続化する
- バーコード検索はSecondary Indexを使う
- `findById` / `findByBarcode`にはCaffeineキャッシュを使う
- 書き込みエンドポイントは`MANAGER`ロールで保護する
- 重複バーコードはScalarDBトランザクション開始前に拒否する
- ドメインイベントを定義するが、Outbox発行は後続フェーズに回す
- `./mvnw clean verify -Pquality -pl services/catalog-service -am`を完了基準にする

Inventory Serviceのplanでは、`Stock`集約、`StockMovement`、在庫引当、在庫戻し、入荷、ScalarDBのread-modify-writeパターン、ユニットテスト、Dockerfileまでが実装単位に分かれています。

Order Serviceのplanでは、注文作成、キャンセル、完了、冪等性キー、Outboxへの`OrderPlacedEvent`書き込みがスコープに含まれています。

この粒度まで落とすことで、AIはそれっぽいサービスを作るのではなく、設計書から導かれた制約を満たすサービスを作りやすくなります。


### 実装された構成

#### 13サービス + フロントエンド

実際に出力された`services/`配下には、以下のサービスが作成されています。

| # | サービス | 役割 |
|---|---|---|
| S1 | `catalog-service` | 商品マスタ |
| S2 | `inventory-service` | 在庫管理 |
| S3 | `order-service` | 注文管理 |
| S4 | `payment-service` | 支払・返金 |
| S5 | `loyalty-service` | 会員・ポイント |
| S6 | `receipt-service` | レシート |
| S7 | `return-service` | 返品 |
| S8 | `checkout-orchestrator` | Checkout Saga |
| S9 | `cart-service` | カート |
| S10 | `identity-service` | 認証・ユーザー管理 |
| S11 | `audit-service` | 監査ログ |
| S12 | `dashboard-service` | CQRS Read Model |
| S13 | `api-gateway` | API Gateway |

さらに、設計書の範囲から実運用に近づけるために、`pos-frontend`も追加されています。

```text
services/
  ├── api-gateway/
  ├── audit-service/
  ├── cart-service/
  ├── catalog-service/
  ├── checkout-orchestrator/
  ├── dashboard-service/
  ├── identity-service/
  ├── inventory-service/
  ├── loyalty-service/
  ├── order-service/
  ├── payment-service/
  ├── pos-frontend/
  ├── receipt-service/
  └── return-service/
```

バックエンドだけを見ると、`src/main/java`配下のJavaファイルは266ファイルありました。

サービスごとのJavaファイル数は以下の通りです。

| サービス | Javaファイル数 |
|---|---:|
| api-gateway | 5 |
| audit-service | 9 |
| cart-service | 15 |
| catalog-service | 16 |
| checkout-orchestrator | 28 |
| dashboard-service | 6 |
| identity-service | 22 |
| inventory-service | 19 |
| loyalty-service | 41 |
| order-service | 22 |
| payment-service | 17 |
| receipt-service | 14 |
| return-service | 23 |

レガシー側では`OrderService`、`CheckoutSaga`、`ReturnSaga`に処理が集中していましたが、実装後はサービス境界ごとに責務が分散されています。


#### 各サービスの内部構造

各サービスの内部構造は、設計レポートで示された通りHexagonal Architectureに寄せられています。

Catalog Serviceを例にすると、以下のような構成です。

```text
services/catalog-service/src/main/java/com/example/pos/catalog/
  ├── application/
  ├── application/dto/
  ├── config/
  ├── domain/
  ├── domain/event/
  ├── infrastructure/
  ├── presentation/
  └── presentation/dto/
```

Checkout Orchestratorでは、外部サービス呼び出しのポートが明示されています。

```text
services/checkout-orchestrator/src/main/java/com/example/pos/checkout/
  ├── application/
  ├── application/port/
  ├── config/
  ├── domain/
  ├── domain/event/
  ├── infrastructure/
  ├── presentation/
  └── presentation/dto/
```

Return Serviceも同様に、アプリケーション層、ポート、ドメイン、インフラ、プレゼンテーションに分かれています。

```text
services/return-service/src/main/java/com/example/pos/returns/
  ├── application/
  ├── application/command/
  ├── application/port/
  ├── config/
  ├── domain/
  ├── domain/event/
  ├── infrastructure/
  ├── presentation/
  └── presentation/dto/
```

ここで重要なのは、単にディレクトリを分けたことではありません。

レガシーコードでは、Controller、Service、DAO、Sagaが相互に強く結合し、特にSagaが複数データベースと複数ドメインを直接操作していました。実装後の構成では、業務ルールはdomain / applicationに置き、ScalarDBやHTTP呼び出しはinfrastructureに閉じ込め、外部からの入力はpresentationに集約しています。

設計レポートで定義した境界コンテキストごとに責務を分けるという考え方が、コード構造として具体化されています。


#### 共通ライブラリ

`shared/`には、各サービスが共通して利用する部品が切り出されています。

```text
shared/
  ├── common/
  ├── outbox/
  ├── saga/
  └── security/
```

実際のJavaファイルは以下のように配置されています。

```text
shared/common/
  ├── DomainEvent.java
  ├── Money.java
  ├── Quantity.java
  ├── PosErrorCode.java
  └── PosException.java

shared/outbox/
  ├── TransactionalOutboxService.java
  ├── OutboxEntry.java
  ├── OutboxRepository.java
  ├── ScalarDbOutboxRepository.java
  └── OutboxPollingPublisher.java

shared/saga/
  ├── SagaContext.java
  ├── SagaOrchestrator.java
  ├── SagaStatus.java
  ├── SagaStep.java
  ├── ScalarDbLeaderLock.java
  └── SagaTimeoutEvaluatorBase.java

shared/security/
  ├── JwtAuthenticationFilter.java
  ├── PosSecurityAutoConfiguration.java
  ├── PosRole.java
  ├── PosUserDetails.java
  ├── RequiresStoreIsolation.java
  └── StoreIdEnforcementAspect.java
```

この共通化は、レガシーの`Utils`的な共通化とは意味が違います。

レガシー側の問題は、便利関数や横断処理が無秩序に広がり、境界を壊していた点にありました。今回の`shared/`は、値オブジェクト、例外体系、Outbox、Saga、Securityという、設計上の明確な理由があるものだけに限定されています。

特に`shared/outbox`と`shared/saga`は、今回のScalarDBベースのマイクロサービス化における中核です。

Order ServiceやPayment Serviceは、外部副作用をScalarDBトランザクションの中で直接実行せず、Outboxに記録します。Checkout OrchestratorやReturn Serviceは、Sagaの状態、タイムアウト、補償処理を扱います。

これにより、設計レビューで重視されていたScalarDB Tx内で外部APIを呼ばない、補償処理を追跡可能にする、二重補償を防ぐ、といった観点が、共通部品として実装に反映されています。

---

## ScalarDB・インフラ・フロントエンド移行を理解する

:::message
実装後のScalarDBスキーマ、Outbox/Saga、CI/CD、Kubernetes、フロントエンド移行をまとめて確認します。
:::

### ScalarDBを中心にしたデータ構成

#### namespaceごとのスキーマ

`db/scalardb-schemas/`には、各境界コンテキストに対応するScalarDBスキーマが配置されています。

```text
db/scalardb-schemas/
  ├── audit-schema.yaml
  ├── catalog-schema.yaml
  ├── checkout-leader-lease-schema.yaml
  ├── coordinator-schema.yaml
  ├── identity-schema.yaml
  ├── inventory-schema.yaml
  ├── loyalty-schema.yaml
  ├── order-schema.yaml
  ├── payment-schema.yaml
  ├── receipt-schema.yaml
  ├── return-leader-lease-schema.yaml
  └── return-schema.yaml
```

Nexus Architectの設計書では、MySQLとPostgreSQLに分かれた既存データを、ScalarDBのnamespaceによって境界コンテキスト単位で扱う方針でした。

実装では、その方針に沿って`catalog`、`inventory`、`order`、`payment`、`loyalty`、`receipt`、`return`、`identity`、`audit`といったnamespaceが作られています。

また、Sagaの分散スケジュールジョブに必要なleader lease用スキーマも追加されています。

```text
checkout-leader-lease-schema.yaml
return-leader-lease-schema.yaml
```

これは実装を進める中で、単にSagaの処理を作るだけでは足りないことが分かったためです。

複数レプリカでCheckout OrchestratorやReturn Serviceを動かす場合、タイムアウトしたSagaを複数プロセスが同時に検出し、同じ補償イベントを二重に発行する可能性があります。そのため、ScalarDBのCASを使ったleader lockを共通化し、単一レプリカだけが補償スキャンを実行する構成にしています。


#### OutboxとSagaの扱い

Order Serviceのplanでは、注文作成時に`OrderPlacedEvent`を同一ScalarDBトランザクション内でOutboxに書き込むことが明示されています。

Payment Serviceでは、`charge`、`refund`、`reverse`の各操作が状態遷移とOutbox書き込みを同じトランザクションで行うように計画されています。

Loyalty Serviceでも、会員登録、会員停止、ポイント加算、ポイント取消などのイベントをOutboxに記録する方針です。

これにより、以下の境界が明確になります。

```text
ScalarDB Tx内:
  - 業務データ更新
  - Outboxレコード書き込み

ScalarDB Tx外:
  - 外部API呼び出し
  - メール送信
  - 決済SaaS連携
  - 他サービスへの通知
```

設計・レビュー編で確認した手書きSagaの問題は、複数データベース更新と外部副作用と補償処理がひとつのクラスに混ざっていたことでした。

今回の実装では、その問題を以下のように分解しています。

- 業務データの原子的な更新はScalarDBが担う
- 副作用の起点はOutboxに記録する
- Sagaの進行や補償はOrchestrator / Return Serviceが担う
- タイムアウト検出はSagaTimeoutEvaluatorが担う
- 多重実行防止はScalarDbLeaderLockが担う

この分解によって、Sagaは巨大な手続きではなく、追跡可能な状態遷移として扱えるようになります。


### CI/CDと実行環境

#### Docker Compose

ローカル実行用には、`infrastructure/docker-compose/docker-compose.yml`が作られています。

`docs/solutions/conventions/docker-compose-local-startup.md`には、起動対象として以下が整理されています。

```text
5 infra containers
1 schema-loader
13 services
```

合計19コンテナを前提に、PostgreSQL、MySQL、Redis、ScalarDB Cluster、各サービスを起動する構成です。

ここまで整えると、コードだけでなく、サービス間通信、スキーマ適用、ヘルスチェック、起動順序の問題も見えてきます。

実際に`docs/solutions/`には、ScalarDB Clusterのhealthcheck不備によるstartup deadlockや、Dashboard Serviceのdatasource URLの誤りなど、実装中に遭遇した問題が記録されています。


#### GitHub ActionsとGitLabマルチレポ

今回の出力物には、GitHub ActionsだけでなくGitLabマルチレポ構成も含まれています。

```text
.github/workflows/ci.yml
ci-templates/
shared/.gitlab-ci.yml
services/*/.gitlab-ci.yml
infrastructure/.gitlab-ci.yml
docs/gitlab-group-structure.md
```

`docs/gitlab-group-structure.md`では、以下のようなグループ構成が定義されています。

```text
pos-microservice/
  ├── shared/
  ├── services/
  └── infrastructure/
```

サービスごとに独立したGitLabプロジェクトを持ち、共有ライブラリはPackage Registryで配布し、各サービスは共通CIテンプレートをincludeする構成です。

最初はモノレポとして実装を始めつつ、将来的にマルチレポへ分割できるようにしている点が特徴です。

これは、設計レポートで示された段階的移行と一致しています。最初から組織構造まで一気に分割すると運用コストが高くなりますが、コード、CI、パッケージ境界をあらかじめ整えておくことで、後から分割しやすい状態を作っています。


#### KubernetesとGitOps

`infrastructure/manifests/`には、Kustomizeを前提にしたKubernetesマニフェストも作られています。

また、後続のplanではArgo Rolloutsによるカナリアリリース、PrometheusのSLIを使った自動ロールバック、Argo CDのselfHealといった運用面の改善も扱っています。

このあたりは、単なるリファクタリングというより、マイクロサービス化後の運用設計です。

実装が進むほど、コードを分けるだけでは不十分であることが分かります。

サービスが増えると、障害時の切り戻し、監視、タイムアウト、リトライ、デプロイ順序、設定管理が重要になります。Compound Engineeringのplanは、その運用上の不足を後続タスクとして拾い上げ、実装に反映しています。


### フロントエンドもStrangler Figで移行する

今回の実装では、バックエンドだけでなく`pos-frontend`も作成されています。

これは、レガシーのThymeleaf画面をすぐに廃止するのではなく、Next.js App RouterベースのSPAを並走させ、段階的に置き換える方針です。

`docs/plans/2026-05-20-015-feat-pos-frontend-spa-phase1-plan.md`では、Phase 1として以下が実装対象になっています。

- ログイン画面
- HttpOnly Cookie認証
- RBACルートガード
- レジ画面
- カートの小計・税額・税込合計表示
- チェックアウト成功後のレシート画面
- 管理ダッシュボードのスタブ

続くPhase 2では、管理画面が追加されています。

- 商品管理
- 在庫管理
- 注文管理
- 返品処理
- ポイント管理
- ユーザー管理
- レシート検索
- パスワード変更
- KPIダッシュボード

実際の`services/pos-frontend/src/app/`には、以下のような画面が作られています。

```text
app/
  ├── login/
  ├── register/
  ├── register/receipt/
  └── admin/
      ├── dashboard/
      ├── inventory/
      ├── loyalty/
      ├── orders/
      ├── products/
      ├── receipts/
      ├── returns/
      └── users/
```

これも重要なポイントです。

マイクロサービス化の話では、バックエンドの境界だけに注目しがちです。しかし、実際のレガシーシステムでは、画面もまた密結合の一部です。

今回のケースでは、レジ画面と管理画面をSPAとして切り出し、API Gateway経由で新しいサービス群に接続することで、UI側もStrangler Figパターンで移行できるようにしています。

---

## 実装で得た学習とアーキテクチャの変化を理解する

:::message
実装中に蓄積した解決知識と、God Service・手書きSaga・DAO直接依存からの構造変化を整理します。
:::

### 実装中に蓄積された学習

#### docs/solutionsの役割

Compound Engineeringを使った実装で特に面白いのは、問題解決の記録が`docs/solutions/`に蓄積されている点です。

```text
docs/solutions/
  ├── architecture-patterns/
  ├── best-practices/
  ├── build-errors/
  ├── conventions/
  ├── database-issues/
  ├── integration-issues/
  ├── logic-errors/
  ├── runtime-errors/
  ├── security-issues/
  └── workflow/
```

実装中に起きた問題は、単にその場で修正されるだけでなく、再利用できる知見として記録されています。

たとえば、以下のようなメモがあります。

- `ScalarDB Result API returns primitives, not Optionals`
- `ScalarDB Put upsert partial column overwrite`
- `ScalarDB Transaction Lifecycle Pitfalls`
- `Identity Service Auth Security Bugs`
- `storeId Tenant Isolation`
- `Resilience4j Circuit Breaker + Bulkhead`
- `docker-compose local startup`
- `paths-filter gitignored directory`
- `zustand selector derived state loop`

これは、AIエージェントによる実装でありがちな同じ失敗を何度も繰り返す問題を抑えるうえで有効です。

たとえばScalarDBの`Put`による部分更新の扱い、トランザクションのabort/commitライフサイクル、`CommitConflictException`のリトライ方針などは、一度理解しても別のサービス実装で再発しやすいポイントです。

`docs/solutions/`に残しておくことで、後続のplanや実装時に参照できるようになります。


#### レビューで見つかった問題も実装に戻す

後半のplanには、単純な機能追加ではなく、レビューや運用観点から見つかった不足を補うものが含まれています。

たとえば、`inventory-service`のOCCリトライplanでは、ScalarDBの`CommitConflictException`がそのまま500系エラーになる問題が扱われています。

修正方針は以下です。

- `CommitConflictException`発生時に最大3回リトライする
- 各リトライでは新しいScalarDBトランザクションを開始する
- 上限超過時は`TOO_MANY_REQUESTS`として扱う
- HTTP 429にマッピングする

また、Circuit Breakerのplanでは、api-gatewayとcheckout-orchestratorにResilience4jのCircuit Breaker / Bulkhead / TimeLimiterを追加しています。

これは、下流サービス障害時にGatewayやOrchestratorがスレッドを使い切り、カスケード障害を起こすリスクを抑えるためです。

さらに、Saga Timeout Evaluatorのplanでは、タイムアウトしたSagaの二重補償、DLQ後の凍結、補償完了の未監視といった問題が扱われています。

つまり、実装後半では設計通りに作るだけでなく、実際に動かすと何が壊れるかを前提に、耐障害性と運用性を追加していっています。


### レガシー構造から何が変わったか

#### God Serviceから境界コンテキストへ

レガシーシステムでは、`OrderService`が976行に肥大化し、注文CRUD、統計、カート計算、返品チェックなどを抱えていました。

また、`CheckoutSaga`と`ReturnSaga`は、それぞれ複数データベース更新、外部副作用、補償処理をひとつのクラスで扱っていました。

実装後は、責務が以下のように分かれています。

| レガシー側の責務 | 実装後の配置 |
|---|---|
| 商品マスタ | Catalog Service |
| 在庫引当・戻し | Inventory Service |
| 注文作成・状態遷移 | Order Service |
| 決済・返金 | Payment Service |
| 会員・ポイント | Loyalty Service |
| レシート生成 | Receipt Service |
| 返品 | Return Service |
| チェックアウト全体の進行 | Checkout Orchestrator |
| カート | Cart Service |
| 認証・ユーザー管理 | Identity Service |
| 監査ログ | Audit Service |
| 集計・KPI | Dashboard Service |

これは、クラスを分割しただけではありません。

各サービスが、自分のデータ、API、ドメインモデル、テスト、Dockerfile、CI設定を持つ単位になっています。

設計レポートで定義した境界コンテキストが、実行可能な単位として実装されたと言えます。


#### 手書きSagaから追跡可能なSagaへ

レガシーのSagaでは、処理が失敗したときにどこまで成功したのか、どの補償が必要なのか、補償が二重に走らないかが追跡しにくい構造でした。

今回の実装では、Sagaの基底クラス、Sagaの状態、Saga Step、Timeout Evaluator、Leader Lock、Outboxイベントが分かれています。

これにより、以下のような運用上の問いに答えやすくなります。

- どのSagaがタイムアウトしているか
- どの補償ステップが未完了か
- どのレプリカが補償スキャンを実行しているか
- 補償イベントがOutboxに記録されたか
- 失敗時に再試行すべきか、エスカレーションすべきか

Sagaは、コード上の巨大な`try-catch`ではなく、観測可能な業務プロセスになります。


#### DAO直接依存からポート/アダプタへ

レガシーコードでは、ServiceやSagaがDAO、JdbcTemplate、複数DBを直接扱っていました。

実装後は、各サービスでdomain / application / infrastructure / presentationが分かれ、データアクセスはrepositoryポートとScalarDBアダプタに閉じ込められています。

たとえば、ドメイン層はScalarDBのAPIを知る必要がありません。アプリケーション層はユースケースを表現し、インフラ層がScalarDBの`Get`、`Put`、`Scan`、トランザクション境界を扱います。

この構成により、テストではアプリケーション層やドメイン層をDBなしで検証しやすくなります。

また、ScalarDB APIの細かい落とし穴は`docs/solutions/`に蓄積され、次のサービス実装に再利用されています。


### Compound Engineeringを使って分かったこと

#### 設計書は実装の終点ではなく、実装の起点になる

今回の実装で最も大きな気づきは、設計書を読むための文書で終わらせず、実装エージェントに渡すための入力として使えたことです。

Nexus Architectの設計書には、境界コンテキスト、ターゲットアーキテクチャ、ScalarDBスキーマ、Saga、Outbox、API Gateway、DR、インフラ構成が書かれていました。

それをそのまま巨大な実装指示にするのではなく、Compound Engineeringのplanによって、サービス単位・機能単位・リスク単位に分解しました。

その結果、設計書は以下のように使われました。

- `CLAUDE.md`に恒常的な実装ルールとして反映する
- `docs/design-references.md`に参照索引として整理する
- `docs/plans/`に作業単位として分解する
- `docs/solutions/`に実装中の知見を蓄積する
- CI/CDやDocker Composeで実行可能性を検証する

設計書が最初に作って終わりのものではなく、実装中に何度も参照される制約になっています。


#### AIに任せるほど、境界と完了条件が重要になる

AIエージェントは、広い範囲のコードを高速に生成できます。

しかし、だからこそ、境界が曖昧なまま実装させると、レガシーと同じような密結合を別の形で再生産してしまいます。

今回うまく機能したのは、各planに以下が含まれていたからです。

- 何を実装するか
- 何を実装しないか
- どの設計書を参照するか
- どのサービスに責務を置くか
- どのAPI / schema / eventを作るか
- どのコマンドが通れば完了とするか
- 後続作業に何をdeferするか

特にDeferred to Follow-Up Workが明示されている点は重要です。

AIに作業を任せると、親切心でスコープ外のものまで作ろうとすることがあります。今回のplanでは、PDF生成、メール送信、Kafka移行、外部決済SaaS連携、DRリージョンなどを明確に後回しにしていました。

結果として、各実装単位が大きくなりすぎず、段階的に進めることができています。


#### 実装知識をcompoundingする

Compound Engineeringという名前の通り、今回の実装では知識が複利的に効いています。

最初にCatalog Serviceで作ったHexagonal Architectureのパターンは、Inventory、Order、Paymentに再利用されます。

ScalarDBのResult APIやPutの挙動でつまずいた内容は、`docs/solutions/`に残り、後続サービスで同じ間違いを避けるために使われます。

フロントエンドで発見したNext.js App Router、Server Actions、RBAC、Cookieセキュリティのパターンも、管理画面の追加実装に再利用されます。

この解いた問題を次の問題の足場にする流れが、今回のような大規模なリファクタリング実装ではかなり効いています。

---

## この記事のまとめ

- Compound Engineeringでは、設計書をそのまま実装へ投げるのではなく、実装計画、サービス構成、共通ライブラリ、禁止事項へ分解してAIエージェントに渡します。
- リファクタリング計画では、13サービス構成、ScalarDBスキーマ、Outbox、CI/CD、Kubernetes、API Gateway、フロントエンド移行を段階的に整理しました。
- 実装中の学習を設計へ戻すことで、AI駆動開発を一方通行のコード生成ではなく、設計と実装が相互に改善されるプロセスとして扱えるようになります。

## 用語解説

### planディレクトリ
Compound Engineeringにおいて、AIエージェントに実装を進めさせるための作業計画の起点となるファイル群を配置するディレクトリです。対象、順序、制約、完了条件を明確にします。

### docs/solutionsディレクトリ
Compound Engineeringにおいて、実装中に発生した問題の解決策や得られた知見を蓄積し、後続の作業で前提情報として再利用するためのディレクトリです。

### 共通ライブラリ
複数サービスで共有するドメイン部品、ScalarDB接続、Outbox処理、エラー処理などをまとめる部品群です。

### API Gateway
複数サービスへの入口を集約し、認証、ルーティング、レート制限、ログ、BFF責務などを担うコンポーネントです。

### Kubernetes
複数サービスをコンテナとしてデプロイ、スケール、管理するための基盤です。
