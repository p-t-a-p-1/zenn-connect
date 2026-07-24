---
title: "5観点レビューでP0ブロッカーを見つける"
---

:::message
本番設計を一通り作ったあと、Consistency、Data Integrity、Operations、Risk、Businessの5観点でレビューしました。
:::

## 5観点と重み

RADARのレビューは、次の5つです。

| 観点 | 重み | 主に確認すること |
| :--- | :---: | :--- |
| Consistency | 0.15 | 文書、API、用語、図の整合性 |
| Data Integrity | 0.25 | トランザクション、制約、同時実行 |
| Operations | 0.20 | 監視、デプロイ、復旧、運用手順 |
| Risk | 0.25 | セキュリティ、障害モード、依存リスク |
| Business | 0.15 | 価値、継続性、計画、要求との対応 |

初回の集約スコアは2.67で、判定はFAILでした。Criticalは7件あり、すべて次工程へ進む前に解消すべきP0 Blockerとして扱われました。

## 初回レビューで不足していたもの

機能、ドメイン、APIの設計は進んでいました。一方、本番運用に必要な基盤設計が体系的に不足していました。

| P0 | 不足していた設計 |
| :--- | :--- |
| SYN-001 | ヘルスチェック |
| SYN-002 | アラート閾値とrunbook |
| SYN-003 | バックアップとRPO/RTO |
| SYN-004 | Multi-AZとフェイルオーバー |
| SYN-005 | シークレット管理 |
| SYN-006 | デプロイ戦略 |
| SYN-007 | OIDC障害時の挙動 |

MVPで価値を確認したあとでも、本番設計へ移るときにはこの空白を埋める必要があります。

## 4つの設計文書でP0を解消する

P0 Blockerに対応するため、次の文書を追加しました。

- `security-design.md`
- `observability-design.md`
- `disaster-recovery-design.md`
- `infrastructure-architecture.md`

一つの文書が一つのP0に対応するのではありません。たとえばobservability設計が、ヘルスチェック、メトリクス、アラート、runbook参照をまとめて扱います。

## レビュー中に実バグを見つける

レビューは文章の採点だけではありません。実際に検証スクリプトを動かし、2件のバグを見つけました。

1. `auth-api.yaml`末尾へMarkdownが混入し、YAMLとしてパースできなかった
2. データ設計のコード例が、存在しない`DealStatus`列挙値を参照していた

どちらもレビューと同時に修正しています。

## 再レビューの結果

P0対応後、同じ5観点で再レビューしました。

| 指標 | 初回 | 2回目 |
| :--- | :---: | :---: |
| aggregate | 2.67 | 3.62 |
| critical | 7 | 0 |
| major | 26 | 18 |
| 判定 | FAIL | FAIL |

全観点が最低3.0を超え、集約スコアもPASS基準3.5へ到達しました。しかし、CONDITIONAL_PASSの条件であるmajor 8件以下を満たさず、判定はFAILのままでした。

合否だけを見ると変化がないように見えますが、Criticalが0になり、本番運用基盤の空白が埋まったことは大きな前進です。

## 本章のまとめ

- レビューはConsistency、Data Integrity、Operations、Risk、Businessの5観点です。
- 初回はaggregate 2.67、Critical 7件でFAILでした。
- 4つの本番運用設計文書を追加し、P0 Blockerをすべて解消しました。
- 再レビューは3.62、Critical 0件まで改善しましたが、major件数によりFAILが続きました。
