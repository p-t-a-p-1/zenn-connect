---
title: "【連載】（第2回）POSとマイクロサービス：ScalarDBで実現する異種DB間の2PC実装"
emoji: "🍣"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["pos", "マイクロサービス", "scalardb", "2pc", "分散トランザクション"]
published: false
publication_name: "scalar_sol_blog"
---

# はじめに

前回の記事では、POSシステムをマイクロサービス化する意義と、そこで直面する分散トランザクション（特にSagaパターン）の課題について解説しました。

@[card](https://zenn.dev/scalar_sol_blog/articles/pos-microservices-saga-pattern)

Sagaパターンは強力な手法ですが、個別の補償処理の実装や複雑な状態管理によって、
コードの複雑化や保守コストの増大という辛さがありました..！

今回は、その辛さを解消する、異種データベース（PostgreSQL / Cassandra / DynamoDBなど）間の分散トランザクションをシンプルかつ安全に実現する**ScalarDB**の実装方法を紹介します！！

## 対象読者

- マイクロサービスにおける分散トランザクションの実装に悩んでいる人
- Sagaパターンの複雑さに限界を感じている人
- PostgreSQL、Cassandra、DynamoDBなどの異なるDBをまたいだトランザクションを実現したい人
- **AI駆動開発と相性の良いアーキテクチャを探している人**

## 用語解説

### 2PC（2フェーズコミット）とは

分散トランザクションを実現するためのプロトコルの一つです。
すべてのデータベースに対してコミットの準備ができるかを確認するフェーズ（Prepare）と、全員が準備完了と答えた場合にのみ実際にコミットするフェーズ（Commit）の2段階に分けて処理を行います。
Sagaパターンのように途中で処理を元に戻す（補償する）のではなく、全員が成功できる状態になってから一斉に処理を確定させるため、データの一貫性を強固に保つことができます。

### ACIDトランザクションとは

データベースの処理が満たすべき4つの特性（Atomicity: 原子性、Consistency: 一貫性、Isolation: 独立性、Durability: 永続性）の頭文字をとったものです。
簡単に言うと、すべての処理が成功するか、すべて失敗するかのどちらかが保証され、処理途中の矛盾したデータが他から見えないようにする仕組みです。

こちらの方の記事がわかりやすかったです。
@[card](https://zenn.dev/atusi/articles/b74c3644cb8de5)

### OCC（楽観的並行性制御）とは

複数の処理が同時に同じデータを更新しようとした場合の競合を防ぐ仕組みの一つです。
たぶん他の処理とぶつからないだろうと楽観的に処理を進め、最後にデータを更新（コミット）する直前に他の処理によってデータが書き換えられていないかをチェックします。もし書き換えられていれば、処理をロールバックして最初からやり直す（リトライする）アプローチです。

# Sagaパターンの辛さ おさらい

分散トランザクションといえばSagaパターンを使うことがありますが、
POSシステムのような在庫引当と決済が絡むケースでは、
実際に運用に乗せようとすると多くの課題に直面します。

たとえば、在庫引当（DBはCassandra）が成功した後に決済（DBはDynamoDB）でエラーが起きたとします。
Sagaパターンの場合、在庫を元に戻す補償トランザクションを自前で実装し、確実に実行させなければなりません...

もし在庫を戻す処理自体がネットワークエラーで失敗したら？リトライキューやバッチでの定期実行など、インフラからアプリまで考慮事項がたくさんあるのかなと思います...

:::message alert
**Sagaパターンの辛いところ**

- 各サービスごとに取り消し処理（補償ロジック）を自作しないといけない
- どこまで処理が進んだかの状態管理（ステートマシン）が必要
- 補償処理自体が失敗した時の対応（リトライ、デッドレター等）が複雑
  :::

特に最近の開発ではAI（Claude CodeやCursor、Antigravityなど）にコードを書かせることが多いと思いますが、Sagaパターンの場合、**冪等性の担保や部分的な失敗時のロールバック手順など、AIに渡すべき前提条件（コンテキスト）が複雑すぎ**て、うまく自動生成できないという課題がありました。

AI駆動開発の観点からも、ロジックは極力シンプルに保ちたいところです...！

# ScalarDBの強み

そういった問題を解決できるのが**ScalarDB**です。
POSシステムでは商品管理（PostgreSQL）、在庫管理（Cassandra）、注文・支払管理（DynamoDB）のようにデータの性質に合わせて最適なDBを選びますが、ScalarDBを使えばこれらを一つの**ACIDトランザクション**として扱えます。

| 比較項目           | Sagaパターン                     | ScalarDB（2PC）                      |
| ------------------ | -------------------------------- | ------------------------------------ |
| **補償処理の実装** | 各サービスに個別実装が必須       | **不要**（自動ロールバック）         |
| **APIの統一性**    | 各DBの独自APIを扱う              | **統一API**で抽象化                  |
| **データ整合性**   | 障害時のリカバリ設計が複雑       | OCC + 2PCで強固な整合性を保証        |
| **AIとの親和性**   | 詳細な指示が必要で自動生成が困難 | ロジックがシンプルで**AI支援が容易** |

ScalarDBを導入して補償ロジックを手放せることで、**本質的なビジネスロジックの開発に集中できる**ようになります...！！

@[card](https://scalardb.scalar-labs.com/ja-jp/docs/latest/overview#scalardb-%E3%82%92%E9%81%B8%E3%81%B6%E7%90%86%E7%94%B1)

# POSシステムにおける異種DB間の2PC実装について

では、実際にPOSシステムのデモアプリケーションでどのようにScalarDBを利用しているのかを見ていきましょう。
チェックアウト処理では、以下の3つのサービスが協調して2PCによるトランザクションを実行します。

![2PCのアーキテクチャ図](/images/pos-microservices-scalardb-2pc/2pc-architecture.png)

### サービスの役割とフロー

| 役割            | サービス          | ストレージ | 概要                                                                         |
| --------------- | ----------------- | ---------- | ---------------------------------------------------------------------------- |
| **Coordinator** | order-service     | DynamoDB   | `begin()`でトランザクションを開始し、トランザクションIDを各Participantに渡す |
| **Participant** | inventory-service | Cassandra  | 渡されたIDでトランザクションに参加（`join()`）し、在庫引当を実行             |
| **Participant** | payment-service   | DynamoDB   | 渡されたIDでトランザクションに参加（`join()`）し、支払処理を実行             |

Coordinatorである`order-service`が中心となり、ParticipantのサービスへgRPCを通じてトランザクションID（`txId`）を伝播させます。

# ScalarDBを利用した実際のコード例

Sagaパターンでは異常系の処理で大量のコードを書く必要がありました。
しかし、ScalarDBの2PCを利用したデモアプリの `CheckoutCoordinatorService.java` を見てみると、チェックアウト処理はシンプルに記述されていることがわかります...！

```java
// order-service: CheckoutCoordinatorService.java
public CheckoutResponse doCheckout(CheckoutRequest request) {
    int retryCount = 0;
    while (true) {
        TwoPhaseCommitTransaction tx = null;
        try {
            // 1. トランザクション開始
            tx = txManager.begin();
            String txId = tx.getId();

            // 2. Coordinator ローカル操作（注文 + 明細登録）
            orderRepository.save(tx, order);

            // 3. Inventory Participant（在庫引当 — gRPC 2PC）
            // トランザクションID（txId）を渡して同一トランザクションに参加させる
            ReserveStockResponse stockResp = inventoryClient.reserveStock(txId, reservations);
            if (!stockResp.getSuccess()) {
                tx.rollback();
                return CheckoutResponse.failed("在庫不足です");
            }

            // 4. Payment Participant（決済 — gRPC 2PC）
            ProcessPaymentResponse payResp = paymentClient.processPayment(txId, ...);
            if (!payResp.getSuccess()) {
                tx.rollback();
                return CheckoutResponse.failed("決済に失敗しました");
            }

            // 5. 2PCのコミットフェーズ
            tx.prepare();
            tx.validate();
            tx.commit();  // 成功時はすべて一括コミットされる

            return CheckoutResponse.success(...);

        } catch (CrudConflictException | CommitConflictException | PreparationConflictException e) {
            // OCC（楽観的並行性制御）による競合時は自動ロールバックしてリトライ
            rollbackQuietly(tx);
            if (++retryCount > MAX_RETRIES) throw e;
            sleepQuietly(backoff); // バックオフを入れてリトライ
        } catch (Exception e) {
            // 予期せぬエラー時も、tx.rollback() だけで安全に全DBを元の状態に戻せる
            rollbackQuietly(tx);
            throw new TransactionFailedException("Checkout failed", e);
        }
    }
}
```

:::message
**ポイント**
開発者が在庫を戻す支払いをキャンセルするといった**補償ロジックを書く必要は一切ない**です...！！
エラーが発生した場合は`tx.rollback()`を呼び出すだけで、ScalarDBが自動的にPostgreSQL、Cassandra、DynamoDB間の整合性を保ちながらロールバックを行ってくれます。さらにOCC（楽観的並行性制御）の競合時リトライもシンプルに書けます！
:::

# おわりに

本記事では、Sagaパターンが抱えていた複雑なエラーハンドリングや補償ロジックの課題を、ScalarDBの分散トランザクション（2PC）によっていかにシンプルに解決できるかを解説しました。

マイクロサービスアーキテクチャにおいて、ScalarDBを使えば各サービスに最適なDBを選択する自由とシステム全体のデータの一貫性をシンプルに両立させられます。
さらに、ロジックから複雑な状態管理が消え去るため、AIにコーディングを任せやすいシンプルな構造を作れるScalarDBは、これからのAI駆動開発において非常に強力な選択肢になると考えています...！
