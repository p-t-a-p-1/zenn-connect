---
title: "【Nexus Architect本レビュー用】（第11回）評価・採点レポートを読み解く"
emoji: "📘"
type: "tech"
topics: ["architecture","scalardb","refactoring"]
published: false
publication_name: "scalar_sol_blog"
---


:::message
MMI・DDD・統合評価レポートから、スコアの意味、構造的な弱点、改善アクションの優先順位を読み解きます。
:::

## 評価・採点

### 各レポート内容の概要

| ファイル | 内容 |
| :--- | :--- |
| mmi-overview.md | MMI スコア（マイクロサービス成熟度指数）の概要 |
| mmi-by-module.md | モジュールごとの MMI 詳細採点 |
| ddd-strategic-evaluation.md | DDD 戦略層の評価（境界コンテキスト・ユビキタス言語など）。 |
| ddd-tactical-architecture-evaluation.md | DDD 戦術層（集約・リポジトリ・ドメインイベントなど）の評価 |
| integrated-evaluation.md | MMI と DDD を統合した総合スコア（33.6%）とクロス評価マトリクス |
| unified-improvement-plan.md | 評価結果から導いた改善アクション一覧（P0〜P3 の優先度付き） |


### MMI スコア（マイクロサービス成熟度指数）の概要

![mmi-overview](/images/legacy-refactoring-nexus-scalardb/mmi-overview.png)
*MMI スコア（マイクロサービス成熟度指数）の概要*

::::details レポート全文
---
title: "MMI評価: 概要"
schema_version: 1
phase: "Phase 2: Evaluation"
skill: evaluate-mmi
generated_at: 2026-05-14T00:00:00Z
input_files:
  - reports/01_analysis/domain-code-mapping.md
  - reports/before/legacy-pos-monolith/issues-and-debt.md
---

# MMI評価: 概要

## サマリ

| 指標 | 値 |
|---|---|
| システム平均 MMI | **46%** |
| 成熟度レベル | **Needs Improvement** |
| 評価モジュール数 | 14 |
| Mature（80-100%） | 0 |
| Moderate（60-80%） | 3 (Catalog 68%, Identity 72%, Audit 72%) |
| Needs Improvement（40-60%） | 5 (Cart 58%, Inventory 52%, Receipt 52%, Point 46%, util 42%) |
| Immature（0-40%） | 6 (Order 32%, Payment 32%, Return 32%, web 32%, CheckoutSaga 26%, ReturnSaga 26%) |

---

## モジュール別 MMI スコア

```
Audit          ████████████████████████████████████████████████████████████████████░░░ 72%
Identity       ████████████████████████████████████████████████████████████████████░░░ 72%
Catalog        ████████████████████████████████████████████████████████████████░░░░░░░ 68%
Cart           ████████████████████████████████████████████████████████░░░░░░░░░░░░░░ 58%
Inventory      ████████████████████████████████████████████████████░░░░░░░░░░░░░░░░░░ 52%
Receipt        ████████████████████████████████████████████████████░░░░░░░░░░░░░░░░░░ 52%
Point          ██████████████████████████████████████████████░░░░░░░░░░░░░░░░░░░░░░░░ 46%
util           ██████████████████████████████████████████░░░░░░░░░░░░░░░░░░░░░░░░░░░░ 42%
Order          ████████████████████████████████░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░ 32%
Payment        ████████████████████████████████░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░ 32%
Return         ████████████████████████████████░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░ 32%
web            ████████████████████████████████░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░ 32%
CheckoutSaga   ██████████████████████████░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░ 26%
ReturnSaga     ██████████████████████████░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░ 26%
               ├─────────────────────────┼──────────────────────────┼──────────────────┤
               0%                       40%                         80%              100%
               [─────── Immature ───────][── Needs Improvement ─────][── Moderate ─][Ready]
```

---

## 軸別の全体傾向

| 軸 | 平均スコア | 所感 |
|---|---|---|
| 凝集性 (C, weight 30%) | **2.71 / 5** | God Service/Class が複数存在し、全体平均を大きく引き下げている |
| 結合性 (K, weight 30%) | **2.43 / 5** | Saga が全 DAO に直接依存。util がグローバル結合点 |
| 独立性 (I, weight 20%) | **1.86 / 5** | モノリス構造・クロスDB Saga により独立性は全モジュールで低い |
| 再利用性 (R, weight 20%) | **1.79 / 5** | PaymentService/ReturnService 不在、内部クラス多用で再利用性が極めて低い |

---

## 改善優先度（上位5項目）

### 1. CheckoutSaga / ReturnSaga の解体（最優先）
- 現MMI: **26%**（最低ランク）
- 問題: 7つのDAOに直接依存するGod Class、クロスDB操作、Sagaステート消失
- **ScalarDB 導入で Saga 自体を不要化**するか、ステップクラスへの分割が急務
- 推定改善後MMI: +30〜40ポイント（ScalarDB 適用 + ステップ分割で 60%+ が目標）

### 2. OrderService の分割（God Service 解体）
- 現MMI: **32%**
- 問題: 976行、注文CRUD/統計/ランキング/在庫リスク/返品可否が混在
- 分割候補: OrderQueryService, OrderCommandService, DashboardService
- 推定改善後MMI: +25〜30ポイント（60%台が目標）

### 3. PaymentService / ReturnService の新設
- 現MMI: Payment **32%**, Return **32%**
- 問題: Service層が存在せず、Saga内に直書き
- CheckoutSaga/ReturnSaga 解体と同時に実施
- 推定改善後MMI: +20〜25ポイント

### 4. web 層の分割（HTML画面 vs REST API）
- 現MMI: **32%**
- 問題: RegisterController が画面遷移・JSON API・業務ロジックを混在
- 分割候補: RegisterViewController（Thymeleaf）+ RegisterApiController（REST）
- 推定改善後MMI: +15〜20ポイント

### 5. util のリファクタリング（ビジネスロジック抽出）
- 現MMI: **42%**
- 問題: 税計算・ポイント計算がUtils内に混在かつ4箇所で重複
- 対策: 税計算は TaxCalculator、ポイント計算は PointCalculator へ分離
- 推定改善後MMI: +15ポイント

---

## マイクロサービス移行準備度の判定

**総合判定: ❌ マイクロサービス移行には重大な前処理が必要**

| 判定観点 | 結果 | 根拠 |
|---|---|---|
| Matureモジュール存在 | ❌ 0件 | 80%以上のモジュールが存在しない |
| Saga の DB 跨ぎ | ❌ 解消必須 | CheckoutSaga/ReturnSaga が MySQL+PG を直接更新、ScalarDB なしでは分離不可 |
| Service 層の完備 | ❌ 不完全 | PaymentService/ReturnService が存在しない |
| ドメイン境界の明確さ | ⚠️ 要整理 | 10 BC 候補は識別できるが、God Service が境界を崩壊させている |
| テスタビリティ | ❌ 極めて低い | God Class による単体テスト不可 |
| 依存管理 | ❌ 問題あり | Saga から全DAO直接依存、util がグローバル結合点 |

### 推奨移行経路

```
Phase 1（前提条件整備）: God Service/Saga 解体 + ScalarDB 導入
    ↓
Phase 2（BC 境界強化）: PaymentService/ReturnService 新設、パッケージ再編
    ↓
Phase 3（マイクロサービス分離）: Catalog, Identity, Audit から独立サービス化
    ↓
Phase 4（コア分離）: Order, Inventory, Payment の段階的分離
```

---

## 計算根拠

| モジュール | C | K | I | R | MMI計算 | MMI% |
|---|---|---|---|---|---|---|
| Catalog | 4 | 4 | 2 | 3 | (0.3×4+0.3×4+0.2×2+0.2×3)/5×100 | 68% |
| Inventory | 3 | 3 | 2 | 2 | (0.3×3+0.3×3+0.2×2+0.2×2)/5×100 | 52% |
| Order | 2 | 2 | 1 | 1 | (0.3×2+0.3×2+0.2×1+0.2×1)/5×100 | 32% |
| Cart | 4 | 3 | 2 | 2 | (0.3×4+0.3×3+0.2×2+0.2×2)/5×100 | 58% |
| Payment | 2 | 2 | 1 | 1 | (0.3×2+0.3×2+0.2×1+0.2×1)/5×100 | 32% |
| Point | 3 | 2 | 2 | 2 | (0.3×3+0.3×2+0.2×2+0.2×2)/5×100 | 46% |
| Receipt | 3 | 3 | 2 | 2 | (0.3×3+0.3×3+0.2×2+0.2×2)/5×100 | 52% |
| Return | 2 | 2 | 1 | 1 | (0.3×2+0.3×2+0.2×1+0.2×1)/5×100 | 32% |
| Identity | 4 | 4 | 3 | 3 | (0.3×4+0.3×4+0.2×3+0.2×3)/5×100 | 72% |
| Audit | 5 | 3 | 3 | 3 | (0.3×5+0.3×3+0.2×3+0.2×3)/5×100 | 72% |
| CheckoutSaga | 2 | 1 | 1 | 1 | (0.3×2+0.3×1+0.2×1+0.2×1)/5×100 | 26% |
| ReturnSaga | 2 | 1 | 1 | 1 | (0.3×2+0.3×1+0.2×1+0.2×1)/5×100 | 26% |
| web | 2 | 2 | 1 | 1 | (0.3×2+0.3×2+0.2×1+0.2×1)/5×100 | 32% |
| util | 1 | 2 | 4 | 2 | (0.3×1+0.3×2+0.2×4+0.2×2)/5×100 | 42% |
| **平均** | **2.71** | **2.43** | **1.86** | **1.79** | | **46%** |
::::


### モジュールごとの MMI 詳細採点


![mmi-by-module](/images/legacy-refactoring-nexus-scalardb/mmi-by-module.png)
*モジュールごとの MMI 詳細採点*

::::details レポート全文
---
title: "MMI評価: モジュール別詳細"
schema_version: 1
phase: "Phase 2: Evaluation"
skill: evaluate-mmi
generated_at: 2026-05-14T00:00:00Z
input_files:
  - reports/01_analysis/domain-code-mapping.md
  - reports/01_analysis/system-overview.md
  - reports/before/legacy-pos-monolith/issues-and-debt.md
---

# MMI評価: モジュール別詳細

## スコア一覧

| モジュール | 凝集性 (C) | 結合性 (K) | 独立性 (I) | 再利用性 (R) | MMI | 成熟度 |
|---|---|---|---|---|---|---|
| Catalog | 4/5 | 4/5 | 2/5 | 3/5 | **68%** | Moderate |
| Inventory | 3/5 | 3/5 | 2/5 | 2/5 | **52%** | Needs Improvement |
| Order | 2/5 | 2/5 | 1/5 | 1/5 | **32%** | Immature |
| Cart | 4/5 | 3/5 | 2/5 | 2/5 | **58%** | Needs Improvement |
| Payment | 2/5 | 2/5 | 1/5 | 1/5 | **32%** | Immature |
| Point | 3/5 | 2/5 | 2/5 | 2/5 | **46%** | Needs Improvement |
| Receipt | 3/5 | 3/5 | 2/5 | 2/5 | **52%** | Needs Improvement |
| Return | 2/5 | 2/5 | 1/5 | 1/5 | **32%** | Immature |
| Identity | 4/5 | 4/5 | 3/5 | 3/5 | **72%** | Moderate |
| Audit | 5/5 | 3/5 | 3/5 | 3/5 | **72%** | Moderate |
| CheckoutSaga | 2/5 | 1/5 | 1/5 | 1/5 | **26%** | Immature |
| ReturnSaga | 2/5 | 1/5 | 1/5 | 1/5 | **26%** | Immature |
| web | 2/5 | 2/5 | 1/5 | 1/5 | **32%** | Immature |
| util | 1/5 | 2/5 | 4/5 | 2/5 | **42%** | Needs Improvement |

> MMI = (0.3×C + 0.3×K + 0.2×I + 0.2×R) / 5 × 100

---

## モジュール別詳細

### Catalog（商品カタログ）— MMI: 68% / Moderate

- **凝集性 (4/5)**: ProductService・ProductDao・domain.Productの3層が商品マスタ管理に集中しており責務は明確。バーコードスキャンがRegisterControllerから呼ばれる程度の軽微な依存はあるが、全体的には凝集度が高い。
- **結合性 (4/5)**: ProductService → ProductDao という単純な一方向依存のみ。RegisterControllerからProductServiceへの依存はあるが、Catalog自体は他モジュールへの依存が少なく比較的独立している。
- **独立性 (2/5)**: ProductServiceはOrderServiceから直接参照され、単一JARモノリス内にデプロイされるため独立したデプロイは不可能。PostgreSQL依存は限定的だが、Controller層との混在により単体テストも困難。
- **再利用性 (3/5)**: Spring/JdbcTemplateに依存しており、他フレームワークへの移植には修正が必要。ただしドメインモデル（Product）は比較的独立しており、インターフェースも明確で部分的な再利用は可能。
- **MMI**: (0.3×4 + 0.3×4 + 0.2×2 + 0.2×3) / 5 × 100 = **68%**（Moderate）

---

### Inventory（在庫管理）— MMI: 52% / Needs Improvement

- **凝集性 (3/5)**: 在庫引当・在庫戻しがInventoryServiceではなくCheckoutSaga/ReturnSagaから直接DAOを呼ぶ形になっており、在庫切れリスク判定がOrderServiceに誤配置されるなど責務漏れが複数存在する。
- **結合性 (3/5)**: InventoryServiceは自律的だが、在庫引当・戻しはCheckoutSaga/ReturnSagaがInventoryDaoに直接依存する形で呼び出され、サービス層をバイパスしている。在庫切れリスク判定がOrderServiceに誤配置されており双方向の暗黙的依存が生じている。
- **独立性 (2/5)**: InventoryDaoはOrderServiceとCheckoutSaga/ReturnSagaから直接呼び出されており、独立したデプロイやバージョニングは不可能。MySQL依存は単一DBに限定されるが、モノリス内での結合度が高い。
- **再利用性 (2/5)**: 在庫引当・戻しのコアロジックがCheckoutSaga/ReturnSagaから直接DAOを呼ぶ形で分散しており、在庫切れリスク判定がOrderServiceに誤配置されているため、Inventoryとして独立した再利用が困難。
- **MMI**: (0.3×3 + 0.3×3 + 0.2×2 + 0.2×2) / 5 × 100 = **52%**（Needs Improvement）

---

### Order（注文管理）— MMI: 32% / Immature

- **凝集性 (2/5)**: OrderServiceが976行のGod Serviceで、注文CRUD・ステータス遷移だけでなく、ダッシュボード統計・ランキング・時間帯別売上・在庫リスク判定・返品可否チェックまで含む複数の無関係な責務が混在している。
- **結合性 (2/5)**: OrderService（976行）がダッシュボード統計・返品可否・ポイント計算・在庫リスクなど他BCのロジックを取り込んでおり、ファンアウトが非常に高い。また直接JdbcTemplateを使用するレイヤ違反もあり変更時の影響範囲が広い。
- **独立性 (1/5)**: OrderService（976行）がInventoryDao・PointDao・ReceiptDaoを直接呼び出すGod Serviceであり、MySQL/PostgreSQL両方に依存する。単独でのデプロイ・テスト・バージョニングは実質不可能。
- **再利用性 (1/5)**: OrderServiceは976行のGod Serviceであり、注文CRUD・ダッシュボード統計・ベストセラーランキング・在庫切れリスク・税計算・返品可否チェックなど無関係な責務が混在しており、コンテキストを超えた再利用はほぼ不可能。
- **MMI**: (0.3×2 + 0.3×2 + 0.2×1 + 0.2×1) / 5 × 100 = **32%**（Immature）

---

### Cart（カート）— MMI: 58% / Needs Improvement

- **凝集性 (4/5)**: CartServiceはセッションスコープのカート操作に集中しており、CartItemも内包クラスとして管理されている。カート→注文変換がRegisterControllerに漏れている点のみが惜しい。
- **結合性 (3/5)**: CartServiceはセッションスコープで比較的独立しているが、RegisterControllerがCheckoutSagaへの変換ロジックを直接実装しており、Cart→Checkout変換の責務がControllerに漏れ出している。
- **独立性 (2/5)**: CartServiceはDB非依存のインメモリセッション状態だが、RegisterControllerと密結合している。DBなしで単体テスト可能だが、モノリスから切り離したデプロイはできない。
- **再利用性 (2/5)**: CartServiceが@SessionScopeでHTTPセッションに密結合しており、バッチ処理やAPI専用文脈では再利用できない。CartItemがインナークラスとして閉じ込められている点も独立した利用を阻む。
- **MMI**: (0.3×4 + 0.3×3 + 0.2×2 + 0.2×2) / 5 × 100 = **58%**（Needs Improvement）

---

### Payment（支払）— MMI: 32% / Immature

- **凝集性 (2/5)**: PaymentServiceが存在せず、支払処理・取消がCheckoutSaga/ReturnSagaに直書きされており、PaymentDaoとdomain.Paymentだけでは支払責務が完結しない。Service層の欠落が凝集度を著しく下げている。
- **結合性 (2/5)**: PaymentServiceが存在せず、支払処理ロジックがCheckoutSagaとReturnSagaに直接埋め込まれている。PaymentDaoへの依存がSagaから直接発生しており、支払BCを独立して変更することが困難。
- **独立性 (1/5)**: PaymentServiceが存在せずPaymentDaoのみで、CheckoutSagaとReturnSagaから直接操作される。独立したビジネスロジック境界がなく、単独テスト・デプロイは不可能。
- **再利用性 (1/5)**: PaymentServiceが存在せず、支払処理ロジックがCheckoutSaga/ReturnSagaのstepに直書きされているため、抽象化・インターフェースが皆無であり再利用性は最低レベル。
- **MMI**: (0.3×2 + 0.3×2 + 0.2×1 + 0.2×1) / 5 × 100 = **32%**（Immature）

---

### Point（ポイント管理）— MMI: 46% / Needs Improvement

- **凝集性 (3/5)**: PointServiceにルール管理・残高照会・計算が揃っているが、ポイント計算ロジックがUtils・CheckoutSaga・PointServiceの3箇所に重複し、ポイント加算・取消もSaga内に散在するため境界が曖昧。
- **結合性 (2/5)**: ポイント計算ロジックがUtils・CheckoutSaga・PointServiceの3箇所に重複実装されており、暗黙的な依存が広がっている。CheckoutSaga/ReturnSagaがPointDaoに直接依存しており、PointServiceを経由しない呼び出しパスが存在する。
- **独立性 (2/5)**: PointServiceは存在するがCheckoutSagaから直接呼び出され、パッケージも`repository/PointDao`と命名揺れがある。PostgreSQL単一DB依存だが、Sagaとの密結合で独立デプロイは不可能。
- **再利用性 (2/5)**: PointServiceは一応Service層として存在するが、ポイント計算ロジックがUtils・CheckoutSaga・PointServiceの3箇所に重複しており、どれが正規実装か不明瞭で安全な再利用が難しい。
- **MMI**: (0.3×3 + 0.3×2 + 0.2×2 + 0.2×2) / 5 × 100 = **46%**（Needs Improvement）

---

### Receipt（レシート）— MMI: 52% / Needs Improvement

- **凝集性 (3/5)**: ReceiptServiceとReceiptDaoはレシート閲覧・永続化に集中しているが、レシート発行ロジックがCheckoutSaga/ReturnSagaのStep5に誤配置されており、核心的な責務がモジュール外に存在する。
- **結合性 (3/5)**: ReceiptServiceは閲覧に限定されており比較的独立しているが、レシート発行ロジックがCheckoutSaga/ReturnSagaに直接埋め込まれており、Saga側からReceiptDaoへの直接依存が存在する。
- **独立性 (2/5)**: ReceiptServiceは存在するがCheckoutSaga/ReturnSagaおよびOrderServiceから直接参照される。`repository/ReceiptDao`と命名揺れがあり、PostgreSQL単一DB依存だが独立デプロイは不可能。
- **再利用性 (2/5)**: ReceiptServiceはレシート閲覧に特化しているが、レシート発行ロジックはCheckoutSaga/ReturnSagaに散在しており、レシートモジュール単体として完結していないため再利用には大幅な改修が必要。
- **MMI**: (0.3×3 + 0.3×3 + 0.2×2 + 0.2×2) / 5 × 100 = **52%**（Needs Improvement）

---

### Return（返品）— MMI: 32% / Immature

- **凝集性 (2/5)**: ReturnServiceが存在せず、返品ロジックの大半がReturnSagaに散在し、返品可否チェックがOrderService・ReturnSaga・Saga内インラインの3箇所に重複している。DAOとdomain classだけでは返品モジュールとして機能しない。
- **結合性 (2/5)**: ReturnServiceが存在せず返品ロジックがReturnSagaに集約され、返品可否チェックがOrderServiceとReturnSagaの2箇所に重複している。ReturnDaoへの依存がSaga経由のみとなり、返品BCを独立して操作できない。
- **独立性 (1/5)**: ReturnServiceが存在せずReturnDaoのみで、ReturnSagaが全ロジックを担う。独立したビジネスロジック層がなく、MySQL依存も他モジュールと共有されており単独テスト・デプロイは不可能。
- **再利用性 (1/5)**: ReturnServiceが存在せず、返品ロジックがReturnSagaに全て埋め込まれており、返品可否チェックも3箇所に重複している。独立した再利用は実質不可能。
- **MMI**: (0.3×2 + 0.3×2 + 0.2×1 + 0.2×1) / 5 × 100 = **32%**（Immature）

---

### Identity（ユーザー認証）— MMI: 72% / Moderate

- **凝集性 (4/5)**: SecurityConfig・PosUserDetailsService・UserService・UserDaoがユーザー認証・認可に集中しており、ロールがString直書きである点を除けば責務の凝集度は高い。
- **結合性 (4/5)**: SecurityConfig・PosUserDetailsService・UserService・UserDaoの依存関係が一方向で整理されており、他モジュールからIdentityへの依存は認証フィルター経由に限定されている。比較的疎結合。
- **独立性 (3/5)**: SecurityConfig・UserService・UserDaoはSpring Security設定として比較的独立しており、PostgreSQL単一DB依存。他モジュールからの直接呼び出しは少ないが、同一JARデプロイのため完全な独立性はない。
- **再利用性 (3/5)**: SecurityConfigとUserServiceはSpring Security固有の実装だが、認証・認可の責務は比較的明確に分離されており、Spring Securityを維持する限り他プロジェクトへの転用は現実的。ロールがString型である点はマイナス。
- **MMI**: (0.3×4 + 0.3×4 + 0.2×3 + 0.2×3) / 5 × 100 = **72%**（Moderate）

---

### Audit（監査）— MMI: 72% / Moderate

- **凝集性 (5/5)**: AuditService・AuditLogDao・domain.AuditLogの3層が監査ログ記録のみに特化しており、呼び出し漏れはあるものの、モジュール内部の責務は単一かつ明確に保たれている。
- **結合性 (3/5)**: AuditServiceはシンプルな単方向依存（AuditService→AuditLogDao）で自律的だが、複数のController・Saga・Serviceから直接呼び出されるファンインが高く、呼び出し漏れも存在する。
- **独立性 (3/5)**: AuditService・AuditLogDaoはPostgreSQL単一DBに依存し、他モジュールへの依存がない単方向関係。ただしモノリスJAR内に閉じており独立デプロイは不可能。
- **再利用性 (3/5)**: AuditServiceはrecord(action, target, payload)というシンプルなインターフェースで薄いラッパー構造のため、他コンテキストからも呼びやすい。ただし例外を握り潰す実装と呼び出し漏れにより信頼性が低い。
- **MMI**: (0.3×5 + 0.3×3 + 0.2×3 + 0.2×3) / 5 × 100 = **72%**（Moderate）

---

### CheckoutSaga — MMI: 26% / Immature

- **凝集性 (2/5)**: 557行の単一executeメソッドに6ステップ＋補償処理が集約されており、支払処理・ポイント加算・レシート発行・冪等性チェックなど本来は別モジュールの責務が混在するGod Classとなっている。
- **結合性 (1/5)**: OrderDao・InventoryDao・PaymentDao・PointDao・ReceiptDao・IdempotencyKeyDao・AuditServiceすべてに直接依存し、557行のGod ClassとしてDBを跨いで複数BCを一手に担っている。結合度は最高レベル。
- **独立性 (1/5)**: MySQL（orders・inventory・payments）とPostgreSQL（receipts・point_balances）の両方を直接更新するクロスDB God Class。Sagaステート永続化なしでプロセス再起動時にロスト。完全にモノリス内に閉じている。
- **再利用性 (1/5)**: CheckoutRequest・CheckoutItem・CheckoutResultがSaga内インナークラスとして閉じ込められており、税計算・ポイント計算・在庫引当・支払処理が一つのGod Classに混在しているため他コンテキストへの再利用は不可能。
- **MMI**: (0.3×2 + 0.3×1 + 0.2×1 + 0.2×1) / 5 × 100 = **26%**（Immature）

---

### ReturnSaga — MMI: 26% / Immature

- **凝集性 (2/5)**: 448行に6ステップ＋補償が集約され、返品・支払取消・ポイント取消・レシート発行が単一クラスに混在。ReturnServiceが存在しない穴を埋める形で責務過多になっている。
- **結合性 (1/5)**: ReturnDao・OrderDao・InventoryDao・PaymentDao・PointDao・ReceiptDao・AuditServiceすべてに直接依存し、448行のGod Classとして返品に関わる全BCを横断的に操作している。結合度は最高レベル。
- **独立性 (1/5)**: CheckoutSagaと同様にMySQL・PostgreSQL両方を直接操作するクロスDB God Class。補償処理の例外握り潰しとSagaステート永続化なしにより、独立したデプロイ・テスト・バージョニングは完全に不可能。
- **再利用性 (1/5)**: ReturnRequest・ReturnResultがSaga内インナークラスとして閉じ込められており、返品ロジック全体がSagaのif/elseフローに直書きされているため抽象化が皆無で再利用は不可能。
- **MMI**: (0.3×2 + 0.3×1 + 0.2×1 + 0.2×1) / 5 × 100 = **26%**（Immature）

---

### web（Controller層）— MMI: 32% / Immature

- **凝集性 (2/5)**: RegisterControllerに画面遷移・JSON API・チェックアウト業務ロジックが混在し、AdminOrderControllerにはダッシュボード集計の呼び出しが含まれるなど、Controller層全体でWeb表示とビジネスロジックの境界が曖昧。
- **結合性 (2/5)**: RegisterControllerがCartService・CheckoutSaga・ProductService・ReceiptService・AuditService・OrderDaoに依存し、画面遷移とJSON APIが混在している。Controller層全体のファンアウトが非常に高い。
- **独立性 (1/5)**: 全コントローラが単一JARに同居し、画面遷移とJSON APIが混在。RegisterController・DashboardController等が複数サービス層を直接参照しており、独立したデプロイは構造的に不可能。
- **再利用性 (1/5)**: ThymeleafのSSR画面遷移とJSON REST APIが同一Controllerクラスに混在しており、Spring MVC・Thymeleaf・HTTPセッションへの強依存があるため、他フレームワークや他UIコンテキストへの再利用は不可能。
- **MMI**: (0.3×2 + 0.3×2 + 0.2×1 + 0.2×1) / 5 × 100 = **32%**（Immature）

---

### util — MMI: 42% / Needs Improvement

- **凝集性 (1/5)**: Utils・CommonUtilに税計算・ポイント計算・MD5ハッシュ・JSON直列化・ログサニタイズなど無関係な処理が雑多に詰め込まれており、単一責務がなくユーティリティダンプグラウンドのパターンを示している。
- **結合性 (2/5)**: Utils/CommonUtilは税計算・ポイント計算・ID生成・JSONシリアライズなど複数BCのビジネスロジックを静的メソッドとして提供しており、コードベース全体から依存されるグローバルな暗黙的結合点となっている。
- **独立性 (4/5)**: Utils・CommonUtilはDB依存のない純粋なユーティリティクラスだが、System.out.printlnの残存やsanitizeForLogの未実装などの品質問題がある。単体テスト・単独ビルドは技術的に可能。
- **再利用性 (2/5)**: 税計算・ポイント計算・ID生成・MD5ハッシュなど汎用的な関数が含まれるが、同一ロジックが他4箇所に重複して実装されており、staticメソッドで業務ロジックが混在しているため信頼できる単一の再利用可能ライブラリとしては機能していない。
- **MMI**: (0.3×1 + 0.3×2 + 0.2×4 + 0.2×2) / 5 × 100 = **42%**（Needs Improvement）
::::

モジュール別の採点結果を見ると、システム平均のMMIが46%で要改善レベルに留まっていることが分かります。
ここで注目すべきは、商品カタログやユーザー認証、監査ログといった周辺モジュールは比較的高いスコアを記録している一方で、注文管理や支払、返品といったシステムの主要なビジネスロジックを担うモジュールが、ことごとく32%以下の最低ランクである「未成熟」に分類されている点です。
これは、本来であれば最も丁寧に設計されるべきコアドメインに技術的負債が集中している状況を示しています。
特に、複数データベースに直接依存するCheckoutSagaとReturnSagaが26%という極めて低いスコアになっており、これがシステム全体の結合性を高め、独立性を大きく引き下げる要因となっていると言えます。


### DDD 戦略層の評価（境界コンテキスト・ユビキタス言語など）


![ddd-strategic-evaluation](/images/legacy-refactoring-nexus-scalardb/ddd-strategic-evaluation.png)
*DDD 戦略層の評価（境界コンテキスト・ユビキタス言語など）*

::::details レポート全文
---
title: "DDD評価: 戦略的設計"
schema_version: 1
phase: "Phase 2: Evaluation"
skill: evaluate-ddd
generated_at: 2026-05-14T00:00:00Z
input_files:
  - reports/01_analysis/ubiquitous-language.md
  - reports/01_analysis/domain-code-mapping.md
  - reports/01_analysis/system-overview.md
---

# DDD戦略的設計評価

## レイヤースコア: 1.3/5（加重貢献度: 7.8%）

## 基準別スコア

| # | 基準 | スコア | 根拠 |
|---|---|---|---|
| 1 | ユビキタス言語 | 2/5 | 基本用語は英語で反映されているが、補償系語彙・ステータス値が String リテラルで散在し一元管理が失われている |
| 2 | 境界コンテキスト | 1/5 | 10 の BC 候補は識別可能だが、コード上に明示的境界が存在せず Saga が 6 BC を直接横断する |
| 3 | サブドメイン分類 | 1/5 | Core / Supporting / Generic の分類が皆無。コアドメイン（チェックアウト）が最低品質の God Class に実装されている |

---

## 詳細所見

### 基準 1 — ユビキタス言語（2/5）

**根拠**: ドメイン用語の英語名称はコード上に概ね反映されているが、ポイントトランザクションの type 値（EARN/EARN_REVERSED/REVERSED/EARN_REVERSED_CANCEL）や在庫移動の reason 値（COMPENSATE/RETURN_COMPENSATE）など補償系の語彙が不統一で、ビジネス語彙とコードの一致が崩れている。全ステータス値が enum 化されずに String リテラルとして散在しており、言語の一元管理が失われている。

**所見**:
- ポイントトランザクション type 値に `REVERSED` / `EARN_REVERSED` / `EARN_REVERSED_CANCEL` が混在し、取消の概念に統一された用語がない
- 在庫移動 reason 値の `COMPENSATE` / `RETURN_COMPENSATE` は動詞・名詞が揺れており、ドメイン語彙として未整備
- `OrderStatus` / `PaymentMethod` / `TaxCategory` 等すべてのステータスが String リテラルのみで定義され、辞書的な一元管理が存在しない
- パッケージ命名が `dao/` と `repository/` に分裂し、同一概念（データアクセス層）に異なる名称が使われている
- `ユーザー（User）` と `会員（Member）` の概念分離はドメイン的に適切だが、会員マスタテーブルが存在しないため概念が宙に浮いている

---

### 基準 2 — 境界コンテキスト（1/5）

**根拠**: 10 の BC 候補領域は分析により識別できるが、コード上に明示的な境界は一切存在せず、すべての BC が単一パッケージ群に混在している。CheckoutSaga と ReturnSaga が Catalog・Inventory・Order・Payment・Point・Receipt の 6 BC を直接操作しており、コンテキスト間の依存が明示されないまま深く結合している。コンテキストマップ・ACL・公開インタフェースは定義されていない。

**所見**:
- `PaymentService` と `ReturnService` が存在せず、支払と返品の責務が CheckoutSaga/ReturnSaga に吸収され BC 境界が消失している
- 在庫切れリスク判定ロジックが `InventoryService` ではなく `OrderService` に実装されており、BC 間の責務漏洩が発生している
- ダッシュボード集計（ベストセラー・時間帯別売上・月次統計）が `OrderService` に混在し、Analytics/Dashboard BC が未分離
- 物理 DB 境界（PG: 5 BC 混在 / MySQL: 4 BC 混在）が Bounded Context 境界と無関係に設計されており、DB がコンテキスト境界の代替にもなっていない
- コンテキストマップ・統合パターン（ACL, Open Host 等）がコードにも設計ドキュメントにも存在しない

---

### 基準 3 — サブドメイン分類（1/5）

**根拠**: Core / Supporting / Generic のサブドメイン分類が設計上も実装上も行われておらず、POS システムの競争優位の源泉である「チェックアウト・ポイント運用」と、汎用的な「ユーザー認証・監査ログ」が同等の粗末な実装水準に置かれている。コアドメインへの投資集中という DDD の原則が適用されていない。

**所見**:
- チェックアウト（Core Domain 候補）が 500 行超の God Class である `CheckoutSaga` に実装され、最も重要な業務ロジックが最も低品質な構造になっている
- ポイント計算（Core Domain 候補）が `Utils` / `PointService` / `CheckoutSaga` の 4 箇所に重複し、コアドメインルールが散在している
- 監査ログ・ユーザー認証（Generic Subdomain 候補）が専用 Service を持つ一方、`PaymentService`・`ReturnService`（Supporting Subdomain 候補）が欠如しており投資配分が逆転している
- `OrderService` が注文管理（Supporting）・在庫リスク判定（Supporting）・ダッシュボード集計（Generic）を 800 行以上で抱える God Service となっており、サブドメイン別の責務割当が機能していない
- サブドメイン分類ドキュメントが存在せず、どの機能が競争優位に直結するかの設計判断が記録されていない

---

## 改善提言（戦略的設計）

### 提言 1 — ユビキタス言語の一元管理（ステータス enum 化）
全ステータス値（`OrderStatus`, `PaymentMethod`, `TaxCategory`, `PointTransactionType`, `StockMovementReason`）を enum として定義し、String リテラルの散在を排除する。これだけでコンパイル時検出が効くようになり言語の一貫性が大幅に向上する。

### 提言 2 — コンテキストマップの作成とパッケージ境界の明示
`com.example.legacypos.catalog`, `com.example.legacypos.order`, `com.example.legacypos.inventory` 等、BC ごとのパッケージを設け、BC 間の依存を明示的なインタフェース（アンチコラプションレイヤ等）経由に制限する。

### 提言 3 — サブドメイン分類の明文化とコアドメインへの投資集中
チェックアウト・ポイント運用を Core Domain と位置づけ、設計品質（テスト・ドメインモデルの明確さ）を最優先で向上させる。監査ログ・認証は Generic Subdomain として外部化・標準実装の利用も検討する。
::::

DDD戦略層の評価レポートは、本システムのドメイン設計が本来あるべき状態から大きく乖離していることを示しています。

最も大きな課題は、ドメインの優先順位付けと投資のバランスが崩れている点です。ビジネスの差別化要因となるチェックアウトやポイント計算といったコアドメインが、巨大なサービスクラスやSagaの中に手続き型コードとして実装され、品質が著しく低下しています。その一方で、汎用的なユーザー認証や監査ログなどの支援的なドメインに優先してサービス層が構築されているという非対称性があります。

また、10個の境界コンテキストの候補は識別できるものの、それらがパッケージやモジュールなどの物理的な境界としてコード上で表現されていないため、コンテキスト間の依存関係が不明瞭なまま密結合を誘発していると言えます。



### DDD 戦術層の評価（集約・リポジトリ・ドメインイベントなど）

![ddd-tactical-architecture-evaluation](/images/legacy-refactoring-nexus-scalardb/ddd-tactical-architecture-evaluation.png)
*DDD 戦術層の評価（集約・リポジトリ・ドメインイベントなど）*

::::details レポート全文
---
title: "DDD評価: 戦術的設計とアーキテクチャ"
schema_version: 1
phase: "Phase 2: Evaluation"
skill: evaluate-ddd
generated_at: 2026-05-14T00:00:00Z
input_files:
  - reports/01_analysis/domain-code-mapping.md
  - reports/before/legacy-pos-monolith/issues-and-debt.md
---

# DDD戦術的設計・アーキテクチャ評価

## 総合 DDD スコア: **24.6%（Minimal DDD Adoption）**

| レイヤー | 重み | 平均スコア | 加重貢献度 |
|---|---|---|---|
| 戦略的設計 | 30% | 1.3/5 | 0.39 |
| 戦術的設計 | 45% | 1.3/5 | 0.59 |
| アーキテクチャ | 25% | 1.0/5 | 0.25 |
| **DDD スコア** | 100% | | **(0.39+0.59+0.25)/5×100 = 24.6%** |

> 成熟度: **Minimal DDD Adoption**（0-40%）— 根本的な再設計が必要

---

## 戦術的設計レイヤー: 1.3/5（重み: 45%）

| # | 基準 | スコア | 根拠 |
|---|---|---|---|
| 4 | 値オブジェクト | 1/5 | Money/TaxCategory/OrderStatus/PaymentMethod が全て primitive/String。不変性・ドメインルール封じ込めが皆無 |
| 5 | エンティティ | 2/5 | エンティティクラスは存在するが POJO のみ。ステータス遷移ガードが存在しない |
| 6 | 集約 | 1/5 | 集約ルートが未定義。OrderService が複数 DAO を直接操作し集約境界が崩壊 |
| 7 | リポジトリ | 2/5 | DAO が存在し一定の抽象化があるが、コレクションセマンティクスなし・SQL インジェクション・パッケージ分裂 |
| 8 | ドメインサービス | 1/5 | God Service（OrderService 976行）、PaymentService/ReturnService 不在、ドメインとアプリ層の区別なし |
| 9 | ドメインイベント | 1/5 | ApplicationEventPublisher 不使用、イベント駆動設計なし、クロスコンテキスト通信の仕組みが皆無 |

---

### 基準 4 — 値オブジェクト（1/5）

**根拠**: Money・TaxCategory・OrderStatus・PaymentMethod のいずれも値オブジェクトとして実装されておらず、プリミティブ型（int/String）のままコード全体に散在している。不変性・等値セマンティクス・ドメインルール封じ込めが一切ない。

**所見**:
- `Money` 値オブジェクトが存在せず、金額は `int` 型のプリミティブで表現されている
- `TaxCategory` は 1/2 の整数リテラル、`OrderStatus`・`PaymentMethod` は文字列リテラルが 4〜5 箇所以上に散在（TD-015, TD-016）
- 税計算ロジックが 4 箇所（Utils, CartService, CheckoutSaga, OrderService）に重複実装されており、値オブジェクトへの集約が皆無

---

### 基準 5 — エンティティ（2/5）

**根拠**: Order・Product 等の主要エンティティは POJO として存在し ID フィールドも持つが、ライフサイクル管理・ステータス遷移ガードが一切なく、不正遷移（COMPLETED→PENDING への巻き戻し等）も防げない。

**所見**:
- Order・OrderItem・Payment・Product 等のエンティティクラスは存在するが、すべてミュータブルな POJO でドメインロジックを持たない
- `OrderStatus` の遷移正当性チェックが存在せず、任意のステータスに直接書き換え可能
- ロール（`User.role`）が String フィールドで表現されており、不正なロール値を防ぐ手段がない

---

### 基準 6 — 集約（1/5）

**根拠**: 集約ルートの概念が存在せず、OrderService が OrderDao・InventoryDao・PointDao・ReceiptDao を直接操作し、トランザクション境界は集約単位ではなく DB 単位で管理されている。

**所見**:
- `Order` が `OrderItem` を集約管理しておらず、集約ルートとしての境界が定義されていない
- CheckoutSaga が 6 つ以上の DAO を直接操作しており、集約の一貫性不変条件を個々の集約が保証できない（TD-006）
- `@Transactional` が Service と DAO の両方に付与されており、トランザクション境界が不明確（TD-014）
- 在庫引当の `SELECT FOR UPDATE` 未使用により、集約レベルの並行制御が欠如している

---

### 基準 7 — リポジトリ（2/5）

**根拠**: DAO クラスが存在し永続化を一定程度抽象化しているが、コレクションセマンティクスではなくクエリビルダー的な操作（ORDER BY 文字列結合等）が露出しており、パッケージ命名も dao/repository に分裂している。

**所見**:
- `OrderDao` 等の DAO は JdbcTemplate による永続化を抽象化しているが、コレクションとしてのセマンティクスを持たない
- `ORDER BY` 句を文字列結合で組み立てる SQL インジェクション脆弱性が存在する（TD-004）
- `PointDao` と `ReceiptDao` が `repository` パッケージに、その他が `dao` パッケージに分散しており命名規則に一貫性がない（TD-013）
- `OrderService` が `getHourlySales()` で JdbcTemplate を直接使用するなど、リポジトリをバイパスする箇所がある

---

### 基準 8 — ドメインサービス（1/5）

**根拠**: ドメインサービスとアプリケーションサービスの区別が存在せず、OrderService は 976 行の God Service としてクエリ・コマンド・統計・ドメインルールが混在しており、PaymentService・ReturnService といった必要なドメインサービスが欠落している。

**所見**:
- `OrderService` が 976 行の God Service で、注文 CRUD・統計・在庫リスク判定・返品可否チェックを一手に担う（TD-005）
- `PaymentService` が存在せず、支払処理ロジックが CheckoutSaga の Step 3 に直書きされている
- `ReturnService` が存在せず、返品ロジックが ReturnSaga 内に散在している
- ポイント計算が Utils・CheckoutSaga・PointService の 3 箇所に重複実装されており、ドメインサービスとしての集約ができていない

---

### 基準 9 — ドメインイベント（1/5）

**根拠**: ApplicationEventPublisher・イベントバス・メッセージングの仕組みが一切存在せず、チェックアウト完了・返品完了・在庫変更といった重要な状態変化がイベントとして発行されていない。

**所見**:
- `ApplicationEventPublisher` の使用箇所がコードベース全体に存在しない
- チェックアウト完了時にポイント加算・レシート発行・在庫引当を同期的に CheckoutSaga 内で直接呼び出しており、ドメインイベント駆動型の疎結合設計になっていない
- 監査ログの呼び出し漏れが複数箇所に存在し（TD-029）、イベント駆動であれば横断的に解決できる問題が手動呼び出しに依存している
- クロスコンテキスト通信の仕組みが皆無であり、将来のマイクロサービス分離に対して大きな障壁となる

---

## アーキテクチャレイヤー: 1.0/5（重み: 25%）

| # | 基準 | スコア | 根拠 |
|---|---|---|---|
| 10 | レイヤリング | 1/5 | Application Service 層が完全欠落。Controller→Service→DAO の粗い 3 層すら守られていない |
| 11 | 依存方向 | 1/5 | DIP 未適用。全 Service/Saga が具象 DAO クラスに直接依存 |
| 12 | ポート＆アダプタ | 1/5 | DB・セッション・テンプレートを抽象化するポートが皆無。ヘキサゴナルアーキテクチャ的要素が存在しない |

---

### 基準 10 — レイヤリング（1/5）

**根拠**: Domain / Application / Infrastructure / Presentation の 4 層分離が存在せず、Application Service（Use Case）層が完全に欠落している。コントローラ・サービス・Saga・DAO が責務を互いに侵食し合っており、レイヤ境界が事実上崩壊している。

**所見**:
- Application Service / Use Case 層が存在しない — Controller が直接 Service と Saga を呼び出す
- `RegisterController.checkout()` にカート→注文変換のビジネスロジックが直書きされており、Presentation 層に業務ロジックが混在
- Thymeleaf テンプレート内に税計算ロジックが存在し、UI 層がドメインルールを保持している
- `OrderService` が `JdbcTemplate` を直接呼び出し（DAO 層をバイパス）しており、Service 層と Infrastructure 層の境界が崩れている
- `CheckoutSaga` / `ReturnSaga` が Payment・Point・Receipt など複数 BC を直接操作し、Saga が Application 層の役割を兼務している

---

### 基準 11 — 依存方向（1/5）

**根拠**: 依存の方向が内側（ドメイン）に向かっておらず、上位レイヤが下位の具象クラスに直接依存する構造が全域に渡っている。依存性逆転原則（DIP）は一切適用されていない。

**所見**:
- 全 Service・Saga がインターフェースではなく具象 DAO クラスに直接依存しており、DIP が適用されていない
- Controller が具象 Service クラスをフィールドに持ち、抽象に依存していない
- CheckoutSaga が 6〜7 個の具象 DAO に直接依存し、ドメイン層への依存逆転が皆無
- `domain` パッケージは純粋な POJO だが、集約ルートや値オブジェクトの概念がなくドメインモデルとして機能していない
- `DriverManagerDataSource`（コネクションプールなし）が config 層に直書きされており、インフラ実装がコアに漏出している

---

### 基準 12 — ポート＆アダプタ（1/5）

**根拠**: 外部システム（DB・セッション・テンプレートエンジン）との接続を抽象化するポートが定義されておらず、アダプタパターンは完全に不在である。ヘキサゴナル/クリーン/オニオンアーキテクチャのいずれのパターンも適用されていない。

**所見**:
- `JdbcTemplate` が DAO 全体に直接使用されており、DB アクセスを抽象化するリポジトリインターフェース（ポート）が存在しない
- `dao` パッケージと `repository` パッケージの命名が混在しており、アダプタ層の境界が曖昧
- `PaymentService`・`ReturnService` が存在せず、外部決済・返品処理を抽象化するポートが定義されていない
- セッションスコープの `CartService` が Service 層に直書きされ、セッションというインフラ関心事がドメイン層に混入している
- `CheckoutSaga` が MySQL・PostgreSQL の両 DB に直接依存しており、DB 間の接続を隠蔽するアダプタが存在しない

---

## 改善提言（戦術的設計・アーキテクチャ）

### 提言 1 — 値オブジェクトの導入（Money, TaxCategory, OrderStatus）
`Money(int yen)`, `TaxCategory(STANDARD/REDUCED)`, `OrderStatus(PENDING/COMPLETED/...)` を不変の値オブジェクト/enum として定義し、プリミティブ散在を解消する。税計算は `Money.withTax(TaxCategory)` に集約する。

### 提言 2 — 集約ルートの定義（OrderAggregate）
`Order` を集約ルートとして位置づけ、`OrderItem` の操作は必ず `Order` 経由とする。`Order.addItem()`, `Order.complete()` 等のドメインメソッドを定義し、ステータス遷移ガードを集約内に閉じ込める。

### 提言 3 — PaymentService / ReturnService の新設
CheckoutSaga Step 3 の支払処理を `PaymentService.charge(orderId, amount, method)` へ、ReturnSaga Step 3 の支払取消を `PaymentService.refund(paymentId)` へ切り出す。ReturnService も同様に ReturnSaga から分離する。

### 提言 4 — Application Service 層（Use Case）の導入
`CheckoutUseCase`, `ReturnUseCase` 等のアプリケーションサービスを新設し、Controller から直接 Saga を呼ぶ構造を解消する。Use Case 層がトランザクション境界を保持し、ドメインサービスを呼び出す役割を担う。

### 提言 5 — リポジトリインターフェースの定義（DIP 適用）
各 DAO にインターフェース（`OrderRepository`, `InventoryRepository` 等）を定義し、Service / Saga がインターフェース経由で DAO に依存するよう変更する。これにより DIP が適用され、テスタビリティが大幅に向上する。

---

## DDD スコア詳細計算

| レイヤー | 基準 | スコア |
|---|---|---|
| 戦略的設計 | ユビキタス言語 | 2 |
| 戦略的設計 | 境界コンテキスト | 1 |
| 戦略的設計 | サブドメイン分類 | 1 |
| 戦術的設計 | 値オブジェクト | 1 |
| 戦術的設計 | エンティティ | 2 |
| 戦術的設計 | 集約 | 1 |
| 戦術的設計 | リポジトリ | 2 |
| 戦術的設計 | ドメインサービス | 1 |
| 戦術的設計 | ドメインイベント | 1 |
| アーキテクチャ | レイヤリング | 1 |
| アーキテクチャ | 依存方向 | 1 |
| アーキテクチャ | ポート＆アダプタ | 1 |

戦略的設計 layer_avg = (2+1+1)/3 = **1.3** → 加重貢献 = 1.3 × 0.30 = **0.39**
戦術的設計 layer_avg = (1+2+1+2+1+1)/6 = **1.3** → 加重貢献 = 1.3 × 0.45 = **0.59 (≈0.585)**
アーキテクチャ layer_avg = (1+1+1)/3 = **1.0** → 加重貢献 = 1.0 × 0.25 = **0.25**

**DDD Score = (0.39 + 0.585 + 0.25) / 5 × 100 ≈ 24.5%**
::::

戦術的設計およびアーキテクチャの評価レポートからは、ドメインモデルを保護するためのオブジェクト指向設計やカプセル化のパターンが機能していないことが読み取れます。

特に、集約の概念が定義されていないため、OrderとOrderItemの関係が暗慢的になっており、Orderが自分の配下にある明細の一貫性を維持するようなガードロジックが存在しません。
また、値オブジェクトの不在により、金額などのドメイン概念が型安全に保護されていません。
さらに、アプリケーションサービス層が完全に欠落しているため、コントローラが直接Sagaを呼び出してカート情報を注文データに変換するようなレイヤ境界の侵害が発生しています。

依存方向も具象DAOへの直接参照となっており、依存性逆転の原則が適用されていないため、システム全体のテスタビリティが低下する要因になっていると言えます。



### MMI と DDD を統合した総合スコア（33.6%）とクロス評価マトリクス

![integrated-evaluation](/images/legacy-refactoring-nexus-scalardb/integrated-evaluation.png)
*MMI と DDD を統合した総合スコア（33.6%）とクロス評価マトリクス*

::::details レポート全文
---
title: 統合評価レポート
schema_version: 1
phase: "Phase 2: Evaluation"
skill: integrate-evaluations
generated_at: 2026-05-14T00:00:00Z
input_files:
  - reports/02_evaluation/mmi-overview.md
  - reports/02_evaluation/mmi-by-module.md
  - reports/02_evaluation/ddd-strategic-evaluation.md
  - reports/02_evaluation/ddd-tactical-architecture-evaluation.md
---

# 統合評価レポート — legacy-pos-monolith

## エグゼクティブサマリ

| 評価指標 | スコア | 成熟度レベル |
|---|---|---|
| **システム平均 MMI** | **46%** | Needs Improvement |
| **DDD スコア** | **24.5%** | Minimal DDD Adoption |
| **総合判定** | | **❌ 根本的な再設計が必要** |

両評価が「Immature / Minimal」という極めて低い水準で一致しており、このシステムはマイクロサービス移行の前に**大規模なアーキテクチャ改善**が不可欠である。

---

## MMI × DDD 統合マトリクス

各モジュールの MMI スコアと、対応する DDD 課題の深刻度をマッピングする。

| モジュール | MMI | MMI成熟度 | 主なDDD課題 | 総合優先度 |
|---|---|---|---|---|
| CheckoutSaga | 26% | Immature | BC横断・集約未定義・ドメインイベントなし | 🔴 **最高** |
| ReturnSaga | 26% | Immature | BC横断・集約未定義・ReturnService欠如 | 🔴 **最高** |
| Order | 32% | Immature | God Service・ドメインサービス未分離・集約未定義 | 🔴 **最高** |
| Payment | 32% | Immature | PaymentService欠如・値オブジェクトなし | 🔴 **最高** |
| Return | 32% | Immature | ReturnService欠如・ロジック3重散在 | 🔴 **最高** |
| web | 32% | Immature | Application層欠如・DIP未適用・責務混在 | 🔴 **高** |
| util | 42% | Needs Impr. | ユビキタス言語汚染・ビジネスロジック混入 | 🟡 **中** |
| Point | 46% | Needs Impr. | ポイント計算4重重複・パッケージ分裂 | 🟡 **中** |
| Receipt | 52% | Needs Impr. | 発行ロジックSaga散在・ReceiptService不完全 | 🟡 **中** |
| Inventory | 52% | Needs Impr. | 在庫引当がSagaに散在・在庫リスク誤配置 | 🟡 **中** |
| Cart | 58% | Needs Impr. | アプリ層欠如・Cart→注文変換がController | 🟢 **低** |
| Catalog | 68% | Moderate | リポジトリI/F欠如・DIP未適用 | 🟢 **低** |
| Identity | 72% | Moderate | ロールがString・メソッドレベル認可なし | 🟢 **低** |
| Audit | 72% | Moderate | 呼び出し漏れ・例外握り潰し | 🟢 **低** |

---

## 横断的課題の特定（両評価で重複する問題）

### 課題 A: 手書き Saga の構造的欠陥（MMI + DDD 双方で最低評価）

**MMI観点**:
- CheckoutSaga/ReturnSaga: 凝集性 2/5、結合性 1/5、独立性 1/5、再利用性 1/5 → MMI 26%

**DDD観点**:
- 境界コンテキスト 1/5: Saga が 6 BC を直接横断
- 集約 1/5: 集約ルート未定義、Saga が DAO を直接操作
- ドメインイベント 1/5: イベント駆動なし、同期直接呼び出し

**統合見解**: Saga の解体と ScalarDB による分散トランザクション化は、MMI・DDD 両方の最大の改善ドライバーである。

---

### 課題 B: God Service / Service 層の崩壊（MMI + DDD 双方で重要）

**MMI観点**:
- OrderService: MMI 32%（976 行）
- PaymentService 不在: Payment MMI 32%
- ReturnService 不在: Return MMI 32%

**DDD観点**:
- ドメインサービス 1/5: God Service、必要なサービスの欠落
- レイヤリング 1/5: Application Service 層の完全欠落

**統合見解**: OrderService の分割と PaymentService/ReturnService の新設は、モジュール MMI を +20〜30% 向上させると同時に DDD スコアの最大の引き上げポイントとなる。

---

### 課題 C: ドメインモデルの貧困（Anemic Domain Model）

**MMI観点**:
- 再利用性の全体平均 1.79/5（最低軸）
- 値オブジェクトがないため、ビジネスロジックが Service/Saga/Util に散在

**DDD観点**:
- 値オブジェクト 1/5: Money/TaxCategory/OrderStatus が primitive/String
- 集約 1/5: 集約ルート未定義
- エンティティ 2/5: POJO のみ、ドメインロジックなし

**統合見解**: 値オブジェクト導入と集約ルート定義は、スコープが限定的で着手しやすいが、DDD スコア向上への貢献が大きい。

---

### 課題 D: アーキテクチャ層の欠落

**MMI観点**:
- 独立性の全体平均 1.86/5（全モジュールで低い主因）
- Controller が Service/Saga を直接呼び出す = テスト不可能な構造

**DDD観点**:
- レイヤリング 1/5・依存方向 1/5・ポート＆アダプタ 1/5（アーキテクチャ層 3 基準すべて最低）

**統合見解**: Application Service（Use Case）層の導入とリポジトリインターフェース定義は、MMI の独立性・再利用性軸と DDD アーキテクチャ層を同時に改善する。

---

### 課題 E: ユビキタス言語の不統一

**MMI観点**:
- util 凝集性 1/5: ビジネスロジックが static メソッドで散在
- dao/repository パッケージ分裂: Inventory/Receipt 凝集性に影響

**DDD観点**:
- ユビキタス言語 2/5: ステータス値の String リテラル散在、補償系語彙の揺れ

**統合見解**: Enum 化とパッケージ統一はコストが低く、即効性があるクイックウィン。

---

## 強み（改善のベースライン）

両評価でも相対的に高い評価を得た領域は、マイクロサービス分離の**最初の候補**となる。

| モジュール | MMI | 強み |
|---|---|---|
| Identity | 72% | 認証・認可が比較的独立。Spring Security 標準実装 |
| Audit | 72% | 単一責務が明確。`record(action, target, payload)` の薄いインタフェース |
| Catalog | 68% | 商品マスタが他 BC から書き込まれない。Read-heavy な安定 BC |

これら 3 モジュールはリファクタリング後に**最初にマイクロサービス化できる候補**である。

---

## 数値サマリ

### MMI スコア分布

```
Mature (80-100%)       : 0 モジュール
Moderate (60-80%)      : 3 モジュール (Catalog, Identity, Audit)
Needs Improvement (40-60%) : 5 モジュール
Immature (0-40%)       : 6 モジュール
```

### DDD スコア詳細

```
戦略的設計  (30%): 1.3/5 → 加重 7.8%
戦術的設計  (45%): 1.3/5 → 加重 11.7%
アーキテクチャ(25%): 1.0/5 → 加重 5.0%
総合 DDD          → 24.5%
```

### 最低スコア基準（根本的な再設計が必要な箇所）

| DDD基準 | スコア |
|---|---|
| 境界コンテキスト | 1/5 |
| サブドメイン分類 | 1/5 |
| 値オブジェクト | 1/5 |
| 集約 | 1/5 |
| ドメインサービス | 1/5 |
| ドメインイベント | 1/5 |
| レイヤリング | 1/5 |
| 依存方向 | 1/5 |
| ポート＆アダプタ | 1/5 |
::::

統合評価レポートおよびMMI・DDDのクロス評価マトリクスを見ると、本システムが抱える課題がより有機的に結びついて見えてきます。

システム全体の総合スコアが33.6%という極めて低い評価に留まり、根本的な再設計が必要と判定された最大の要因は、CheckoutSagaやReturnSagaといった複数DBを跨ぐ手書きSagaの存在です。
これらがMMIの結合性・独立性とDDDの境界コンテキスト・集約のすべての面でボトルネックとなっており、システム全体のアーキテクチャ移行を妨げています。

その一方で、IdentityやAudit、Catalogといった周辺ドメインは、この壊滅的な状況下にあっても比較的安定した設計品質を維持しています。これらは、密結合なコアドメインの解体を進めた後に、最初のマイクロサービス分離候補として十分に機能するポテンシャルを持っていると言えます。


### 評価結果から導いた改善アクション一覧

![unified-improvement-plan](/images/legacy-refactoring-nexus-scalardb/unified-improvement-plan.png)
*改善アクション一覧*

::::details レポート全文
---
title: 統合改善計画
schema_version: 1
phase: "Phase 2: Evaluation"
skill: integrate-evaluations
generated_at: 2026-05-14T00:00:00Z
input_files:
  - reports/02_evaluation/integrated-evaluation.md
  - reports/02_evaluation/mmi-overview.md
  - reports/02_evaluation/ddd-tactical-architecture-evaluation.md
---

# 統合改善計画 — legacy-pos-monolith

## 計画概要

本計画は MMI・DDD 両評価の結果を統合し、**スコアへの影響度・ビジネスリスク低減・実装難易度**の 3 軸で改善項目を優先順位付けした実行ロードマップである。

全体方針:
1. **ScalarDB 導入による分散トランザクション基盤の確立**（最大の技術的負債解消）
2. **God Class / God Service の段階的解体**（モジュール独立性の確保）
3. **DDD 戦術パターンの段階的導入**（将来のマイクロサービス化の基盤）

---

## フェーズ 1 — 基盤整備（短期・1〜3ヶ月）

### P1-1: 接続プール導入（CRITICAL 優先）
**対象**: `DataSourceConfig.java`
**問題**: `DriverManagerDataSource` が本番運用不可（TD-001）
**アクション**:
- `DriverManagerDataSource` → `HikariDataSource` に切り替え
- 接続プール設定（最大接続数・タイムアウト）の適切な定義

**期待効果**: CRITICAL 技術的負債の解消。本番運用可能状態への移行。
**MMI影響**: 独立性軸 +0.5（全モジュール）
**難易度**: 低

---

### P1-2: ユビキタス言語の enum 化（クイックウィン）
**対象**: コードベース全体
**問題**: ステータス値が String リテラルで散在（TD-015, TD-016、DDD 基準1=2/5）
**アクション**:
```java
// 新規作成
enum OrderStatus    { PENDING, COMPLETED, CANCELLED, RETURNED, FAILED }
enum PaymentMethod  { CASH, CARD }
enum TaxCategory    { STANDARD(10), REDUCED(8) }
enum PaymentStatus  { COMPLETED, REVERSED, REFUNDED }
enum ReceiptKind    { SALE, RETURN }
enum StockMovementReason { RECEIVE, SELL, RETURN, COMPENSATE, RETURN_COMPENSATE }
enum PointTransactionType { EARN, EARN_REVERSED, REVERSED, EARN_REVERSED_CANCEL }
```
- 既存の String 比較をすべて enum 参照に置き換え

**期待効果**: コンパイル時ステータス値の検証が可能になる。DDD「ユビキタス言語」スコア 2→3 相当。
**MMI影響**: Catalog/Order/Point など凝集性 +0.5
**難易度**: 低（機械的置き換え）

---

### P1-3: パッケージ命名統一（クイックウィン）
**対象**: `repository/PointDao`, `repository/ReceiptDao`
**問題**: `dao/` と `repository/` の並存（TD-013）
**アクション**:
- `repository/PointDao` → `dao/PointDao`
- `repository/ReceiptDao` → `dao/ReceiptDao`
- 参照箇所の一括修正

**期待効果**: DDD「リポジトリ」スコア向上の前提。IA の統一。
**MMI影響**: Point・Receipt 凝集性 +0.5
**難易度**: 低

---

### P1-4: SQL インジェクション修正（セキュリティ）
**対象**: `OrderDao.findAll()` — ORDER BY 文字列結合
**問題**: `sortBy` パラメータが SQL に直接結合（TD-004）
**アクション**:
```java
// ホワイトリストマップで ORDER BY を制御
private static final Map<String, String> ALLOWED_SORT = Map.of(
    "ordered_at_desc", "ordered_at DESC",
    "ordered_at_asc",  "ordered_at ASC",
    "total_yen_desc",  "total_yen DESC"
);
```

**期待効果**: CRITICAL セキュリティ脆弱性の解消。
**難易度**: 低

---

## フェーズ 2 — God Class/Service 解体（中期・3〜6ヶ月）

### P2-1: ScalarDB 導入（最大インパクト）
**対象**: CheckoutSaga, ReturnSaga, DataSourceConfig
**問題**: MySQL+PostgreSQL 跨ぎ Saga の原子性未保証（TD-002, TD-003, TD-009）
**アクション**:
1. ScalarDB を依存関係に追加
2. MySQL・PostgreSQL を ScalarDB のバックエンドとして設定
3. `CheckoutSaga.execute()` を ScalarDB トランザクション内の処理に書き換え
4. 手書き補償処理（catch ブロック群）を削除
5. ReturnSaga も同様に書き換え

**期待効果**:
- TD-002（DB 跨ぎ原子性未保証）を根本解決
- TD-003（Saga ステート永続化なし）を不要化
- TD-009（補償例外握り潰し）を不要化
- CheckoutSaga/ReturnSaga の MMI: 26% → 推定 50%+

**MMI影響**: CheckoutSaga/ReturnSaga の全軸で +2〜3 相当
**DDD影響**: 集約スコア・ドメインサービススコアの前提整備
**難易度**: 高（ScalarDB の学習コストあり）

---

### P2-2: OrderService の分割（God Service 解体）
**対象**: `OrderService.java`（976 行）
**問題**: 注文 CRUD・統計・ランキング・在庫リスク・返品可否が混在（TD-005）
**アクション**: 責務ごとに分割

| 新クラス | 責務 | 移動するメソッド |
|---|---|---|
| `OrderQueryService` | 注文 CRUD・検索 | findAll, findById, findByDateRange, findTodayOrders, findByIds |
| `OrderCommandService` | ステータス遷移・キャンセル | updateStatus, cancelOrder, markFailed |
| `DashboardService` | 統計・ランキング・売上分析 | getTodaySalesTotal, getTodayOrderCount, getBestSellerRanking, getHourlySales, getStockoutRisk, getThisMonthSalesTotal |
| `ReturnEligibilityService` | 返品可否チェック | canReturn（OrderService + ReturnSaga の重複を統合） |

**期待効果**:
- OrderService MMI: 32% → 推定 60%+
- DashboardController から DashboardService 経由に整理
- ドメインサービス境界の明確化（DDD スコア向上）

**MMI影響**: Order 凝集性 2→4、結合性 2→3
**DDD影響**: ドメインサービス 1→3 相当
**難易度**: 中（依存関係の整理が必要）

---

### P2-3: PaymentService / ReturnService の新設
**対象**: CheckoutSaga Step 3、ReturnSaga Step 3
**問題**: PaymentService/ReturnService が存在しない（DDD ドメインサービス 1/5）
**アクション**:
```java
// PaymentService（新規）
@Service
public class PaymentService {
    public Payment charge(String orderId, int amountYen, PaymentMethod method) { ... }
    public void refund(String paymentId) { ... }
}

// ReturnService（新規）
@Service
public class ReturnService {
    public boolean isReturnable(String orderId) { ... }
    public Return createReturn(String orderId, List<ReturnItem> items, String requestedBy) { ... }
}
```

- CheckoutSaga Step 3 → `PaymentService.charge()` 呼び出しに置き換え
- ReturnSaga Step 3 → `PaymentService.refund()` 呼び出しに置き換え
- `OrderService.canReturn()` と `ReturnSaga.isOrderReturnable()` の重複を `ReturnService.isReturnable()` に統合

**期待効果**:
- Payment MMI: 32% → 推定 55%+
- Return MMI: 32% → 推定 55%+
- BC 境界の明確化

**難易度**: 中

---

### P2-4: CheckoutSaga / ReturnSaga のステップ分割
**対象**: `CheckoutSaga.execute()`（557 行）、`ReturnSaga.execute()`（448 行）
**問題**: 6 ステップ + 補償処理が単一メソッドに集約（TD-006）
※ P2-1（ScalarDB 導入）後に補償処理が不要になるため、その後に実施

**アクション**:
```java
// CheckoutSaga をステップクラスに分解
class CreateOrderStep { ... }
class ReserveInventoryStep { ... }
class ProcessPaymentStep { ... }  // → PaymentService.charge() 呼び出し
class EarnPointsStep { ... }      // → PointService 呼び出し
class IssueReceiptStep { ... }    // → ReceiptService 呼び出し
class ConfirmOrderStep { ... }

// CheckoutUseCase（Application Service）
@Service
public class CheckoutUseCase {
    @Transactional  // ScalarDB トランザクション
    public CheckoutResult execute(CheckoutRequest request) {
        // 各ステップを順番に実行
    }
}
```

**期待効果**:
- CheckoutSaga/ReturnSaga MMI: 推定 60%+（P2-1 後）
- Application Service 層の導入（DDD レイヤリング向上）

**難易度**: 高（P2-1 完了後）

---

## フェーズ 3 — DDD 戦術パターン導入（中期〜長期・6〜12ヶ月）

### P3-1: 値オブジェクトの導入
**対象**: domain パッケージ
**問題**: Money/TaxCategory/OrderStatus が primitive/String（DDD 値オブジェクト 1/5）
**アクション**:
```java
// Money 値オブジェクト（不変）
public final class Money {
    private final int yen;
    public Money add(Money other) { return new Money(this.yen + other.yen); }
    public Money withTax(TaxCategory category) {
        return new Money((int)(yen * (1 + category.getRate())));
    }
}
```
- `Order.totalYen` → `Money totalYen`
- 税計算ロジックを `TaxCategory.calculateTax(Money base)` に集約

**期待効果**: DDD 値オブジェクト 1→3 相当。税計算の 4 重重複を解消。
**難易度**: 中

---

### P3-2: 集約ルートの定義
**対象**: `domain.Order` + `domain.OrderItem`
**問題**: 集約ルートが未定義、OrderItem が Order から独立（DDD 集約 1/5）
**アクション**:
```java
// Order を集約ルートとして強化
public class Order {
    private List<OrderItem> items = new ArrayList<>();

    public void addItem(Product product, int quantity) {
        // 集約内のバリデーションここで実施
    }
    public void complete() {
        if (this.status != OrderStatus.PENDING) {
            throw new IllegalStateException("PENDING 状態のみ確定可能");
        }
        this.status = OrderStatus.COMPLETED;
    }
}
```

**期待効果**: DDD 集約 1→3 相当。不正なステータス遷移の防止。
**難易度**: 高（OrderService との役割再整理が必要）

---

### P3-3: リポジトリインターフェースの定義（DIP 適用）
**対象**: 全 DAO
**問題**: Service/Saga が具象 DAO に直接依存（DDD 依存方向 1/5）
**アクション**:
```java
// インターフェース（domain または application レイヤ）
public interface OrderRepository {
    Optional<Order> findById(String orderId);
    void save(Order order);
    List<Order> findByDateRange(LocalDate from, LocalDate to);
}

// 実装（infrastructure レイヤ）
@Repository
public class JdbcOrderRepository implements OrderRepository { ... }
```
- Service / UseCase がインターフェース経由で依存するよう変更

**期待効果**: DDD 依存方向 1→3 相当。単体テストが可能になる。
**難易度**: 高（全 DAO の変更が必要）

---

### P3-4: Application Service（Use Case）層の導入
**対象**: Controller → Service/Saga の直接呼び出しパス
**問題**: Application Service 層が完全欠落（DDD レイヤリング 1/5）
**アクション**:
- P2-4 で作成した `CheckoutUseCase`・`ReturnUseCase` をベースに拡張
- `RegisterController` → `CheckoutUseCase` 経由に変更
- `AdminReturnController` → `ReturnUseCase` 経由に変更

**期待効果**: DDD レイヤリング 1→3 相当。Controller のスリム化。
**難易度**: 中（P2-4 後に実施）

---

### P3-5: Web 層の分割（HTML vs REST API）
**対象**: `RegisterController`（303 行）
**問題**: 画面遷移と JSON API が混在（DDD アーキテクチャ観点）
**アクション**:
```
RegisterViewController  — GET /register, GET /register/receipt/{orderId}
RegisterApiController   — POST /api/register/checkout, POST /api/register/cart/**
```

**期待効果**: web MMI: 32% → 推定 50%+
**難易度**: 低〜中

---

## フェーズ 4 — マイクロサービス候補の分離（長期・12ヶ月+）

フェーズ 1〜3 完了後、以下の順序でマイクロサービス化を検討する。

| 順序 | サービス候補 | 前提条件 | 期待 MMI |
|---|---|---|---|
| 1 | **Identity Service** | フェーズ 1〜3 完了 | 72% → 85%+ |
| 2 | **Catalog Service** | P3-3（リポジトリI/F）完了 | 68% → 82%+ |
| 3 | **Audit Service** | フェーズ 1〜3 完了 | 72% → 80%+ |
| 4 | **Inventory Service** | P2-1（ScalarDB）+ P2-2 完了 | 52% → 72%+ |
| 5 | **Order Service** | P2-2（分割）+ P3-2（集約）完了 | 32% → 65%+ |
| 6 | **Payment Service** | P2-3 + P3-1（値オブジェクト）完了 | 32% → 68%+ |

---

## 改善効果の推定

| フェーズ | 推定 MMI 向上 | 推定 DDD 向上 | 期間目安 |
|---|---|---|---|
| フェーズ 1 完了後 | 46% → **50%** | 24.5% → **28%** | 1〜3ヶ月 |
| フェーズ 2 完了後 | 50% → **62%** | 28% → **40%** | 3〜6ヶ月 |
| フェーズ 3 完了後 | 62% → **72%** | 40% → **58%** | 6〜12ヶ月 |
| フェーズ 4 完了後 | 72% → **82%** | 58% → **70%** | 12ヶ月+ |

---

## 優先度マトリクス（全改善項目）

| 項目 | ビジネスリスク | 技術的影響 | 難易度 | 優先度 |
|---|---|---|---|---|
| P1-1: 接続プール | 🔴 CRITICAL | 高 | 低 | **最高** |
| P2-1: ScalarDB 導入 | 🔴 CRITICAL | 最高 | 高 | **最高** |
| P1-4: SQL Injection 修正 | 🔴 HIGH | 中 | 低 | **最高** |
| P1-2: Enum 化 | 中 | 中 | 低 | **高** |
| P2-2: OrderService 分割 | 中 | 高 | 中 | **高** |
| P2-3: PaymentService/ReturnService 新設 | 中 | 高 | 中 | **高** |
| P1-3: パッケージ統一 | 低 | 低 | 低 | **中** |
| P2-4: Saga ステップ分割 | 中 | 高 | 高 | **中** |
| P3-1: 値オブジェクト導入 | 低 | 中 | 中 | **中** |
| P3-2: 集約ルート定義 | 中 | 高 | 高 | **中** |
| P3-3: リポジトリ I/F 定義 | 低 | 高 | 高 | **中** |
| P3-4: Application Service 層 | 中 | 高 | 中 | **中** |
| P3-5: Web 層分割 | 低 | 中 | 低 | **低** |
::::

統合改善計画のレポートは、これまで指摘されてきた多くの技術的負債を解消し、システムを現代的な設計へと再構築するための現実的なロードマップを示しています。

この計画の重要なポイントは、システム全体を一度に作り直すのではなく、段階的なフェーズに分けて改善を進める点です。フェーズ1では接続プールの導入やステータス値のEnum化、パッケージ名の統一といった比較的低コストで効果の出やすい部分から着手し、開発の安全性とコードの可読性を高めます。

その後、手書きのSaga処理を分散トランザクションで制御する環境を整え、OrderServiceなどの巨大なクラスを責務ごとに分割し、値オブジェクトや集約といったDDDの戦術パターンを順次適用していきます。
このように内側からドメインの境界と設計を整理していくことで、将来的に各モジュールをマイクロサービスとして安全に切り出せる状態を作り出すことができると言えます。

## 本章のまとめ

* 評価レポートでは、MMI、DDD戦略設計、DDD戦術設計、統合評価、改善計画を順番に読み解きました。
* スコアは低く見えますが、重要なのは弱点が数値と根拠つきで可視化され、改善順序を決められるようになったことです。
* 統合改善計画では、接続プールやSQL Injection対策などの基盤整備から、ScalarDB導入、God Service解体、DDD戦術パターン適用へ進む道筋が示されました。

## 用語解説

### DDD戦略設計
境界コンテキスト、サブドメイン、ユビキタス言語など、システム全体の業務境界を整理する設計です。

### DDD戦術設計
エンティティ、値オブジェクト、集約、リポジトリ、ドメインイベントなど、コードに近い設計パターンを扱う設計です。

### Anemic Domain Model
ドメインモデルがデータを保持するだけで、業務ルールがServiceなどに散らばっている状態です。変更漏れや重複実装の原因になります。

### DIP
Dependency Inversion Principleの略で、上位の業務ロジックが具体的なDB実装などに直接依存しないようにする設計原則です。
