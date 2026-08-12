---
title: "データ・API・正式化トランザクション"
---

:::message
RADARの設計で最も重要な技術的論点は、仮アサインの正式化を、再送や同時実行が起きても正しく完了させることでした。
:::

## 正式化APIを業務操作として表す

正式化は、Assignmentの種別を1件ずつ更新するCRUDではありません。

```http
POST /v1/deals/{dealId}/formalize
Idempotency-Key: <UUID>
```

このAPIは、商談を案件へ移し、紐づく仮アサインをまとめて正式化します。

業務上の意味を持つ操作としてAPIを定義することで、認可、監査、冪等性、トランザクションの単位を合わせられます。

## トランザクション内で行ロックを取る

現在の`AssignmentFormalizationService`は、最初にDealを`SELECT ... FOR UPDATE`相当で取得します。

```java
@Transactional
public FormalizationResult formalizeDealAssignments(
        UUID dealId, UUID actorUserId, String idempotencyKey) {
    var cached = idempotencyStore.find(idempotencyKey);
    if (cached.isPresent()) return cached.get();

    Deal deal = dealRepository.findByIdForUpdate(dealId)
            .orElseThrow(() -> new DealNotFoundException(dealId));

    // 事前条件をロック取得後に確認する
    // Deal更新、Assignment正式化、履歴、監査を同一トランザクションで実行
}
```

事前条件チェックをロックの外で行うと、確認後から更新までの間に別リクエストが状態を変えるTOCTOU競合が起きます。設計レビューでこの問題が指摘され、ロック取得後に確認する形へ修正しました。

## Idempotency-Keyを画面から保持する

バックエンドがIdempotency-Keyに対応していても、画面がクリックごとに新しいキーを作ると再送を検知できません。

RADARの`FormalizeAction`では、コンポーネントのマウント時に一度だけ生成します。

```tsx
const [idempotencyKey] = useState(() => crypto.randomUUID());
```

失敗後に再送しても同じキーを使います。フロントエンドとバックエンドを一つの冪等性設計として扱っています。

## PostgreSQLを最後の防波堤にする

正式アサインの期間重複は、アプリケーションの事前チェックだけでは完全に防げません。並行リクエストが同じ空き期間を確認し、同時に保存する可能性があるためです。

そこでPostgreSQLの`EXCLUDE USING gist`を利用します。

```sql
EXCLUDE USING gist (
  user_id WITH =,
  daterange(start_date, end_date, '[]') WITH &&
)
```

現在のマイグレーションでは、業務変更に合わせて対象を **正式かつ終了日があり、準委任同士** に絞っています。

## flushで例外の発生位置を制御する

JPAはSQL発行をトランザクション終端まで遅らせることがあります。その場合、`try-catch`の外で制約違反が起き、業務エラーへ変換できません。

RADARでは`saveAllAndFlush()`を使い、排他制約違反を捕捉したいブロック内で確実に発生させます。これも5観点レビューから実装へ反映された修正です。

正式化を専用の業務APIとして扱い、行ロック、Idempotency-Key、PostgreSQL排他制約、明示的flushを組み合わせました。次章では、本番運用へ進める設計かをレビューします。
