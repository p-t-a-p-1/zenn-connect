---
title: "Spring BootとPostgreSQLの実装"
---

:::message
本番バックエンドは、Java 21、Spring Boot 3.3、PostgreSQL 16を使い、境界コンテキストごとに段階的に実装しました。
:::

## Dockerでビルド環境をそろえる

![Dockerでビルド環境をそろえるを補助する白黒線画](/images/nexus-product-new-development/section/section-19-docker-build.png)


実装開始時、ホストにはJDKとGradleがありませんでした。

選択肢は、ホストへインストールする、コードだけ作る、Dockerでビルドする、の3つです。ユーザーはDockerを選びました。

現在のリポジトリでも、`gradle:8.11-jdk21`イメージを使う手順が残っています。

```bash
docker run --rm -v "$PWD":/app -w /app \
  -v gradle-cache:/root/.gradle \
  gradle:8.11-jdk21 gradle build -x test --no-daemon
```

ローカル環境を勝手に変更せず、再現可能な実行方法を選んだ例です。

## フェーズ1で実装したもの

![フェーズ1で実装したものを補助する白黒線画](/images/nexus-product-new-development/section/section-19-phase-one.png)


最初に、BC-001、BC-002、BC-003を実装しました。

| BC | Javaパッケージ | 主な実装 |
| :--- | :--- | :--- |
| BC-001 | `dealassignment` | Deal、Assignment、正式化、履歴 |
| BC-002 | `risk` | RiskFlag、RiskResponse、検知 |
| BC-003 | `member` | User、ロール、メンバー情報 |

Controller、Service、Repositoryを分離し、Flywayだけがスキーマを変更する構成にしました。Hibernateは`ddl-auto: validate`で、マイグレーションとの差分があれば起動を失敗させます。

## ドメインルールを集約へ置く

![ドメインルールを集約へ置くを補助する白黒線画](/images/nexus-product-new-development/section/section-19-domain-aggregate.png)


`Assignment`は、単なるJPAのデータ入れ物ではありません。

現在の実装には、次のルールがあります。

- 開始日は月初、終了日は月末へ正規化する
- 終了日がない場合は **未定** などのラベルを必要とする
- BPアサインにはBP単価が必要
- 取消済みアサインは変更できない
- 正式化は二重実行しても状態を壊さない

```java
public void formalize() {
    if (this.type == AssignmentType.正式アサイン) {
        return;
    }
    this.type = AssignmentType.正式アサイン;
    this.formalizedAt = Instant.now();
}
```

業務ルールをControllerやフォームだけへ置かず、集約自身が不正な状態を拒否します。

## テストでレビュー課題を確認する

![テストでレビュー課題を確認するを補助する白黒線画](/images/nexus-product-new-development/section/section-19-tests-review.png)


初期フェーズでは、重複アサイン防止とIdempotency-Keyを統合テストで確認しました。

その後、認証、通知、監査、予実、営業依頼、調達へ実装が広がり、現在は40を超えるバックエンドテストクラスがあります。

テスト数そのものより、レビューで見つけたリスクがテストケースへ変わったことが重要です。

- 同じIdempotency-Keyなら同じ結果を返す
- 権限のないロールは操作できない
- SALESは他人の商談を取得できない
- 期間重複をDBが拒否する
- 重要な業務操作で監査ログが作られる

Dockerで再現可能なビルド環境を作り、業務ルールとレビュー指摘をSpring Bootのコードとテストへ段階的に変換しました。フェーズ1以降、BC-004（通知）、BC-005（認証・権限）、BC-006（予実）、調達コンテキストへと実装が広がります。その詳細は第21章でまとめます。次章では、認証とデータ単位の権限境界をフロントエンドまでつなぎます。
