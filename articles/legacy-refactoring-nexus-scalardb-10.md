---
title: "【Nexus Architect本レビュー用】（第10回）ドメイン分析レポートを読み解く"
emoji: "📘"
type: "tech"
topics: ["architecture","scalardb","refactoring"]
published: false
publication_name: "scalar_sol_blog"
---


:::message
ドメイン分析レポートから、業務スコープ、アクター/権限、ユビキタス言語、ドメイン概念とコードの対応を読み解きます。
:::

## ドメイン分析

前フェーズの現状把握から、よりビジネスの観点で整理したレポートをドメイン分析フェーズとして読み進めていきます。

### 各レポート内容の概要

| ファイル | 内容 |
| :--- | :--- |
| system-overview.md | システム全体の目的・スコープ・主要機能の整理 |
| actors-roles-permissions.md | アクター（ユーザー種別）・ロール・権限マトリクスの定義 |
| ubiquitous-language.md | ドメインの共通語彙集（ユビキタス言語辞書） |
| domain-code-mapping.md | ドメインモデルの概念と実際のコード（クラス・テーブル等）の対応マッピング |

### システム全体の目的・スコープ・主要機能の整理

![system-overview](/images/legacy-refactoring-nexus-scalardb/system-overview.png)
*ビジネスコンテキストや主要機能など*


::::details レポート全文
# システム概要 — legacy-pos-monolith

## ビジネスコンテキスト

小売店舗向けの POS（Point of Sale, レジ販売）システム。
レジ係（Cashier）・店舗マネージャ（Manager）・システム管理者（Admin）の 3 ロールで運用される。

### 解決対象のビジネス課題

1. **店頭での販売処理**: バーコードスキャン → カート → 会計 → レシート発行までの一連のフロー
2. **在庫管理**: 入荷登録、在庫水準モニタリング、販売による自動在庫減算
3. **会員ポイント運用**: 注文金額に応じたポイント付与、返品時のポイント取消
4. **返品対応**: 完了済み注文に対する返品処理、在庫戻し、支払取消、ポイント取消
5. **店舗運営の可視化**: 日次・月次の売上、ベストセラーランキング、時間帯別売上、在庫切れリスク
6. **業務監査**: 重要オペレーション（注文・返品・商品変更等）の監査ログ記録

---

## 主要機能

### 1. レジ販売（Cashier 機能）

| 機能 | エンドポイント | 説明 |
|---|---|---|
| レジ画面表示 | `GET /register` | カート内容と合計金額を表示 |
| バーコードスキャン | `GET /api/register/scan?barcode=X` | バーコードから商品検索 |
| カート追加 | `POST /api/register/cart/add` | 商品をカートに追加 |
| カート削除 | `POST /api/register/cart/remove` | カートから商品削除 |
| 数量変更 | `POST /api/register/cart/update-qty` | カート内商品の数量更新 |
| カートクリア | `POST /api/register/cart/clear` | カートを空にする |
| カート取得 | `GET /api/register/cart` | カート内容取得 |
| **チェックアウト** | `POST /api/register/checkout` | **CheckoutSaga 実行** |
| レシート表示 | `GET /register/receipt/{orderId}` | 完了注文のレシート表示 |
| 会員ポイント残高照会 | `GET /api/points/{memberId}/balance` | 会員のポイント残高 |

### 2. 商品管理（Manager 機能）

| 機能 | エンドポイント |
|---|---|
| 商品一覧 | `GET /admin/products` |
| 商品詳細 | `GET /admin/products/{id}` |
| 商品登録 | `POST /admin/products/new` |
| 商品編集 | `POST /admin/products/{id}/edit` |
| 商品終売 | `POST /admin/products/{id}/discontinue` |

### 3. 在庫管理（Manager 機能）

| 機能 | エンドポイント |
|---|---|
| 在庫一覧 | `GET /admin/inventory` |
| 在庫詳細 | `GET /admin/inventory/{productId}` |
| 入荷登録 | `POST /admin/inventory/receive` |
| 在庫移動履歴 | `GET /admin/inventory/{productId}/movements` |

### 4. 注文管理（Manager 機能）

| 機能 | エンドポイント |
|---|---|
| 注文一覧 | `GET /admin/orders` |
| 注文詳細 | `GET /admin/orders/{orderId}` |
| ステータス更新 | `POST /admin/orders/{orderId}/status` |

### 5. 返品処理（Manager 機能）

| 機能 | エンドポイント |
|---|---|
| 返品検索 | `GET /admin/returns/search?orderId=X` |
| 返品入力画面 | `GET /admin/returns/new?orderId=X` |
| **返品実行** | `POST /admin/returns/new` | **ReturnSaga 実行** |
| 返品一覧 | `GET /admin/returns` |

### 6. ポイント管理（Manager 機能）

| 機能 | エンドポイント |
|---|---|
| ポイント履歴 | `GET /admin/points/history` |
| ポイントルール一覧 | `GET /admin/point-rules` |
| ポイントルール作成 | `POST /admin/point-rules/new` |
| ポイントルール編集 | `POST /admin/point-rules/{id}/edit` |

### 7. ダッシュボード（Manager 機能）

| 機能 | エンドポイント |
|---|---|
| ダッシュボード | `GET /admin/dashboard` |
| 主な指標 | 今日の売上、今日の注文件数、ベストセラー、時間帯別売上、在庫切れリスク |

### 8. レシート閲覧（Manager 機能）

| 機能 | エンドポイント |
|---|---|
| レシート一覧 | `GET /admin/receipts` |
| レシート詳細 | `GET /admin/receipts/{receiptId}` |

### 9. ユーザー管理（Admin 機能）

| 機能 | エンドポイント |
|---|---|
| ユーザー一覧 | `GET /admin/users` |
| ユーザー作成 | `POST /admin/users/new` |
| ユーザー編集 | `POST /admin/users/{id}/edit` |

### 10. 共通機能

| 機能 | エンドポイント |
|---|---|
| ログイン | `POST /login` |
| ログアウト | `POST /logout` |
| 自パスワード変更 | `POST /me/password` |

---

## システム境界

### 内部システム

- **アプリケーション本体**: Spring Boot モノリス（単一 JAR）
- **PostgreSQL（pos_pg）**: 商品マスタ・ユーザー・ポイント・レシート・監査ログ
- **MySQL（pos_mysql）**: 在庫・注文・支払・返品・冪等性キー

### 外部システムとの連携

- **なし** — 決済代行サービス・実物決済端末・配送・会員管理 SaaS 等への外部連携はモック化されている（`System.out.println("支払処理: ...")`）。

---

## 重要なビジネス制約

| 制約 | 内容 |
|---|---|
| 税率 | 標準税率 10%（taxCategory=1）/ 軽減税率 8%（taxCategory=2） |
| ポイント付与 | 注文金額 × multiplier ÷ 100（デフォルト 1%） |
| 返品条件 | 元注文が COMPLETED ステータス、かつ既存の COMPLETED 返品が存在しないこと |
| 冪等性 | チェックアウトは idempotencyKey でリクエスト重複を排除 |
| 同時セッション | 1 ユーザーあたり最大 1 セッション |

---

## 主要ビジネスフロー

### A. チェックアウトフロー（D4）

```
[カート] → CheckoutSaga.execute()
  ├ Step 0: 冪等性キーチェック (MySQL: idempotency_keys)
  ├ Step 1: 注文ヘッダ・明細作成 (MySQL: orders, order_items)
  ├ Step 2: 在庫引当 (MySQL: inventory, stock_movements)
  ├ Step 3: 支払処理 (MySQL: payments)
  ├ Step 4: ポイント加算 (PostgreSQL: point_balances, point_transactions)
  ├ Step 5: レシート発行 (PostgreSQL: receipts)
  └ Step 6: 注文確定 (MySQL: orders.status=COMPLETED)

異常系: 補償処理を逆順で実行（補償失敗は握り潰す）
```

### B. 返品フロー（E3）

```
[返品リクエスト] → ReturnSaga.execute()
  ├ Step 0: 注文バリデーション (MySQL: orders)
  ├ Step 1: 返品ヘッダ・明細作成 (MySQL: returns, return_items)
  ├ Step 2: 在庫戻し (MySQL: inventory, stock_movements)
  ├ Step 3: 支払取消 (MySQL: payments)
  ├ Step 4: ポイント取消 (PostgreSQL: point_balances, point_transactions)
  ├ Step 5: 返品レシート発行 (PostgreSQL: receipts)
  └ Step 6: 返品確定 (MySQL: returns.status=COMPLETED, orders.status=RETURNED)

異常系: 補償処理を逆順で実行
```

---

## 利用シーン

| シーン | アクター | 主要操作 |
|---|---|---|
| 店頭レジ会計 | Cashier | スキャン → カート → 会計 → レシート発行 |
| 入荷作業 | Manager | 在庫詳細 → 入荷登録 |
| 返品対応 | Manager | 返品検索 → 返品入力 → 返品実行 |
| 日次締め | Manager | ダッシュボード閲覧、注文・売上確認 |
| 商品マスタメンテ | Manager | 商品登録・編集・終売 |
| ユーザー管理 | Admin | ユーザー作成・ロール変更 |
::::

システム概要のレポートを見ると、本システムはレジ販売や商品管理、在庫管理、ポイント運用、返品対応など、小売店舗におけるPOSシステムに必要な機能群を一通り網羅していることが分かります。
しかし、システム境界に目を向けると、単一のモノリスアプリケーションでありながら、PostgreSQLとMySQLという2つの異なる物理データベースを同時に抱えているという特徴的な構成になっています。
また、決済や配送といった外部システムとの連携はすべて実質的なモック処理になっており、外部への直接的なI/Oは発生していません。

このような構成であるため、チェックアウト処理や返品処理において、2つのデータベースを跨ぐ一連の書き込みが必要となります。

フレームワーク標準のトランザクション管理機能（@Transactionalなど）では、この複数DBにまたがる原子性を保証できません。
そのため、メモリ上のローカル変数を利用した手書きのSagaパターンで協調動作を制御せざるを得ません。
これが、システム全体の整合性管理を複雑にする大きな要因となっています。

### アクター（ユーザー種別）・ロール・権限マトリクスの定義


![actors-roles-permissions](/images/legacy-refactoring-nexus-scalardb/actors-roles-permissions.png)
*アクター・ロール定義・権限マトリクス* 



::::details レポート全文
# アクター・ロール・パーミッション — legacy-pos-monolith

## アクター（人間ユーザー）

| アクター | 説明 | システムロール | 主な目的 |
|---|---|---|---|
| **Cashier（レジ係）** | 店頭でレジ会計を行うスタッフ | `CASHIER` | 商品スキャン・カート操作・チェックアウト・レシート発行 |
| **Manager（店舗マネージャ）** | 店舗運営の責任者 | `MANAGER` | 商品マスタ管理・在庫管理・注文管理・返品処理・ポイント管理・ダッシュボード閲覧 |
| **Admin（システム管理者）** | システム全体の管理者 | `ADMIN` | 上記すべて + ユーザー管理 |
| **Member（会員）** | ポイント獲得対象の顧客 | （ロールなし — ログインしない） | 商品購入時に member_id を提示してポイントを蓄積 |

**注意**:
- Member は **システムにログインしない**（会員自身の認証フローは未実装）。
- Cashier がレジ操作中に member_id を入力することでポイント加算が行われる。

---

## ロール定義

ロールは `users.role` カラム（VARCHAR）に格納され、Spring Security の `ROLE_X` 形式で扱われる。

| ロール | 階層 | 包含関係 |
|---|---|---|
| `CASHIER` | レジ係レベル | 自分自身のパスワード変更のみ |
| `MANAGER` | 店舗管理レベル | CASHIER ができる全操作 + 管理機能 |
| `ADMIN` | システム管理レベル | MANAGER ができる全操作 + ユーザー管理 |

**注意**: Spring Security の設定 (`SecurityConfig.java`) では各 URL ごとに `hasAnyRole` で個別指定されており、**ロールの階層継承は手動で設定されている**。

---

## パーミッション・マトリクス

### URL パターン × ロール

| URL パターン | CASHIER | MANAGER | ADMIN |
|---|---|---|---|
| `/login`, `/error/**`, `/actuator/health` | ✅ permitAll | ✅ permitAll | ✅ permitAll |
| `/me/password` | ✅ | ✅ | ✅ |
| `/register/**`（レジ画面） | ✅ | ✅ | ✅ |
| `/api/register/**`（レジ API） | ✅ | ✅ | ✅ |
| `/api/products/scan` | ✅ | ✅ | ✅ |
| `/api/points/*/balance` | ✅ | ✅ | ✅ |
| `/api/receipts**` | ✅ | ✅ | ✅ |
| `/receipts/**`（レシート閲覧） | ✅ | ✅ | ✅ |
| `/admin/dashboard` | ❌ | ✅ | ✅ |
| `/admin/products/**` | ❌ | ✅ | ✅ |
| `/admin/inventory/**` | ❌ | ✅ | ✅ |
| `/admin/orders/**` | ❌ | ✅ | ✅ |
| `/admin/returns/**` | ❌ | ✅ | ✅ |
| `/admin/points/**` | ❌ | ✅ | ✅ |
| `/admin/point-rules/**` | ❌ | ✅ | ✅ |
| `/admin/receipts/**` | ❌ | ✅ | ✅ |
| `/admin/users/**` | ❌ | ❌ | ✅ |

---

## 機能カテゴリ別パーミッション

### A. レジ販売機能

| 操作 | CASHIER | MANAGER | ADMIN |
|---|:---:|:---:|:---:|
| バーコードスキャン | ✅ | ✅ | ✅ |
| カート追加・削除・数量変更・クリア | ✅ | ✅ | ✅ |
| カート取得 | ✅ | ✅ | ✅ |
| **チェックアウト実行** | ✅ | ✅ | ✅ |
| レシート閲覧（自店分） | ✅ | ✅ | ✅ |
| 会員ポイント残高照会 | ✅ | ✅ | ✅ |

### B. 商品マスタ管理

| 操作 | CASHIER | MANAGER | ADMIN |
|---|:---:|:---:|:---:|
| 商品一覧閲覧 | ❌ | ✅ | ✅ |
| 商品詳細閲覧 | ❌ | ✅ | ✅ |
| 商品登録 | ❌ | ✅ | ✅ |
| 商品編集 | ❌ | ✅ | ✅ |
| 商品終売 | ❌ | ✅ | ✅ |

### C. 在庫管理

| 操作 | CASHIER | MANAGER | ADMIN |
|---|:---:|:---:|:---:|
| 在庫一覧 | ❌ | ✅ | ✅ |
| 入荷登録 | ❌ | ✅ | ✅ |
| 在庫移動履歴閲覧 | ❌ | ✅ | ✅ |

### D. 注文管理

| 操作 | CASHIER | MANAGER | ADMIN |
|---|:---:|:---:|:---:|
| 注文一覧閲覧 | ❌ | ✅ | ✅ |
| 注文詳細閲覧 | ❌ | ✅ | ✅ |
| 注文ステータス手動変更 | ❌ | ✅ | ✅ |
| 注文キャンセル | ❌ | ✅ | ✅ |

### E. 返品処理

| 操作 | CASHIER | MANAGER | ADMIN |
|---|:---:|:---:|:---:|
| 返品検索 | ❌ | ✅ | ✅ |
| **返品実行** | ❌ | ✅ | ✅ |
| 返品一覧 | ❌ | ✅ | ✅ |

### F. ポイント管理

| 操作 | CASHIER | MANAGER | ADMIN |
|---|:---:|:---:|:---:|
| ポイント履歴閲覧 | ❌ | ✅ | ✅ |
| ポイントルール作成 | ❌ | ✅ | ✅ |
| ポイントルール編集 | ❌ | ✅ | ✅ |

### G. ダッシュボード・レシート

| 操作 | CASHIER | MANAGER | ADMIN |
|---|:---:|:---:|:---:|
| ダッシュボード閲覧 | ❌ | ✅ | ✅ |
| 全店レシート閲覧 | ❌ | ✅ | ✅ |

### H. ユーザー管理

| 操作 | CASHIER | MANAGER | ADMIN |
|---|:---:|:---:|:---:|
| ユーザー一覧 | ❌ | ❌ | ✅ |
| ユーザー作成 | ❌ | ❌ | ✅ |
| ユーザー編集（ロール変更） | ❌ | ❌ | ✅ |

### I. 共通機能

| 操作 | CASHIER | MANAGER | ADMIN |
|---|:---:|:---:|:---:|
| 自パスワード変更 | ✅ | ✅ | ✅ |
| ログイン・ログアウト | ✅ | ✅ | ✅ |

---

## 認証・認可の実装

### 認証
- Spring Security フォームログイン
- `PosUserDetailsService` が `users` テーブルからユーザー情報を取得
- パスワードは BCrypt ハッシュで保存・検証
- セッション最大同時接続数: 1

### 認可
- ロールベース（RBAC）
- URL パターンマッチによる粗粒度認可（`SecurityConfig` で一括定義）
- メソッドレベルの `@PreAuthorize` / `@Secured` は **使用されていない**

### 認可上の問題点（意図的レガシー臭）

| 問題 | 場所 | 影響 |
|---|---|---|
| メソッドレベル認可なし | OrderService 等 | サービス層から呼ばれるとロール検証なし |
| `OrderService.cancelOrder()` にチェックなし | `OrderService.java:448` | コントローラレベルのチェックに依存。リフレクション・内部呼び出しで回避可能 |
| `OrderService.updateStatus()` にチェックなし | `OrderService.java:166` | 同上 |
| 会員(Member)の認証フローなし | — | Cashier が任意の member_id を入力できる（なりすまし可能） |

---

## 初期データ（テストアカウント）

| ユーザー名 | パスワード | ロール |
|---|---|---|
| `admin` | `admin1234` | ADMIN |
| `manager` | `manager1234` | MANAGER |
| `cashier` | `cashier1234` | CASHIER |

**会員 ID**: `member-001` 〜 `member-010` （`point_balances` テーブルに事前投入）

---

## システム外アクター

| 外部アクター | 連携方法 | 現状 |
|---|---|---|
| 決済代行サービス | （未連携） | `CheckoutSaga` 内で `System.out.println("支払処理: ...")` のみのモック |
| 会員管理 SaaS | （未連携） | 会員マスタ自体がシステム内にない |
| 配送業者 | （該当なし） | 店頭販売モデル |
| 経理システム | （未連携） | 売上データの外部連携なし |
::::

アクターとロール、パーミッションの設定に関するレポートからは、本システムのセキュリティ認可における構造的な課題が読み取れます。
特に顕著な問題は、Spring Securityを利用した認可制御がコントローラレベルのURLパターンマッチング（粗粒度認可）のみで行われている点です。

サービス層やデータアクセス層におけるメソッドレベルの認可ガード（細粒度認可）が全く適用されていません。
例えば、注文キャンセル処理やステータス更新処理などの重要な操作を実行するサービスメソッドにおいて、ロールの検証が行われていません。
そのため、コントローラをバイパスするような予期しない呼び出し経路が存在した場合に、権限のないロールからでも処理が実行できてしまうというリスクが残されています。
さらに、会員のポイント管理において、システムの利用時に会員自身の認証プロセスが存在しません。

レジ係が入力した会員IDを検証なしでそのまま受け入れて処理する仕組みになっているため、なりすましなどに対する防衛機構が機能していないと言えます。


### ドメインの共通語彙集（ユビキタス言語辞書）

![ubiquitous-language](/images/legacy-refactoring-nexus-scalardb/ubiquitous-language.png)
*ドメイン用語辞書*



::::details レポート全文
# ユビキタス言語辞書 — legacy-pos-monolith

## 用語の使い分けに関する注意

このコードベースには **同じ概念に複数の名称が使われている** ケースが存在する。
そうした「揺れ」も以降のテーブル中に記録する。

---

## ドメイン用語辞書

### 商品・カタログ領域

| 用語（日本語） | 用語（英語） | コード上の表現 | 定義 |
|---|---|---|---|
| 商品 | Product | `domain.Product`, `products` テーブル | 販売対象。商品IDをキーにバーコード・名称・税カテゴリ・価格を持つ |
| バーコード | Barcode | `Product.barcode` | 商品識別用のバーコード（UNIQUE） |
| 価格 | Price | `Product.priceYen`, `unit_price_yen` | 円建ての税抜価格（int） |
| 税カテゴリ | Tax Category | `Product.taxCategory`（int: 1 or 2）| 1=標準税率10%、2=軽減税率8%。**Enum 化されていない** |
| カテゴリ | Category | `Product.category`（VARCHAR） | 商品の大分類 |
| 商品ステータス | Product Status | `Product.status`（VARCHAR）| 'ACTIVE' / 'DISCONTINUED' |

### 在庫領域

| 用語 | 英語 | コード上の表現 | 定義 |
|---|---|---|---|
| 在庫 | Inventory | `domain.Inventory`, `inventory` テーブル | 商品IDごとの在庫情報 |
| 利用可能在庫 | Available Quantity | `Inventory.availableQuantity` | 引当可能な在庫数 |
| 引当済在庫 | Reserved Quantity | `Inventory.reservedQuantity` | カラムは存在するが現実装では未使用 |
| 在庫移動 | Stock Movement | `domain.StockMovement`, `stock_movements` テーブル | 在庫の増減履歴。delta（差分）と reason（理由）を持つ |
| 入荷 | Receive | `InventoryService.receive()`, reason='RECEIVE' | 入荷（在庫増） |
| 引当 | Allocate / Reserve | `inventoryDao.decreaseStock()`, reason='SELL' | 販売による在庫減 |
| 在庫戻し | Restock | `inventoryDao.increaseStock()`（返品時）, reason='RETURN' | 返品による在庫増 |
| 補償 | Compensate | reason='COMPENSATE' / 'RETURN_COMPENSATE' | Saga 補償処理 |

### 注文領域

| 用語 | 英語 | コード上の表現 | 定義 |
|---|---|---|---|
| 注文 | Order | `domain.Order`, `orders` テーブル | レジでの会計1回に対応する1取引 |
| 注文明細 | Order Item | `domain.OrderItem`, `order_items` テーブル | 注文に含まれる商品明細（line_no で識別） |
| 注文ID | Order ID | `Order.orderId`（"ORD-..."） | UUID 由来の文字列ID |
| 注文ステータス | Order Status | `Order.status`（String）| PENDING / COMPLETED / CANCELLED / RETURNED / FAILED。**Enum 化されていない** |
| 合計金額 | Total Yen | `Order.totalYen` | 税込合計金額（円） |
| 税額 | Tax Yen | `Order.taxYen` | 消費税額（円） |
| レジID | Register ID | `Order.registerId` | 会計を行ったレジ端末ID |
| オペレータID | Operator ID | `Order.operatorId` | 会計を行った担当者（Cashier）のユーザー名 |
| 会員ID | Member ID | `Order.memberId` | 会員注文の場合の会員識別子 |

### カート領域

| 用語 | 英語 | コード上の表現 | 定義 |
|---|---|---|---|
| カート | Cart | `service.CartService`（@SessionScope） | 会計前の商品リスト。ユーザーセッションごとに保持 |
| カートアイテム | Cart Item | `CartService.CartItem` | カート内の1商品エントリ |

### 支払領域

| 用語 | 英語 | コード上の表現 | 定義 |
|---|---|---|---|
| 支払 | Payment | `domain.Payment`, `payments` テーブル | 注文に対する支払情報 |
| 支払方法 | Payment Method | `Payment.method` (String) | "CASH" / "CARD" 等の文字列。**Enum 化されていない** |
| 支払ステータス | Payment Status | `Payment.status` (String) | COMPLETED / REVERSED / REFUNDED |
| 支払金額 | Amount Yen | `Payment.amountYen` | 支払金額（円） |
| 支払ID | Payment ID | `Payment.paymentId`（"PAY-..."） | UUID 由来の文字列ID |

### ポイント領域

| 用語 | 英語 | コード上の表現 | 定義 |
|---|---|---|---|
| ポイント残高 | Point Balance | `domain.PointBalance`, `point_balances` テーブル | 会員の現在ポイント |
| ポイントトランザクション | Point Transaction | `domain.PointTransaction`, `point_transactions` テーブル | ポイントの増減履歴 |
| トランザクションタイプ | Type | `PointTransaction.type` | "EARN" / "EARN_REVERSED" / "REVERSED" / "EARN_REVERSED_CANCEL" 等の **不統一な文字列** |
| ポイントルール | Point Rule | `domain.PointRule`, `point_rules` テーブル | 倍率（multiplier）とルールタイプ・有効期間 |
| 倍率 | Multiplier | `PointRule.multiplier`（DECIMAL(4,2)） | ポイント計算倍率（1.00 = 1%） |
| 付与（獲得） | Earn | type='EARN' | ポイント付与 |
| 取消 | Reverse / Cancel | type='REVERSED' / 'EARN_REVERSED' | ポイント取消（**用語が混乱**） |
| 有効期限 | Expires At | `PointTransaction.expiresAt` | **現実装では常に null** |

### レシート領域

| 用語 | 英語 | コード上の表現 | 定義 |
|---|---|---|---|
| レシート | Receipt | `domain.Receipt`, `receipts` テーブル | 注文・返品ごとに発行される紙レシート相当の電子記録 |
| レシート種別 | Kind | `Receipt.kind`（"SALE" / "RETURN"） | **Enum 化されていない** |
| レシートステータス | Status | `Receipt.status` | "ISSUED" / "VOID" |
| レシート本文 | Body JSON | `Receipt.bodyJson` | JSON 文字列。**`Utils.toJson()` が toString() を呼ぶだけのため不完全** |

### 返品領域

| 用語 | 英語 | コード上の表現 | 定義 |
|---|---|---|---|
| 返品 | Return | `domain.Return`, `returns` テーブル | 完了済注文に対する返品処理単位 |
| 返品明細 | Return Item | `domain.ReturnItem`, `return_items` テーブル | 返品に含まれる商品明細 |
| 返品ステータス | Status | `Return.status` | "PENDING" / "COMPLETED" / "CANCELLED" |
| 返品金額 | Refund Yen | `ReturnItem.refundYen` | **現実装では常に 0**（重大欠陥） |
| 返品リクエスタ | Requested By | `Return.requestedBy` | 返品操作を行った Manager のユーザー名 |

### Saga / 冪等性領域

| 用語 | 英語 | コード上の表現 | 定義 |
|---|---|---|---|
| Saga | Saga | `saga.CheckoutSaga`, `saga.ReturnSaga` | 手書きオーケストレーション型 Saga |
| ステップ | Step | `step1Done`, `step2Done`... | Saga 内のローカル変数。永続化なし |
| 補償 | Compensation | catch ブロック内の処理 | Saga 失敗時の逆順巻き戻し |
| 冪等性キー | Idempotency Key | `domain.IdempotencyKey`, `idempotency_keys` テーブル | チェックアウトの重複排除用キー |
| 冪等性目的 | Purpose | `IdempotencyKey.purpose` | "CHECKOUT" 等 |
| 結果ステータス | Result Status | `IdempotencyKey.resultStatus` | "SUCCESS" / "FAILED" |
| 結果ペイロード | Result Payload | `IdempotencyKey.resultPayload` | "orderId\|receiptId" の **パイプ区切り文字列** |

### ユーザー・認証領域

| 用語 | 英語 | コード上の表現 | 定義 |
|---|---|---|---|
| ユーザー | User | `domain.User`, `users` テーブル | システムにログインする操作者 |
| ユーザー名 | Username | `User.username` (UNIQUE) | ログインID |
| ロール | Role | `User.role`（String） | "CASHIER" / "MANAGER" / "ADMIN"。**Enum 化されていない**（Spring Security 上は `ROLE_X` で扱う） |
| パスワードハッシュ | Password Hash | `User.passwordHash` | BCrypt ハッシュ |
| 有効状態 | Enabled | `User.enabled` | アカウント有効/無効 |
| 会員 | Member | `member_id`（VARCHAR） | ポイント運用対象の顧客。**`User` とは別概念**（`users` テーブルとは独立、外部マスタを想定） |

### 監査領域

| 用語 | 英語 | コード上の表現 | 定義 |
|---|---|---|---|
| 監査ログ | Audit Log | `domain.AuditLog`, `audit_logs` テーブル | 重要オペレーションの記録 |
| アクション | Action | `AuditLog.action` | "CHECKOUT" / "RETURN" / "ORDER_CANCELLED" 等の文字列 |
| 対象 | Target | `AuditLog.target` | アクション対象（注文ID、商品名等） |
| ペイロード | Payload | `AuditLog.payload` | 補足情報（追加コンテキスト） |

### ダッシュボード・分析領域

| 用語 | 英語 | コード上の表現 | 定義 |
|---|---|---|---|
| 今日の売上 | Today's Sales Total | `OrderService.getTodaySalesTotal()` | 当日の COMPLETED 注文合計 |
| 注文件数 | Order Count | `OrderService.getTodayOrderCount()` | COMPLETED 注文の件数 |
| ベストセラー | Best Seller | `OrderService.getBestSellerRanking()` | 商品別販売数ランキング |
| 時間帯別売上 | Hourly Sales | `OrderService.getHourlySales()` | 時間帯ごとの売上集計（24時間分） |
| 在庫切れリスク | Stockout Risk | `OrderService.getStockoutRisk()` | 在庫水準と直近販売量からのリスクスコア（HIGH/LOW） |

---

## 用語の不統一・重複（語彙のズレ）

### 1. パッケージ命名の揺れ
- `dao/`（OrderDao, InventoryDao 等）と `repository/`（PointDao, ReceiptDao）が並存
- 概念は同じ「データアクセス層」だが命名が分裂

### 2. ポイントトランザクションの type 値の揺れ
| 値 | 意味 |
|---|---|
| `EARN` | 通常の付与 |
| `EARN_REVERSED` | チェックアウト失敗時の補償 |
| `REVERSED` | 返品時の取消 |
| `EARN_REVERSED_CANCEL` | 返品の補償（取消の取消） |

→ 「reverse」「cancel」「rewind」相当のセマンティクスが入り乱れている

### 3. 在庫移動 reason 値の揺れ
| 値 | 意味 |
|---|---|
| `RECEIVE` | 入荷 |
| `SELL` | 販売 |
| `RETURN` | 返品 |
| `COMPENSATE` | チェックアウト補償 |
| `RETURN_COMPENSATE` | 返品補償 |

→ 補償系の用語が動詞化されておらず一貫性がない

### 4. 「ユーザー」と「会員」の概念分離
- `User`: システム操作者（Cashier/Manager/Admin）→ `users` テーブル（PG）
- `Member`: 会員ID 文字列（`member-001` 等）→ `point_balances` テーブル（PG）にしか実体がない
- 会員マスタとしての `members` テーブルは存在しない（外部マスタを想定？仕様上不明確）

### 5. ステータス文字列の重複定義
すべてのステータス値（OrderStatus, PaymentStatus, ReturnStatus, ReceiptStatus, PointTxType）が
String リテラルとしてコード中に散在。`enum` 化されていない（意図的なレガシー臭）。

---

## 用語数サマリ

- **正式用語**: 60+ 件
- **Bounded Context 候補**: 8 領域（商品・在庫・注文・カート・支払・ポイント・レシート・返品・ユーザー認証・監査）
- **不統一/重複**: 5 系統
::::

ユビキタス言語辞書のレポートを見ると、ドメインの概念を示す言葉や定義がコード上で十分に統一されておらず、複数の不整合が生じていることが分かります。

例えば、ポイント取引における取引タイプにEARNやEARN_REVERSED、REVERSED、EARN_REVERSED_CANCELといった用語が混在しています。
何がどのような操作を指しているのかが、一目で分かりにくい状態です。
また、在庫移動の理由を示す区分値などにおいても、動詞と名詞の使い分けが定まっておらず、一貫性がありません。
さらに、これらのステータスや区分を示す値がJavaのEnumとして定義されず、コードベースの各所に文字列として直接書かれて散在しています。そのため、タイポによるバグを検知しにくい状態です。

データアクセス層のクラス名においても、daoパッケージとrepositoryパッケージが並存するなど、システムの言葉としての統一感が欠けていると言えます。


### ドメインモデルの概念と実際のコード（クラス・テーブル等）の対応マッピング


![domain-code-mapping](/images/legacy-refactoring-nexus-scalardb/domain-code-mapping.png)
*ドメイン概念とコードのギャップなど*


::::details レポート全文
# ドメイン-コードマッピング — legacy-pos-monolith

## 概要

本ドキュメントは、ドメイン概念とコード実装の対応関係を整理し、
- ドメイン概念がどのレイヤに散在しているか
- ドメインルールがどこで実装されているか
- 概念とコードのギャップ
を可視化する。

---

## ドメイン概念 × コード対応マトリクス

### 1. Catalog（商品カタログ）BC 候補

| ドメイン概念 | コード実装 | レイヤ |
|---|---|---|
| 商品 (Product) | `domain.Product` | Domain |
| 商品マスタ管理 | `service.ProductService` | Service |
| 商品データ永続化 | `dao.ProductDao` | DAO |
| 商品 UI | `web.AdminProductController`, `templates/admin/products/*.html` | Web |
| バーコードスキャン | `RegisterController.scanBarcode()` → `ProductService.findByBarcode()` | Web→Service |
| 商品検索 | `ProductService.search(keyword, sortBy)` | Service |

### 2. Inventory（在庫管理）BC 候補

| ドメイン概念 | コード実装 | レイヤ |
|---|---|---|
| 在庫 (Inventory) | `domain.Inventory` | Domain |
| 在庫移動 (StockMovement) | `domain.StockMovement` | Domain |
| 入荷処理 | `InventoryService.receive()` | Service |
| 在庫引当 | `inventoryDao.decreaseStock()` ← `CheckoutSaga` から呼ぶ | DAO（Saga 経由） |
| 在庫戻し | `inventoryDao.increaseStock()` ← `ReturnSaga` から呼ぶ | DAO（Saga 経由） |
| 在庫切れリスク判定 | `OrderService.getStockoutRisk()` ⚠️ **OrderService に混在** | Service（誤配置） |
| 在庫水準 UI | `AdminInventoryController` | Web |

### 3. Order（注文）BC 候補

| ドメイン概念 | コード実装 | レイヤ |
|---|---|---|
| 注文 (Order) | `domain.Order` | Domain |
| 注文明細 (OrderItem) | `domain.OrderItem` | Domain |
| 注文クエリ | `OrderService.findById/findAll/findByDateRange/findTodayOrders` | Service |
| 注文ステータス遷移 | `OrderService.updateStatus()`, `cancelOrder()`, `markFailed()` | Service |
| 注文統計（日次・月次） | `OrderService.getTodaySalesTotal`, `getThisMonthSalesTotal` | Service（誤配置） |
| ベストセラーランキング | `OrderService.getBestSellerRanking()` | Service（誤配置） |
| 時間帯別売上 | `OrderService.getHourlySales()` | Service（誤配置） |
| 注文一覧 UI | `AdminOrderController`, `templates/admin/orders/*.html` | Web |

### 4. Cart（カート）BC 候補

| ドメイン概念 | コード実装 | レイヤ |
|---|---|---|
| カート (Cart) | `service.CartService`（@SessionScope） | Service（実体はセッション） |
| カート明細 (CartItem) | `CartService.CartItem` インナークラス | 内包 |
| カート操作 | `CartService.addItem/removeItem/updateQuantity/clear` | Service |
| カート→注文変換 | `RegisterController.checkout()` ⚠️ **コントローラに直書き** | Web（誤配置） |

### 5. Payment（支払）BC 候補

| ドメイン概念 | コード実装 | レイヤ |
|---|---|---|
| 支払 (Payment) | `domain.Payment` | Domain |
| 支払処理 | `CheckoutSaga.execute()` Step 3 ⚠️ **Saga に直書き** | Saga（誤配置） |
| 支払取消 | `ReturnSaga.execute()` Step 3 + `paymentDao.updateStatus(REFUNDED)` | Saga（誤配置） |
| 支払検索 | `PaymentDao.findByOrderId()` | DAO |
| **PaymentService が存在しない** | — | ❌ Service 層がない |

### 6. Point（ポイント）BC 候補

| ドメイン概念 | コード実装 | レイヤ |
|---|---|---|
| ポイント残高 (PointBalance) | `domain.PointBalance` | Domain |
| ポイント取引 (PointTransaction) | `domain.PointTransaction` | Domain |
| ポイントルール (PointRule) | `domain.PointRule` | Domain |
| ポイント残高照会 | `PointService.getBalance()` | Service |
| ポイント計算 | `PointService.calculatePoints()` ⚠️ + `Utils.calculatePointsEarned()` ⚠️ + `CheckoutSaga` 内で重複 | Service / Util / Saga（**3箇所重複**） |
| ポイント加算 | `CheckoutSaga.execute()` Step 4 ⚠️ | Saga（誤配置） |
| ポイント取消 | `ReturnSaga.execute()` Step 4 ⚠️ | Saga（誤配置） |
| ルール管理 | `PointService.createRule/updateRule` | Service |
| ポイント永続化 | `repository.PointDao` ⚠️ **repository パッケージ** | DAO |

### 7. Receipt（レシート）BC 候補

| ドメイン概念 | コード実装 | レイヤ |
|---|---|---|
| レシート (Receipt) | `domain.Receipt` | Domain |
| レシート発行 | `CheckoutSaga` Step 5 / `ReturnSaga` Step 5 ⚠️ | Saga（誤配置） |
| レシート閲覧 | `ReceiptService` | Service |
| レシート永続化 | `repository.ReceiptDao` ⚠️ **repository パッケージ** | DAO |
| レシート UI | `AdminReceiptController`, `templates/register/receipt.html` | Web |

### 8. Return（返品）BC 候補

| ドメイン概念 | コード実装 | レイヤ |
|---|---|---|
| 返品 (Return) | `domain.Return` | Domain |
| 返品明細 (ReturnItem) | `domain.ReturnItem` | Domain |
| 返品オーケストレーション | `saga.ReturnSaga` ⚠️ | Saga |
| 返品可能性チェック | `OrderService.canReturn()` ⚠️ + `ReturnSaga.isOrderReturnable()` ⚠️ **2箇所重複** | Service / Saga |
| 返品永続化 | `dao.ReturnDao` | DAO |
| 返品 UI | `AdminReturnController`, `templates/admin/returns/*.html` | Web |
| **ReturnService が存在しない** | — | ❌ Service 層がない |

### 9. Identity（ユーザー認証）BC 候補

| ドメイン概念 | コード実装 | レイヤ |
|---|---|---|
| ユーザー (User) | `domain.User` | Domain |
| ロール (Role) | `User.role` (String) | Domain（**String 直）** |
| 認証 | `security.PosUserDetailsService` | Security |
| 認可 | `security.SecurityConfig` | Security |
| ユーザー管理 | `service.UserService` | Service |
| ユーザー UI | `web.AdminUserController` | Web |
| ログイン UI | `web.LoginController` | Web |

### 10. Audit（監査）BC 候補

| ドメイン概念 | コード実装 | レイヤ |
|---|---|---|
| 監査ログ (AuditLog) | `domain.AuditLog` | Domain |
| 監査ログ記録 | `service.AuditService.record()` | Service |
| 監査ログ永続化 | `dao.AuditLogDao` | DAO |

---

## ビジネスルールの実装箇所トレース

### ルール 1: 税計算（標準10% / 軽減8%）

**ドメインルール**: tax_category=1 → 10%, tax_category=2 → 8%

**実装箇所（4箇所に重複）**:
1. `Utils.calculateTax(priceYen, taxCategory)` — 汎用ユーティリティ
2. `CartService.getTaxYen()` — カート集計時
3. `CheckoutSaga.execute()` 内インライン — Step 1 注文作成時
4. `OrderService.calculateOrderTax()` — 公開ヘルパー

**問題**: 同一ロジックが 4 箇所、税率変更時の修正漏れリスク。

---

### ルール 2: ポイント計算（金額 × multiplier ÷ 100）

**ドメインルール**: 注文金額 × multiplier ÷ 100 = 付与ポイント。デフォルト multiplier=1.00（=1%）。

**実装箇所（4箇所に重複）**:
1. `Utils.calculatePointsEarned(totalYen, multiplier)` — 汎用ユーティリティ
2. `CheckoutSaga.execute()` Step 4 — multiplier=1.00 ハードコード（ルールが空のとき）
3. `CheckoutSaga.determinePointsEarned()` — 未使用ヘルパー
4. `PointService.calculatePoints()` — Service 層実装、`totalYen / 100` ハードコード

**問題**: ルール選択ロジック（カテゴリ別、期間限定）が `CheckoutSaga` で省略されており、常に最初のルールが使われる。

---

### ルール 3: 注文ステータス遷移

**ドメインルール**: PENDING → COMPLETED / CANCELLED / FAILED → RETURNED（COMPLETED から）

**実装箇所**: `OrderService.updateStatus(orderId, status)`

**問題**:
- ステータス遷移の正当性チェックが **存在しない**（COMPLETED → PENDING に戻すことも可能）
- ロールチェックが **存在しない**（任意のロールから呼べる）
- 遷移グラフがコード上に明示されていない

---

### ルール 4: 返品可能性

**ドメインルール**:
- 注文が COMPLETED であること
- 既存の COMPLETED または PENDING な返品が存在しないこと

**実装箇所（2箇所に重複）**:
1. `OrderService.canReturn(orderId)` — 公開メソッド
2. `ReturnSaga.isOrderReturnable(orderId)` — 内部メソッド（実際は使われていない）

加えて、`ReturnSaga.execute()` 内に **再びインラインで同様のチェック** が書かれている（Step 0）。

**問題**: 3 箇所に類似ロジックが分散。`ReturnSaga.execute()` の Step 0 が事実上のオーソリティだが、外部からは `OrderService.canReturn()` が使われる。

---

### ルール 5: チェックアウトの冪等性

**ドメインルール**: 同一 idempotencyKey のリクエストは1度のみ処理し、以降は前回結果を返す

**実装箇所**: `CheckoutSaga.execute()` Step 0

**問題**:
- `IdempotencyKey.resultPayload` が `"orderId|receiptId"` のパイプ区切り文字列（壊れやすい）
- FAILED 時はキー削除して再試行を許可するが、これは冪等性の保証としては弱い

---

### ルール 6: 在庫引当の競合制御

**ドメインルール**: 並行チェックアウト時に在庫不足を検知すること

**実装箇所**: `CheckoutSaga.execute()` Step 2

**問題**:
- `SELECT FOR UPDATE` を使っていない
- `findByProductId()` → `availableQuantity` 比較 → `decreaseStock()` の間に競合があり、過剰引当が発生しうる
- `inventory.reserved_quantity` カラムがあるのに使われていない

---

### ルール 7: 監査ログの記録対象

**実装箇所**: 各種オペレーションで `auditService.record(action, target, payload)` を呼ぶ

**問題**:
- 呼び出し漏れ箇所が存在する
- `record()` 自体が例外を握り潰しているため、記録失敗が検知できない

---

## ドメイン概念とコードのギャップ

### ❌ コードに存在しないドメイン概念

| ドメイン概念 | 期待される実装 | 現状 |
|---|---|---|
| **Money（金額）** 値オブジェクト | `Money(amount, currency)` 等 | int の primitive |
| **TaxCategory** Enum | `TaxCategory.STANDARD / REDUCED` | int 1/2 |
| **OrderStatus** Enum | `OrderStatus.PENDING / COMPLETED ...` | String 文字列 |
| **PaymentMethod** Enum | `PaymentMethod.CASH / CARD ...` | String 文字列 |
| **Member** ドメイン | 会員マスタ・ドメイン | `member_id` String のみ。`members` テーブルなし |
| **PaymentService** | 支払責務の集約 | Saga 内に散在 |
| **ReturnService** | 返品責務の集約 | Saga 内に散在 |
| **Cashier UseCase / OrderUseCase** | アプリケーション層 | Service と Saga と Controller に分散 |

### ⚠️ ドメイン概念の誤配置

| 概念 | 本来の所属 | 現状の所属 |
|---|---|---|
| 在庫切れリスク判定 | InventoryService | `OrderService.getStockoutRisk()` |
| ベストセラー集計 | DashboardService（あるべき） | `OrderService.getBestSellerRanking()` |
| 時間帯別売上 | DashboardService（あるべき） | `OrderService.getHourlySales()` |
| 月次・日次統計 | DashboardService（あるべき） | `OrderService.getThisMonthSalesTotal()` 等 |
| カート→Checkout 変換 | アプリケーションサービス | `RegisterController.checkout()` |
| 支払処理 | PaymentService（あるべき） | `CheckoutSaga` Step 3 |
| 返品ロジック | ReturnService（あるべき） | `ReturnSaga.execute()` |

### ⚠️ DB と BC 境界の不一致

| 物理 DB | 含まれるテーブル | 対応 BC |
|---|---|---|
| pos_pg (PostgreSQL) | users, products, point_*, receipts, audit_logs | Identity / Catalog / Point / Receipt / Audit（**5 BC が混在**） |
| pos_mysql (MySQL) | inventory, stock_movements, orders, order_items, payments, returns, return_items, idempotency_keys | Inventory / Order / Payment / Return（**4 BC が混在**） |

→ DB 境界は分類軸が「データ管理者」ベース（おそらく歴史的経緯）で、Bounded Context 境界とは無関係。

---

## レイヤ違反一覧

| 違反 | 場所 |
|---|---|
| Service が DAO 層をスキップして直接 JdbcTemplate を使用 | `OrderService.getHourlySales()` (line 392) |
| コントローラに業務ロジック | `RegisterController.checkout()` |
| Thymeleaf テンプレートに業務ロジック（税計算等） | templates 各所（CLAUDE.md にも明記） |
| Saga が複数 BC を直接操作 | `CheckoutSaga`, `ReturnSaga` |
| 集約ルートが定義されていない | Order が OrderItem を集約管理していない |

---

## サマリ

- **Bounded Context 候補**: 10 領域
- **コードに欠落しているドメイン概念**: 8 件
- **重複実装されているドメインルール**: 5 件（税計算 4 重、ポイント計算 4 重、返品可否 3 重 ほか）
- **誤配置されているドメインロジック**: 7 件
- **DB 境界と BC 境界の不一致**: 2 つの物理 DB がそれぞれ複数 BC を内包
::::

ドメインとコードの対応関係のレポートを分析すると、ビジネスルールがドメインモデル内にカプセル化されず、システム全体に分散している状況が明確になります。

具体的には、消費税の計算ロジックがUtilsやCartService、CheckoutSaga、OrderServiceといった4つの異なる場所に重複して記述されています。
そのため、ビジネスルールの変更があった場合に修正漏れが発生しやすい構造になっています。

また、金額を表現するMoney型や、税カテゴリを表現するEnumなどのドメイン概念を表現するクラスが存在せず、すべてプリミティブ型や文字列で扱われています。
このようにドメインモデルが単なるデータ保持用のPOJOになってしまっています。
そのため、注文ステータスの不正な遷移を防ぐガードロジックや、並行アクセス時における在庫引当の競合制御といった重要なビジネスルールが、サービスやSagaの処理フローの中に手続き的に記述されていると言えます。

## 本章のまとめ

* ドメイン分析では、業務スコープ、アクター、権限、ユビキタス言語、ドメイン概念とコードの対応を確認しました。
* 税計算、ポイント計算、返品可否などのビジネスルールが複数箇所に分散し、ドメインモデルに閉じ込められていないことが分かりました。
* DB境界とBounded Context境界が一致していないため、設計改善では業務境界とデータ配置をあらためて整理する必要があります。

## 用語解説

### アクター
システムを利用する人や外部システムを表す概念です。レジ担当者、店舗管理者、システム管理者などが該当します。

### ロール
ユーザーに割り当てられる役割です。どの画面や機能を使えるかを決める認可の単位になります。

### パーミッション
具体的な操作権限です。たとえば商品の登録、在庫の更新、返品処理の実行など、機能単位で整理されます。

### ドメインコードマッピング
業務上の概念と、実際のクラス、テーブル、メソッドがどのように対応しているかを整理したものです。

### 複数DBにまたがる原子性
複数のデータベースに対する一連の更新を、すべて成功するか、すべて失敗するかのどちらかにそろえる性質です。本章のチェックアウトでは、MySQL側の注文・在庫・支払と、PostgreSQL側のポイント・レシート更新をまとめて扱う必要があります。そのため、この原子性が課題になります。

### POJO
Plain Old Java Objectの略で、特別なフレームワーク機能に依存しない素朴なJavaオブジェクトです。本章では、単にデータを入れるだけで、注文ステータス遷移や在庫引当のようなビジネスルールを持たないドメインクラスという意味で使っています。
