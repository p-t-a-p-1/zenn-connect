---
title: "Nexus Architectで調査と分析を実行する"
---

:::message
Claude Code上でNexus Architectの調査・分析パイプラインを起動し、生成される初期レポートまでの流れを追います。
:::

## 解析コマンドの実行手順とインタラクティブな対話

プラグインが準備できたら、Claude Code のチャットセッションをレガシーPOSモノリスのディレクトリで開始し、以下のコマンドを入力して自動解析パイプラインを起動します。

```bash
/architect:start
```

コマンドを入力すると、AIエージェントは対話を開始し、アウトプット言語と言律設定（ScalarDBを適用するかどうか）について質問を投げかけてきます。

```text
I have a clear picture of the project. This is a legacy refactoring path — an existing Java monolith POS system.

  Before I initialize the pipeline, two quick questions:

  1. Output language?
  - English (default)
  - Japanese

  2. ScalarDB?
  The codebase uses PostgreSQL + MySQL with hand-written Saga coordination — no ScalarDB. Should I:
  - A) Keep no ScalarDB — use design-data-layer path (matches the current architecture)
  - B) Include ScalarDB — design the target architecture with ScalarDB handling cross-DB distributed transactions (the Saga pain
  points would be eliminated)

  Option B is the more interesting refactoring target given the cross-DB Saga complexity in CheckoutSaga and ReturnSaga. Which do
  you prefer?
```

ここでは、日本語での出力と、今回のリファクタリングの主眼である **Option B（ScalarDBを採用してクロスDB分散トランザクションのSaga複雑性を解消する設計）** を選択して進めます。

対話を終えると、AIエージェントはまず `/architect:init-output` を呼び出してパイプラインを初期化します。

```text
まず /architect:init-output でパイプラインを初期化します。
⏺ 初期化完了しました。次は /architect:investigate でコードベース調査を開始します。
```

### 1. 調査フェーズ（/architect:investigate）

初期化が完了すると、自動的に調査（investigate）フェーズへと進みます。
このフェーズでは、プロジェクトのコードベース全体の静的解析と技術的負債の洗い出しが行われます。

```text
⏺ investigate フェーズが完了しました。

  ---
  調査サマリ

  4つのレポートを生成しました：
```

| レポート | 主要発見 |
| :--- | :--- |
| テクノロジースタック | Spring Boot 2.7.18 (EOL), Java 11 (EOL), DriverManagerDataSource（接続プールなし） |
| コードベース構造 | 56 Java ファイル、OrderService（976行）God Service、CheckoutSaga（557行）God Class、dao/+repository/ パッケージ命名揺れ |
| 技術的負債 | CRITICAL 4件・High 8件・Medium 13件・Low 5件、計30件。最重大は DB 跨ぎ原子性未保証 |
| DDD 準備度 | 総合 29/100。潜在的 BC（8個）は識別可能だが God Service/Saga が境界を崩壊。ScalarDB で手書き Saga を置き換えると TD-002/003/009 が根本解決 |

### 2. 分析フェーズ（/architect:analyze）

続いて、`/architect:analyze` を実行し、システム全体の機能概要、ユビキタス言語、アクター/ロール、およびコードとドメインの対応関係のドキュメント化を行います。

```text
⏺ analyze フェーズが完了しました。

  ---
  analyze 完了サマリ

  4つのドキュメントを生成しました：
```

| ドキュメント | 主要発見 |
| :--- | :--- |
| system-overview | 10 機能カテゴリ、主要フロー: CheckoutSaga（6ステップ） / ReturnSaga（6ステップ）。決済は System.out.println モック |
| ubiquitous-language | 60+ ドメイン用語。不統一: dao/repository 並存、ポイント type 文字列の混乱、User vs Member の概念分離 |
| actors-roles-permissions | 3 ロール（CASHIER/MANAGER/ADMIN）+ Member。SecurityConfig の URL パターン認可のみ。メソッドレベル認可なし |
| domain-code-mapping | 10 BC 候補、税計算 4 重・ポイント計算 4 重・返品可否 3 重、PaymentService/ReturnService 欠如、DB 境界が BC 境界と不一致 |

#### レポート出力例

##### システム概要 - system-overview.md

![system-overview-01](/images/legacy-refactoring-nexus-scalardb/system-overview-01.png)
*ビジネスコンテキストや主要機能*

![system-overview-02](/images/legacy-refactoring-nexus-scalardb/system-overview-02.png)
*ビジネス制約や主要なビジネスフロー*

##### ユビキタス言語辞書 - ubiquitous-language.md

![ubiquitous-language-01](/images/legacy-refactoring-nexus-scalardb/ubiquitous-language-01.png)
*ドメイン用語辞書*

![ubiquitous-language-02](/images/legacy-refactoring-nexus-scalardb/ubiquitous-language-02.png)
*用語の不統一・重複レビュー結果*

##### アクター・ロール・パーミッション - actors-roles-permissions.md

![actors-roles-permissions-01](/images/legacy-refactoring-nexus-scalardb/actors-roles-permissions-01.png)
*アクターやロールの定義状況*

![actors-roles-permissions-02](/images/legacy-refactoring-nexus-scalardb/actors-roles-permissions-02.png)
*認証認可の実装分析*

##### ドメインコードマッピング - domain-code-mapping.md

![domain-code-mapping-01](/images/legacy-refactoring-nexus-scalardb/domain-code-mapping-01.png)
*ドメイン概念とコード対応のマトリクス表*

![domain-code-mapping-02](/images/legacy-refactoring-nexus-scalardb/domain-code-mapping-02.png)
*ビジネスルールの実装箇所トレース一覧*

![domain-code-mapping-03](/images/legacy-refactoring-nexus-scalardb/domain-code-mapping-03.png)
*ドメイン概念とコードのギャップレビュー結果*


---

この分析結果を受けて、AIエージェントは次のステップとしてモジュラリティ成熟度（MMI）とDDD適合度（DDD）の並列評価、およびドメイン分類・BCマッピングなどのステップを提案します。

## 本章のまとめ

* `/architect:start`を起点に、Nexus Architectの調査・分析パイプラインを開始しました。
* 実行前の対話では、出力言語とScalarDBを設計対象に含めるかを選択し、今回の方向性を決めています。
* 調査フェーズと分析フェーズにより、コード構造、技術的負債、ドメイン概念、権限、用語のズレがレポートとして可視化されました。

## 用語解説

### 調査フェーズ
コードベース、設定ファイル、DB構成、主要クラスなどを静的に読み取り、現状を把握するフェーズです。後続の分析や評価の前提情報になります。

### 分析フェーズ
調査結果をもとに、業務領域、ドメイン概念、権限、ビジネスルール、コードとの対応関係を整理するフェーズです。

### 静的解析
アプリケーションを実行せず、ソースコードや設定ファイルを読んで構造や問題点を調べる方法です。レガシーシステムの全体像をつかむ入口になります。

### ユビキタス言語
開発者と業務担当者が同じ意味で使う共通語彙です。用語の揺れを減らすことで、設計と実装のズレを小さくできます。
