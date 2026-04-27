---
title: "【連載】（第3回）POSとマイクロサービス：ScalarDB Analyticsで実現する異種DB間の横断データ分析"
emoji: "🏬"
type: "tech"
topics: ["pos", "マイクロサービス", "scalardb", "データ分析", "sql"]
published: false
publication_name: "scalar_sol_blog"
---

# はじめに

過去の記事では、POSシステムのマイクロサービスにおける分散トランザクション（更新処理）の課題と、その解決策について解説してきました。

@[card](https://zenn.dev/scalar_sol_blog/articles/pos-microservices-saga-pattern)
@[card](https://zenn.dev/scalar_sol_blog/articles/pos-microservices-scalardb-2pc)

今回はトランザクションの話からは少し離れて、データ分析における分散DBの課題と、その解決策として**ScalarDB Analytics**の活用方法を紹介します！！

## 対象読者

- マイクロサービス化によってデータが分散し、分析や集計に苦労している人
- 複数のDB（RDBやNoSQL）をまたいだデータ抽出を効率化したい人
- アプリケーション側での複雑なJOIN処理（インメモリ結合）をなくしたい人
- **AI駆動開発で分析機能を実装したい人**

## 用語解説

### N+1クエリとは

データベースからデータを取得する際、1回のクエリでN件のデータを一覧取得した後、そのN件それぞれの関連データを取得するためにN回の追加クエリが実行されてしまう問題です。
結果としてN + 1回のクエリが発行され、通信オーバーヘッドによって著しいパフォーマンス低下を引き起こします。

### インメモリ結合とは

データベース側でJOIN処理（テーブル同士の結合）を行うのではなく、複数のデータベースからそれぞれデータをアプリケーションサーバーのメモリ上に全件取得し、プログラム（JavaのStream APIなど）でデータを突き合わせて結合する処理のことです。
データ量が増えるとメモリを圧迫しやすくなります。

### OOM（Out Of Memory）とは

アプリケーションが利用できるメモリ容量の限界を超えてしまい、システムが強制終了したり正常に動作しなくなるエラーのことです。

### MCP（Model Context Protocol）とは

ClaudeやCursorなどのAIエージェントが、外部のデータソースやツールと安全に連携するための標準プロトコルです。
これを使うことで、AIが直接データベースのスキーマ情報を読み取ったり、クエリを実行してデータを取得したりできるようになります。

# マイクロサービスにおけるデータ分析の辛さ

POSシステムでは、商品マスタ（PostgreSQL）、在庫データ（Cassandra）、注文履歴（DynamoDB）のように、各マイクロサービスがそれぞれのドメインに最適なデータベースを選択しています。

それぞれのサービスが独立して動いてはいますが、運用フェーズで問題になるのが分析です。
例えば、店舗運営において非常に重要な欠品リスクの分析（どの商品がいつ売り切れそうか）を行いたい場合、これら3つの異なるDBすべてのデータが必要になります。

従来のアプローチでは、アプリケーション側で以下のような実装をせざるを得ませんでした。

```java
// 1. PostgreSQL から商品情報を取得
List<Product> products = productRepository.findAll();

// 2. Cassandra から在庫情報を取得
List<Inventory> inventories = inventoryRepository.findAll();

// 3. DynamoDB から直近の注文履歴を取得
List<Order> orders = orderRepository.findByDateRange(startDate, endDate);

// 4. アプリケーションコードで無理やり結合・集計...（辛い）
Map<String, Integer> salesByProduct = orders.stream()
    .flatMap(order -> order.getItems().stream())
    .collect(Collectors.groupingBy(
        OrderItem::getProductId,
        Collectors.summingInt(OrderItem::getQuantity)
    ));

// 5. 在庫と突き合わせて欠品リスクを計算
List<StockoutRisk> risks = products.stream()
    .map(product -> {
        int sales = salesByProduct.getOrDefault(product.getId(), 0);
        int stock = inventories.stream()
            // ※ここでN+1的な処理が発生したり、ループが複雑になる
            .filter(inv -> inv.getProductId().equals(product.getId()))
            .findFirst()
            .map(Inventory::getQuantity)
            .orElse(0);
        return new StockoutRisk(product, sales, stock);
    })
    .collect(Collectors.toList());
```

:::message alert
**インメモリ結合の辛いところ**

- 各DBから個別に大量のデータを引っ張ってくるためネットワーク負荷が高い -> N+1クエリと通信オーバーヘッド
- Stream APIなどを駆使して手動でJOINや集計を書く必要があり、可読性が低くバグを生みやすい -> 実装の複雑さ
- メモリ上でデータを突き合わせるため、データ量が増えると一瞬でOOM（Out Of Memory）の危機に瀕する -> パフォーマンス劣化
  :::

# ScalarDB Analyticsの凄さ

そういった課題を解決できるのが**ScalarDB Analytics**です。
ScalarDB Analyticsを使えば、PostgreSQL、Cassandra、DynamoDBといった**異なるデータベースを横断するクエリを単一のSQLで記述**できます。

エンジン内部でApache Spark等の技術が使われており、分散JOINや集計をDBエンジン側で最適化して実行してくれます。

### 具体的なSQLの実装例

先ほどJavaのStream APIで書いていた**欠品リスク分析**の処理を、ScalarDB Analyticsで記述すると以下のようなSQLになります。シンプルです..！

```java
// Spark SQL で 3DB を横断クエリ（集計・結合はエンジンが最適化！！）
Dataset<Row> result = spark.sql("""
    SELECT
        p.product_name,
        COUNT(oi.product_id) AS total_sales,
        i.available_quantity AS current_stock,
        ROUND(i.available_quantity / (COUNT(oi.product_id) / 30.0), 1) AS days_until_stockout
    FROM order_service.order_items oi
    JOIN product_service.products p ON oi.product_id = p.product_id
    JOIN inventory_service.inventories i ON oi.product_id = i.product_id
    WHERE oi.ordered_at >= current_date() - INTERVAL 30 DAYS
    GROUP BY p.product_id, p.product_name, i.available_quantity
    HAVING days_until_stockout < 7
    ORDER BY days_until_stockout ASC
""");

List<StockoutRisk> risks = result.as(Encoders.bean(StockoutRisk.class)).collectAsList();
```

複雑なJavaのロジックが消え去り、宣言的なSQLだけが残りました。

<!-- 画像メモ: 従来のJavaコード（複雑なスパゲッティ状態）と、ScalarDB AnalyticsのシンプルなSQL（スマートな状態）を比較する図解 -->

# Claude CodeとScalarDB Analyticsの統合（MCPサーバー化）

ScalarDB Analyticsの利用において、**MCP（Model Context Protocol）サーバー化によるAIとの連携**が可能です。

MCPは、ClaudeやCursorなどのAIエージェントが、外部のデータソースやツールと安全に連携するための標準プロトコルです。
ScalarDB AnalyticsをこのMCPサーバーとしてAIに繋ぐと、これまでのデータ分析の常識が変わります...！

### 複数DBをまたいだ分析をAIに丸投げできる

通常、AIに現在の在庫リスクを教えてと聞いても、AIはデータベースに直接アクセスできないため答えることができません。
しかし、ScalarDB AnalyticsをMCPサーバーとして提供すると、以下のようなフローが実現します。

1. **人間のプロンプト**: 直近30日の売上トレンドを元に、1週間以内に在庫切れしそうな商品をリストアップして！
2. **AIの自律アクション**:
   - MCP経由でスキーマ情報を取得する
   - 目的を達成するための横断SQL（Spark SQL）を自ら組み立てる
   - MCP経由でScalarDB AnalyticsにSQLを実行させる
   - 取得した結果を自然言語で分析し、人間に回答する

## 実際のデモ

少し脱線しますが、今回はMCPサーバーとの連携にKeycloakを利用して、OAuth 2.1による自動認証を行いました。
Claude Code上で、`/mcp`と入力し、MCPの設定で、`Authentication`を選択すると、ブラウザが自動起動し Keycloak ログイン画面が表示されます。

![MCPサーバーの認証1](/images/pos-microservices-scalardb-analytics/demo_sa_mcp_auth_1.png)

![MCPサーバーの認証2](/images/pos-microservices-scalardb-analytics/demo_sa_mcp_auth_2.png)

認証後は Bearer トークンが自動付与されツールが即座に使用可能になります。

実際には、以下のようなプロンプトを入力し、AIに分析タスクを依頼します。

![利用例1](/images/pos-microservices-scalardb-analytics/demo_sa_mcp_usage_1.png)

AIはMCP経由でScalarDB Analyticsに横断SQLを投げ、結果を取得。そして、自然言語で回答してくれます。

![利用例2](/images/pos-microservices-scalardb-analytics/demo_sa_mcp_usage_2.png)

時間帯別の売上分析はこのように自然言語ベースで依頼して、集計結果をグラフで表示させることも可能です。

単なる分析機能にとどまらず、AIエージェントに直接業務分析を行わせるフローが、ScalarDB AnalyticsとMCPの組み合わせによって手軽に実現できるようになります！！

# まとめ

全3回にわたり、POSシステムを例にマイクロサービス化における課題とその解決策を紹介してきました。

マイクロサービスアーキテクチャは各サービスに最適なDBを選べるという大きなメリットがある反面、データの一貫性（更新）やデータの結合（参照）といった重い課題を抱えがちです。

トランザクション処理に特化したScalarDBと、分析処理に特化したScalarDB Analyticsはそれぞれ独立したプロダクトですが、これらを活用することでマイクロサービス特有の課題を大きく軽減できます。

ぜひ、本質的なビジネスロジックの開発や、AIを活用した生産性向上を目指すチームでScalarDBの活用を検討してみてください！！
