---
title: "【Nexus Architect本レビュー用】【連載】（第9回）現状把握レポートを読み解く"
emoji: "📘"
type: "tech"
topics: ["architecture","scalardb","refactoring"]
published: false
publication_name: "scalar_sol_blog"
---


:::message
現状把握レポートから、コード構造、DB構成、DDD準備度、技術的負債、ScalarDB適用余地を読み解きます。
:::


ここからは、Nexus Architectを実行して生成されたレポートを数ページに分けて深掘りしていきます。

```
reports/
  ├── before/
  ├── 00_summary/
  ├── 01_analysis/
  ├── 02_evaluation/
  ├── 03_design/
  └── review/
```

```
現状把握（00_summary/） → ドメイン分析（01_analysis） → 評価（02_evaluation） → 設計（03_design） → レビュー（review/）
```

このページでは、最初の現状把握レポートを扱います。ドメイン分析と評価・採点のレポートは、後続ページで読み解きます。


## 現状把握

### 各レポート内容の概要

Nexus Architectを実行して生成されたreports/before/配下のマークダウン文書群を詳細に分析すると、既存システムが抱える構造的および実装レベルの問題が、具体的な指標やログとして明確に記述されていることが分かります。


出力されるレポートは以下の4種類です。

| ファイル | 内容 |
| :--- | :--- |
| codebase-structure.md | ディレクトリ構成・モジュール構造・主要クラスのマッピング |
| ddd-readiness.md | DDD 適用可能性の評価（集約・境界コンテキスト・ユビキタス言語の整備状況） |
| issues-and-debt.md | 技術的負債・コード品質問題・アーキテクチャ上の課題一覧 |
| technology-stack.md | 使用フレームワーク・ライブラリ・インフラ構成のインベントリ |

実際の出力結果を全てお見せして、一つずつみていきます。

### ディレクトリ構成・モジュール構造・主要クラスのマッピング

![codebase-structure](/images/legacy-refactoring-nexus-scalardb/codebase-structure.png)
*ファイル統計やディレクトリ構造の調査*

::::details レポート全文
# コードベース構造分析 — legacy-pos-monolith

## ファイル統計

| 種別 | 件数 |
|---|---|
| Java ソースファイル | 56 |
| Thymeleaf テンプレート | 29 |
| SQL マイグレーション（PG） | 6 |
| SQL マイグレーション（MySQL） | 5 |
| テストファイル | 3 |
| 設定ファイル | 2（application.properties + application.yml の混在） |

---

## ディレクトリ構成

```
src/main/java/com/example/legacypos/
├── LegacyPosApplication.java           # エントリポイント
├── config/
│   ├── DataSourceConfig.java           # 2 DataSource + 2 JdbcTemplate + 2 Flyway の手動設定
│   └── WebMvcConfig.java               # 静的リソース設定
├── web/                                # ⚠️ Controller（画面 + JSON API 混在）
│   ├── RegisterController.java         # レジ画面 + /api/register/** 混在
│   ├── AdminInventoryController.java
│   ├── AdminOrderController.java
│   ├── AdminPointController.java
│   ├── AdminProductController.java
│   ├── AdminReceiptController.java
│   ├── AdminReturnController.java
│   ├── AdminUserController.java
│   ├── DashboardController.java
│   ├── LoginController.java
│   └── GlobalExceptionHandler.java
├── service/
│   ├── OrderService.java               # ⚠️ God Service（976行）— 注文全責務
│   ├── CartService.java                # カート（セッションスコープ相当）
│   ├── InventoryService.java
│   ├── PointService.java
│   ├── ProductService.java
│   ├── ReceiptService.java
│   ├── AuditService.java
│   └── UserService.java
├── saga/
│   ├── CheckoutSaga.java               # ⚠️ God Class（557行）— チェックアウト全ステップ
│   └── ReturnSaga.java                 # ⚠️ God Class（448行）— 返品全ステップ
├── dao/                                # ⚠️ パッケージ命名揺れ（dao と repository 両方存在）
│   ├── AuditLogDao.java
│   ├── IdempotencyKeyDao.java
│   ├── InventoryDao.java
│   ├── OrderDao.java
│   ├── PaymentDao.java
│   ├── ProductDao.java
│   ├── ReturnDao.java
│   └── UserDao.java
├── repository/                         # ⚠️ dao と並存する 2 つ目の DAO パッケージ
│   ├── PointDao.java
│   └── ReceiptDao.java
├── domain/                             # POJO（エンティティ/DTO 混在）
│   ├── AuditLog.java
│   ├── IdempotencyKey.java
│   ├── Inventory.java
│   ├── Order.java
│   ├── OrderItem.java
│   ├── Payment.java
│   ├── PointBalance.java
│   ├── PointRule.java
│   ├── PointTransaction.java
│   ├── Product.java
│   ├── Receipt.java
│   ├── Return.java
│   ├── ReturnItem.java
│   ├── StockMovement.java
│   └── User.java
├── security/
│   ├── SecurityConfig.java
│   └── PosUserDetailsService.java
└── util/
    ├── Utils.java                      # ⚠️ 手書き JSON, MD5, static initializer で System.out.println
    └── CommonUtil.java                 # ⚠️ Utils と重複するユーティリティ
```

---

## データベーススキーマ構成

### PostgreSQL（pos_pg）— 7テーブル

| テーブル | 主な内容 |
|---|---|
| `users` | ユーザー（username, password_hash, role） |
| `products` | 商品マスタ（name, price_yen, tax_category, barcode） |
| `point_balances` | 会員ポイント残高 |
| `point_transactions` | ポイント取引履歴 |
| `point_rules` | ポイント付与ルール（multiplier） |
| `receipts` | レシート（kind: SALE/RETURN, body_json） |
| `audit_logs` | 監査ログ |

### MySQL（pos_mysql）— 8テーブル

| テーブル | 主な内容 |
|---|---|
| `inventory` | 在庫（product_id, available_quantity） |
| `stock_movements` | 在庫移動履歴 |
| `orders` | 注文ヘッダ |
| `order_items` | 注文明細 |
| `payments` | 支払情報 |
| `returns` | 返品ヘッダ |
| `return_items` | 返品明細 |
| `idempotency_keys` | 冪等性キー |

---

## 主要クラスの行数と責務

| クラス | 行数 | 問題 |
|---|---|---|
| `OrderService` | 976 | God Service — 注文 CRUD・統計・カート計算・返品チェックが混在 |
| `CheckoutSaga` | 557 | God Class — 6 ステップ + 補償処理が 1 メソッドに |
| `ReturnSaga` | 448 | God Class — 6 ステップ + 補償処理が 1 メソッドに |
| `RegisterController` | 303 | 画面 + REST API 混在、業務ロジックを直接記述 |

---

## アーキテクチャレイヤ

```
Browser/Client
    │
    ▼
[web/] Controller
    │  ⚠️ REST API と画面遷移が同一クラス
    │  ⚠️ コントローラが業務ロジックを直接記述
    ▼
[service/] Service
    │  ⚠️ OrderService が God Service
    │  ⚠️ Service が DAO 層をスキップして直接 JdbcTemplate を使用
    ▼
[saga/] Saga Coordinator
    │  ⚠️ CheckoutSaga / ReturnSaga が DB 跨ぎ (MySQL + PG) を直接協調
    │  ⚠️ Saga ステート永続化なし
    ▼
[dao/ / repository/] DAO
    │  ⚠️ 2 パッケージに分断（dao, repository）
    │  ⚠️ @Transactional が Service と DAO の両方に付いている
    ▼
PostgreSQL(pos_pg)   MySQL(pos_mysql)
```

---

## エントリポイント・主要パス

| パス | Controller | 説明 |
|---|---|---|
| `GET /register` | RegisterController | レジ画面 |
| `POST /api/register/checkout` | RegisterController | チェックアウト（CheckoutSaga 呼び出し） |
| `POST /admin/returns/new` | AdminReturnController | 返品（ReturnSaga 呼び出し） |
| `GET /admin/dashboard` | DashboardController | ダッシュボード（OrderService 統計系メソッド） |
| `GET /admin/orders/**` | AdminOrderController | 注文管理 |
| `GET /admin/inventory/**` | AdminInventoryController | 在庫管理 |

---

## 設定ファイル構成

| ファイル | 内容 |
|---|---|
| `application.properties` | server.port, thymeleaf.cache, error handling |
| `application.yml` | DataSource (PG/MySQL), Flyway 設定（⚠️ 実際は dead config — Flyway は DataSourceConfig.java で手動設定） |

**⚠️ `application.properties` と `application.yml` が共存している。**
::::

このレポートをみて特に気になったのは、**複数データベース（MySQLとPostgreSQL）への依存と、それらを跨ぐ処理をサービス（Saga）層で強引に解決している点**です。

具体的には、注文処理を行う`OrderService`が900行を超える巨大なサービスクラスになっており、その内部でチェックアウト（`CheckoutSaga`）や返品（`ReturnSaga`）といった複数DBを跨ぐ処理が、永続化もされないメモリ上のローカル変数を使った手書きのSagaで協調されていました。

さらに、DAOの命名規則も`dao/`と`repository/`が混在するなど乱れており、レイヤ間の境界が曖昧になっていました。

このような複雑な結合を解消し、どこか一部に手を加えてもシステム全体に影響が及ばないようにするためには、まずはこの巨大なクラスの解体とレイヤの整理が最優先の課題と言えます。

### DDD適用可能性の評価

![ddd-readiness](/images/legacy-refactoring-nexus-scalardb/ddd-readiness.png)
*ドメイン境界の分析、集約の粒度の評価やビジネスイベントの洗い出し等*


::::details レポート全文
# DDD 移行準備度評価 — legacy-pos-monolith

## 総合評価

| 観点 | スコア | 根拠 |
|---|---|---|
| ドメイン概念の認識 | 🟡 40/100 | POJOは存在するがDDDエンティティ・値オブジェクトの設計ではない |
| 境界の明確さ | 🔴 20/100 | God Service・God Class により境界が崩壊している |
| ユビキタス言語の統一 | 🟡 50/100 | 日本語ビジネス用語は存在するが、ステータス値が文字列で散在 |
| テスタビリティ | 🔴 15/100 | God Class と密結合により単体テストが実質不可能 |
| モジュール独立性 | 🔴 20/100 | 全サービスが相互依存、パッケージ命名も不統一 |
| **総合** | **🔴 29/100** | DDD 移行にはまず解体が必要 |

---

## ドメイン境界の分析

### 認識できる潜在的 Bounded Context

| Bounded Context（候補） | 現在のコード配置 | 独立性 |
|---|---|---|
| **Catalog（商品カタログ）** | `ProductService`, `ProductDao`, `domain/Product.java` | 🟢 比較的独立 |
| **Inventory（在庫管理）** | `InventoryService`, `InventoryDao`, MySQL | 🟡 OrderService と結合 |
| **Order（注文管理）** | `OrderService`(976行), `OrderDao`, `CheckoutSaga` | 🔴 God Service で境界崩壊 |
| **Payment（支払）** | `PaymentDao`, MySQL | 🔴 CheckoutSaga に埋め込み |
| **Point（ポイント管理）** | `PointService`, `PointDao`, `repository/PointDao.java` | 🟡 Saga と密結合 |
| **Receipt（レシート）** | `ReceiptService`, `repository/ReceiptDao.java`, PG | 🟡 Saga と密結合 |
| **Return（返品処理）** | `ReturnSaga`, `ReturnDao`, MySQL | 🔴 OrderService と ReturnSaga に分散 |
| **Identity（認証・ユーザー）** | `SecurityConfig`, `UserService`, `UserDao`, PG | 🟢 比較的独立 |

---

## ドメインモデルの現状

### エンティティ候補（domain/ パッケージ）

| クラス | 集約の明確さ | 問題点 |
|---|---|---|
| `Order` | 🟡 | OrderItem との関係が暗黙的（外部キーのみ） |
| `OrderItem` | 🟡 | Order の一部だが独立 POJO |
| `Payment` | 🟡 | Order との 1:1 関係が設計に現れていない |
| `Return` + `ReturnItem` | 🟡 | Order との関係が設計上明示されていない |
| `Inventory` | 🟢 | 比較的単純 |
| `Product` | 🟢 | 商品マスタとして独立性あり |
| `PointBalance` + `PointTransaction` | 🟡 | 集約ルートが不明確 |
| `Receipt` | 🟡 | body_json が非構造化 |

### 値オブジェクト候補（現状は存在しない）

| 概念 | 現状 | 理想 |
|---|---|---|
| 金額（円） | `int totalYen`, `int taxYen` | `Money` 値オブジェクト |
| 税カテゴリ | `int taxCategory`（1=標準, 2=軽減） | `TaxCategory` enum |
| 注文ステータス | `String status` | `OrderStatus` enum |
| 支払方法 | `String method` | `PaymentMethod` enum |

---

## DDD 移行の主要障壁

### 障壁1: God Service が集約境界を破壊している
`OrderService` が注文・返品・在庫・ポイント・ダッシュボード統計のすべてを担当。
これをそのままにした状態では集約ルートを定義しても意味をなさない。

### 障壁2: Saga の粒度が集約境界を越えている
`CheckoutSaga` が単一メソッドで Order, Inventory, Payment, Point, Receipt の 5 つの潜在的集約を直接操作する。
DDD では各集約は独立したトランザクション境界を持つべきだが、現在は手書き Saga で一括処理している。

### 障壁3: DB 設計が Bounded Context 境界と不一致
- MySQL に: Order, Inventory, Payment, Return（本来は分離されるべき複数 BC）
- PostgreSQL に: User, Product, Point, Receipt（異なる分類基準でまとめられている）

DDD の Bounded Context でドメインを整理するとき、DB の境界と一致させる必要があり、
現在の PG/MySQL 分割は BC 境界とほぼ一致しない。

### 障壁4: ドメインロジックが至る所に散在
税計算ロジックが `Utils.java`, `CheckoutSaga.java`, `OrderService.java`, Thymeleaf テンプレートの 4 箇所に重複。
ドメインロジックをドメインモデル内に集約するため、まず重複を除去する必要がある。

### 障壁5: パッケージ命名の不統一
`dao/` と `repository/` が並存しており、リポジトリパターン適用時に混乱を招く。

---

## ScalarDB 適用への適合性評価

### 現状の DB 跨ぎ処理

| 処理 | MySQL テーブル | PostgreSQL テーブル | 現在の整合性保証 |
|---|---|---|---|
| CheckoutSaga | orders, inventory, payments | receipts, point_balances, point_transactions | ❌ なし（手書き補償） |
| ReturnSaga | returns, inventory, payments | receipts, point_balances, point_transactions | ❌ なし（手書き補償） |

### ScalarDB 適用の効果

- `CheckoutSaga` と `ReturnSaga` の手書き補償ロジックを ScalarDB の分散トランザクションで置き換えることで、原子性を保証できる
- Saga ステート永続化問題が解消される
- MySQL-PostgreSQL 跨ぎの 2PC が ScalarDB によって透過的に処理される

### ScalarDB 移行で解決される問題

| 問題 | 解決 |
|---|---|
| TD-002: DB 跨ぎトランザクション原子性未保証 | ✅ ScalarDB 分散トランザクションで解決 |
| TD-003: Saga ステート永続化なし | ✅ 手書き Saga 自体が不要になる |
| TD-009: 補償処理の例外握り潰し | ✅ 補償処理自体が不要になる |

---

## 移行ロードマップ提言

### フェーズ 1（前提条件）— God Service の解体
1. `OrderService` を `OrderQueryService` / `OrderCommandService` / `OrderStatsService` に分割
2. `CheckoutSaga` をステップクラスに分割（`CreateOrderStep`, `ReserveInventoryStep` 等）
3. パッケージ命名を統一（`dao/` に統合）

### フェーズ 2 — ScalarDB 導入と Bounded Context 整理
1. ScalarDB を導入し、MySQL・PostgreSQL を ScalarDB 管理下に置く
2. CheckoutSaga / ReturnSaga の補償ロジックを ScalarDB トランザクションで置き換え
3. Bounded Context ごとにパッケージを再構成

### フェーズ 3 — ドメインモデルの強化
1. 値オブジェクト（Money, TaxCategory, OrderStatus 等）の導入
2. 集約ルートの明確化（Order が OrderItem を管理、等）
3. ドメインイベントの導入（将来的なマイクロサービス化の基盤）

---

## 結論

このコードベースは DDD 移行の **出発点としての最低条件（ドメイン概念の分離意図）は存在する** が、
God Service・God Class・DB 跨ぎ手書き Saga によって境界が崩壊しており、
**そのまま DDD を適用することはできない。**

まず God Service/God Class の解体と ScalarDB による分散トランザクション管理の導入を行い、
段階的に Bounded Context を明確化していくアプローチが現実的。
::::

このレポートでは、**ドメインロジックの極端な散在と、整合性保証の致命的な欠如**が気になりました。スコアも低い...

ドメインモデルと呼べるPOJOは存在するものの、金額を表現する`int`やステータスを表現する`String`などプリミティブ型がそのまま使われており、ビジネスルールがコード上で表現されていません。

特に税計算ロジックで、4箇所（`Utils`、`CheckoutSaga`、`OrderService`、Thymeleaf）に同じようなコードがコピペで散在しています。

また、MySQLとPostgreSQLという異なるデータベースにまたがるトランザクションの原子性が保証されておらず、手書きの補償トランザクションで強引に辻褄を合わせている状態です。

しかし、レポートでも提言されているように、ここに**ScalarDB**を導入して分散トランザクションを適用すれば、手書きの不安定なSagaや補償処理そのものを綺麗に消し去ることができるため、ScalarDBはリファクタリングを進める上での強力なアプローチであることがわかります！

### 技術的負債・コード品質問題・アーキテクチャ上の課題一覧

![issues-and-debt](/images/legacy-refactoring-nexus-scalardb/issues-and-debt.png)
*CRITICAL/High/Medium/Lowで分類*

::::details レポート全文
# 技術的負債・課題一覧 — legacy-pos-monolith

## CRITICAL（本番運用不可レベル）

### TD-001: 接続プールなし（DriverManagerDataSource）
- **場所**: `DataSourceConfig.java`
- **内容**: `DriverManagerDataSource` を使用。毎リクエストで新規 JDBC 接続を生成。高トラフィック環境ではコネクション枯渇・タイムアウトが発生する。
- **対策**: HikariCP に切り替え（Spring Boot 2.x の標準）

### TD-002: DB 跨ぎトランザクションの原子性未保証
- **場所**: `CheckoutSaga.java`, `ReturnSaga.java`
- **内容**: チェックアウト・返品は MySQL と PostgreSQL の両方を更新するが、2PC/XA/分散トランザクションマネージャは使用していない。補償トランザクションも例外を全て握り潰すため、障害時にデータ不整合が残る。
- **対策**: ScalarDB による分散トランザクション管理

### TD-003: Saga ステート永続化なし
- **場所**: `CheckoutSaga.java`, `ReturnSaga.java`
- **内容**: Saga の進行状態はメモリ上のローカル変数 (`step1Done`, `step2Done`...) にのみ保持。プロセス再起動で進行中 Saga はロスト。中途半端な状態が DB に残る。
- **対策**: Saga ステートを永続化するか、ScalarDB による分散トランザクションで Saga 不要にする

### TD-004: SQL インジェクション脆弱性（ORDER BY）
- **場所**: `OrderDao.java:61` — `sql = sql + " ORDER BY " + sortBy;`
- **内容**: `sortBy` パラメータをそのまま SQL に文字列結合。呼び出し元がユーザー入力を渡した場合に SQL インジェクションになる。
- **対策**: ORDER BY はホワイトリストバリデーション + 定数マップで制御

---

## High（機能・品質上の重大問題）

### TD-005: God Service — OrderService（976行）
- **場所**: `OrderService.java`
- **内容**: 注文 CRUD・統計・カート計算・返品可否チェック・ポイント計算・ダッシュボード用集計など、あらゆる関心事が1クラスに集約。テストが実質不可能。
- **内訳**: 注文クエリ系、ステータス遷移、統計・ランキング、ヘルパー（デッドコード含む）

### TD-006: God Class — CheckoutSaga（557行）・ReturnSaga（448行）
- **場所**: `CheckoutSaga.java`, `ReturnSaga.java`
- **内容**: 6ステップ + 補償処理が単一 `execute()` メソッドに集約。ステップ追加・変更が困難。

### TD-007: N+1 クエリ — 注文一覧・ランキング・在庫リスク
- **場所**: `OrderService.getOrderListWithDetails()`, `OrderService.getBestSellerRanking()`, `OrderService.getStockoutRisk()`
- **内容**: 注文一覧表示で注文数 N 回の SELECT を発行。ランキング計算でも注文ごとに明細 SELECT。
- **対策**: JOIN クエリまたはバッチ IN クエリに書き換え

### TD-008: 時間帯別売上で 24 回のクエリ発行
- **場所**: `OrderService.getHourlySales()` — `for (int hour = 0; hour < 24; hour++) { jdbcTemplate.queryForMap(...) }`
- **内容**: 1時間帯ずつ個別 SQL を実行。GROUP BY SQL 1本で代替可能。

### TD-009: 補償処理の例外全握り潰し
- **場所**: `CheckoutSaga.java:286-345`, `ReturnSaga.java:251-324`
- **内容**: 各補償ステップの catch ブロックで `log.error()` のみ実行し、補償失敗を上位に伝播しない。補償が途中で失敗してもサイレントに続行するため、データ不整合が検知できない。

### TD-010: コントローラに業務ロジック混在
- **場所**: `RegisterController.java:249-302` — checkout エンドポイント
- **内容**: CheckoutRequest 構築、カートアイテム変換、エラーハンドリングがコントローラ内に直書き。

### TD-011: ポイント有効期限の設定漏れ
- **場所**: `CheckoutSaga.java:222` — `ptx.setExpiresAt(null);`
- **内容**: ポイント付与時に有効期限が常に null。ポイント管理の業務ルール上は問題になる可能性がある。

### TD-012: 返品時の返金金額が常に 0
- **場所**: `ReturnSaga.java:125` — `ri.setRefundYen(0);`
- **内容**: 返品明細の返金金額が計算されず 0 のまま保存される。返品レシートも totalYen=0。

---

## Medium（品質・保守性問題）

### TD-013: パッケージ命名揺れ（dao と repository の並存）
- **場所**: `src/main/java/com/example/legacypos/dao/` + `src/main/java/com/example/legacypos/repository/`
- **内容**: MySQL 側 DAO は `dao/`、PostgreSQL 側（PointDao, ReceiptDao）は `repository/` に配置。命名規則に一貫性がない。

### TD-014: @Transactional の二重付与
- **場所**: `OrderDao.java`（全メソッドに `@Transactional`）+ `OrderService.java`（`updateStatus` に `@Transactional`）
- **内容**: Service と DAO の両方に `@Transactional` が付いており、トランザクション境界が不明確。

### TD-015: Magic Numbers（税率 10%, 8%）
- **場所**: `CheckoutSaga.java:117-121`, `OrderService.java:262-270`, `Utils.java:50-57`
- **内容**: 標準税率 10%、軽減税率 8% がコード中に数値リテラルとして散在。定数化されていない。

### TD-016: ステータス値の文字列リテラル散在
- **場所**: コードベース全体
- **内容**: `"PENDING"`, `"COMPLETED"`, `"CANCELLED"`, `"RETURNED"`, `"FAILED"` 等を文字列比較。enum 不使用（意図的）。

### TD-017: isOrderCompleted / isOrderPending / isOrderCancelled / isOrderReturned の重複
- **場所**: `OrderService.java:569-791`
- **内容**: 同一構造のメソッドが 4 つ並存。`isOrderInStatus(orderId, status)` 1 本で代替可能。

### TD-018: Map<String, Object> の多用
- **場所**: `RegisterController.java`, `OrderService.java`, `DashboardController.java` 他
- **内容**: View や JSON レスポンスに型安全でない `Map<String, Object>` を渡す。コンパイル時エラーが検出できない。

### TD-019: application.properties + application.yml の混在
- **場所**: `src/main/resources/`
- **内容**: server.port 等は `.properties`、DataSource は `.yml` に分散。設定ファイルの一元化がされていない。

### TD-020: 手書き JSON シリアライザ
- **場所**: `Utils.java:80-106`
- **内容**: `Utils.toJson()` が `toString()` を呼ぶだけ。レシートの `body_json` に格納されるデータが不正確な可能性。

### TD-021: MD5 ハッシュの使用
- **場所**: `Utils.hashString()` — 冪等性キーの requestHash 計算
- **内容**: MD5 は暗号学的に弱い。冪等性キー用途では衝突リスクは低いが、セキュリティベストプラクティス上問題。

### TD-022: System.out.println の残存
- **場所**: `CheckoutSaga.java:184` — `System.out.println("支払処理: " + ...)`、`Utils.java:17` — static initializer
- **内容**: 本番コードに `System.out.println` が残存。ログ基盤を迂回する。

### TD-023: log.error() での文字列連結
- **場所**: `CheckoutSaga.java:276`, `ReturnSaga.java:246` 他
- **内容**: `log.error("失敗: " + e.getMessage(), e)` — 文字列連結はパラメータ化ログより非効率。

### TD-024: デッドコードの残存
- **場所**: `OrderService.java:673-708` — `isValidOrderNote()`, `isValidTotalYen()`, `isValidOperatorId()`、コメントアウトされた旧実装複数箇所
- **内容**: 呼び出し元がないメソッド、TODO コメントのまま放置された未実装機能。

### TD-025: ベストセラーランキングをアプリ層で計算
- **場所**: `OrderService.getBestSellerRanking()`
- **内容**: 全注文を取得後、Java で GROUP BY・ソートを実装。DB の集計関数（`GROUP BY`, `ORDER BY`, `LIMIT`）で代替可能。

---

## Low（改善推奨）

### TD-026: ページネーションなし
- **場所**: `OrderService.findAll()`
- **内容**: 全件取得。大量データで OOM のリスク。

### TD-027: CartService がシングルトンスコープ
- **場所**: `CartService.java`
- **内容**: `@Service`（=シングルトン）として定義されているが、カートはユーザーごとに異なるべき。セッションスコープ（`@SessionScope`）への変更が必要。

### TD-028: Lombok がデコイとして pom.xml に含まれるが未使用
- **場所**: `pom.xml`
- **内容**: Lombok 依存が定義されているがアプリコードでは一切使用されていない。意図的なデコイ。

### TD-029: 監査ログの呼び出し漏れ
- **場所**: 複数箇所（商品更新、在庫調整等）
- **内容**: 一部の業務オペレーションで `auditService.record()` が呼ばれていない。

### TD-030: sanitizeForLog が何もしない
- **場所**: `Utils.sanitizeForLog()`
- **内容**: ログインジェクション対策のメソッドが入力をそのまま返すだけ。

---

## サマリ

| 重大度 | 件数 |
|---|---|
| CRITICAL | 4 |
| High | 8 |
| Medium | 13 |
| Low | 5 |
| **合計** | **30** |
::::

検出された負債は合計30件に上りますが、その中にはリクエストごとに新規JDBC接続を作る（接続プールなし）やSQLインジェクション脆弱性といった、Webアプリケーションとしては致命的な課題が含まれていました。

また、カート管理用の`CartService`がシングルトンになっている（＝全ユーザーでカートの中身が共有されてしまう）、返品時の返金金額が常に0で保存されるといった、重大な業務上のバグが仕様のように埋め込まれている点です。
このシステムをリファクタリングするには、単にコードをきれいに再構成するだけでなく、これらの致命的な不具合やパフォーマンスボトルネックを同時に解消する治療としてのリファクタリングが必要になります。

### 使用フレームワーク・ライブラリ・インフラ構成のインベントリ

![technology-stack](/images/legacy-refactoring-nexus-scalardb/technology-stack-01.png)
*開発言語、フレームワーク等*

![technology-stack](/images/legacy-refactoring-nexus-scalardb/technology-stack-02.png)
*評価サマリ*


::::details レポート全文
# テクノロジースタック分析 — legacy-pos-monolith

## 概要

レガシー Java モノリス POS システム。意図的にリファクタリング評価のターゲットとして設計されており、現実の現場で見られるレガシー構成を再現している。

---

## 言語・ランタイム

| 項目 | 値 | 評価 |
|---|---|---|
| 言語 | Java 11 (LTS) | EOL 済み（2024年9月）。Java 17/21 LTS への移行が必要 |
| ビルドツール | Maven 3（フラット構成） | サブモジュールなし。モノリス分割には構成変更が必要 |
| パッケージング | Fat JAR (Spring Boot Plugin) | Docker コンテナには適合しているが、WAR 配置とのハイブリッドな記述が混在 |

---

## フレームワーク

| 項目 | バージョン | 評価 |
|---|---|---|
| Spring Boot | **2.7.18** | EOL 寸前。Spring Boot 3.x（Spring Framework 6）への移行で `javax` → `jakarta` 名前空間変更が必要 |
| Spring MVC | Spring Boot 2.7 付属 | `WebSecurityConfigurerAdapter` は Spring Security 5.7 で deprecated |
| Spring Security | 5.x (Spring Boot 2.7 付属) | `WebSecurityConfigurerAdapter` + `antMatchers` → Spring Security 6 で削除済み |
| Thymeleaf | 3.x (Spring Boot 2.7 付属) | `thymeleaf-extras-springsecurity5` を使用。Security 6 対応時に変更要 |
| Spring JDBC | JdbcTemplate のみ | ORM 不使用（意図的）。`DriverManagerDataSource` を使用 ⚠️ |

---

## 永続化

| 項目 | 詳細 | 評価 |
|---|---|---|
| DB（Primary） | PostgreSQL 15（port 5432, DB: `pos_pg`） | ユーザー・商品・ポイント・レシート・監査ログ |
| DB（Secondary） | MySQL 8.0（port 3306, DB: `pos_mysql`） | 在庫・注文・支払・返品・冪等性キー |
| 接続管理 | `DriverManagerDataSource` | **CRITICAL: 接続プールなし** — 毎リクエストで新規 JDBC 接続を生成。本番環境では HikariCP 等が必須 |
| ORM | 不使用（意図的） | JdbcTemplate + 手書き SQL のみ |
| マイグレーション | Flyway（手動 Bean 設定） | `FlywayAutoConfiguration` を exclude し、`DataSourceConfig` で Flyway Bean を手動定義 |
| トランザクション | 単一 DB: `@Transactional` / DB 跨ぎ: 手書き Saga | DB 跨ぎの ACID 保証なし |

---

## UI / フロントエンド

| 項目 | 詳細 |
|---|---|
| テンプレートエンジン | Thymeleaf 3 (SSR) |
| JavaScript | jQuery + Vanilla JS（`app.js`, `register.js`）|
| CSS | カスタム `app.css`（フレームワーク不使用） |
| SPA 要素 | レジ画面（`/register`）のみ AJAX ベース |

---

## 認証・認可

| 項目 | 詳細 |
|---|---|
| 認証方式 | フォームログイン + BCrypt パスワードハッシュ |
| セッション管理 | Cookie セッション（最大同時セッション数 1） |
| 認可 | ロールベース（CASHIER / MANAGER / ADMIN） |
| 実装 | `WebSecurityConfigurerAdapter`（deprecated） |

---

## 依存ライブラリ一覧

| ライブラリ | バージョン | 用途 | 備考 |
|---|---|---|---|
| spring-boot-starter-web | 2.7.18 | Web MVC | |
| spring-boot-starter-thymeleaf | 2.7.18 | SSR テンプレート | |
| spring-boot-starter-security | 2.7.18 | 認証・認可 | |
| spring-boot-starter-jdbc | 2.7.18 | JdbcTemplate | |
| spring-boot-starter-actuator | 2.7.18 | ヘルスチェック | |
| flyway-core | Spring Boot 管理 | DBマイグレーション | |
| flyway-mysql | Spring Boot 管理 | MySQL Flyway サポート | |
| postgresql | Spring Boot 管理 | PostgreSQL ドライバ | |
| mysql-connector-java | 8.0.33 | MySQL ドライバ | |
| lombok | provided | （pom.xml にあるが **一切使用されていない**） | 意図的なデコイ |
| spring-boot-starter-test | test | JUnit 5 + Spring Boot Test | |
| spring-security-test | test | セキュリティテスト | |

---

## インフラ・運用

| 項目 | 詳細 |
|---|---|
| コンテナ化 | Docker Compose（app + postgres + mysql） |
| ポート | 8080 (app), 5432 (PG), 3306 (MySQL) |
| ロギング | Logback（`logback-spring.xml` で設定） |
| 設定ファイル | `application.properties` + `application.yml` の **混在**（意図的欠陥） |
| テスト | JUnit 5 + Spring Boot Test（最小限・3ファイルのみ） |

---

## テクノロジースタック評価サマリ

| 観点 | 評価 | 主な問題点 |
|---|---|---|
| アップグレードリスク | 🔴 高 | Spring Boot 2.7 EOL、Java 11 EOL、javax→jakarta 変更コスト大 |
| 接続管理 | 🔴 CRITICAL | DriverManagerDataSource — 本番利用不可 |
| データ整合性 | 🔴 CRITICAL | 2 DB 跨ぎトランザクション未保証 |
| セキュリティ | 🟡 要注意 | MD5 ハッシュ使用、log injection 対策なし、SQL injection リスク（OrderDao） |
| テスト | 🔴 不十分 | テストファイル 3 つのみ、God Class はテスト困難 |
| 可観測性 | 🟡 最小限 | Actuator ヘルスのみ、メトリクス・分散トレーシングなし |
::::

既存システムは Java 11 や Spring Boot 2.7 といった、すでにEOLを迎えている（あるいは寸前の）古いバージョンで動作しています。
これらを現代の Java 17/21 や Spring Boot 3.x へアップグレードするには`javax`から`jakarta`へのパッケージ名移行や、Spring Security の非推奨クラスの廃止など、アーキテクチャの根幹に関わる大きな書き換えが必要になります。

最大の問題は、テストコードがほぼ皆無（3ファイル）である点です。
動作を保証する自動テストがない状態では、リファクタリングやバージョンアップによる予期せぬデグレを検知できず、壊す恐怖から既存コードの修正を避けてコピペで機能を追加するという負債の悪循環を招きます。

## 本章のまとめ

* 現状把握レポートでは、ディレクトリ構成、主要クラス、DBスキーマ、技術スタックが整理されました。
* `OrderService`や`CheckoutSaga`に責務が集中し、DB跨ぎ更新、例外握り潰し、N+1問題などの技術的負債が明確になりました。
* Java 11、Spring Boot 2.7、テスト不足といったアップグレードリスクも、モダナイゼーション前に把握すべき重要な制約です。

## 用語解説

### God Service
業務ロジック、DBアクセス、外部連携など多くの責務を抱え込みすぎたServiceクラスです。変更影響が広がりやすく、テストも難しくなります。

### 技術的負債
短期的な都合で積み上がった設計・実装上の問題です。放置すると、機能追加や障害対応のたびに余計なコストとして現れます。

### EOL
End of Lifeの略で、ソフトウェアのサポート終了を意味します。EOL後はセキュリティ修正や不具合修正を受けにくくなります。

### ScalarDB適用余地
複数DBをまたぐ更新や一貫性制御をScalarDBで置き換えることで、どの課題を解消できるかを見る観点です。
