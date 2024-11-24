# 트랜잭션의 ACID 속성에 대한 설명과 중요성

## Atomicity (원자성)
트랜잭션의 모든 연산은 전부 실행되거나 전혀 실행되지 않아야 합니다 (All or Nothing).

**예시**: 계좌 이체 트랜잭션
```sql
BEGIN TRANSACTION;
    UPDATE account SET balance = balance - 1000 WHERE user_id = 'sender';
    UPDATE account SET balance = balance + 1000 WHERE user_id = 'receiver';
COMMIT;
```
이 경우 두 업데이트 쿼리 중 하나라도 실패하면 전체가 롤백되어야 합니다. 송금자의 계좌에서 돈이 빠져나갔는데 수취인의 계좌에 입금되지 않는 상황은 있어서는 안 됩니다.

## Consistency (일관성)
트랜잭션이 완료된 후에도 데이터베이스가 일관된 상태를 유지해야 합니다. 모든 데이터는 정해진 규칙을 만족해야 합니다.

**예시**: 재고 관리 시스템
```sql
BEGIN TRANSACTION;
    UPDATE inventory SET quantity = quantity - 10 WHERE product_id = 1;
    INSERT INTO orders (product_id, quantity) VALUES (1, 10);
COMMIT;
```
재고는 절대 마이너스가 될 수 없다는 규칙이 있다면, 이 트랜잭션은 재고가 10개 미만일 때 실행되지 않아야 합니다.

## Isolation (격리성)
동시에 실행되는 트랜잭션들은 서로 영향을 미치지 않고 독립적으로 수행되어야 합니다.

**예시**: 동시 주문 처리
```sql
-- Transaction 1
BEGIN TRANSACTION;
    SELECT quantity FROM inventory WHERE product_id = 1 FOR UPDATE;
    -- quantity: 5
    UPDATE inventory SET quantity = quantity - 3 WHERE product_id = 1;
COMMIT;

-- Transaction 2 (동시 실행)
BEGIN TRANSACTION;
    SELECT quantity FROM inventory WHERE product_id = 1 FOR UPDATE;
    -- Transaction 1이 완료될 때까지 대기
    -- quantity: 2
    UPDATE inventory SET quantity = quantity - 2 WHERE product_id = 1;
COMMIT;
```

## Durability (지속성)
트랜잭션이 성공적으로 완료(커밋)된 후에는 해당 결과가 영구적으로 보장되어야 합니다.

**예시**: 주문 데이터 보존
```sql
BEGIN TRANSACTION;
    INSERT INTO orders (id, user_id, total_amount) VALUES (1, 'user1', 50000);
    INSERT INTO order_items (order_id, product_id, quantity) VALUES (1, 100, 2);
COMMIT;
```
시스템이 갑자기 중단되더라도, 커밋된 주문 데이터는 보존되어야 합니다.

## 격리 수준에 따른 주의사항
격리성은 다음과 같은 레벨로 구분됩니다:
1. READ UNCOMMITTED
2. READ COMMITTED
3. REPEATABLE READ
4. SERIALIZABLE