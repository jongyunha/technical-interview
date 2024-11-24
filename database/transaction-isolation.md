# 트랜잭션 격리 수준(Transaction Isolation Level)과 발생 가능한 문제

## 격리 수준별 발생 가능한 문제점 요약표

| 격리 수준 | Dirty Read | Non-Repeatable Read | Phantom Read |
|-----------|------------|-------------------|--------------|
| READ UNCOMMITTED | 발생 | 발생 | 발생 |
| READ COMMITTED | 방지 | 발생 | 발생 |
| REPEATABLE READ | 방지 | 방지 | MySQL: 방지¹, PostgreSQL: 발생 |
| SERIALIZABLE | 방지 | 방지 | 방지 |

¹ MySQL의 InnoDB 엔진은 REPEATABLE READ 레벨에서도 Phantom Read를 방지합니다.

## 각 문제점 상세 설명과 예시

### 1. Dirty Read
커밋되지 않은 데이터를 다른 트랜잭션이 읽을 수 있는 문제입니다.

```sql
-- Transaction 1
BEGIN;
UPDATE accounts SET balance = balance - 100 WHERE id = 1;  -- 아직 커밋하지 않음

-- Transaction 2
BEGIN;
SELECT balance FROM accounts WHERE id = 1;  -- Dirty Read 발생: 커밋되지 않은 감소된 잔액을 읽음

-- Transaction 1
ROLLBACK;  -- Transaction 1이 롤백되어 실제로는 차감되지 않았어야 할 금액
```

### 2. Non-Repeatable Read
한 트랜잭션 내에서 같은 쿼리를 두 번 실행했을 때 결과가 다른 문제입니다.

```sql
-- Transaction 1
BEGIN;
SELECT balance FROM accounts WHERE id = 1;  -- 잔액: 1000원

-- Transaction 2
BEGIN;
UPDATE accounts SET balance = balance - 100 WHERE id = 1;
COMMIT;

-- Transaction 1
SELECT balance FROM accounts WHERE id = 1;  -- 잔액: 900원
-- 같은 트랜잭션 내에서 다른 결과 조회
COMMIT;
```

### 3. Phantom Read
한 트랜잭션 내에서 같은 쿼리를 실행했을 때 이전에 없던 레코드가 나타나거나, 있던 레코드가 사라지는 현상입니다.

```sql
-- Transaction 1
BEGIN;
SELECT * FROM accounts WHERE balance BETWEEN 1000 AND 2000;  -- 10개 레코드 조회

-- Transaction 2
BEGIN;
INSERT INTO accounts (id, balance) VALUES (5, 1500);
COMMIT;

-- Transaction 1
SELECT * FROM accounts WHERE balance BETWEEN 1000 AND 2000;  -- 11개 레코드 조회 (Phantom)
COMMIT;
```

## DBMS별 특이사항

### MySQL (InnoDB)
- REPEATABLE READ가 기본 격리 수준입니다
- REPEATABLE READ에서 Phantom Read도 방지됩니다
- Gap Lock과 Next-Key Lock을 사용하여 Phantom Read 방지

### PostgreSQL
- READ COMMITTED가 기본 격리 수준입니다
- REPEATABLE READ에서 Phantom Read가 발생할 수 있습니다
- SERIALIZABLE은 SSI(Serializable Snapshot Isolation)를 사용합니다

### Oracle
- READ COMMITTED가 기본 격리 수준입니다
- REPEATABLE READ 대신 SERIALIZABLE을 사용합니다
- READ UNCOMMITTED를 지원하지 않습니다

## 근거 자료
1. MySQL 공식 문서:
    - [Transaction Isolation Levels](https://dev.mysql.com/doc/refman/8.0/en/innodb-transaction-isolation-levels.html)
    - [InnoDB Locking](https://dev.mysql.com/doc/refman/8.0/en/innodb-locking.html)

2. PostgreSQL 공식 문서:
    - [Transaction Isolation](https://www.postgresql.org/docs/current/transaction-iso.html)
    - [13.2. Transaction Isolation](https://www.postgresql.org/docs/current/tutorial-transactions.html)

3. Oracle 공식 문서:
    - [Oracle Database Concepts - Data Concurrency and Consistency](https://docs.oracle.com/en/database/oracle/oracle-database/19/cncpt/data-concurrency-and-consistency.html)

4. 학술 자료:
    - [A Critique of ANSI SQL Isolation Levels](https://www.microsoft.com/en-us/research/wp-content/uploads/2016/02/tr-95-51.pdf) - Microsoft Research
    - [Serializable Snapshot Isolation in PostgreSQL](https://drkp.net/papers/ssi-vldb12.pdf)

각 DBMS마다 격리 수준의 구현이 다르기 때문에, 실제 프로젝트에서는 사용 중인 DBMS의 특성을 정확히 이해하고 있어야 합니다. 또한 격리 수준이 높아질수록 동시성이 떨어지므로, 비즈니스 요구사항에 맞는 적절한 격리 수준을 선택하는 것이 중요합니다.

## 참고 문헌 및 근거
- [MySQL 공식 문서 - Transaction ACID Properties](https://dev.mysql.com/doc/refman/8.0/en/mysql-acid.html)
- [PostgreSQL 공식 문서 - Transaction Isolation](https://www.postgresql.org/docs/current/transaction-iso.html)
- [Martin Kleppmann - Designing Data-Intensive Applications](https://dataintensive.net/) - Chapter 7: Transactions
- [Oracle 공식 문서 - Database Concepts: Data Concurrency and Consistency](https://docs.oracle.com/cd/E11882_01/server.112/e40540/consist.htm)

실제 면접에서는 이론적인 설명과 함께 자신이 실무에서 겪은 구체적인 사례를 함께 언급하면 더욱 효과적입니다. 예를 들어, 격리 수준을 조정하여 성능을 개선한 경험이나 데드락을 해결한 경험 등을 공유할 수 있습니다.