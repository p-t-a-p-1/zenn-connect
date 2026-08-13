---
title: "設計をEpic・Sub-Epic・Issueへ分解する"
---

本番設計ができたら、次は実装できる大きさへ分解します。Nexus Architectでは、設計内容をEpic、Sub-Epic、Issueの階層へ変換し、Issueを実装単位として扱います。

ここで紹介するGitHub連携とmanifestの例は、Nexus Architectの一般化された出力例です。RADARの実際のIssue情報は非公開のため、プロダクト名、URL、パッケージ名を匿名化した同等構造の例で説明します。

## バックログを開発の契約にする

バックログは作業一覧ではありません。設計と実装をつなぎ、AIと人が同じ完了条件を確認するための契約です。

| 階層 | 役割 |
| :--- | :--- |
| Epic | プロダクト全体の目的と背景 |
| Sub-Epic | 境界コンテキストや機能領域 |
| Issue | 実装できる大きさの作業 |

Issueには、実装内容、受入基準、参照する設計書、作業規模を記載します。設計IDも残すため、IssueからFeature、API、Entityへ戻れます。

![EpicからSub-EpicとIssueへ分解し、Issueを実装につなげるバックログの構造](/images/nexus-architect-ai-devops-loop/backlog-hierarchy.png)
*EpicをSub-EpicとIssueへ分解し、Issue単位で実装する構造*

## `reports/`をバックログへ変換する

![`reports/`をバックログへ変換するを補助する白黒線画](/images/nexus-product-new-development/section/section-17-reports-to-backlog.png)


Nexus Architectは、企画・設計の成果物を`reports/`へ保存します。代表的な構成は次のとおりです。

```text
reports/
  00_core/       # 構想、開発範囲、成功指標、仮説
  01_ux/         # ペルソナ、ジャーニー、ドメインストーリー
  02_spec/       # UIモック、機能一覧、データモデル
  03_domain/     # 境界コンテキスト、API、アーキテクチャ
  04_quality/    # SLA、非機能要件
  backlog/       # 計画、管理情報、レビュー、実装ログ
```

設計の項目には、種類を表す接頭辞と連番を付けます。

| ID | 意味 |
| :--- | :--- |
| `VIS-001` | Vision。実現したい状態や解決する課題 |
| `FEAT-001` | Feature。ユーザーへ提供する機能 |
| `ENT-001` | Entity。データモデルで扱う業務概念 |
| `API-001` | API。機能を実現する操作やエンドポイント |

たとえば、1つのIssueが`FEAT-001`と`API-001`を参照していれば、実装対象の機能とAPI設計をたどれます。この関係は`work/traceability.json`にも記録し、企画の判断がどの実装へ届いたかを確認できるようにします。

## `export-backlog`は計画を先に作る

![`export-backlog`は計画を先に作るを補助する白黒線画](/images/nexus-product-new-development/section/section-17-export-backlog.png)


設計がそろったら、Claude Codeで次のワークフローを実行します。

```text
/architect:export-backlog
```

この処理は、いきなりIssueを作るのではなく、まず人が確認する計画を作ります。

- `backlog-plan.md`: 登録前に確認するバックログ計画
- `backlog-manifest.json`: 階層、設計参照、リモートURL、実装状態を管理するデータ
- GitHub上のEpic、Sub-Epic、Issue

`backlog-plan.md`には、構想、成功指標、境界コンテキスト、機能一覧、API、データモデル、非機能要件を入力として、次のような階層を記載します。

```text
Epic
├── Sub-Epic: Deal Management
│   ├── Issue: 案件を新規作成する
│   ├── Issue: 担当案件を一覧表示する
│   └── Issue: 進捗ステータスを更新する
├── Sub-Epic: Team Visibility
│   └── Issue: 更新が滞った案件を強調する
└── Sub-Epic: Identity & Access Management
    ├── Issue: Spring Securityを導入する
    └── Issue: オブジェクト単位の認可を追加する
```

目標値が決まっていない指標は`TBD`として残します。仮説検証を通過していても、需要や採算性まで実証されたとは限りません。未確定の情報を埋めず、検証方法が決まっているかを確認してから登録します。

## manifestで設計と実装の対応を残す

![manifestで設計と実装の対応を残すを補助する白黒線画](/images/nexus-product-new-development/section/section-17-manifest-traceability.png)


実際のmanifestは多数のノードを含むため、本書ではEpicとIssueを1件ずつ抜粋します。ほかのSub-Epic、長い本文、全URL、全実装ファイルは省略しています。

:::details backlog-manifest.jsonの代表例

```json
[
  {
    "local_id": "E1",
    "level": "epic",
    "title": "プロダクトMVP",
    "labels": ["type::epic", "status::todo"],
    "parent_local_id": null,
    "source_reports": [
      "reports/00_core/vision-mission-value.md",
      "reports/00_core/scope-definition.md",
      "reports/00_core/success-metrics.md"
    ],
    "traceability": ["VIS-001", "NSM-001"],
    "remote_github": {
      "number": 1,
      "url": "https://github.com/example/project/issues/1"
    },
    "impl": {"status": "done"}
  },
  {
    "local_id": "I1.1.1",
    "level": "issue",
    "title": "案件を新規作成する",
    "labels": [
      "type::issue",
      "domain::deal-management",
      "status::todo"
    ],
    "parent_local_id": "SE1.1",
    "source_reports": [
      "reports/02_spec/feature-list.md",
      "reports/03_domain/api-design.md",
      "reports/02_spec/data-model.md"
    ],
    "traceability": ["FEAT-001", "API-001", "ENT-001"],
    "remote_github": {
      "number": 6,
      "url": "https://github.com/example/project/issues/6"
    },
    "impl": {
      "status": "done",
      "files": ["backend/src/main/java/example/CreateDealService.java"]
    }
  }
]
```

`local_id`と`parent_local_id`が親子関係を表し、`traceability`が設計ID、`remote_github`が登録先、`impl`が実装状態と変更ファイルを表します。全量のmanifestでは、この構造をすべてのEpic、Sub-Epic、Issueへ適用します。

:::

Issueを作成する前に`backlog-plan.md`を人が確認します。再実行時にリモートURLが存在する項目をスキップできるため、同じIssueの重複作成も防げます。

設計をIssueへ変換したことで、次章ではIssueを起点に実装、レビュー、マージを進められます。
