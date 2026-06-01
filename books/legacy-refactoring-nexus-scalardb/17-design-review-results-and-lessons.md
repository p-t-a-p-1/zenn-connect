---
title: "設計レビュー結果と学びを読み解く"
---

:::message
初回レビューのFAIL理由からv3での承認までを追い、設計レビューが何を見つけ、どう設計を強くしたかを整理します。
:::

## レビュー結果を読み解く

### 初回レビューはFAIL

`review-synthesis.md` では、初回設計に対して以下の判定が出ています。

![review-synthesis](/images/legacy-refactoring-nexus-scalardb/review-synthesis.png)
*初回レビュー結果*

::::details レポート全文
---
title: レビュー統合レポート
schema_version: 1
phase: "Phase 4: Review"
skill: review-synthesizer
generated_at: 2026-05-14T00:00:00Z
---

# レビュー統合レポート — legacy-pos-monolith

## 判定: ❌ FAIL

**重大度 CRITICAL が 4 件あり、CONDITIONAL_PASS の上限（2 件）を超過しています。**
P0 Blockers をすべて解消してから再審査を実施してください。

---

## スコアサマリ

| 観点 | 判定 | スコア | 重み |
|---|---|---|---|
| Consistency（一貫性） | PASS_WITH_RECOMMENDATIONS | 3.0 | 15% |
| ScalarDB（DB制約） | PASS_WITH_RECOMMENDATIONS | 3.0 | 25% |
| Operations（運用） | PASS_WITH_RECOMMENDATIONS | 3.0 | 20% |
| **Risk（リスク）** | **❌ FAIL** | **1.5** | **25%** |
| Business（ビジネス） | PASS_WITH_RECOMMENDATIONS | 3.0 | 15% |
| **総合スコア** | | **2.625 / 5.0** | |

**品質ゲート判定根拠**:
- aggregate 2.625 ≥ 2.5 ✅
- CRITICAL 4 件 ≤ 2（CONDITIONAL_PASS 上限）❌
- major 17 件 ≤ 8（CONDITIONAL_PASS 上限）❌

**再審査条件**: CRITICAL 4 件（SYN-001〜004）をすべて解消し CRITICAL 0 件になれば、P1 の解消状況次第で CONDITIONAL_PASS に引き上げ可能。

---

## 発見事項サマリ

| 優先度 | 件数 | 説明 |
|---|---|---|
| P0 - Blocker | **4** | CRITICAL: 解消必須 |
| P1 - Must Fix | **15** | High: 実装前に解消推奨 |
| P2 - Should Fix | **16** | Medium: 品質向上のため解消推奨 |
| P3 - Consider | **13** | Low/Info: 改善提案 |
| **合計（dedup 後）** | **48** | |

---

## P0 — Blockers（解消必須・設計進行不可）

### SYN-001: ScalarDB Cluster の HA 構成が未定義 ⚠️
**観点**: Risk
**影響**: Phase 2-4 において単一 ScalarDB Cluster がシステム全体の SPOF。Cluster 障害で全機能（チェックアウト・返品・在庫・ポイント・監査）が同時停止。HA・フェイルオーバ・リトライポリシーが未定義。

**対策**: ScalarDB Cluster を最低 3 ノード構成（indirect モード + ロードバランサ）にし、coordinator namespace を HA RDB クラスタ（Patroni/Galera）に乗せる。Cluster 障害検知・サーキットブレーカ・読み取り系 API の degraded モードを設計に追加。RTO/RPO を明記する。

---

### SYN-002: Outbox を Phase 2-3 でオプション化しているため at-least-once すら保証されない ⚠️
**観点**: Risk + Operations
**影響**: `@TransactionalEventListener` は AFTER_COMMIT で発火するため、メイン Tx 成功後の publisher 例外でイベントが完全にロストする。Audit 監査ログ・Dashboard 集計が黙って欠落 = TD-029（監査ログ呼び出し漏れ）の構造的解消という主張が成立しない。

**対策**: Outbox を Phase 2 から必須化。各 BC の ScalarDB namespace に `outbox_events` テーブルを置き、メイン Tx 内で同一 Tx として書く。別プロセスの Polling Publisher が `published=false` を取得して発火し、消費側はイベント ID で冪等化する。

---

### SYN-003: 外部決済 SaaS 連携で ScalarDB rollback だけの補償は破綻する ⚠️
**観点**: Risk
**影響**: target-architecture.md はSaga 不要、補償は ScalarDB rollback に委ねると明言しているが、Payment Context は外部決済 SaaS（Stripe 等）への ACL を担う想定。外部 API の charge は ScalarDB Tx の rollback では取り消せない。ScalarDB rollback と副作用ある外部呼び出しを同一 Tx に混在させた瞬間、原子性の幻想が崩れる。

**対策**: 副作用境界（side-effect boundary）の概念を設計に導入する。外部呼び出しを伴う処理は Outbox 経由でcommit 後ワーカが外部 API 呼び出し + 失敗時に Compensation コマンドを発行するハイブリッド Saga を Payment の外部連携 ACL 導入時点で必須化する。

---

### SYN-004: Dashboard の GROUP BY 集計が ScalarDB Standard では実現不可 ⚠️
**観点**: Risk + ScalarDB
**影響**: ScalarDB Core/Cluster は GROUP BY/集約関数を提供しない（ScalarDB Analytics は Premium のみ）。エディション選定 Standard のままSQL の GROUP BY で完結（TD-008/025 修正）と主張するのは技術的に成立しない。最悪のケースで全注文を Scan してアプリ側で集約する旧来と同等の問題が再発する。

**対策**: Dashboard 専用の Read Model（CQRS Read DB）を Phase 3 から必須化し、ScalarDB の Tx 経路ではなく直接 RDB に集計クエリを発行する別経路にする。または ScalarDB Analytics（Premium）を選定する旨を明記する。

---

## P1 — Must Fix（実装前に解消推奨）

| ID | タイトル | 観点 |
|---|---|---|
| SYN-005 | 1テーブルに複数 Secondary Index を設定している | ScalarDB |
| SYN-006 | 2PC 参加者 6 名の許容と stuck Tx 運用が未設計 | ScalarDB + Risk |
| SYN-007 | DR 戦略・RPO/RTO・バックアップリストア手順の欠如 | Operations |
| SYN-008 | 可観測性が一覧レベルに留まり SLI/SLO/アラート閾値が未定義 | Operations |
| SYN-009 | シークレット管理・JWT 鍵ローテーション・監査ログ保管要件の不足 | Operations |
| SYN-010 | マイクロサービス分離後の独立 CI/CD パイプライン設計が不在 | Operations |
| SYN-011 | JWT へのロール埋め込みでロール変更の伝播が遅延する | Risk |
| SYN-012 | Phase 4 で Identity 単独分離時に API Gateway + Identity が二重 SPOF 化 | Risk |
| SYN-013 | 冪等性キーの衝突検知（同一キーで内容が異なる場合）が未設計 | Risk |
| SYN-014 | ScalarDB Tx タイムアウト・OCC ストーム時の挙動が未設計 | Risk |
| SYN-015 | Phase 移行中の機能フラグ切替で Tx 境界が分断される | Risk |
| SYN-016 | Cart Service の Redis 障害時のフォールバックが未設計 | Risk |
| SYN-017 | 会員（Member）マスタ不在問題が新設計で解決されていない | Business |
| SYN-018 | CASHIER の既存権限がマイクロサービス分離後に維持される具体的設計が不在 | Business |
| SYN-019 | NFR ターゲット数値の根拠と測定方法が不明 | Business |

---

## P2 — Should Fix（品質向上のため解消推奨）

| ID | タイトル | 観点 |
|---|---|---|
| SYN-020 | audit_logs / receipts のキー設計が時系列クエリに弱い | ScalarDB |
| SYN-021 | member_points ホットスポット評価が楽観的 | ScalarDB |
| SYN-022 | Flyway DDL と ScalarDB 管理テーブルの競合リスク | ScalarDB |
| SYN-023 | coordinator.state テーブル配置と独立 Tx の整合性 | ScalarDB |
| SYN-024 | キャパシティ計画・接続プール・リソース要件の数値が未提示 | Operations |
| SYN-025 | Docker Compose → Kubernetes 移行のデプロイ進化計画が抽象的 | Operations |
| SYN-026 | Phase 横断のロールバック計画と機能フラグ管理不足 | Operations |
| SYN-027 | オンコール体制・Runbook・KPI 計測手段が未整備 | Operations |
| SYN-028 | Kafka 移行時のイベントスキーマ進化・Exactly-once 戦略が未設計 | Risk |
| SYN-029 | 日付範囲スキャンが ScalarDB の Scan 制約と整合していない | Risk |
| SYN-030 | API Gateway 認可と Service 側 @PreAuthorize のドリフトリスク | Risk |
| SYN-031 | 24 ヶ月計画のビジネス価値提示が技術指標中心 | Business |
| SYN-032 | 既存機能互換性保証の具体策不足（移行中の無停止保証） | Business |
| SYN-033 | 分析フェーズの BC 候補数と設計フェーズの BC 数の不一致が説明されていない | Consistency |
| SYN-034 | サービスカタログのサービス数とフェーズ別 KPI の最終サービス数が不一致 | Consistency |
| SYN-035 | OpenAPI の productId 型と BC 設計の値オブジェクト方針が不整合 | Consistency |

---

## P3 — Consider（改善提案）

| ID | タイトル | 観点 |
|---|---|---|
| SYN-036 | テーブル数表記が schema.md と migration.md で不整合（15 vs 13） | ScalarDB |
| SYN-037 | BIGINT 採番戦略（receipt_id 等）が未指定 | ScalarDB |
| SYN-038 | API Gateway のフェイルオーバ・冗長構成・タイムアウトが未記述 | Operations |
| SYN-039 | ScalarDB スキーマ変更のロールバック計画不足 | Risk |
| SYN-040 | TaxCategory の値マッピング表なし（int 1/2 → STANDARD/REDUCED） | Consistency |
| SYN-041 | context-map.md の Checkout→Return 矢印が bounded-contexts-redesign.md と矛盾 | Consistency |
| SYN-042 | Loyalty Context の 3 集約の責務境界が context-map に表現されていない | Consistency |
| SYN-043 | ダッシュボード指標とレシート閲覧スコープのカバレッジ細部が欠落 | Business |
| SYN-044 | 既存欠陥（TD-011/012/029）の修正検証手段の言及が弱い | Business |
| SYN-045 | Phase 1 の対象 CRITICAL 負債の根拠が外部ドキュメント依存 | Business |
| SYN-046 | CheckoutItem の taxCategory 責務分担（サーバ決定 vs 外部入力）が曖昧 | Consistency |
| SYN-047 | Gateway のレート制限値の根拠・WAF/mTLS の運用方針が不足 | Operations |
| SYN-048 | Phase 5 の MMI 82% が target-architecture.md の 75%+ を超過する説明がない | Consistency |

---

## 再審査ロードマップ

```
【現在】FAIL（CRITICAL 4件）
   ↓  SYN-001〜004 を解消（ScalarDB HA, Outbox 必須化, 外部SaaS Saga設計, Dashboard Read Model）
【Step 1】CRITICAL 0件 → CONDITIONAL_PASS 候補
   ↓  SYN-005〜019（P1 Must Fix 15件）を解消
【Step 2】major 8件以下 → CONDITIONAL_PASS 確定
   ↓  P2/P3 を順次解消
【Step 3】aggregate ≥ 3.5, all perspectives ≥ 3.0 → PASS
```

### P0 解消後の追加スキル実行推奨

| 優先順位 | スキル | 解消する主な P0/P1 |
|---|---|---|
| 1 | `/architect:design-disaster-recovery` | SYN-001（HA/RTO/RPO）, SYN-007 |
| 2 | `/architect:design-observability` | SYN-008, SYN-027 |
| 3 | `/architect:design-security` | SYN-009, SYN-011, SYN-012, SYN-018 |
| 4 | `/architect:design-infrastructure` | SYN-010, SYN-025 |
| 5 | `/architect:estimate-cost` | SYN-024 |
::::


| 観点 | スコア |
| :--- | :---: |
| Business | 3.0 |
| Architecture | 3.0 |
| Operations | 2.5 |
| Risk | 1.5 |
| ScalarDB | 2.5 |
| **総合** | **2.625/5.0** |

判定は **FAIL** です。

理由は、P0 Blockerが4件あったためです。

| ID | P0 Blocker | 問題 |
| :--- | :--- | :--- |
| SYN-001 | ScalarDB ClusterのHA構成が未定義 | 本番可用性の根拠がない |
| SYN-002 | Outboxがオプション扱い | at-least-onceすら保証できない |
| SYN-003 | 外部決済SaaSの補償が破綻 | ScalarDB rollbackでは外部副作用を戻せない |
| SYN-004 | Dashboard集計がScalarDB Standard制約に違反 | `GROUP BY` 前提が成立しない |

このFAIL判定は、かなり価値があります。

AIが生成した設計書は、一見するとそれらしい構成図やサービス分割を含んでいるため、そのまま進めたくなります。しかしレビューにかけると、運用できない配信保証がない外部副作用を戻せない採用技術の制約に反しているという、実装前に潰すべきリスクが見つかっています。

設計レビューがなければ、これらは実装フェーズや本番障害で発覚していた可能性があります。


### v2でP0を解消

`review-synthesis-v2.md` では、P0 4件がすべて構造的に解消され、判定は **CONDITIONAL_PASS** へ上がっています。

![review-synthesis-v2](/images/legacy-refactoring-nexus-scalardb/review-synthesis-v2.png)
*P0解消後の再レビュー結果*

::::details レポート全文
---
title: レビュー統合レポート v2 (P0 ブロッカー対応後)
schema_version: 1
phase: "Phase 4: Review (v2)"
skill: review-synthesizer
generated_at: "2026-05-14T13:50:00+09:00"
verdict: CONDITIONAL_PASS
aggregate_score: 3.65
previous_verdict: FAIL
previous_aggregate_score: 2.625
---

# レビュー統合レポート v2

## 1. 最終判定

**判定: CONDITIONAL_PASS**（aggregate score 3.65 / 5.0）

- 前回 P0 ブロッカー 4 件は **すべて構造的に解消** された。
- 新規 3 ドキュメント（`disaster-recovery.md` / `saga-design.md` / `read-model-design.md`）が既存設計に統合され、Risk 観点のスコアは **1.5 → 4.0（+2.5）** と大きく改善。
- ただし、新旧ドキュメントの整合と運用詳細に **HIGH 8 件** が残っており、PASS の品質ゲート（major <= 3）を満たさないため CONDITIONAL_PASS 判定。
- aggregate >= 3.5、critical 0、全観点 >= 3.0、major <= 8 を満たすため、Phase 5 設計フェーズ（design-implementation 等）への進行は可能。並行して P1 改修を進めることを推奨。

## 2. スコアサマリ

| 観点 | 重み | v1 スコア | v2 スコア | 増減 | Verdict |
|---|---:|---:|---:|---:|---|
| Consistency | 15% | 3.0 | **4.0** | +1.0 | PASS_WITH_RECOMMENDATIONS |
| ScalarDB    | 25% | 3.0 | **4.0** | +1.0 | PASS_WITH_RECOMMENDATIONS |
| Operations  | 20% | 3.0 | 3.0 | 0   | PASS_WITH_RECOMMENDATIONS |
| Risk        | 25% | 1.5 | **4.0** | +2.5 | PASS_WITH_RECOMMENDATIONS |
| Business    | 15% | 3.0 | 3.0 | 0   | PASS_WITH_RECOMMENDATIONS |
| **加重平均**| 100% | **2.625** | **3.65** | **+1.025** | **CONDITIONAL_PASS** |

### 品質ゲート評価

| ゲート条件 | 結果 |
|---|---|
| aggregate >= 3.5 | PASS（3.65） |
| CRITICAL 0 | PASS（0 件） |
| HIGH（major）<= 3 | FAIL（8 件） |
| 全観点 >= 3.0 | PASS（最低 3.0） |

PASS の major <= 3 を満たさないため CONDITIONAL_PASS。一方、CONDITIONAL_PASS の上限（aggregate >= 2.5、CRITICAL <= 2、major <= 8）はすべて満たす。

## 3. 前回 CRITICAL 解消状況

| ID | 内容 | v1 評価 | v2 結果 | 反映ドキュメント |
|---|---|---|---|---|
| SYN-001 | ScalarDB Cluster の HA / DR 戦略不在 | CRITICAL | **RESOLVED** | `disaster-recovery.md`（3 ノード Multi-AZ + Patroni 3 ノード + indirect モード + 4 ティア RTO/RPO 表） |
| SYN-002 | 二重書き込み問題（Outbox 不在） | CRITICAL | **RESOLVED** | `scalardb-transaction.md`（Phase 2 から必須化）+ `scalardb-schema.md`（outbox_events / outbox_dlq / processed_event_ids 追加） |
| SYN-003 | Saga 設計の欠陥（補償の握り潰し / 副作用境界曖昧） | CRITICAL | **RESOLVED** | `saga-design.md`（Pure Tx 領域と副作用境界を分離、ハイブリッド Saga + saga_state テーブル） |
| SYN-004 | Standard vs Premium の判断不在 | CRITICAL | **RESOLVED** | `read-model-design.md`（Standard 維持 + 独立 PostgreSQL の Read Model + Polling Publisher + Projector） |

## 4. 統合 findings サマリ

- 生 findings 合計: 49（v2 の 3 観点 + v1 維持の 2 観点）
- 重複排除後: 38（型不整合・DR・TTL バッチで 3 件をマージ）
- severity 内訳: CRITICAL 0 / HIGH 8 / MEDIUM 18 / LOW 11 / INFO 1
- 優先度内訳: P0 0 / P1 8 / P2 6 / P3 24

## 5. 優先度別 findings

### 5.1 P0（CRITICAL）— 0 件

なし。

### 5.2 P1（HIGH from 2+ perspectives, or HIGH from risk/scalardb）— 8 件

| ID | severity | 観点 | タイトル | 推奨アクション |
|---|---|---|---|---|
| SYN-V2-001 | HIGH | Consistency / ScalarDB / Risk | outbox_events.event_id の型不整合（TEXT/UUID v4 vs BIGINT） | UUID v4(TEXT) に統一、read-model-design 全 DDL とコード例を修正、リプレイは created_at 範囲で行う |
| SYN-V2-002 | HIGH | Consistency | saga_state テーブルが scalardb-schema.md に欠落 | Saga Namespace セクション追加、合計 36→37 に更新 |
| SYN-V2-003 | HIGH | Consistency | payment_dead_letter / notification / shipment dead letter のスキーマ未定義 | Payment Namespace に payment_dead_letter DDL 追加、Phase 5 オプション分は注記 |
| SYN-V2-004 | HIGH | Consistency | context-map / BC redesign の Audit/Dashboard 記述が Outbox/Read Model 必須化を未反映 | Audit/Dashboard Context に Phase 2/3 必須化を明記、context-map の Mermaid に Outbox/Projector/Read DB を追加 |
| SYN-V2-005 | HIGH | Risk | Polling Publisher / Projector の HA・リーダ選出が未確定 | K8s Lease + 2 レプリカ active-standby に固定、saga-design OPEN-1 をクローズ、SLO に重複処理上限明示 |
| SYN-V2-006 | HIGH | Risk | Outbox 保持期間が文書間で 7 日 vs 90 日と矛盾 | 二段構成（本体 7 日 + アーカイブ 90 日）に統一、Replayer は両ソースをマージ |
| SYN-V2-007 | HIGH | Risk / Operations | Dashboard Read DB の DR リージョン跨ぎ戦略が未定義 | DR async replica or S3 cross-region Outbox アーカイブから再構築する Runbook、§6.2 表に Read DB 行追加 |
| SYN-V2-008 | HIGH | Risk | saga_state atomicity と PaymentACL Worker の二段書込ギャップ | タイムアウト監視バッチを独立 Runbook 化、AWAITING_EXTERNAL 4h で ESCALATED、起動時クラッシュリカバリ実装 |

### 5.3 P2（HIGH from 1 perspective, MEDIUM common across 3+）— 6 件

| ID | severity | 観点 | タイトル | 推奨アクション |
|---|---|---|---|---|
| SYN-V2-009 | HIGH | Operations | 可観測性の SLI/SLO・アラート閾値が未定義（v1 残存） | /architect:design-observability 実施、各サービス Golden Signals + Burn-rate アラート + ScalarDB ダッシュボード |
| SYN-V2-010 | HIGH | Operations | シークレット管理・JWT 鍵ローテーション・監査ログ WORM（v1 残存） | Phase 2 までに Vault 必須化、kid ローテーション、refresh 失効ストア、WORM 保管 SLA |
| SYN-V2-011 | HIGH | Operations | マイクロサービス独立 CI/CD パイプライン設計が不在（v1 残存） | Phase 1 末で CI/CD ブループリント作成、Phase 4 Identity Wave で参照実装 |
| SYN-V2-012 | HIGH | Business | Member 集約不在問題が新設計で未解決（v1 残存） | Loyalty 内サブ集約として Member（属性、本人確認、停止/退会）を定義 |
| SYN-V2-013 | HIGH | Business | CASHIER 既存権限のマイクロサービス分離後維持設計が不在（v1 残存） | target-architecture にロール × API 権限マトリクスを追加 |
| SYN-V2-014 | HIGH | Business | NFR ターゲット数値の根拠と測定方法が未定義（v1 残存） | NFR ごとに『ビジネス要求 → SLO/SLI → 目標値』のトレース表追加 |

### 5.4 P3（MEDIUM/LOW）— 24 件

| ID | severity | 観点 | タイトル |
|---|---|---|---|
| SYN-V2-015 | MEDIUM | ScalarDB | outbox_dlq の Secondary Index が運用上の必要性に対し過剰 |
| SYN-V2-016 | MEDIUM | ScalarDB / Risk | processed_event_ids の TTL 削除バッチ運用負荷 |
| SYN-V2-017 | MEDIUM | ScalarDB | coordinator namespace を専用 Patroni に向ける multi-storage 設定の不整合 |
| SYN-V2-018 | MEDIUM | ScalarDB | 1 業務 Tx で複数 Outbox イベントを書く際の順序保証が未定義 |
| SYN-V2-019 | MEDIUM | ScalarDB | Projector DLQ の責務分離・再投入経路が曖昧 |
| SYN-V2-020 | MEDIUM | Risk | OCC ストーム時の Adaptive Concurrency しきい値が未定量化 |
| SYN-V2-021 | MEDIUM | Risk | レシートプリンタ印字（SE-RECEIPT-1）の現行運用との整合が未記載 |
| SYN-V2-022 | MEDIUM | Risk | Read Model の順序逆転耐性が部分的にしか担保されていない |
| SYN-V2-023 | MEDIUM | Consistency | context-map の Mermaid に CheckoutUC→Return が残存（CON-V1-006 未解消） |
| SYN-V2-024 | MEDIUM | Consistency | Phase 5 でのインフラ拡張（Cluster 5 ノード）の transformation-plan 未連携 |
| SYN-V2-025 | MEDIUM | Consistency | Outbox 進化ロードマップの表現粒度がドキュメント間で揃っていない |
| SYN-V2-026 | MEDIUM | Consistency | dashboard.processed_event_ids の配置（ScalarDB vs Read Model）が曖昧 |
| SYN-V2-027 | MEDIUM | Operations | キャパシティ計画・接続プール・リソース要件の数値が未提示（v1 残存） |
| SYN-V2-028 | MEDIUM | Operations | Docker Compose → Kubernetes 移行のデプロイ進化計画が抽象的（v1 残存） |
| SYN-V2-029 | MEDIUM | Operations | Phase 横断のロールバック計画と機能フラグ責任分界が不足（v1 残存） |
| SYN-V2-030 | MEDIUM | Operations | オンコール体制・Runbook・KPI 計測手段が未整備（v1 残存） |
| SYN-V2-031 | MEDIUM | Operations | イベント駆動の可観測性・整合性検証が手薄（v1 一部解消） |
| SYN-V2-032 | MEDIUM | Business | 24 ヶ月計画のビジネス価値提示が技術指標中心（v1 残存） |
| SYN-V2-033 | MEDIUM | Business | 既存機能の互換性保証メカニズムが言及のみで具体策不足（v1 残存） |
| SYN-V2-034 | MEDIUM | Business | ダッシュボード指標とレシート閲覧のカバレッジに細部欠落（v1 残存） |
| SYN-V2-035 | LOW | Consistency | system 全体図と DR Multi-AZ 図のコンポーネント表記の食い違い |
| SYN-V2-036 | LOW | Consistency | ハイブリッド Saga と副作用境界の英日表記揺れ |
| SYN-V2-037 | LOW | Consistency | transformation-plan の Phase 別 KPI 表に Read Model / Outbox SLO 不在 |
| SYN-V2-038 | LOW | Consistency | 新規 3 ドキュメントから既存ドキュメントへのバックリンクが片側のみ |

（その他、SYN-V2-039〜049 は ScalarDB / Risk / Operations / Business の LOW・INFO 残課題。詳細は JSON 参照。）

## 6. v1 からの差分

| 項目 | v1 | v2 | 差分 |
|---|---|---|---|
| 判定 | FAIL | CONDITIONAL_PASS | **昇格** |
| aggregate | 2.625 | 3.65 | +1.025 |
| CRITICAL | 4 | 0 | -4（**全件解消**） |
| HIGH（dedup 後） | 17 | 8 | -9 |
| Risk スコア | 1.5 | 4.0 | +2.5（最大改善） |
| Consistency スコア | 3.0 | 4.0 | +1.0 |
| ScalarDB スコア | 3.0 | 4.0 | +1.0 |
| Operations スコア | 3.0 | 3.0 | 変化なし（再レビュー対象外） |
| Business スコア | 3.0 | 3.0 | 変化なし（再レビュー対象外） |

### 主な改善内容

1. **HA/DR 戦略の整備**: ScalarDB Cluster 3 ノード + Patroni 3 ノード Multi-AZ、indirect モード、4 ティア RTO/RPO（Tier 0/1/2/3）が `disaster-recovery.md` で網羅。
2. **Outbox の必須化**: Phase 2 から `outbox_events` + Polling Publisher を必須経路化。Audit / Dashboard / Notification すべてが Outbox 経由に統一。
3. **副作用境界の分離**: ハイブリッド Saga として Pure Tx と外部副作用（Stripe / Printer / Mail）を `saga-design.md` で明確化、`saga_state` テーブル + Payment ACL Worker で永続化。
4. **Read Model の独立 PostgreSQL 化**: Standard 維持のまま GROUP BY/JOIN 要件を独立 PG に外出しすることで Premium 不要と判断。`read-model-design.md` で Projector / projector_offset / projector_dlq を定義。

## 7. 残存リスクと推奨次ステップ

### 7.1 Phase 2 着手前に必ず解消すべき P1 課題（8 件）

これらは構造的な型・スキーマ・運用整合の問題で、Phase 2 実装に直結する。

1. **SYN-V2-001**: event_id 型を UUID v4(TEXT) に統一。read-model-design.md の DDL とコード例を全面修正。
2. **SYN-V2-002, SYN-V2-003**: scalardb-schema.md に saga_state、payment_dead_letter の DDL を追加し合計テーブル数を更新。
3. **SYN-V2-004**: context-map.md / bounded-contexts-redesign.md に Outbox / Read Model 必須化を反映。
4. **SYN-V2-005**: Polling Publisher / Projector の HA リーダ選出方式を 1 つに固定（推奨: K8s Lease）。
5. **SYN-V2-006**: Outbox 保持期間を二段構成（本体 7 日 + アーカイブ 90 日）に統一。
6. **SYN-V2-007**: Dashboard Read DB の DR 戦略を明文化し RTO/RPO 表に追加。
7. **SYN-V2-008**: saga_state タイムアウト監視バッチの Runbook 化と起動時クラッシュリカバリ実装。

### 7.2 Phase 1 並行で進める P2 課題（6 件）

Operations / Business の v1 HIGH 残存。次フェーズで以下を実施:

- `/architect:design-observability` で SLI/SLO / アラート設計を完成
- `/architect:design-security` で Vault / JWT / 監査ログ WORM を明文化
- `/architect:design-infrastructure` で CI/CD ブループリント
- bounded-contexts-redesign.md に Member 集約を追加
- target-architecture.md にロール × API 権限マトリクスを追加
- NFR トレース表（ビジネス要求 → SLO/SLI → 目標値）を整備

### 7.3 推奨進行ジャッジ

- **PASS WITH CONDITIONS**: 上記 P1 8 件を Phase 2 着手前マイルストーンとして固定し、解消確認後に design-implementation / generate-scalardb-code に進む。
- P3（MEDIUM/LOW 24 件）は Phase 2-3 中の改修として取り扱い、次回 review-synthesizer 実行時に diff チェック。
- 次の review チェックポイントは Phase 2 完了時（Outbox 動作 + saga_state 運用が立ち上がった時点）。
::::


| P0 | 解消方法 |
| :--- | :--- |
| SYN-001 | `disaster-recovery.md` に3ノードMulti-AZ、Patroni、RTO/RPOを追加 |
| SYN-002 | `scalardb-transaction.md` と `scalardb-schema.md` でOutboxをPhase2から必須化 |
| SYN-003 | `saga-design.md` でPure Txと副作用境界を分離し、ハイブリッドSagaを追加 |
| SYN-004 | `read-model-design.md` でStandard + 独立PostgreSQL Read ModelのCQRS設計へ変更 |

スコアも、2.625から3.65へ改善しています。

| 指標 | v1 | v2 |
| :--- | :---: | :---: |
| 判定 | FAIL | CONDITIONAL_PASS |
| 総合スコア | 2.625 | 3.65 |
| CRITICAL | 4 | 0 |
| HIGH | 17 | 8 |

ここで面白いのは、レビューが単にダメ出しで終わっていないことです。

各P0に対して、どの設計文書を追加・修正すれば解消できるのかが明確に紐づいています。

たとえばOutboxの問題は、トランザクション設計とスキーマ設計の両方を修正しないと解消できません。Sagaの問題は、ターゲットアーキテクチャの説明だけでは足りず、外部副作用を扱う専用のSaga設計が必要でした。

このように、レビュー結果がそのまま次に作るべき設計書のバックログになっています。


### v3で本番運用の設計を補強

`review-synthesis-v3.md` では、さらにP1課題への対応が進み、総合スコアは **4.275/5.0** まで上がっています。

![review-synthesis-v3](/images/legacy-refactoring-nexus-scalardb/review-synthesis-v3.png)
*P1解消後の最終レビュー結果*

::::details レポート全文
---
title: 最終レビュー統合レポート v3
schema_version: 1
phase: "Phase 4: Review (v3)"
skill: review-synthesizer
generated_at: "2026-05-14T00:00:00Z"
---

# 最終レビュー統合レポート v3

## 1. 最終判定

```
============================================================
  VERDICT: CONDITIONAL_PASS
  AGGREGATE SCORE: 4.275 / 5.00  (PASS 閾値 3.50 を超過)
  CRITICAL: 0   HIGH: 8   MEDIUM: 12   LOW: 5
  全 P0 (4/4) RESOLVED   全 P1 追跡対象 (12/12) RESOLVED
  進捗: FAIL (v1, 2.625) → CONDITIONAL_PASS (v2, 3.65) → CONDITIONAL_PASS (v3, 4.275)
============================================================
```

**結論**: 集約スコア・全観点最低スコア・CRITICAL 0 件は PASS 基準を満たすが、PASS の `major <= 3` 条件に対し HIGH 8 件が残るため **CONDITIONAL_PASS**。残 HIGH はすべて v2 で新規発見されたスキーマ整合・Outbox/Saga 運用詳細であり、Risk / Consistency 観点の v3 再レビュー（次サイクル）でクローズすれば PASS 到達が見込める。

---

## 2. スコアサマリ表（v1 / v2 / v3 推移）

| 観点 | 重み | v1 | v2 | v3 | Δ(v1→v3) | v3 verdict | ソース |
|---|---|---|---|---|---|---|---|
| Consistency | 15% | 3.0 | 4.0 | **4.0** | +1.0 | PASS_WITH_RECOMMENDATIONS | v2 流用 |
| ScalarDB | 25% | 3.0 | 4.0 | **4.0** | +1.0 | PASS_WITH_RECOMMENDATIONS | v2 流用 |
| Operations | 20% | 3.0 | 3.0 | **5.0** | +2.0 | **PASS** | v3 |
| Risk | 25% | 1.5 | 4.0 | **4.0** | +2.5 | PASS_WITH_RECOMMENDATIONS | v2 流用 |
| Business | 15% | 3.0 | 3.0 | **4.5** | +1.5 | PASS_WITH_RECOMMENDATIONS | v3 (SYN-017 解消反映) |
| **加重集約** | 100% | **2.625** | **3.65** | **4.275** | **+1.65** | **CONDITIONAL_PASS** | — |

### 品質ゲート評価

| ゲート | 閾値 | 実測 | 判定 |
|---|---|---|---|
| 集約スコア | >= 3.5 | 4.275 | PASS |
| CRITICAL | 0 | 0 | PASS |
| HIGH (major) | <= 3 | 8 | **FAIL → CONDITIONAL** |
| 観点最低スコア | >= 3.0 | 4.0 (Consistency / ScalarDB / Risk) | PASS |
| CONDITIONAL_PASS（aggregate>=2.5, critical<=2, major<=8） | — | 全条件達成 | PASS |

---

## 3. 全 CRITICAL（SYN-001〜004）解消状況

| ID | 概要 | 状態 | 解消経路 |
|---|---|---|---|
| SYN-001 | ScalarDB Cluster の HA 構成が未定義 | **RESOLVED** | disaster-recovery.md で 3 ノード Multi-AZ + Patroni 3 ノード + indirect モード + 4 Tier RTO/RPO 表 |
| SYN-002 | Outbox を Phase 2-3 でオプション化 | **RESOLVED** | scalardb-transaction.md で Phase 2 から必須化、scalardb-schema.md に outbox_events / outbox_dlq / processed_event_ids 追加 |
| SYN-003 | 外部決済 SaaS で ScalarDB rollback だけの補償が破綻 | **RESOLVED** | saga-design.md で Pure Tx 領域と副作用境界分離、ハイブリッド Saga + saga_state テーブル |
| SYN-004 | Dashboard の GROUP BY が ScalarDB Standard 不可 | **RESOLVED** | read-model-design.md で Standard 維持 + 独立 PostgreSQL Read Model + Polling Publisher + Projector |

**全 4 件 構造的に解消**（v2 時点で達成、v3 でも維持）。

---

## 4. 全 P1（HIGH 12 件）解消状況

| ID | 概要 | 状態 | v3 での主たる解消経路 |
|---|---|---|---|
| SYN-007 | DR 戦略・RPO/RTO・バックアップリストア手順の欠如 | **RESOLVED** | disaster-recovery-design.md で Tier 別 RTO/RPO、Patroni HA、リストア手順、DR ドリル計画 |
| SYN-008 | 可観測性の SLI/SLO/アラート閾値未定義 | **RESOLVED** | observability-design.md で SLO 13 件、Burn Rate アラート 15 件、Golden Signals、ScalarDB ダッシュボード、Loki 監査ログ保管 |
| SYN-009 | シークレット管理・JWT 鍵ローテ・監査ログ保管 | **RESOLVED** | security-design.md で Vault HA、DB クレデ自動ローテ、JWT kid 30 日ローテ、refresh 失効ストア、監査 WORM |
| SYN-010 | 独立 CI/CD パイプライン設計が不在 | **RESOLVED** | deployment-guide.md で GitHub Actions テンプレ、Argo CD/Rollouts canary、Trivy/Sonar、Pact、Flyway+schema-loader 連動 |
| SYN-011 | JWT のロール伝播遅延（権限剥奪が効かない） | **RESOLVED** | security-design.md §JWT で kid ローテ 30 日 + refresh-token blacklist + 短寿命 access token |
| SYN-012 | Phase 4 で API Gateway + Identity 二重 SPOF 化 | **RESOLVED** | deployment-guide.md / infrastructure 設計で HA レプリカ + HPA + Argo Rollouts canary |
| SYN-017 | 会員（Member）マスタ不在 | **RESOLVED** | bounded-contexts-redesign.md / scalardb-schema.md / target-architecture.md / context-map.md に Member 集約（属性・status・verificationLevel・PII 暗号化・API ロール権限）と member テーブルを追加（v3 追加修正） |
| SYN-018 | CASHIER 既存権限の MS 化後維持設計が不在 | **RESOLVED** | security-design.md §4.2 で 75 行のロール×API 権限マトリクスと CASHIER 既存 URL→新 API の 1:1 マッピング |
| SYN-019 | NFR ターゲット数値の根拠と測定方法 | **RESOLVED** | observability-design.md でサービス別 SLO 13 件 + ビジネス KPI マッピング、disaster-recovery-design.md で Tier 別 RTO/RPO |
| SYN-024 | キャパシティ計画・接続プール・リソース要件 | **RESOLVED** | scalardb-sizing.md で HikariCP / Pod sizing / HPA 数値、ピーク TPS、容量モデルを Phase 別に提示 |
| SYN-025 | Docker Compose → K8s デプロイ進化計画 | **RESOLVED** | deployment-guide.md で Compose→K8s 進化、Helm 構造、Argo Rollouts、values 設計 |
| SYN-027 | オンコール・Runbook・KPI 計測手段 | **RESOLVED** | observability-design.md / disaster-recovery-design.md で Runbook 6 種、輪番、深刻度、ポストモーテム、KPI 自動計測 |

**全 12 件 RESOLVED（解消率 100%）**。

---

## 5. 残存 findings リスト（P1/P2/P3）

### P1（HIGH） 8 件 — すべて v2 で新規発見されたスキーマ整合・Outbox/Saga 運用詳細

| ID | 観点 | カテゴリ | 概要 |
|---|---|---|---|
| SYN-V3-001 | Consistency / ScalarDB / Risk | Schema Type | outbox_events.event_id の型不整合（TEXT/UUID v4 vs BIGINT） |
| SYN-V3-002 | Consistency | Schema | saga_state テーブルが scalardb-schema.md のテーブル一覧に欠落 |
| SYN-V3-003 | Consistency | Schema | payment_dead_letter / notification_dead_letter / shipment_dead_letter の DDL 欠落 |
| SYN-V3-004 | Consistency | Traceability | context-map.md / BC redesign の Audit/Dashboard が Outbox/Read Model 必須化を未反映 |
| SYN-V3-005 | Risk | SPOF / Eventing | Polling Publisher / Projector の HA・リーダ選出が未確定 |
| SYN-V3-006 | Risk | Retention | Outbox 保持期間の文書間矛盾（7 日 vs 90 日） |
| SYN-V3-007 | Risk | DR / Read Model | Dashboard Read DB の DR リージョン跨ぎ戦略が未定義（Risk 残存） |
| SYN-V3-008 | Risk | Saga Atomicity | saga_state atomicity と PaymentACL Worker の二段書込ギャップ |

### P2（MEDIUM） 6 件

| ID | 観点 | 概要 |
|---|---|---|
| SYN-V3-009 | Business | memberId 参照整合性の実装方針が STRIDE 脅威モデルと未連携 |
| SYN-V3-010 | ScalarDB | outbox_dlq の Secondary Index が運用上過剰（reviewed が適切） |
| SYN-V3-011 | ScalarDB / Risk | processed_event_ids の TTL 削除バッチ運用負荷（1.5 億行規模） |
| SYN-V3-012 | ScalarDB | coordinator namespace の multi-storage 設定が schema.md 側と不整合 |
| SYN-V3-013 | ScalarDB | 1 業務 Tx で複数 Outbox イベントを書く際の順序保証が未定義 |
| SYN-V3-014 | ScalarDB | Projector DLQ の責務分離・再投入経路が曖昧 |

### P3（LOW + 一部 MEDIUM） 11 件（抜粋）

- SYN-V3-015 OCC ストーム時の Adaptive Concurrency しきい値未定量化（Risk/Operations）
- SYN-V3-016 レシートプリンタ印字（SE-RECEIPT-1）の現行運用との整合未記載
- SYN-V3-017 Read Model の順序逆転耐性（OrderCancelled 先行）が部分的
- SYN-V3-018 context-map.md Mermaid に CheckoutUC→Return が残存
- SYN-V3-019 Phase 5 Cluster 5 ノード化が transformation-plan に未連携
- SYN-V3-020 dashboard.processed_event_ids 配置（ScalarDB 側 vs Read Model 側）の整合不明
- SYN-V3-021 新設 Runbook（RB-006-A/B/C, RB-007）が disaster-recovery-design.md 本文に未統合（OBS-OPEN-1）
- SYN-V3-022 Helm values スキーマと cert-manager / Vault Operator の Helm 構成が未確定（DEP-OPEN-2/3）
- SYN-V3-023 MTTD の自動計測実装が運用ハンドオフ段階
- SYN-V3-024 売上機会損失の金額換算根拠が cost-summary.md と observability で独立
- SYN-V3-025 経営向けビジネス価値の Phase 別アウトカムが定量で示されていない

完全な findings 一覧は `review-synthesis-v3.json` を参照。

---

## 6. v1 → v3 の主要な差分・追加成果物リスト

### v1 → v2 の構造変化（CRITICAL 解消）

- **新規 3 ドキュメント追加**: `disaster-recovery.md` / `saga-design.md` / `read-model-design.md`
- **既存ドキュメント更新**: `target-architecture.md` / `scalardb-schema.md` / `scalardb-transaction.md` に新規ドキュメントへの前向き参照
- **集約スコア**: 2.625 → 3.65（+1.025）
- **判定**: FAIL → CONDITIONAL_PASS

### v2 → v3 の構造変化（P1 完全解消）

#### Operations 観点の構造的解消（HIGH 7 件 + P2 残存 4 件 → HIGH 0 件）

新規 8 ドキュメント:
- `08_infrastructure/disaster-recovery-design.md`（DR/RPO/RTO Tier 別、Patroni HA、Runbook 整備）
- `08_infrastructure/observability-design.md`（SLO 13 件、Burn Rate アラート 15 件、Golden Signals、ScalarDB ダッシュボード、Runbook 6 種）
- `08_infrastructure/security-design.md`（Vault HA、JWT 30 日鍵ローテ、refresh 失効、監査ログ WORM、ロール×API マトリクス 75 行）
- `08_infrastructure/deployment-guide.md`（Compose→K8s 進化、Helm、Argo CD/Rollouts canary、Trivy/Sonar、Pact、Flyway+schema-loader 連動）
- `08_infrastructure/scalardb-sizing.md`（HikariCP / Pod sizing / HPA 数値、ピーク TPS、容量モデル）
- `05_estimate/cost-summary.md` 等 3 ドキュメント（TCO 比較、機会損失削減 1.48 億円、MTTR/Deploy Frequency/リードタイム指標）

#### Business 観点の SYN-017 追加修正

bounded-contexts-redesign.md / scalardb-schema.md / target-architecture.md / context-map.md に:
- **Member 集約**（属性、status、verificationLevel、PII 暗号化）
- **member テーブル**（DDL）
- **API ロール権限**（CASHIER/MANAGER/ADMIN × Member CRUD のマトリクス）

を追加し、SYN-017（会員マスタ不在）を完全クローズ。

### 追加成果物の累計（v1 起点）

| カテゴリ | v1 時点 | v3 時点 | 増分 |
|---|---|---|---|
| アーキテクチャ設計 | 7 ドキュメント | 10 | +3（disaster-recovery / saga-design / read-model-design） |
| インフラ設計 | 0 | 5 | +5（08_infrastructure 配下） |
| コスト見積 | 0 | 3 | +3（05_estimate 配下） |
| 集約 namespace 数 | 9 | 9（saga / dlq の不整合は P1 残存） | — |
| 集約テーブル数 | 13〜15（揺れ） | 36〜37（揺れ）| 約 +23 |

---

## 7. ROI / 投資判断の総括

### 解消済み価値

- **CRITICAL 4 件解消**: HA/DR、Outbox 必須化、Saga 設計、Read Model 分離 — Phase 2 着手の絶対条件をクリア
- **P1 12 件全解消**: 運用基盤（DR/可観測性/CI-CD/セキュリティ/キャパシティ/オンコール）と業務基盤（Member 集約/CASHIER 権限/NFR 数値）が揃い、本番投入前ゲートを満たす
- **集約スコア +1.65**（2.625 → 4.275）: 加重平均で約 63% 改善
- **Operations 観点 +2.0**（3.0 → 5.0）: 単一観点での最大改善幅
- **TCO 試算**: cost-summary.md §4 で Legacy 継続 vs 新アーキの 3 年 TCO 比較、機会損失削減 1.48 億円、MTTR/Deploy Frequency/リードタイム指標を提示

### 残存リスクと投資要否

残 HIGH 8 件はいずれも:
- (a) **既存設計の局所的な整合不足**（event_id 型、saga_state DDL、payment_dead_letter DDL、context-map / BC の文言更新）
- (b) **Outbox/Saga 運用詳細の積み残し**（Polling Publisher HA、Outbox 保持期間、Read DB DR、saga_state atomicity）

であり、構造的な再設計は不要。Phase 2 着手前のドキュメント整合改修（Risk + Consistency の v3 再レビュー）で **PASS 到達が現実的**。

**投資判断**: 既に大規模な追加投資（v1→v3 で 11 ドキュメント追加）に対して `+1.65` のスコア改善を得ており、限界費用に対する限界効用は依然として高い。残作業はドキュメント整合（推定 2〜3 人日）であり、Phase 2 着手前に完了可能。**実装フェーズへの移行を強く推奨**（CONDITIONAL_PASS の条件付き）。

---

## 8. 次のステップ

### 直近（Phase 2 着手前 必須）

1. **Risk / Consistency 観点の v3 再レビュー実施**
   - SYN-V3-001（event_id 型）統一: read-model-design.md の DDL を TEXT(UUID v4) に修正
   - SYN-V3-002（saga_state）: scalardb-schema.md に Saga Namespace + saga_state DDL 追記、合計 36→37
   - SYN-V3-003（payment_dead_letter）: scalardb-schema.md に DLQ DDL 追記
   - SYN-V3-004（BC redesign）: Audit/Dashboard Context に Outbox/Read Model 必須化を反映
   - SYN-V3-005（Polling Publisher HA）: K8s Lease + 2 レプリカ active-standby に確定
   - SYN-V3-006（Outbox 保持期間）: 7 日（本体）+ 90 日（アーカイブ）の二段構成に統一
   - SYN-V3-007（Dashboard Read DB DR）: DR リージョン async replica or S3 cross-region リプレイ Runbook 化
   - SYN-V3-008（saga_state atomicity）: タイムアウト監視バッチを独立 Runbook 化、AWAITING_EXTERNAL 4h 上限

2. **再レビュー後の v4 集約予測**: HIGH 8 件解消で `major <= 3` 達成 → **PASS 確実**

### Phase 2 着手判断

- **CONDITIONAL_PASS の条件**: 上記 8 件のドキュメント整合改修を Phase 2 開始前に完了
- **完了確認手段**: review-synthesizer の v4 サイクルで PASS verdict を取得
- 完了後は **Phase 1（Strangler Fig 準備 / Identity Wave）→ Phase 2（ScalarDB 全面切替）** に進む

### 中期（Phase 2-3 並行）

- P2/P3 残課題（SYN-V3-009〜025）を Phase 2/3 の開始判定に組み込む
  - SYN-V3-015 Adaptive Concurrency しきい値: Phase 2 性能 POC で実測値投入
  - SYN-V3-021 新設 Runbook 統合: Phase 2 Saga / Outbox 運用開始までに完了
  - SYN-V3-022 Helm values スキーマ: Phase 3 Catalog 独立化前にレビュー完了
  - SYN-V3-024 売上機会損失根拠: 経営報告タイミングまでに cost-summary.md / observability.md で相互参照

### Phase 5 完了時の再評価

- ScalarDB Premium / Analytics 移行の判定基準（ADR）を残す（SDB-V2-012）
- Read Model 維持コスト > Premium ライセンス、HTAP 要件発生など

---

## 補足: ファイル参照

- 集約 JSON: `/Users/mavs022/Desktop/work/scalar/ai-driven-devops/refactoring-agents/nexus-architect/work/reports/review/review-synthesis-v3.json`
- v3 個別レビュー: `individual/review-operations-v3.json` / `individual/review-business-v3.json`
- v2 流用個別レビュー: `individual/review-risk-v2.json` / `individual/review-scalardb-v2.json` / `individual/review-consistency-v2.json`
- v1 / v2 集約: `review-synthesis.json` / `review-synthesis-v2.json`
::::


| 観点 | v1 | v2 | v3 |
| :--- | :---: | :---: | :---: |
| Business | 3.0 | 3.5 | 4.0 |
| Architecture | 3.0 | 4.0 | 4.25 |
| Operations | 2.5 | 3.0 | 5.0 |
| Risk | 1.5 | 3.5 | 4.0 |
| ScalarDB | 2.5 | 4.0 | 4.0 |
| **加重集約** | **2.625** | **3.65** | **4.275** |

最終判定は **CONDITIONAL_PASS** です。

PASSではなくCONDITIONAL_PASSなのは、HIGHがまだ8件残っているためです。ただし、CRITICALは0件であり、P1追跡対象はすべて解消されています。

v3で強化された主な内容は以下です。

| 領域 | 追加・改善 |
| :--- | :--- |
| Operations | EKS、Helm、Argo CD、CI/CD、Runbook |
| Security | JWT短命化、失効リスト、鍵ローテーション、WORM監査 |
| Observability | SLI/SLO、Burn Rateアラート、OpenTelemetry、オンコール |
| Business | Member集約、CASHIER権限維持、NFRトレース |
| Cost | AWS + ScalarDB + 運用体制の概算コスト |

特にOperationsが5.0点まで上がっているのは、設計がコードの分割案から運用できるシステム案へ進化したことを示しています。


### 残存リスク

v3でも、完全にすべての課題が消えたわけではありません。

残ったHIGH 8件は、主に以下のような整合性・運用詳細です。

- Outbox関連スキーマの型や保持期間の整合
- `saga_state` やDLQテーブルのスキーマ整合
- Polling Publisher/ProjectorのHA詳細
- Dashboard Read DBのDR戦略
- Saga timeoutやクラッシュリカバリの運用詳細

ただし、これらはそもそも設計の前提が破綻しているという種類の問題ではなく、実装前に詰めるべき詳細に近いものです。

`review-synthesis-v3.md` では、残作業は推定2〜3人日程度のドキュメント整合改修であり、Phase2着手前に完了可能と評価されています。

つまり、ここでのCONDITIONAL_PASSはまだ条件はあるが、実装フェーズへ進む判断は現実的という意味です。


## この設計レポートから学べること

### 1. リファクタリング設計は理想図だけでは足りない

境界コンテキストを分け、サービス一覧を作り、きれいなアーキテクチャ図を書くこと自体は難しくありません。

しかし、実際に重要なのはその先です。

- トランザクション境界はどこか
- 外部副作用はどこで発生するか
- イベントは欠落しないか
- 採用技術の制約に反していないか
- 障害時に復旧できるか
- 誰がどのRunbookで対応するか
- その運用コストはいくらか

今回のレポート群は、初回設計ではそこが弱く、レビューで指摘され、v2/v3で補強されていく構造になっていました。


### 2. ScalarDB導入で消えるSagaと、消えないSagaがある

ScalarDBを導入すると、MySQLとPostgreSQLを跨いだ業務更新をACIDトランザクションとして扱えるようになります。

そのため、既存の手書きSagaのうち、DB更新だけを補償していた部分は大きく簡素化できます。

しかし、外部決済SaaS、メール送信、SMS送信、レシート印字のような副作用は、ScalarDBの管理外です。

ここには依然としてSaga的な状態管理と補償設計が必要です。

この違いを、レポートではPure Tx領域とSide-Effect Boundaryとして整理していました。

この整理がないと、分散トランザクション基盤を入れたから全部解決という危険な誤解につながります。


### 3. DashboardはWrite Modelから切り離す

ScalarDB Standardでは、分析系SQLをそのまま実行する設計は成立しません。

ここで無理にWrite Modelへ集計要件を押し込むと、スキャンやアプリケーション集計が増え、性能問題が再発します。

今回の設計では、OutboxからDashboard Read DBへイベントを投影し、集計専用のPostgreSQLで `GROUP BY` や `SUM` を実行する方針に切り替えています。

これは、技術制約を避けるというより、**更新系と参照系の責務を分ける**という意味で自然な設計です。


### 4. レビューを設計プロセスに組み込む価値

今回最も印象的だったのは、レビューが設計の品質ゲートとして機能している点です。

初回設計はFAILでした。

しかし、そのFAILは失敗ではなく、むしろ必要な発見でした。

P0 Blockerが明確になり、それぞれに対して設計文書が追加され、再レビューでスコアが改善していく。これは、人間がアーキテクチャレビューを回すときと非常に近い流れです。

AIエージェントを使う場合でも、生成された設計をそのまま信用するのではなく、**別の視点でレビューし、品質ゲートを通し、指摘を設計へ戻す**ことが重要だと分かります。


## Nexus Architectを使うメリット

ここまでを見ると、Nexus Architectの価値は、人の経験や勘だけに頼りがちな設計判断を、レポート、スコア、レビュー結果として残せる点にあります。

現状分析では、God Service、手書きSaga、複数DB更新、境界の崩れといった問題を、個別の印象ではなく構造として確認できました。

さらに、MMIやDDD評価、設計レビューを通すことで、どこがP0で、どこが後続フェーズでもよいのかを整理できます。これは、人がレビューする場合にも必要な作業ですが、Nexus Architectを使うことで、同じ観点を繰り返し適用しやすくなります。

もう一つ大きいのは、設計の改善履歴がv1、v2、v3のように残ることです。なぜOutboxを必須にしたのか、なぜDashboardをRead Modelへ分けたのか、なぜ外部決済を副作用境界として扱うのかが、後から追える形になります。

つまり、Nexus Architectは人の判断を不要にするものではありません。むしろ、人が判断するための材料をそろえ、抜け漏れを見つけ、実装へ渡せる粒度まで設計を具体化するための補助線として機能します。


## 本章のまとめ

今回の設計・レビュー文書を通して、Nexus Architectは単にリファクタリング案を出すだけでなく、以下のような流れを支援していることが分かりました。

1. 現状のGod Service / 手書きSaga/DB跨ぎ更新を分析する
2. 境界コンテキストと集約を再定義する
3. モジュラモノリスからマイクロサービスへ段階的に移行する道筋を描く
4. ScalarDBでPure Tx領域の一貫性を担保する
5. Outboxで監査ログ・Dashboard・外部Workerへのイベント欠落を防ぐ
6. 外部副作用はハイブリッドSagaで明示的に扱う
7. CQRS Read ModelでScalarDB Standardの集計制約を回避する
8. レビューでP0/P1を検出し、設計をv1→v2→v3へ改善する
9. 最後にインフラ、セキュリティ、可観測性、コストまで含めて本番運用可能性を確認する

ここで見た設計は、まだコードそのものではありません。

しかし、次にAIエージェントへ実装を依頼するための地図としては、かなり具体的です。

次章からは、この設計をもとに、実際にレガシーPOSシステムをリファクタリングしていくプロセスへ進んでいきます。

## 用語解説

### FAIL
レビューで重大な設計欠陥が見つかり、次の工程へ進む前に修正が必要な状態です。本章では初回レビューがこの判定でした。

### CONDITIONAL_PASS
重要な欠陥は解消されているものの、実装前に対応すべき課題が残っている状態です。条件付き合格として扱われます。

### PASS
主要な品質ゲートを満たし、次の工程へ進める状態です。ただし、実装や運用で継続的な確認が不要になるわけではありません。

### Blocker
そのまま進めると設計や実装が成立しない重大な問題です。P0として扱われ、優先的に解消する必要があります。

### Runbook
障害や運用作業が発生したときの手順書です。Saga補償失敗やDR切り替えのような場面で、対応のばらつきを減らします。

### 加重集約
複数の評価観点を、重要度に応じた重み付きでまとめる計算方法です。本章では、Consistency、ScalarDB、Operations、Risk、Businessの各スコアを重みに応じて集約し、全体のレビュー判定に使っています。
