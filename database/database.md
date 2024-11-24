# 데이터베이스 인덱스의 동작 원리와 장단점

## 동작원리
- B-Tree 구조를 기반으로 구현되어 있어 O(log n)의 시간 복잡도로 데이터 검색이 가능합니다.
- 인덱스는 실제 데이터의 정렬된 복사본을 별도로 저장하고, 실제 데이터의 위치를 가리키는 포인터를 함께 보관합니다.
- 예를 들어, 사용자 테이블에서 이메일로 검색할 때, 이메일 컬럼에 인덱스가 있다면 전체 테이블을 스캔하지 않고 인덱스를 통해 빠르게 데이터를 찾을 수 있습니다.

## 장점
- 소량의 데이터 검색 시(일반적으로 전체 데이터의 5% ~ 20% 미만) 매우 빠른 검색 속도를 제공합니다.
- WHERE 절의 조건, ORDER BY, JOIN 연산 시 성능이 향상됩니다.
- 테이블 전체를 스캔하지 않아도 되므로 디스크 I/O가 감소합니다.

## 단점
- INSERT, UPDATE, DELETE 시 인덱스도 함께 수정해야 하므로 쓰기 작업의 성능이 저하됩니다.
- 가장 중요한 제한사항으로, 전체 데이터의 5~20% 이상을 조회하는 경우 오히려 성능이 저하될 수 있습니다. 이는 다음과 같은 이유 때문입니다:
    - Index Range Scan은 Random Access 방식으로 동작합니다.
    - 즉, 디스크의 여러 위치를 건너뛰면서 데이터를 읽어야 하므로 많은 I/O가 발생합니다.
    - 반면 Full Table Scan은 Sequential Access로 디스크를 순차적으로 읽기 때문에, 대량의 데이터를 읽을 때 더 효율적일 수 있습니다.


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


# 데이터베이스 정규화와 반정규화

## 정규화가 필요한 이유

### 1. 데이터 무결성 보장
- 데이터 중복을 제거하여 업데이트 이상(Update Anomaly) 방지
- 삽입 이상(Insertion Anomaly)과 삭제 이상(Deletion Anomaly) 방지
- 데이터의 일관성 유지

**예시**: 학생 수강 정보 테이블
```sql
-- 정규화 전
CREATE TABLE student_courses (
    student_id INT,
    student_name VARCHAR(100),
    student_address VARCHAR(200),  -- 학생 주소가 중복 저장됨
    course_id INT,
    course_name VARCHAR(100),
    professor_name VARCHAR(100)    -- 교수 정보가 중복 저장됨
);

-- 정규화 후
CREATE TABLE students (
    student_id INT PRIMARY KEY,
    student_name VARCHAR(100),
    student_address VARCHAR(200)
);

CREATE TABLE courses (
    course_id INT PRIMARY KEY,
    course_name VARCHAR(100),
    professor_name VARCHAR(100)
);

CREATE TABLE enrollments (
    student_id INT,
    course_id INT,
    FOREIGN KEY (student_id) REFERENCES students(student_id),
    FOREIGN KEY (course_id) REFERENCES courses(course_id)
);
```

### 2. 정규화 단계
1. 제1정규형(1NF): 원자값으로 구성
2. 제2정규형(2NF): 부분 함수적 종속 제거
3. 제3정규형(3NF): 이행적 함수적 종속 제거
4. BCNF: 결정자이면서 후보키가 아닌 것 제거
5. 제4정규형(4NF): 다치 종속 제거
6. 제5정규형(5NF): 조인 종속성 제거

## 반정규화를 고려해야 하는 상황

### 1. 조회 성능이 중요한 경우
- 데이터 조회가 매우 빈번한 경우
- 여러 테이블을 조인해야 하는 경우

**예시**: 주문 조회 시스템
```sql
-- 반정규화 전
SELECT o.order_id, o.order_date, c.customer_name, p.product_name
FROM orders o
JOIN customers c ON o.customer_id = c.customer_id
JOIN order_items oi ON o.order_id = oi.order_id
JOIN products p ON oi.product_id = p.product_id;

-- 반정규화 후
CREATE TABLE order_summary (
    order_id INT PRIMARY KEY,
    order_date TIMESTAMP,
    customer_name VARCHAR(100),  -- 중복 허용
    product_names TEXT,          -- 제품명들을 하나의 필드에 저장
    total_amount DECIMAL(10,2)   -- 계산된 총액 저장
);
```

### 2. 자주 사용되는 통계 데이터
- 실시간 집계가 필요한 경우
- 대규모 데이터에서의 집계 연산이 부담되는 경우

```sql
-- 반정규화 예시: 주문 총액을 미리 계산하여 저장
CREATE TABLE daily_sales_summary (
    date DATE PRIMARY KEY,
    total_orders INT,
    total_amount DECIMAL(10,2),
    average_order_value DECIMAL(10,2)
);
```

### 3. 반정규화 고려 시 체크포인트
1. 데이터 일관성 vs 조회 성능
2. 저장 공간 사용량
3. 데이터 업데이트 빈도
4. 트랜잭션의 복잡도

## 근거 자료

### 학술 자료
1. E.F. Codd's Paper on Normalization:
    - "A Relational Model of Data for Large Shared Data Banks" (1970)
    - [Communications of the ACM](https://dl.acm.org/doi/10.1145/362384.362685)

2. Database System Concepts (7th Edition):
    - Chapter 7: Database Design and Normalization
    - Authors: Abraham Silberschatz, Henry F. Korth, S. Sudarshan
    - [Book Link](https://www.db-book.com/)

### 기술 문서
1. Microsoft SQL Server Documentation:
    - [Database Normalization Description](https://docs.microsoft.com/en-us/office/troubleshoot/access/database-normalization-description)
    - [Design for Performance](https://docs.microsoft.com/en-us/sql/relational-databases/performance/design-for-performance)

2. Oracle Documentation:
    - [Database Design Basics](https://docs.oracle.com/cd/B19306_01/server.102/b14231/design.htm)
    - [Schema Object Management](https://docs.oracle.com/cd/B19306_01/server.102/b14200/schema.htm)

### 실제 사례 연구
1. [Stack Overflow Database Schema](https://meta.stackexchange.com/questions/2677/database-schema-documentation-for-the-public-data-dump-and-sede):
    - 대규모 시스템에서의 정규화와 반정규화 사례
    - 성능 최적화를 위한 전략

### 성능 관련 연구
- [Impact of Database Normalization on Performance](https://arxiv.org/ftp/arxiv/papers/1508/1508.05esdeveniments-socials.pdf)
- [Trade-offs in SQL Database Design](https://www.confluent.io/blog/designing-high-performance-database-schemas/)

정규화와 반정규화는 상황에 따라 적절한 균형을 찾는 것이 중요합니다. 특히 마이크로서비스 아키텍처에서는 각 서비스의 특성에 따라 다른 정규화 수준을 적용하는 것이 일반적입니다.

실무에서는 정규화된 기본 구조를 유지하면서, 성능이 중요한 특정 부분에 대해서만 선택적으로 반정규화를 적용하는 것이 좋은 접근 방법입니다.