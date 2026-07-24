---
title: "成果物を支えるリポジトリ構成"
---

:::message
RADARでは、現在動くプロダクトと、Nexus Architectが生成した設計履歴を分けて保存しています。この構成そのものが、AI駆動開発の重要な成果です。
:::

## 現在動く2つのプロジェクト

本番実装は、`backend/`と`frontend/`に分かれています。

| ディレクトリ | 技術 | 役割 |
| :--- | :--- | :--- |
| `backend/` | Java 21 / Spring Boot 3.3 / PostgreSQL | ドメイン、認証、権限、永続化、監査 |
| `frontend/` | Next.js App Router / React / TypeScript | SSR画面、Server Actions、業務操作 |

バックエンドは、単一のSpring Bootアプリケーションとしてデプロイするモジュラーモノリスです。境界コンテキストごとにJavaパッケージとPostgreSQLスキーマを分けています。

```text
com.radar.app
├── dealassignment   BC-001 商談・アサイン
├── risk             BC-002 遅延リスク
├── member           BC-003 メンバー
├── notification     BC-004 通知
├── auth             BC-005 認証・権限
├── budget           BC-006 予実
├── procurement      調達
└── audit            横断的な監査ログ
```

現在の成果物には、バックエンドのJavaソース約190ファイル、テストクラス40件以上、15個のNext.jsページ、20本以上のFlywayマイグレーションがあります。

## 読み取りはServer Components、更新はServer Actions

フロントエンドは、ブラウザからバックエンドへ直接アクセスする構成ではありません。

Server Componentsからの読み取りは`apiFetch()`へ集約し、ブラウザから届いたCookieをバックエンドへ中継します。すべてのデータがセッションに依存するため、取得には`cache: 'no-store'`を指定しています。

更新操作はServer Actionsを経由します。クライアント側に汎用fetchラッパーを作らず、認証Cookieと業務操作の境界をサーバー側へ寄せています。

## デザインシステムを単一ソースにする

`design-system/default/`には、次の成果物があります。

- `tokens.css` / `tokens.json`
- コンポーネント一覧
- UIガイドライン
- プレビュー

現在のフロントエンドはTailwind CSSとshadcn/uiを利用しますが、色や余白の値をコンポーネントへ直接書くのではなく、デザイントークンを単一ソースとして参照します。

特に重要な原則が、確定と仮の状態を色だけで表現しないことです。

```text
仮アサイン: 破線 + 仮ラベル + 状態色
正式アサイン: 実線 + 正式ラベル + 状態色
```

## 設計履歴をアーカイブする

`docs/design-history/`には、ProductパイプラインとArchitectパイプラインが生成した成果物が残っています。

```text
reports/
├── 00_core           Vision、成功指標、スコープ、仮説
├── 01_ux             ペルソナ、ジャーニー、ドメインストーリー
├── 02_spec           機能一覧、データモデル、UIモック
├── 03_domain         初期ドメイン設計
├── 03_design         本番アーキテクチャ、API、データ設計
├── 04_quality        NFR、SLA
├── 08_infrastructure セキュリティ、監視、DR、AWS
└── review            5観点レビュー
```

`work/traceability.json`は、Vision、仮説、スコープ、機能、要求、API、境界コンテキストのIDをつなぎます。AIが生成した文書をフォルダへ置くだけで終わらず、後から設計判断をたどれる状態にしています。

## MVPは削除せず凍結する

Hono、SQLite、React、Viteで作ったコンシェルジュMVPは、`docs/concierge-mvp-archive/`へ移動しました。

MVPは本番コードの土台ではありません。しかし、何を検証し、どの画面が使われ、どこに技術的負債があったかを示す証拠です。そのため、現在の実装と混ぜず、削除もせず、読み取り専用の履歴として残しています。

## 本章のまとめ

- 現在動くプロダクトはSpring BootバックエンドとNext.jsフロントエンドです。
- バックエンドは境界コンテキストごとに分けたモジュラーモノリスです。
- デザインシステムはトークンを単一ソースとして管理します。
- Nexus Architectの成果物とMVPを`docs/`へアーカイブし、意思決定の履歴を追跡できます。
