---
title: "設計レビューと再レビューで設計を固める"
---

:::message
5つの観点による設計レビュー、HTMLレポート生成、P0指摘の修正と再レビュー承認までの流れを確認します。
:::

## 5大観点による設計レビューと統合レビュー結果（/architect:review-synthesizer）

設計がひと通り完了したところで、AIエージェントはConsistency（一貫性）ScalarDB（DB制約）Operations（運用）Risk（分散システムリスク）Business（ビジネス要件）の5つの異なる観点から個別のレビューエージェントを並列起動し、それらを `review-synthesizer`エージェントで統合評価します。

設計用のエージェントとは別に、レビュー用のエージェントも実行することで**客観的な評価**が可能になります。

```text
⏺ 5 つのレビュー観点を並列実行します。
⏺ 5 agents finished
   ├ review-consistency: design consistency review
   ├ review-scalardb: ScalarDB-specific constraints review
   ├ review-operations: operational readiness review
   ├ review-risk: distributed system risk review
   └ review-business: business requirements review

⏺ review-synthesizer 完了しました。

  ---
  統合レビュー結果

  総合判定: ❌ FAIL（aggregate 2.625、CRITICAL 4 件で CONDITIONAL_PASS 上限超過）
```

P0 Blockers（設計進行不可・解消必須）

| ID | タイトル | 影響 |
| :--- | :--- | :--- |
| SYN-001 | ScalarDB Cluster の HA 構成が未定義 | Phase 2-4 全体の SPOF |
| SYN-002 | Outbox を Phase 2-3 でオプション化 → at-least-once 保証なし | Audit/Dashboard イベントがサイレントにロスト |
| SYN-003 | 外部決済 SaaS 連携で ScalarDB rollback だけの補償は破綻する | Saga 廃止の前提条件が崩れる |
| SYN-004 | Dashboard の GROUP BY が ScalarDB Standard では実現不可 | TD-008/025 の SQL 最適化主張が技術的に成立しない |

システムリスクやScalarDB固有の制約に関するレビューで致命的課題が4件指摘され、総合判定は **❌ FAIL** となりました。

設計を修正する前に、ここまでの成果物（全35ドキュメントに及ぶ膨大なマークダウン）をすべて統合したHTMLレポートを生成します。

## HTMLレポートの統合生成（/architect:report）

`/architect:report` スキルを実行し、全Markdown成果物を見やすく統合したHTMLレポートを生成します。

```text
⏺ Skill(architect:report)
⏺ ✅ HTML レポート生成完了

  - 出力先:
  /Users/mavs022/Desktop/work/scalar/ai-driven-devops/refactoring-agents/nexus-architect/work/reports/00_summary/full-report.html
  - 統合セクション: 23 個
  - 機能: Mermaid 図ダイアグラム対応、自動 TOC 生成、フェーズ別構成、ダーク/ライトテーマ対応など
```

これで、初回実行セッションにおける全設計のインプットと現状診断、初期レビューでFAILとなりHTMLレポートを生成するまでの全プロセスが完了しました。

ここまでで、Nexus Architectの実行が完了しました。

次の章では、これらの生成されたレポート群を読み解き、レガシーPOSシステムが抱える構造的な弱点について深掘りしていきたいと思います。


## 設計レビューの修正と再レビューの承認プロセス

初期設計レビュー（`/architect:review-synthesizer`）による厳格な品質チェックの結果、分散システムにおける4つの致命的な課題が特定され、総合判定は **❌ FAIL** という結果となりました。

ここからは、AIエージェントと対話し、指摘されたP0課題を1つずつ修正して品質ゲートを突破（PASS）するまでの、設計修正プロセスを辿ります。

---

### 設計レビュー指摘事項（P0 ブロッカー）の修正プロセス

初回レビューで指摘された4つのP0ブロッカー（SYN-001〜SYN-004）は、分散アーキテクチャを構築する上でいずれも本番運用に致命的な影響を及ぼす内容でした。
これらに対し、AIエージェントの専門設計スキル（DR設計、Security設計など）を追加で起動し、対話を繰り返し1つずつ設計をアップデートしました。

#### ① SYN-001：ScalarDB ClusterのHA（高可用性）構成の定義
* **課題**：システム全体の分散トランザクションを管理する ScalarDB Cluster の冗長化方針や障害時の耐障害性（HA）が未定義であり、単一障害点（SPOF）となっていた。
* **解消策**：追加スキル `/architect:design-disaster-recovery` を実行し、3ノード構成の ScalarDB Cluster、調整役としての Patroni coordinator、およびクライアント側に Resilience4j サーキットブレーカを配備する構成を決定。さらに、RTO（目標復旧時間）/ RPO（目標復旧時点）のサービス別目標（Tier 1〜3）を定義した災害復旧設計書（`disaster-recovery.md`）を新規作成しました。

![disaster-recovery-01](/images/legacy-refactoring-nexus-scalardb/disaster-recovery-01.png)
*災害復旧/高可用性設計*

#### ② SYN-002：Outboxパターンの必須化によるメッセージの到達保証
* **課題**：会員ポイント更新や監査ログ、ダッシュボードへのイベント通知について、Outbox パターンがオプション扱いとなっていたため、イベント送信の at-least-once（最低1回配信）が保証されず、イベントがサイレントにロストする懸念があった。
* **解消策**：トランザクション設計を全面的に修正し、Phase 2 からの Outbox パターン適用を必須化。ScalarDBトランザクションとアトミックに書き込まれる `outbox_events` テーブルおよび重複排除用の `processed_event_ids` の DDL を定義し、ポーリングPublisherが安全にイベントをパブリッシュする仕組みを確立しました。

#### ③ SYN-003：外部決済SaaS連携における整合性破綻の解消
* **課題**：Spring 側で例外が発生した際、ScalarDBのトランザクションロールバック（`tx.abort()`）のみで外部決済APIの処理もキャンセルできると想定していたが、既に決済ゲートウェイ側で実行されたクレジットカード課金等はDBロールバックだけでは取り消せない。
* **解消策**：アーキテクチャ内に「副作用境界（Side-Effect Boundary）」を導入。決済処理を非同期の決済ACL Workerおよび `PaymentDEADLETTER` 集約に分離し、エラー発生時には `Compensation Queue`（補償処理キュー）を介して明示的な返金API（返金トランザクション）を呼び出すハイブリッドSagaパターン（`saga-design.md`）を新たに設計しました。

![saga-design-01](/images/legacy-refactoring-nexus-scalardb/saga-design-01.png)
*副作用境界の設計*

#### ④ SYN-004：Dashboardの集計におけるScalarDB Standard機能制約の回避
* **課題**：ダッシュボードの売上集計等で SQL の `GROUP BY` によるスキャン集計を前提としていたが、分散キー/ストレージである ScalarDB Standard では GROUP BY スキャンが使用不可であるため、設計が成立していなかった。
* **解消策**：徹底した CQRS（コマンド・クエリ責務分離）を採用。更新系データベースとは完全に物理分離された PostgreSQL を読み取り専用のダッシュボード用 DB（Read Model）として新設し、注文や在庫のドメインイベントをトリガーに Projector（プロジェクター）が非同期（最終一貫性：P95 < 30秒）で集計済みデータを事前生成して保持する設計（`read-model-design.md`）を構築しました。

これらの設計変更により、3つの新規設計書が追加され、既存の5つの設計ドキュメントがアップデートされました。

| 課題 ID | 課題内容 | 主な解消手段 | 関連ドキュメント |
| :--- | :--- | :--- | :--- |
| **SYN-001** | ScalarDB Cluster HA構成未定義 | 3ノードHA構成 + Patroni + サーキットブレーカ採用 | `disaster-recovery.md` |
| **SYN-002** | Outboxがオプションで配信保証なし | Phase 2よりOutbox必須化、ポーリングPublisher導入 | `scalardb-schema.md` |
| **SYN-003** | 外部決済SaaS連携時の補償破綻 | 副作用境界の概念導入、明示的ハイブリッドSaga設計 | `saga-design.md` |
| **SYN-004** | DashboardでGROUP BYが使用不可 | CQRSリードモデルの導入、分析用独立DBとProjector定義 | `read-model-design.md` |

---

### 修正（v2）後の再レビュー

P0ブロッカーを解消した状態（v2設計）で再度、統合レビューコマンド `/architect:review-synthesizer` を実行しました。結果、判定は **⚠️ CONDITIONAL_PASS（条件付き合格）** へと劇的に改善され、品質ゲート通過が見える段階に到達しました。

✅ P0 ブロッカー対応後のレビューサマリ (v2)

| 評価 | スコア |
| :--- | :--- |
| 総合判定 | ⚠️ CONDITIONAL_PASS |
| 総合評価スコア | 3.65 / 5.0 (v1: 2.625) |
| CRITICAL (P0) | 0 件 (v1: 4 件) |
| Risk スコア | 4.0 / 5.0 (v1: 1.5) |

#### 設計完了

ここからさらに、本番適用承認となるPASSへと引き上げるため、AIエージェントのガイドに従って残る12件の P1（HIGH）課題（可観測性のSLI/SLOアラート、シークレット管理、Argo CDを用いたGitOpsパイプライン、3年間のインフラTCOコスト見積もり等）の追加設計を完了させました。

この設計の継続的改善を経て、最終レビューでは総合スコア **4.275 / 5.0** というスコアになり、主要な課題がすべて解消されたためここでは完了としました。

🎉 設計・評価ロードマップの最終推移 (v1 → v3)

| 段階 | 判定 | aggregate | CRITICAL | HIGH | 主な作業 |
| :--- | :--- | :--- | :--- | :--- | :--- |
| v1 | ❌ FAIL | 2.625 | 4 件 | 15 件 | 初回設計レビュー（SPOF、Outbox、決済、集計の欠陥） |
| v2 | ⚠️ CONDITIONAL_PASS | 3.65 | 0 件 | 8 件 | P0課題解消（DR/Saga/CQRS新規設計、Outbox必須化） |
| v3 | ✅ PASS (完全承認) | 4.275 | 0 件 | 0 件 | P1/P2課題完全解消（可観測性・インフラ・コスト詳細） |

## 本章のまとめ

ここまでのNexus Architect実行では、プラグインの導入から現状分析、評価、再設計、ScalarDB設計、レビュー、再レビューまでを一連の流れとして進めました。最終的には、レガシーPOSモノリスを堅牢な分散トランザクションをそなえた13個のマイクロサービス群へ変革するための設計プロセスを、品質ゲートを通しながら固めることができました。

## 用語解説

### 設計レビュー
生成された設計を別の観点から検査し、矛盾、リスク、抜け漏れを見つける工程です。AIエージェントが作った設計でも、レビューを通すことで信頼性を上げられます。

### `review-synthesizer`
複数のレビュー観点を統合し、全体としての判定や優先度を整理するNexus Architectのレビュー支援です。

### P0
設計を進める前に必ず解消すべき最重要指摘です。本章では、HA構成、Outbox、外部決済、Dashboard集計がP0として扱われました。

### HA（高可用性）構成
一部のサーバーやプロセスが落ちても、システム全体を止めずに動かし続けるための構成です。本章では、ScalarDB Clusterを3ノード化し、障害時にも分散トランザクション基盤を継続できるようにする設計を指しています。

### 単一障害点
そこが壊れるとシステム全体が止まってしまう部品や構成要素です。ScalarDB Clusterが1台構成のままだと、そこが単一障害点になり、全サービスのトランザクションに影響します。

### Resilience4j サーキットブレーカ
外部サービスやDB基盤への呼び出しが失敗し続けたときに、一定期間呼び出しを止めて障害の連鎖を防ぐ仕組みです。本章では、ScalarDB Clusterへの接続障害がアプリケーション全体へ広がらないようにするための部品として登場します。

### RTO（目標復旧時間）
障害が発生してから、サービスをどのくらいの時間内に復旧させるかという目標値です。重要な業務ほど短いRTOを設定し、復旧手順や冗長化構成を設計します。

### RPO（目標復旧時点）
障害発生時に、どの時点までのデータを守るかという目標値です。RPOが短いほど、失ってよいデータ量が少ないことを意味します。

### Outboxパターン
DB更新とイベント送信を別々に行うのではなく、まず同じトランザクション内でOutbox用テーブルへイベントを書き込み、あとからPublisherが送信する設計です。これにより、注文更新は成功したのに監査ログやDashboard向けイベントだけ失われる、といった事故を減らせます。

### 品質ゲート
次の工程へ進んでよいかを判断するための基準です。FAIL、CONDITIONAL_PASS、PASSのような判定で設計の成熟度を確認します。
