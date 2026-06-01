---
title: "実装で得た学習とアーキテクチャの変化を理解する"
---

:::message
実装中に蓄積した解決知識と、God Service・手書きSaga・DAO直接依存からの構造変化を整理します。
:::

## 実装中に蓄積された学習

### docs/solutionsの役割

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


### レビューで見つかった問題も実装に戻す

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


## レガシー構造から何が変わったか

### God Serviceから境界コンテキストへ

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


### 手書きSagaから追跡可能なSagaへ

レガシーのSagaでは、処理が失敗したときにどこまで成功したのかどの補償が必要なのか補償が二重に走らないかが追跡しにくい構造でした。

今回の実装では、Sagaの基底クラス、Sagaの状態、Saga Step、Timeout Evaluator、Leader Lock、Outboxイベントが分かれています。

これにより、以下のような運用上の問いに答えやすくなります。

- どのSagaがタイムアウトしているか
- どの補償ステップが未完了か
- どのレプリカが補償スキャンを実行しているか
- 補償イベントがOutboxに記録されたか
- 失敗時に再試行すべきか、エスカレーションすべきか

Sagaは、コード上の巨大な`try-catch`ではなく、観測可能な業務プロセスになります。


### DAO直接依存からポート/アダプタへ

レガシーコードでは、ServiceやSagaがDAO、JdbcTemplate、複数DBを直接扱っていました。

実装後は、各サービスでdomain / application / infrastructure / presentationが分かれ、データアクセスはrepositoryポートとScalarDBアダプタに閉じ込められています。

たとえば、ドメイン層はScalarDBのAPIを知る必要がありません。アプリケーション層はユースケースを表現し、インフラ層がScalarDBの`Get`、`Put`、`Scan`、トランザクション境界を扱います。

この構成により、テストではアプリケーション層やドメイン層をDBなしで検証しやすくなります。

また、ScalarDB APIの細かい落とし穴は`docs/solutions/`に蓄積され、次のサービス実装に再利用されています。


## Compound Engineeringを使って分かったこと

### 設計書は実装の終点ではなく、実装の起点になる

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


### AIに任せるほど、境界と完了条件が重要になる

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


### 実装知識をcompoundingする

Compound Engineeringという名前の通り、今回の実装では知識が複利的に効いています。

最初にCatalog Serviceで作ったHexagonal Architectureのパターンは、Inventory、Order、Paymentに再利用されます。

ScalarDBのResult APIやPutの挙動でつまずいた内容は、`docs/solutions/`に残り、後続サービスで同じ間違いを避けるために使われます。

フロントエンドで発見したNext.js App Router、Server Actions、RBAC、Cookieセキュリティのパターンも、管理画面の追加実装に再利用されます。

この解いた問題を次の問題の足場にする流れが、今回のような大規模なリファクタリング実装ではかなり効いています。


## 本章のまとめ

本ページでは、レビュー完了となったNexus Architectの設計書をもとに、Compound Engineeringを使って実際に`pos-microservices`を実装した流れを見てきました。

最終的な成果物は、13個のSpring Bootマイクロサービス、Next.jsフロントエンド、ScalarDBスキーマ、共通ライブラリ、Docker Compose、GitHub Actions、GitLabマルチレポ構成、Kubernetesマニフェスト、運用改善planまで含む、かなり実践的な構成になっています。

このページで重要なのは、AIが大量のコードを生成できたことそのものではありません。

重要なのは、レビュー済みの設計を、実装可能な小さなplanに分解し、実装中の失敗を`docs/solutions/`に蓄積しながら、段階的にコードベースを育てていったことです。

レガシーリファクタリングにおいて、設計と実装は分離した工程ではありません。

設計は実装の制約になり、実装は設計の不足を発見し、その不足は次のplanとsolutionに反映されます。

この循環を回せる状態を作れたことが、今回のCompound Engineering実践における最大の成果だと感じています。

## 用語解説

### `docs/solutions/`
実装中に見つかった問題と解決策を残す知識ベースです。後続のサービス実装で同じ失敗を繰り返さないために使います。

### OCCリトライ
楽観的同時実行制御で競合が発生したときに、処理を再試行する仕組みです。ScalarDBでは`CommitConflictException`への対応として重要になります。

### Bulkhead
障害が起きた処理の影響を他の処理へ広げないため、リソースや実行枠を分離する考え方です。

### Deferred to Follow-Up Work
今回の実装範囲から明示的に外し、後続作業として扱う項目です。スコープの膨張を防ぐために重要です。
