---
title: "Next.js・OIDC・権限境界"
---

:::message
バックエンドの中核実装後、Google Workspace OIDC、PermissionAggregate、行レベルアクセス制御、Next.jsフロントエンドを追加しました。
:::

## 認証を後付けにしない

![認証を後付けにしないを補助する白黒線画](/images/nexus-product-new-development/section/section-20-auth-first.png)


フェーズ1の次は、AWS基盤よりBC-005認証を優先しました。

業務アプリでは、認証だけでなく **誰がどの操作とデータへアクセスできるか** がAPI設計へ影響します。全員が全件を見られる状態で画面を作り切ると、後から取得条件とテストを広範囲に直すことになります。

## PermissionAggregateで操作権限を表す

![PermissionAggregateで操作権限を表すを補助する白黒線画](/images/nexus-product-new-development/section/section-20-permission-aggregate.png)


RADARは、ロールと操作の対応を`PermissionAggregate`へ集約しています。

```java
MATRIX.put(Role.PM_LEADER, EnumSet.of(
    Operation.DEAL_VIEW,
    Operation.ASSIGNMENT_CREATE_TENTATIVE,
    Operation.ASSIGNMENT_FORMALIZE,
    Operation.RISK_RECORD_RESPONSE,
    Operation.BUDGET_TARGET_MANAGE
));
```

このマトリクスは、Nexus Architectが作った`actors-roles-permissions.md`をコード化したものです。

## 操作権限と行レベル制御を分ける

![操作権限と行レベル制御を分けるを補助する白黒線画](/images/nexus-product-new-development/section/section-20-operation-row-control.png)


**SALESは商談を閲覧できる** という権限だけでは不十分です。営業は、自分が担当する商談または営業依頼で紐づいた商談だけを見られる必要があります。

RADARでは、Controllerの`@PreAuthorize`に加え、Serviceで所有者条件を適用します。

実装後の追加開発では、SALESが`ownerUserId`を省略すると全商談を見られる行レベル制御の欠落も発見されました。現在は、SALESから渡された検索条件にかかわらず、バックエンド側で自分のIDへ上書きします。

フロントエンドでボタンを隠すことはUX上の補助です。実際の認可境界はバックエンドです。

## CookieセッションをSSRで中継する

![CookieセッションをSSRで中継するを補助する白黒線画](/images/nexus-product-new-development/section/section-20-cookie-ssr.png)


Next.jsはApp Routerを使い、読み取りをServer Components、更新をServer Actionsへ寄せています。

SSRのfetchはブラウザを経由しないため、受け取ったCookieをバックエンドへ明示的に中継します。

```ts
const response = await fetch(`${BACKEND_URL}${path}`, {
  ...init,
  headers: {
    ...init?.headers,
    Cookie: cookieStore.toString(),
  },
  cache: "no-store",
});
```

`middleware.ts`は`JSESSIONID`の存在だけを高速確認し、`app/layout.tsx`の`requireSession()`がバックエンドへ問い合わせて実際のセッションを検証します。

## OIDCの契約不一致を実ログインで見つける

![OIDCの契約不一致を実ログインで見つけるを補助する白黒線画](/images/nexus-product-new-development/section/section-20-oidc-login.png)


当初、Google Cloud Consoleでのクライアント登録は開発環境外の作業として手順書へ切り出しました。

その後、実クライアントを登録してログインを試したところ、Google側の認証後に`/auth/callback`で401になりました。

原因は、設定したredirect URIとSpring Securityが実際に待ち受けるコールバックパスの不一致です。

```java
.redirectionEndpoint(r -> r.baseUri("/auth/callback"))
```

この指定を追加して解消し、Google Workspaceアカウントによるログイン成立まで確認しました。

設計書と設定ファイルが一致して見えても、外部サービスとの契約はE2Eで初めて確認できることがあります。

認証、操作権限、行レベル制御をAPIへ組み込み、Next.jsのSSRと実ログインで外部サービスとの契約まで確認しました。次章では、実装後に通知・監査と実利用から設計を更新します。
