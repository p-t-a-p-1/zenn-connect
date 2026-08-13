---
title: "Issue単位で実装・レビュー・マージする"
---

Issueを作っただけでは、開発は完了しません。Issueを実装単位として、共有情報を読み、コードを変更し、レビューを通し、人の承認を得てマージします。

## Epic単位とIssue単位を使い分ける

Epic配下のIssueを順番に進める場合は、上位のオーケストレーターを使います。

```text
/architect:deliver-backlog --epic=E1
```

`deliver-backlog`は、Issueごとに実装、レビュー、マージを順番に進めます。Pull Requestを作った時点で人の承認を待ち、承認後にマージして次のIssueへ移ります。実行内容だけ確認する場合は、次のように`--dry-run`を指定します。

```text
/architect:deliver-backlog --epic=E1 --dry-run
```

個別のIssueだけを進める場合は、次のワークフローを直接実行します。

```text
/architect:implement-backlog
```

`status::doing`のIssueを優先し、なければ`status::todo`から次のIssueを選びます。途中で処理を止めても、Issueと`backlog-manifest.json`に残った状態から再開できます。

![Issueの実装、レビュー、Pull Request、マージ、知識の引き継ぎを繰り返す流れ](/images/nexus-architect-ai-devops-loop/implementation-review-cycle.png)
*Issueごとに実装、レビュー、マージを進め、得た知識を次のIssueへ引き継ぐ*

## 実装前に共有情報を読む

Issueごとにすべての設計背景を会話で説明し直すと、前提が揺れます。実装前に`reports/backlog/shared-context/`へ共有情報を置き、AIエージェントが毎回読む構成にします。

| 用語 | 正式名称 | 説明 |
| :--- | :--- | :--- |
| SLA | Service Level Agreement（サービス品質合意） | 可用性・応答時間などの品質目標を定めた合意文書 |
| NFR | Non-Functional Requirements（非機能要件） | 性能・可用性・セキュリティなど機能以外の品質特性に関する要件 |

| ファイル | 内容 |
| :--- | :--- |
| `architecture-guardrails.md` | パッケージ境界と技術選定の制約 |
| `coding-standards.md` | 命名、例外処理、テスト方針 |
| `data-contracts.md` | テーブルとデータ変換の規則 |
| `nfr-budgets.md` | SLAと非機能要件の目標 |
| `decisions.md` | 実装中に決めたこと |
| `review-knowledge.md` | レビューで得た教訓 |

この共有情報は、Issueの本文よりも広い範囲の制約を持ちます。1つのIssueだけでは見えない境界や、前のIssueで見つかった失敗条件を次の実装へ渡すためです。

代表的なルールは、次のように具体的に記録します。

```text
DealとDealStatusHistoryは、1つのトランザクションで保存する。
外部から受け取ったownerIdを、そのまま所有者条件に使わない。
多対多の中間テーブルには、外部キーの組へ一意性制約を置く。
```

## Issueの完了条件を受入基準にする

Issueには、実装内容、検証可能な受入基準、参照した設計書を記載します。

```markdown
## How
`POST /deals`を実装し、案件を新規作成する。

## Acceptance Criteria
- [ ] 案件名と担当者を指定すると201が返る
- [ ] 存在しない担当者を指定した場合は作成を拒否する
- [ ] 初期ステータスを履歴へ記録する

## References
reports/02_spec/feature-list.md
reports/03_domain/api-design.md

size: M
```

受入基準は、AIへ渡すプロンプトではなく、実装とレビューの共通の完了条件です。設計書を参照元として残すことで、実装の都合で業務要件が変わっていないかも確認できます。

## 実装完了とマージ完了を分ける

実装が終わると、Issueは`status::review`へ進みます。コードを書いた時点では`status::done`にしません。ブランチ名にもIssue IDを含めると、コード、Pull Request、Issueを追跡できます。

```text
feature/<issue-id>-<slug>
feature/I1.1.1-create-deal
```

未マージの変更は、プロダクトへ反映された機能ではありません。この状態を分けることで、AIに実装を任せつつ、リリース済みの範囲を人が正しく把握できます。

## レビューは差分の外側も読む

レビューは次のワークフローで始めます。

```text
/architect:review-issue
```

`review-issue`は変更差分だけでなく、親Epic、Sub-Epic、兄弟Issue、共有情報も確認します。指摘は、修正必須、要確認、質問に分けて扱います。

```text
[B] 修正必須
[S] 要確認
[Q] 質問
```

修正必須の指摘が残っている場合は、修正して再レビューします。問題が解消するとPull Requestを作成し、Issueをレビュー中の状態へ進めます。

![review-issue実行後に作成されたPull Requestとレビュー結果](/images/nexus-architect-ai-devops-loop/github-pull-request.png)
*レビュー完了後に作成されたPull Request。変更内容、関連Issue、テスト結果を確認する*

レビューで得た知識は`review-knowledge.md`へ残します。たとえば、クライアントが渡した`ownerId`を信用しない、多対多の中間テーブルに一意性制約を置く、といったルールです。次のIssueでは、この知識を実装前提として読み込みます。

## マージ後に親の進捗を更新する

Pull Requestがopenであること、修正必須の指摘がないこと、CIが成功していること、コンフリクトがないことを確認してから、次のワークフローでマージします。

```text
/architect:merge-issue
```

マージ後は、次の状態を順番に更新します。

1. Issueをcloseして`status::done`にする
2. Sub-Epicのタスクリストを更新する
3. Sub-Epicが完了したらEpicのタスクリストを更新する
4. `backlog-manifest.json`を`done`へ更新する

Sub-Epicが完了した時点でEpic全体を見直します。個別Issueでは分からなかった非対称な実装や統合テストの不足があれば、新しいIssueとして次のバックログへ戻します。

AIエージェントへ任せる範囲が広がっても、バックログの承認、Pull Requestの確認、マージの判断は人が持ち続けます。これでIssueは単なるタスクではなく、設計と運用をつなぐ境界になります。
