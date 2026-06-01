---
title: "ScalarDB・インフラ・フロントエンド移行を理解する"
---

:::message
実装後のScalarDBスキーマ、Outbox/Saga、CI/CD、Kubernetes、フロントエンド移行をまとめて確認します。
:::

## ScalarDBを中心にしたデータ構成

### namespaceごとのスキーマ

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


### OutboxとSagaの扱い

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


## CI/CDと実行環境

### Docker Compose

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


### GitHub ActionsとGitLabマルチレポ

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


### KubernetesとGitOps

`infrastructure/manifests/`には、Kustomizeを前提にしたKubernetesマニフェストも作られています。

また、後続のplanではArgo Rolloutsによるカナリアリリース、PrometheusのSLIを使った自動ロールバック、Argo CDのselfHealといった運用面の改善も扱っています。

このあたりは、単なるリファクタリングというより、マイクロサービス化後の運用設計です。

実装が進むほど、コードを分けるだけでは不十分であることが分かります。

サービスが増えると、障害時の切り戻し、監視、タイムアウト、リトライ、デプロイ順序、設定管理が重要になります。Compound Engineeringのplanは、その運用上の不足を後続タスクとして拾い上げ、実装に反映しています。


## フロントエンドもStrangler Figで移行する

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

## 本章のまとめ

* 実装後の構成では、ScalarDBスキーマ、Outbox/Saga、Docker Compose、CI/CD、Kubernetes、フロントエンドがそろいました。
* バックエンドだけでなく、管理画面やレジ画面もNext.jsのSPAとして切り出し、API Gateway経由で新サービス群へ接続しています。
* フロントエンドもStrangler Figパターンで移行することで、画面側の密結合も段階的にほどける構成になりました。

## 用語解説

### Strangler Figパターン
既存システムを一度に置き換えず、新しい機能や画面から少しずつ外側に切り出していく移行パターンです。

### Docker Compose
複数のコンテナをまとめて起動・接続するための仕組みです。開発環境でサービス群やDBを動かすときに使います。

### Kubernetes
コンテナ化されたアプリケーションを本番環境で配置、監視、スケールさせるための基盤です。

### CI/CD
テスト、ビルド、デプロイを自動化する仕組みです。マイクロサービス化後は、サービス単位で安全に変更を反映するために重要になります。

### SPA
Single Page Applicationの略で、画面遷移や表示更新をブラウザ側で行うWebアプリケーションです。
