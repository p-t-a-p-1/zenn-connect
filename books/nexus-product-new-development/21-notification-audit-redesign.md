---
title: "通知・監査・実利用からの再設計"
---

:::message
本番実装は、初期設計をコードへ移すだけでは終わりません。通知・監査を追加し、実利用側の業務フローと照合しながらRADAR自体を更新しました。
:::

## コミット後に通知する

![コミット後に通知するを補助する白黒線画](/images/nexus-product-new-development/section/section-21-commit-notify.png)


通知は、遅延リスクなどの業務イベントを受けて作成します。

RADARでは、`@TransactionalEventListener`を`AFTER_COMMIT`で実行します。

```java
@Async("notificationExecutor")
@TransactionalEventListener(phase = TransactionPhase.AFTER_COMMIT)
public void onRiskFlagDetected(RiskFlagDetectedEvent event) {
    // 通知作成
}
```

業務トランザクションがロールバックしたのに通知だけ送られる状態を避けるためです。

通知には`dedup_key`のユニーク制約があり、同じイベントの重複を防ぎます。ドメインモデルには最大3回の再試行回数と`failed`状態があります。専用のbounded thread poolも用意し、通知処理がHTTPリクエスト処理を圧迫しないようにしています。

ただし、現在の配信先はログ出力のプレースホルダーです。失敗時に`retryCount`を増やす処理はありますが、次の配信試行を起動するスケジューラやキューはまだありません。メールやGoogle Chatなど、実チャネルへの接続と合わせて残っている実装課題です。

## 監査ログを同じトランザクションで記録する

![監査ログを同じトランザクションで記録するを補助する白黒線画](/images/nexus-product-new-development/section/section-21-audit-transaction.png)


正式化、仮アサイン、リスク対応では、`AuditService`をコア業務と同じトランザクションから呼びます。

監査ログには、操作者、操作種別、対象リソース、時刻、変更前後の値を記録します。

現在の方針では、監査ログの書き込みに失敗した場合、業務操作もロールバックします。一方、DBユーザーからUPDATE/DELETE権限を剥奪するINSERT-only運用は、RDS構築時の残課題です。

## 初期の背骨を見直す

![初期の背骨を見直すを補助する白黒線画](/images/nexus-product-new-development/section/section-21-revisit-backbone.png)


初期のRADARは、仮アサインをプロダクトの中心に置きました。BC-001〜003の実装後、認証（BC-005）と通知（BC-004）を追加しながら実利用を進めるなかで、業務フロー全体と照合する機会がありました。

その結果、組織が最初に見るのは **グループ別・月別の予算と売上の差** であり、不足を営業、アサイン、調達で埋める流れだと分かりました。

```text
予算と売上実績
  ↓ 不足を把握
営業依頼
  ↓ 商談を作る
アサイン計画
  ↓ 社内で不足
外部人材を調達
```

このギャップを受け、RADARは次を追加しました。

- OrgGroupによる組織構造
- RevenueLineによる月別売上の事実源
- SalesRequestによるリーダーから営業への依頼
- PROCUREMENTロール
- 外部人材の調達コンテキスト

## 設計履歴と実コードの差を管理する

![設計履歴と実コードの差を管理するを補助する白黒線画](/images/nexus-product-new-development/section/section-21-history-code-gap.png)


`docs/design-history/`は初期パイプラインの履歴として凍結しています。後から機能を追加するたびに過去文書を完成形へ書き換えると、当時何を判断したか分からなくなります。

そこで、現在のコードと`PRODUCT.md`を生きた状態として扱い、初期設計との差分は`MEMORY.md`と実装計画へ追記します。

AI駆動開発では、生成物を常に最新へ上書きするより、次の3種類を区別する方が追跡しやすくなります。

| 種類 | RADARでの場所 |
| :--- | :--- |
| 現在動くコード | `backend/`、`frontend/` |
| 現在の原則 | `PRODUCT.md`、`DESIGN.md`、`CLAUDE.md` |
| 過去の判断 | `docs/design-history/`、`docs/plans/` |

通知をコミット後に処理し、監査を重要業務と同じトランザクションで記録しました。同時に、未接続の実チャネルなどの残課題を明示し、実利用の学びからRADARの中心を予実・営業依頼・アサイン・調達へ広げました。次章で、この開発ループを再利用できる型としてまとめます。
