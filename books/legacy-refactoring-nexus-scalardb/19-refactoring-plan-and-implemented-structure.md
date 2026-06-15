---
title: "リファクタリング計画と実装構成を理解する"
---

:::message
Compound Engineeringで作成した実装planと、その結果としてできた13サービス・フロントエンド・共通ライブラリの構成を確認します。
:::

## 実装の進め方

### planを小さく分ける

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


### planの中身はかなり具体的

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


## 実装された構成

### 13サービス + フロントエンド

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


### 各サービスの内部構造

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


### 共通ライブラリ

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

## 本章のまとめ

* 実装は大きな一括作業ではなく、`docs/plans/`に残された小さなplan単位で進められました。
* 成果物として、13個のSpring Bootサービス、Next.jsフロントエンド、ScalarDBスキーマ、共通ライブラリが構成されました。
* `shared/`は便利関数置き場ではなく、値オブジェクト、Outbox、Saga、Securityなど設計上の理由がある共通部品に限定されています。

## 用語解説

### `shared/common`
サービス横断で使う値オブジェクト、例外、識別子などをまとめた共通モジュールです。業務境界を壊さない範囲に絞って使います。

### Hexagonal Architecture
業務ロジックを中心に置き、DBや外部APIなどの技術詳細をポートとアダプターの外側へ分離する設計です。

### Saga Orchestrator
複数サービスにまたがる業務プロセスの順序、状態、補償処理を管理するコンポーネントです。

### Outbox Publisher
Outboxに保存されたイベントを読み取り、外部の購読者や別サービスへ配送する仕組みです。
