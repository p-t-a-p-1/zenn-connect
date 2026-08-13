---
title: "ProductからArchitectへ引き継ぐ"
---

:::message
MVP検証で価値仮説を確認したあと、Productパイプラインの成果物を要求ベースラインとしてArchitectパイプラインへ引き継ぎました。
:::

## 本番化はMVPの延長ではない

コンシェルジュMVPには、実際に使われた業務フローがあります。一方、本番運用に必要な認証、権限、監査、障害復旧、運用監視は備えていません。

そのため、MVPコードをそのまま整えて本番化するのではなく、要求定義から再開しました。

```text
Productパイプライン
  ├─ Vision・成功指標
  ├─ スコープ・仮説
  ├─ ペルソナ・UI
  └─ MVP検証結果
           ↓
Architectパイプライン
  ├─ 本番要求
  ├─ MVPコード調査
  ├─ MMI/DDD評価
  ├─ 境界コンテキスト再設計
  └─ データ・API・運用設計
```

## 引き継いだ要求ベースライン

`reports/00_core`から`04_quality`までの成果物を、Architect側の入力として扱いました。

特に重要な入力は次のとおりです。

- ASM-001/002を通過したこと
- 仮アサインが中核価値であること
- データ移行を行わないこと
- 確定と仮を混同しないこと
- PM/リーダーと営業の業務ストーリー
- 本番で必要な認証、監査、通知、非機能要件

この引き継ぎにより、Architectパイプラインは一般的な業務アプリを設計するのではなく、検証済みのRADAR固有の価値を守る設計へ進めます。

実際の設計は、次の流れで進みました。

```text
architect:define-requirements
  → architect:investigate
  → architect:analyze
  → architect:evaluate-mmi / evaluate-ddd
  → architect:integrate-evaluations
  → architect:redesign
  → architect:design-microservices
  → architect:design-data-layer
  → architect:design-api
  → architect:review-*
```

コード調査と評価を挟んでから再設計するため、最初から理想アーキテクチャを当てはめる流れにはなっていません。

## 本番技術スタックを決める

要求定義で、次の技術スタックを選びました。

| 用語 | 正式名称 | 説明 |
| :--- | :--- | :--- |
| OIDC | OpenID Connect | OAuth 2.0を拡張した認証プロトコル。Googleなどのアイデンティティプロバイダーと連携してログインを実現する |

| 領域 | 技術 |
| :--- | :--- |
| Frontend | React / Next.js |
| Backend | Java 21 / Spring Boot |
| Database | PostgreSQL |
| Infrastructure | AWS |
| Authentication | Google Workspace OIDC |

MVPのHonoとSQLiteを採用し続けなかった理由は、MVPの失敗ではありません。学習用の実装と、継続運用する実装では最適化する対象が違うためです。

## モジュラーモノリスを選ぶ

設計パイプラインでは、マイクロサービス化の必要性も評価しました。

RADARの初期規模では、サービスごとのデプロイ、分散トランザクション、独立運用を持ち込む効果より、複雑さの方が大きいと判断しました。

そこで、単一のSpring Bootアプリケーション内で境界コンテキストをパッケージ分離する、モジュラーモノリスを採用しました。

同じ理由で、Kubernetesも初期構成では過剰投資と判断し、AWS ECS/Fargateを選びました。

## ScalarDBを採用しなかった理由

Nexus ArchitectにはScalarDB適用性を評価する流れもあります。しかしRADARは、単一PostgreSQL内のローカルトランザクションで中核処理を完結できます。

仮アサイン正式化は複数集約をまたぎますが、単一アプリ・単一DBの範囲です。分散トランザクション基盤を追加する必要はないと判断しました。

ツールが候補として扱える技術を、必ず採用するわけではありません。要求と運用規模に合わなければ採用しないことも設計成果です。

Productの成果物を要求ベースラインとしてArchitectへ引き継ぎ、MVPの延長ではなく、初期規模に合う本番構成を選びました。次章では、MVPコードを調査し、再設計の入力を作ります。
