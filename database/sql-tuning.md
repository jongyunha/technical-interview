# SQL 튜닝에 대한 기술 면접 답변

## 주요 질문
"SQL 성능 최적화를 위한 튜닝 방법에 대해 설명해주세요. 실제 프로젝트에서 성능 개선을 위해 어떤 방법들을 적용해보셨나요?"

## 답변 구조

### 1. SQL 튜닝의 기본 원칙

#### 1.1 실행계획 분석
```sql
-- MySQL의 경우
EXPLAIN SELECT * FROM users u 
JOIN orders o ON u.id = o.user_id 
WHERE u.status = 'active';

-- 실행계획 분석 포인트
- type 컬럼: const, eq_ref, ref, range, index, ALL 등의 접근 방식 확인
- key 컬럼: 사용되는 인덱스 확인
- rows 컬럼: 검색되는 행 수 예측
```

#### 1.2 WHERE 절 최적화
```sql
-- 안좋은 예
SELECT * FROM orders 
WHERE YEAR(created_at) = 2024;  -- 인덱스 사용 불가

-- 좋은 예
SELECT * FROM orders 
WHERE created_at >= '2024-01-01' 
AND created_at < '2025-01-01';  -- 인덱스 사용 가능
```

### 2. 주요 튜닝 기법

#### 2.1 인덱스 최적화
```sql
-- 복합 인덱스 생성 시 선택도(Selectivity)가 높은 컬럼을 앞에 배치
CREATE INDEX idx_user_status_created 
ON users(status, created_at);  -- status보다 created_at의 선택도가 높은 경우 순서 변경 고려

-- 커버링 인덱스 활용
SELECT user_id, status, created_at  -- 인덱스에 포함된 컬럼만 조회
FROM users 
WHERE status = 'active' 
AND created_at > '2024-01-01';
```

#### 2.2 조인 최적화
```sql
-- 작은 결과셋을 먼저 조인
SELECT * FROM small_table s  -- 1000 rows
JOIN large_table l          -- 1000000 rows
ON s.id = l.small_id;

-- 조인 조건 최적화
SELECT * FROM orders o
JOIN users u ON u.id = o.user_id  -- PK-FK 조인
WHERE u.status = 'active';
```

#### 2.3 서브쿼리 최적화
```sql
-- 안좋은 예 (상관 서브쿼리)
SELECT *, 
  (SELECT COUNT(*) FROM orders o 
   WHERE o.user_id = u.id) as order_count
FROM users u;

-- 좋은 예 (조인으로 변환)
SELECT u.*, COALESCE(o.order_count, 0) as order_count
FROM users u
LEFT JOIN (
  SELECT user_id, COUNT(*) as order_count
  FROM orders
  GROUP BY user_id
) o ON u.id = o.user_id;
```

### 3. 고급 튜닝 기법

#### 3.1 파티셔닝
```sql
-- 범위 파티셔닝 예시
CREATE TABLE orders (
    id INT,
    created_at DATE,
    amount DECIMAL(10,2)
) PARTITION BY RANGE (YEAR(created_at)) (
    PARTITION p2023 VALUES LESS THAN (2024),
    PARTITION p2024 VALUES LESS THAN (2025)
);
```

#### 3.2 materialized view 활용
```sql
-- PostgreSQL의 경우
CREATE MATERIALIZED VIEW monthly_sales AS
SELECT 
    DATE_TRUNC('month', created_at) as month,
    SUM(amount) as total_sales
FROM orders
GROUP BY DATE_TRUNC('month', created_at)
WITH DATA;
```

## 근거 자료

### 1. 데이터베이스 벤더 공식 문서
- [MySQL Query Optimization](https://dev.mysql.com/doc/refman/8.0/en/optimization.html)
- [PostgreSQL Query Performance Tuning](https://www.postgresql.org/docs/current/performance-tips.html)
- [Oracle Database SQL Tuning Guide](https://docs.oracle.com/en/database/oracle/oracle-database/19/tgsql/index.html)

### 2. 성능 측정 도구 문서
- [MySQL EXPLAIN Format](https://dev.mysql.com/doc/refman/8.0/en/explain-output.html)
- [pg_stat_statements Documentation](https://www.postgresql.org/docs/current/pgstatstatements.html)

### 3. 학술 자료 및 서적
- "High Performance MySQL" by Baron Schwartz, Peter Zaitsev, and Vadim Tkachenko
    - [O'Reilly Link](https://www.oreilly.com/library/view/high-performance-mysql/9781492080503/)
- "SQL Performance Explained" by Markus Winand
    - [Official Website](https://sql-performance-explained.com/)

### 4. 커뮤니티 및 블로그
- [Use The Index, Luke!](https://use-the-index-luke.com/)
- [SQLperformance.com](https://sqlperformance.com/)
- [Percona Database Performance Blog](https://www.percona.com/blog/)

## 실제 튜닝 사례

### 1. 대량 데이터 처리 최적화
```sql
-- 처리 전
DELETE FROM logs WHERE created_at < '2023-01-01';

-- 처리 후 (배치 처리)
REPEAT
    DELETE FROM logs 
    WHERE created_at < '2023-01-01' 
    LIMIT 10000;
    SLEEP(0.1);
UNTIL ROW_COUNT() = 0 END REPEAT;
```

### 2. IN절 최적화
```sql
-- 처리 전
SELECT * FROM orders 
WHERE user_id IN (
    SELECT id FROM users WHERE status = 'active'
);

-- 처리 후
SELECT o.* 
FROM orders o
JOIN users u ON o.user_id = u.id
WHERE u.status = 'active';
```

### 3. GROUP BY 최적화
```sql
-- 처리 전
SELECT user_id, COUNT(*) 
FROM orders 
GROUP BY user_id;

-- 처리 후 (인덱스 활용)
CREATE INDEX idx_user_id ON orders(user_id);
SELECT user_id, COUNT(*) 
FROM orders 
GROUP BY user_id;
```

실제 면접에서는 이러한 이론적인 내용과 함께, 본인이 실제 프로젝트에서 경험한 성능 개선 사례를 구체적으로 설명하는 것이 좋습니다. 예를 들어:
1. 문제 상황 설명
2. 원인 분석 과정
3. 해결 방법 적용
4. 성능 개선 결과

이러한 구체적인 경험을 공유하면 실무 능력을 효과적으로 어필할 수 있습니다.